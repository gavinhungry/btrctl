btrctl
======

Btrfs snapshot and trunk backup tool.

`btrctl` is a single bash script that takes read-only snapshots of a set of
btrfs subvolumes and backs them up, with `btrfs send`/`receive`, to a separately
mounted btrfs trunk device. It can snapshot and back up remote machines over
SSH, tags snapshots, and prunes old snapshots with a tiered retention policy.

Full documentation is in the man page: `man btrctl`, or `man ./btrctl.1` from
a checkout.

Configuration
-------------

`/etc/btrctl.conf` is a shell fragment with four variables:

| Variable            | Default       | Meaning                                                    |
|---------------------|---------------|------------------------------------------------------------|
| `HOST_LABEL`        | `$(hostname)` | Name for this machine in snapshot names and on the trunk   |
| `BTRFS_MOUNT_POINT` | `/btrfs`      | Mount point of the file system holding the subvolumes      |
| `SUBVOLUMES`        | `(@)`         | Bash array of subvolume names, relative to the mount point |
| `TRUNK_MOUNT_POINT` | `/trunk`      | Mount point of the trunk device                            |

Values are limited to a conservative character set because they end up in shell
commands run over SSH; see **CONFIGURATION** in the man page.

A second, independent pool on the same machine can have its own config overlay
selected with the `BTRCTL_CONF` environment variable. Overlays are sourced as
root, so they must be owned by root and live in a root-owned, non-writable
directory such as `/etc/btrctl.d/`; see **ENVIRONMENT** in the man page.

Remote hosts
------------

`--host` runs the same `btrctl` command on another machine over SSH. The
connection is made as the current user (not root), so user SSH keys and
`ssh_config` apply, and the remote side asks for its own sudo password. `trunk
backup --host` streams the remote snapshot straight into `btrfs receive` on the
trunk machine.

Retention policy
----------------

`prune` and `trunk prune` keep the union of:

- everything from the last 24 hours;
- the newest set per hour for 3 days;
- the newest set per day for 7 days;
- the newest set per week (Sunday start) for 4 weeks;
- the newest set per calendar month for 12 months;
- the newest set per calendar year, forever.

Sets pointed to by `latest` or any `trunk_<id>` link, and sets tagged
`noprune`, are never pruned. `--dry-run` shows the whole plan with the reason
each set is kept. See **RETENTION POLICY** in the man page for the fine print
on cutoffs and calendar edge cases.


License
-------
This software is released under the terms of the **MIT license**. See `LICENSE`.
