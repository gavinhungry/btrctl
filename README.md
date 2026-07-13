# btrctl

Btrfs snapshot and trunk backup tool.

```
btrctl - Btrfs snapshot and trunk backup tool

USAGE:
  btrctl <command> [options]

ENVIRONMENT:
  BTRCTL_CONF=FILE
      Overlay config file, sourced after /etc/btrctl.conf. Values set in
      FILE override the base config; anything not set in FILE is
      inherited from /etc/btrctl.conf. Useful for a second, independent
      btrfs pool on the same host (its own HOST_LABEL/BTRFS_MOUNT_POINT/
      SUBVOLUMES) while still sharing TRUNK_*/SSH_USER settings from the
      base config. If set and FILE does not exist, this is a hard error.
      Preserved automatically across the automatic sudo elevation that
      happens when btrctl isn't already running as root.

COMMANDS:
  mount [--host HOST]
      Mount $BTRFS_MOUNT_POINT. Says so and exits if already mounted.

        --host HOST         Run against HOST instead of the local machine

  umount [--host HOST]
  unmount [--host HOST]
      Unmount $BTRFS_MOUNT_POINT. Says so and exits if already unmounted.

        --host HOST         Run against HOST instead of the local machine

  snapshot [--tag NAME] [--host HOST]
      Take a snapshot set of all configured subvolumes under $BTRFS_MOUNT_POINT.
      Mounts $BTRFS_MOUNT_POINT automatically if not already mounted.
      If $BTRFS_MOUNT_POINT/btrctl-pre-snapshot exists and is executable,
      it is run (cwd = $BTRFS_MOUNT_POINT) before snapshotting begins; a
      non-zero exit aborts the snapshot before any subvolumes are touched.
      Records immutable-flagged files/dirs for each snapshot (lost on
      send/receive otherwise).

        --tag NAME          Attach a human-readable tag to this snapshot set
        --host HOST         Run against HOST instead of the local machine

  list [--host HOST]
      List local snapshot sets: timestamp, tag if present, and FLAGS
      ("latest" and/or any trunk_<id> this set was last backed up to).

        --host HOST         Run against HOST instead of the local machine

  latest [--host HOST]
      Print just the timestamp ID of the latest local snapshot set, and
      nothing else -- suitable for capturing in scripts.

        --host HOST         Run against HOST instead of the local machine

  rm SET_NAME [--host HOST]
      Remove a local snapshot set by timestamp name. Prompts for
      confirmation before deleting.

        --host HOST         Run against HOST instead of the local machine

  tag SET_ID TAG_NAME [--host HOST]
      Set or update the tag on an existing local snapshot set. An empty
      TAG_NAME clears the tag.

        --host HOST         Run against HOST instead of the local machine

  trunk mount
      Unlock (LUKS) and mount the trunk backup device at $TRUNK_MOUNT_POINT.
      Each step is skipped (with a message) if already done.

  trunk umount
  trunk unmount
      Unmount $TRUNK_MOUNT_POINT and close the LUKS device. The unmount
      step is skipped (with a message) if already unmounted, but the LUKS
      device is always closed regardless.

  trunk id
      Print the currently mounted trunk device's short identifier (the
      $TRUNK_MOUNT_POINT/.id marker file content used in trunk_<id>
      tracking symlinks), e.g. "a".

  trunk backup [--host HOST]
      Back up the latest snapshot set to the currently mounted trunk device.
      Always runs on the machine trunk is physically attached to.
      If /trunk/@<HOST_LABEL> does not exist, it is created automatically
      as a new subvolume. If it exists but is not a btrfs subvolume,
      aborts with an error.

        --host HOST         Back up HOST's latest snapshot set instead of
                              the local machine's

  trunk list [HOST]
      List snapshot sets stored on trunk for a given host (default: local
      machine's HOST_LABEL). HOST is a HOST_LABEL, not an SSH alias, and
      no SSH connection is ever made (same as trunk rm's HOST).

  trunk rm HOST/SET_NAME
      Remove a snapshot set from trunk. HOST is required and explicit;
      there is no implicit "local machine" default for this destructive
      operation. Prompts for confirmation before deleting.

  config
      Print effective configuration as KEY=value pairs.

  help, -h, --help
      Show this help text.
```
