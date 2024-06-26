# ansible-role-snapraid

An Ansible role which will download, build and install SnapRAID on Debian 12
(other releases are untested).

This role supports defining multiple SnapRAID arrays, each with their own
configuration file, and the syncing/scrubbing process for them can be automated
via the included [`snapraid_sync`][1] script. It is `cron` that is used to
trigger the script on a configurable schedule, and it will send you notification
emails when syncs are successful, threshold levels for deleted/updated files are
exceeded or something else goes wrong.


# Installation

This repository makes use of a submodule, which is just pointer to another
repository, and it needs to be initialized and downloaded as well before this
role will work. Fortunately it is possible to do this in just a single command,
so move into your `roles/` folder and run the following:

```bash
git clone --recursive git@github.com:JonasAlfredsson/ansible-role-snapraid.git snapraid
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull --recurse-submodules
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Install SnapRAID and push out configuration files
  roles:
    - snapraid
```



# Usage

Since the SnapRAID arrays are often unique to each individual host, I usually
prefer to define these individual configurations in their respective
`host_vars/{{ ansible_hostname }}` path. However, if you have multiple
identical machines there should not be any problem to define all of this in
one of the `group_vars/` files.

An important thing to remember is that Ansible will overwrite, and not merge,
these kinds of hashes/dictionaries if there are two with the same name. You can
therefore not have a part of this be defined in the `group_vars/` and then other
parts in the `host_vars/`. If you do not like this behavior you may look into
setting [`hash_behaviour = merge`][2], but be aware that this is not a very
[good solution][3]. Instead you should probably look into the [`combine`][5]
filter or the [`merge_vars`][4] action plugin.


## Example Configuration
There are sort of two parts to this configuration; first SnapRAID itself and
its arrays, and then it is variables related to the [`snapraid_sync`][1] script.
The second one is not necessary if you don't want to, but it will automate the
syncing/scrubbing if configured.

The following examples have all the available variables included, and all the
default values written out. So any field which is not marked with `# Required`
may be left out of your configuration if you are fine with the defaults.


### SnapRAID
> List of available SnapRAID versions/tags may be found [here][6].
  Example: `snapraid_version: "11.3"`

```yaml
snapraid_version:  # Required
snapraid_tmp_dir: "/tmp"
snapraid_arrays:
  - name:  # Required and may only be [a-zA-Z_-] (limit from cron file naming).
    conf_dir: # Required
    exclude_hidden_items: false
    exclude_items:
      - "*.unrecoverable"
      - "/tmp/"
      - "/lost+found/"
    blocksize: 256
    hashsize: 16
    autosave: 500
    parity_drives:
      - mount:  # Required
        content_file: false
    data_drives:
      - mount:  # Required
        name:  # Required and must be unique (space not allowed).
        content_file: true
    snapraid_sync: []  # See next section for more details
```

