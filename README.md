btrctl
======

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

      Because FILE is sourced as root after that elevation, it must come from a
      trusted location: after resolving symlinks, FILE must be a regular file
      owned by root and not group- or world-writable, and every directory on
      its path must be owned by root and not group- or world-writable (sticky
      directories such as /tmp do not qualify). Anything else is a hard error.
      /etc/btrctl.d/ is a good home for overlays. This is what makes it safe to
      grant btrctl through a restricted sudoers rule with SETENV; without the
      check, BTRCTL_CONF would be equivalent to arbitrary root.

CONFIGURATION:
  /etc/btrctl.conf sets HOST_LABEL, BTRFS_MOUNT_POINT, SUBVOLUMES and
  TRUNK_MOUNT_POINT. Because these values are embedded in shell commands run
  over SSH and used to build paths, they are restricted to a conservative
  character set and validated before snapshot and trunk backup run:

    HOST_LABEL          A-Z a-z 0-9 . _ -
    SUBVOLUMES entries  A-Z a-z 0-9 @ . _ + -
    BTRFS_MOUNT_POINT   absolute path of A-Z a-z 0-9 @ . _ + / - with no ".."
                        component

  None may be empty, ".", or "..", or begin with "-". The trunk-id marker file
  follows the HOST_LABEL rules. The same rules are applied to configuration
  fetched from a remote host with --host.

SOURCE MOUNTS:
  Commands that access $BTRFS_MOUNT_POINT require it to already be mounted. With
  --host, the remote host's configured BTRFS_MOUNT_POINT must already be
  mounted.

