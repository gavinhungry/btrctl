# btrctl

Btrfs snapshot and trunk backup tool.

```
btrctl - Btrfs snapshot and trunk backup tool

USAGE:
  btrctl <command> [options]

ENVIRONMENT:
  BTRCTL_CONF=FILE
      Overlay config file, sourced after /etc/btrctl.conf. Values set in FILE
      override the base config; anything not set in FILE is inherited from
      /etc/btrctl.conf. Useful for a second, independent btrfs pool on the same
      host (its own HOST_LABEL/BTRFS_MOUNT_POINT/ SUBVOLUMES) while still
      sharing TRUNK_* settings from the base config. If set and FILE does not
      exist, this is a hard error. Preserved automatically across the automatic
      sudo elevation that happens when btrctl isn't already running as root.

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
      Mounts $BTRFS_MOUNT_POINT automatically if not already mounted. If
      $BTRFS_MOUNT_POINT/.btrctl/pre-snapshot exists and is executable, it is
      run (cwd = $BTRFS_MOUNT_POINT) before snapshotting begins; a non-zero exit
      aborts the snapshot before any subvolumes are touched. If
      $BTRFS_MOUNT_POINT/.btrctl/post-snapshot exists and is executable, it is
      run after the snapshot set and latest link are created. A non-zero exit
      marks the set invalid; btrctl exits with an error and the set must be
      cleaned up manually.

      Both hooks receive BTRFS_MOUNT_POINT and run with it as their cwd.
      post-snapshot also receives SNAPSHOT_DIR, the absolute path to the
      completed snapshot set. pre-snapshot does not receive SNAPSHOT_DIR. Set
      metadata and post-snapshot sidecars belong in SNAPSHOT_DIR/.btrctl. trunk
      backup copies this directory recursively, accepting only regular files and
      directories. On trunk, files are owned by the backup process, have mode
      0644, and receive new timestamps; directories have mode 0755.

        --tag NAME          Attach a human-readable tag to this snapshot set
        --host HOST         Run against HOST instead of the local machine

  list [--host HOST]
      List local snapshot sets: host label, timestamp, tag if present, and FLAGS
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

  trunk id
      Print the currently mounted trunk device's short identifier (the
      $TRUNK_MOUNT_POINT/.btrctl/trunk-id marker file content used in trunk_<id>
      tracking symlinks), e.g. "a".

  trunk backup [SET_NAME] [--host HOST]
      Back up SET_NAME, or the latest snapshot set when omitted, to the
      currently mounted trunk device. Device opening, closing, and mounting
      are managed externally.
      Always runs on the machine trunk is physically attached to.
      If /trunk/@<HOST_LABEL> does not exist, it is created automatically
      as a new subvolume, along with a /trunk/@<HOST_LABEL>/.btrctl marker
      file. If it exists but is not a btrfs subvolume, or exists without
      the .btrctl marker, aborts with an error (the marker distinguishes
      btrctl-managed subvolumes from other, unrelated subvolumes that may
      also live on trunk). To adopt a pre-existing subvolume, manually
      create an empty .btrctl file inside it.
      Snapshot-set metadata under .btrctl is copied after subvolume receive.
      A trunk backup fails if the destination snapshot set already exists.

        --host HOST         Back up HOST's selected snapshot set instead of
                              the local machine's

  trunk list [HOST]
      List snapshot sets stored on trunk for a given host (default: local
      machine's HOST_LABEL). HOST is a HOST_LABEL, not an SSH alias, and
      no SSH connection is ever made (same as trunk rm's HOST). Aborts
      with an error if /trunk/@<HOST_LABEL> exists but is missing its
      .btrctl marker.

  trunk rm HOST/SET_NAME
      Remove a snapshot set from trunk. HOST is required and explicit;
      there is no implicit "local machine" default for this destructive
      operation. Prompts for confirmation before deleting. Aborts with an
      error if /trunk/@HOST exists but is missing its .btrctl marker.
      Removes the entire set directory, including its .btrctl metadata.

  config
      Print effective configuration as KEY=value pairs.

  help, -h, --help
      Show this help text.
```