> Jump directly to the [`snapraid_sync`](#snapraid_sync) section.

As can be seen the `snapraid_arrays` variable is a list, so it is possible to
expand it to as many arrays that you want. You should only make sure that they
have unique names.

The `parity_drives` variable is also a list, and you will need at minimum one
parity drive defined for this role to function (with a maximum of 6 supported
by SnapRAID). The parity mounts must **NOT** be in a data disk.

**Example:**

```yaml
parity_drives:
  - mount: "/mnt/parity1"
  - mount: "/mnt/parity2"
```

There are no limits (that I know of) for how many data disks you may have.
These are also defined as a list, and it is important that all have unique
names since these are used as identifiers by SnapRAID. The name and mount point
association of the data disks is relevant for parity, so do not change them
afterwards.

**Example:**

```yaml
  data_drives:
    - mount: "/mnt/data1"
      name: "D1"
    - mount: "/mnt/data2"
      name: "D2"
```

You must also have at least one content file for each parity file **plus one**.
These content files can be in the disks used for data, parity or boot, but each
file must be in a different disk. The first and primary content file is created
inside the `conf_dir` along with the `.conf` file for this array.

By default it is also configured so that each data disk includes a content file
located at `{{ mount }}/.snapraid_{{ name }}.content`. This has the added
benefit of making total amount of available space on the data disk
**slightly less** than the full disk amount. This is a recommended thing to do,
because the parity file will be **slightly larger** than the amount of synced
data, and this content file is [excluded][10] from the sync, so it creates a
natural buffer to hinder the parity disk from being overfilled.

Then there are the remaining variables and their short explanations:

- `exclude_items` - List of files and directories to exclude.
  - Remember that all the paths are [relative][11] at the mount points.
- `exclude_hidden_items` - Hidden items will be ignored during 'syncs'.
  - In Unix systems this is usually files beginning with a period.
- `autosave` - Number of gigabytes to process before saving the state.
  - This option is useful to avoid having to restart from scratch if a long
    'sync' is interrupted (`0` to disable).
- `blocksize` - The block size in kibi bytes (1024 bytes).
  - WARNING: Changing this value is for experts only!
- `hashsize` - The hash size in bytes.
  - WARNING: Changing this value is for experts only!


### snapraid_sync
This is a script used for automating the syncing and scrubbing process, so
manual intervention will only be necessary when the number of deleted/updated
files exceed your defined thresholds. A detailed explanation of this "manual
intervention" can be found in [its repository][7], along with more information
about the inner workings of this script, but there is also some extra info
[at the bottom](#manual-intervention) of this guide.

Anyway, the automatic syncing is defined on a per-array basis (first mentioned
in the [previous section](#snapraid)), and the `*_schedule` variables are then
normal `cron` expressions.

```yaml
snapraid_arrays:
  - name:  # Defined in previous section.
    config:  # Defined in previous section.
    ...
    snapraid_sync:
      - sync_schedule: "05 9,22 * * 2-7"  # Example -> sync at 09:05 and 22:05 every day except monday.
        scrub_schedule: "00 13 * * mon"  # Example -> scrub at 13:00 on mondays.
        delete_threshold: 0
        update_threshold: -1
        scrub_percent: 8
        scrub_age: 10
        attach_log: "false"
```

If you want to be notified by email, on successful syncs or errors, you should
define the `snapraid_sync_email_address` variable. However, in order to be able
to receive emails over the open internet you will need an account on a trusted
provider and configure the `mutt` email client to use that account
([details here][8]). As of now there is only support for automatically
configuring Gmail accounts in the `muttrc` file, but if you have such an
account the following variables are available:

```yaml
snapraid_sync_email_address: ""
snapraid_sync_email_subject_prefix: "SnapRAID on $(hostname) - "
snapraid_mutt:
  realname: "User Name"
  email: "my-username@gmail.com"
  password: "supersecret"
```

It is also necessary to handle all the log output it creates. To not have it
fill a single file with a million lines after a while, we will use
[`logrotate`][9] to only keep a limited amount of old log files. There are
only three options you will need to be aware of, and these are their default
values:

```yaml
snapraid_sync_log_dir: "/var/log/snapraid_sync"
snapraid_sync_logrotate_interval: "daily"
snapraid_sync_logrotate_count: 7
```

Below are a couple of other variables related to this role, and their default
values. You probably don't need to edit these.

```yaml
snapraid_sync_script_dir: "/root/snapraid_sync"
snapraid_muttrc_path: "/root/.muttrc"
snapraid_sync_mail_bin: "/usr/bin/mutt"
```


## Manual Intervention
In the [`snapraid_sync` repository][7] there are more details regarding the
thoughts behind "manual intervention", but if you have multiple arrays it might
be annoying to always define all the environment variables every time. This role
will therefore create "entrypoints" for each array that you define.

These "entrypoints" are nothing more than small bash scripts, with all your
array specific variables set, which then call upon the original
`snapraid_sync.sh` script. With this it should therefore be possible for you
to run an array specific "force sync" like this:

```bash
sudo /{{ snapraid_sync_script_dir }}/snapraid_sync_entrypoint-{{ snapraid_sync.name }}.sh force
```

e.g.

```bash
sudo /root/snapraid_sync/snapraid_sync_entrypoint-array1.sh force
```






[1]: https://github.com/JonasAlfredsson/snapraid_sync
[2]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour
[3]: https://medium.com/uptime-99/3-things-ive-learned-about-ansible-the-hard-way-bae341524a86
[4]: https://pypi.org/project/ansible-merge-vars/
[5]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries
[6]: https://github.com/amadvance/snapraid/releases
[7]: https://github.com/JonasAlfredsson/snapraid_sync#interactive-intervention
[8]: https://github.com/JonasAlfredsson/snapraid_sync#mutt
[9]: https://linux.die.net/man/8/logrotate
[10]: https://www.snapraid.it/manual#7.4
[11]: https://www.snapraid.it/manual#8
