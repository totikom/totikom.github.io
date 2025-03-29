+++
title = "Backups made easy: btrfs + snapper + snapborg"

description = "Combining btrfs with borg creates a very robust and easy to use backup system with nice extra features."

date = 2025-03-29
template = "page_with_toc.html"
[taxonomies]
tags = ["system-administration", "backups"]
[extra]
show_only_description = true
+++
Almost half a year ago I've switched from Manjaro to ArchLinux. One of the triggers for that switch was an urge to try CoW file system, mainly [btrfs](https://wiki.archlinux.org/title/Btrfs).

# System installation and subvolume layout
Basically, I've followed this [tutorial](https://walian.co.uk/arch-install-with-secure-boot-btrfs-tpm2-luks-encryption-unified-kernel-images.html):

- Created 2GB FAT partition for EFI
- Encrypted the rest of my disk with `cryptsetup`
- Formatted it to `btrfs`
- Created a set of subvolumes, mainly:
    - `@` for rootfs
	- `@home` for home folders
	- `@home_unbacked` user files, which won't be backed up (more about it later)
	- `@var` for `/var`
	- `@var_log` for logs
	- `@var_pkgs` for `pacman` cache

This massive number of subvolumes is needed to allow fine-grained backups without wasting space for unneeded data.

For instance,  you most likely won't need a backup of `pacman` cache, as you can download old versions of the packages form official archives, however you most likely need a backup of your `/bin/`,`/lib/`, `/usr/` and `/etc/` folders, because they contain files, essential for your system operation.

The separation of `/` and `/home/` backups is necessary, because they could have different retention policies.

`@home_unbacked` subvolume is mounted to `/home/.unbacked_files/` directory, where each user have its own subdirectory.
They can symlink to this folder those files, which they don't need to back up.
For instance, my `~/Videos/` , `~/Music/` and `~/.cache` folders are symlinked to `.unbacked_files`, so they will not waste space on the backup drive.

# snapper
Of course, you can do snapshots manually with `btrfs subvolume snapshot <subvolume> <destination path>`, be it is very verbose.

`snapper` is a small program for automating this task.
Basically, it manages 2 things:
- Automatic snapshot creation: time-based and as a pacman hook
- Automatic removal of old snapshot based on different retention policies

My  `snapper` config looks like this:
```bash
# subvolume to snapshot
SUBVOLUME="/"

# filesystem type
FSTYPE="btrfs"

# fraction or absolute size of the filesystems space the snapshots may use
SPACE_LIMIT="0.5"

# fraction or absolute size of the filesystems space that should be free
FREE_LIMIT="0.2"

# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="1"
TIMELINE_LIMIT_DAILY="7"
TIMELINE_LIMIT_WEEKLY="8"
TIMELINE_LIMIT_MONTHLY="4"
TIMELINE_LIMIT_YEARLY="0"
```
With this config you'll have a hourly snapshots and a sane retention policy.

However, **snapshots are not backups**.
They'll save you from accidental file deletion or system misconfiguration, but not from disk failure.

To truly have a backup, you'll need a separate drive, which stores you snapshots.

Luckily, `btrfs` allows you to send subvolumes (or snapshots) to different devices.

With `btrfs send` you can copy the entire subvolume to `stdout` and receive it with `btrfs receive` on the backup drive.

So, with another (ideally, offline) `btrfs` disk your snapshots will be backed up and deduplicated thanks to copy-on-write semantics.

However, this approach has several downsides:
1. Backup drive **must** be formatted as `btrfs`
2. Snapshots have to be moved manually
3. Storing backups of several subvolumes will be a mess
4. You'll have to manually remove old snapshots

# borg
I'm using [borg](https://borgbackup.readthedocs.io/en/stable/) as my main backup utility for several years already.
It has all the backing up features I have ever needed:

- block-level deduplication
- backup compression
-  does not depend on the file system: `borg` archive is just a folder with files, no hardlinks, symlinks of other special features
-  has rich retention policy settings
-  possibility to mount backups as FUSE file systems