COMMANDS:
  snapshot [-t|--tag TAG ...] [-H|--host HOST]
      Take a snapshot set of all configured subvolumes under $BTRFS_MOUNT_POINT.
      If $BTRFS_MOUNT_POINT/.btrctl/pre-snapshot exists and is executable, it is
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

        -t, --tag TAG       Attach a tag; repeat this option for multiple tags
        -H, --host HOST     Run against HOST instead of the local machine

  list [-H|--host HOST]
      List local snapshot sets: host label, timestamp, TAGS if present, and
      FLAGS ("latest" and/or any trunk_<id> this set was last backed up to).

        -H, --host HOST     Run against HOST instead of the local machine

  prune [-H|--host HOST] [-n|--dry-run]
      Prune local snapshot sets using the retention policy below. Always keeps
      sets targeted by latest or any current trunk_* pointer, and sets tagged
      noprune. Uses the local clock/timezone, or HOST's when run remotely.
      Previews deletions and asks for one batch confirmation. Stops on deletion
      failure. --dry-run shows retained sets and reasons without deleting or
      prompting. Protections are checked again immediately before deletion.

  latest [-H|--host HOST]
      Print just the timestamp ID of the latest local snapshot set, and nothing
      else -- suitable for capturing in scripts.

        -H, --host HOST     Run against HOST instead of the local machine

  rm SET_NAME [-H|--host HOST]
      Remove a local snapshot set by timestamp name. Prompts for confirmation
      before deleting. Warns when the set is flagged as latest or by one or more
      trunk_<id> tracking symlinks.

        -H, --host HOST     Run against HOST instead of the local machine

  tag SET_ID {add|rm} TAG... [-H|--host HOST]
      Add or remove one or more tags on an existing local snapshot set. Other
      tags are preserved. Adding an existing tag or removing an absent tag
      succeeds without changing the tag list. Removing the final tag removes the
      tags file.

      Tags are case-sensitive and nonempty. Quote tags containing spaces; tags
      may not begin with "-" or contain commas or newlines. Tags are stored one
      per line in SET_DIR/.btrctl/tags, with duplicates removed and insertion
      order preserved. Listings use a TAGS heading and comma-separated tags. The
      old .btrctl/tag file and tag SET_ID TAG_NAME syntax are no longer used.

        -H, --host HOST     Run against HOST instead of the local machine

  trunk id
      Print the currently mounted trunk device's short identifier (the
      $TRUNK_MOUNT_POINT/.btrctl/trunk-id marker file content used in trunk_<id>
      tracking symlinks), e.g. "a".

  trunk backup [SET_NAME] [-H|--host HOST]
      Back up SET_NAME, or the latest snapshot set when omitted, to the
      currently mounted trunk device. Device opening, closing, and mounting are
      managed externally. Always runs on the machine trunk is physically
      attached to. If /trunk/@<HOST_LABEL> does not exist, it is created
      automatically as a new subvolume, along with a
      /trunk/@<HOST_LABEL>/.btrctl marker file. If it exists but is not a btrfs
      subvolume, or exists without the .btrctl marker, aborts with an error (the
      marker distinguishes btrctl-managed subvolumes from other, unrelated
      subvolumes that may also live on trunk). To adopt a pre-existing
      subvolume, manually create an empty .btrctl file inside it. Snapshot-set
      metadata under .btrctl is copied after subvolume receive. A trunk backup
      fails if the destination snapshot set already exists. Incremental sends
      use the set tracked by trunk_<id> as parent only when that set still
      exists on both the source and trunk; otherwise a full send is used.

      With --host, the remote's configuration (see CONFIGURATION) and the
      snapshot set names its latest and trunk_<id> symlinks resolve to are
      validated before anything is created on trunk; a malformed value aborts
      the backup.

        -H, --host HOST     Back up HOST's selected snapshot set instead of
                            the local machine's

  trunk hosts
      Print the HOST_LABEL of each btrctl-managed host directory on trunk, one
      per line. Directories without a .btrctl marker are skipped.

  trunk list [HOST]
      List snapshot sets stored on trunk across all hosts. Specify HOST to list
      only that host. HOST is a HOST_LABEL, not an SSH alias, and no SSH
      connection is ever made (same as trunk rm's HOST). Automatic discovery
      skips directories without a .btrctl marker; explicitly listing such a HOST
      aborts with an error.

  trunk prune {HOST|-a|--all} [-n|--dry-run]
      Prune one trunk HOST, or all btrctl-managed hosts with --all. HOST is a
      stored HOST_LABEL, not an SSH alias. A scope is required; HOST and --all
      cannot be combined. Retention is independent per host, using the attached
      machine's clock/timezone. Keeps sets tagged noprune. Confirmation and
      --dry-run behave like prune. Explicit HOST selection requires its marker.

  trunk rm HOST/SET_NAME
      Remove a snapshot set from trunk. HOST is required and explicit; there is
      no implicit "local machine" default for this destructive operation.
      Prompts for confirmation before deleting. Aborts with an error if
      /trunk/@HOST exists but is missing its .btrctl marker. Removes the entire
      set directory, including its .btrctl metadata.

  trunk tag HOST/SET_ID {add|rm} TAG...
      Add or remove tags directly on a trunk snapshot set, using the same
      semantics as tag. HOST is a stored HOST_LABEL, not an SSH alias. Requires
      the host directory's .btrctl marker. trunk backup copies tags along with
      all other set metadata. Later tag changes on the source or trunk are
      independent and are not synchronized.

  config
      Print effective configuration as KEY=value pairs.

  help, -h, --help
      Show this help text.
```

Prune retention
---------------
Keep the union of: all snapshots for 24 hours (shown as `recent`); latest per
hour for 72 hours; latest per day for 7 days; latest per Sunday-start week for
4 weeks; latest per month for 12 calendar months; and latest per year forever.
Protected sets also participate in bucket selection. Pruning removes whole sets
and their metadata.

Timestamp directory names determine age and calendar buckets. Each window
includes its cutoff; days/weeks mean elapsed 24-hour days, while 12 months means
the same local date/time one year earlier (February 29 clamps to February 28; a
cutoff in a daylight-saving gap shifts forward through the gap). Future-dated
sets are kept; invalid timestamps are warned about and skipped. The clock is
captured once per command. No timezone metadata is required.

License
-------
This software is released under the terms of the **MIT license**. See `LICENSE`.