Before I've moved to `btrfs`, I've used `borg` to backup my entire system drive like this:
```bash
#!/bin/bash
TARGET=/path/to/borg/archive
BACKUPS= dirs to be backed up
BORG_PASSPHRASE=<passphrase>
borg create --stats --progress \
	$TARGET::{now} \
	$BACKUPS \
	--exclude-from /path/to/exclusions/file
```
And it was very convenient: I've added this script to `cron` and it did everything automatically.

Occasionally, I've run `borg prune -v --list --stats $PRUNE_OPTS $TARGET` to delete old backups.
In this command `PRUNE_OPTS` is a variable, which contained retention policies.
In my case they were:

- `-d 1` keep a single backup for the last day
- `-w 8` keep a single backup for each of the last 8 weeks
- `-m 6` keep a single backup for each of the last 6 months
- `-y 5` keep a single backup for each of the last 5 years

For general file systems it is already a very good solution, but with `btrfs` we can do better:

# snapborg
Actually, `borg` can backup not only directories, but also the entire block devices and even `stdin`[^1].

[^1]:you can pipe some data to it and it will be stored as `stdin` file in the `borg` archive

This means that at least we can pipe `btrfs send` to `borg`, but this way we will not be able to mount subvolumes.

Luckily, we can instruct  `borg`  to backup snapshot volumes discarding path to subvolumes, so from the `borg`'s point of view, snapshots are different states of a single file system instead of different subfolders.

And this is already implemented in [snapborg](https://github.com/enzingerm/snapborg)!

More over, it is implemented in a fail-safe way: before starting back up `snapborg` disables retention policies on snapshots to be backed up, so they will not be removed during backup process.
After backing everything up it will return retention policies back to the previous values.

My `snapborg` config looks like this (it is slightly edited example config `snapborg` repo):
```yaml
# snapborg sample configuration file. Default values are used for the sample
# configuration, thus can be omitted or changed


# List of mappings of snapper configs to their borg repos
configs:
    # MANDATORY: name of the snapper config
  - name: root
    # MANDATORY: borg repo target, e. g. backupuser@backuphost:reponame
    repo: /path/to/borg/repo
    # if this is set to true, borg does not neccessarily fail when a backup fails
    fault_tolerant_mode: false
    # snapborg fails when the most recent snapshot transferred successfully is
    # older than the time period given here. Set to '0d' to disable this behaviour
    last_backup_max_age: 0d
    # archive creation/storage options
    storage:
      # use either none or repokey encryption, defaults to none
      encryption: repokey
      # MANDATORY when using repokey: literal key passphrase or path to file
      # containing the key passphrase. Ignored when using none encryption
      encryption_passphrase: <my borg passphrase>
      # compression configuration, see borg manual
      compression: auto,zstd,4

    # define retention settings for borg independently from snapper
    retention:
      keep_last: 1
      keep_hourly: 0
      keep_daily: 1
      keep_weekly: 8
      keep_monthly: 6
      keep_yearly: 5
    # exclude patterns (see borg help patterns)
    exclude_patterns: []
```
Whey you run `snapborg backup`, it will get a list of local snapshots, merge it with the list of backups from `borg` archive and determine, which snapshots actually  need to be backed up (i.e. will not be immediately deleted by retention policy).
It will disable retention policy for this snapshots and back up them to `borg`.

When the snapshots are backed up to `borg` archive, `snapborg` will run `borg prune` with the retention policy from the config, removing backups, which are no longer needed.

# Final workflow
With `btrfs`, `snapper` and `snapborg` I have a very easy to use backup system:
- `snapper` automatically[^2] create `btrfs` snapshots and take care of removing the old ones
- occasionally I connect my backup hard drive to the laptop and run `snapborg backup`, so that the snapshots are backed up.
[^2]: via `systemd`  service

... and that's it!
That's all the things, which I need to do, to maintain my backup chores.

Thanks to snapshots, I do not need to worry about the moment, when I need to back them up.
If I update some file, the previous version will still be in the snapshots, so it will still be backed up, even if the changes were made _before_ actual backup.

For me, it is a killer-feature and I am very proud, that I've came up with this setup.
