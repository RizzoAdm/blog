---
title: "Home backup, part 8: only the list of what stays out (and a guard for the day the disk shows up empty)"
date: 2026-12-01T08:00:00-06:00
draft: false
translationKey: "home-backup-part-8"
tags:
  - backup
  - restic
  - rclone
  - systemd
  - homelab
summary: "I swapped restic's folder list for an exceptions list and found it opened a new risk: an empty disk passes the checks too. The guard that closes it, and the automatic clean-up of the cloud versions."
comments: true
---

In [part 7](/blog/posts/home-backup-part-7/), I made the rule that every backup takes the whole folder and the only list is the list of exceptions. The cloud copy already worked that way, but restic still used the old list, the one of what goes in. Its turn had come.

## The new list

The old list had eight folders of the main disk plus my home folder. The new one has two lines:

- the whole main disk (`/mnt/hd-dados`);
- the whole `/home` (this PC has only my user).

Everything else went into the exceptions list: the trash, the quarantine, the catalog copies and the games (they can be reinstalled). Two game saves come back. restic only re-includes a subfolder (`!`) when the parent folder is excluded with `/*`: `Jogos/*` works, `Jogos` does not.

I also went through the folders that start with a dot, where programs keep their settings. They go in, except the cache, what can be reinstalled and the secrets. (I found one more secret: a program's login file. It joined the exceptions list, next to the SSH keys.)

Before switching, I ran a dry run (a simulation that writes nothing). It came to 485,799 files, against 485,796 in the snapshot taken minutes before. In other words, the content is the same. The gain is for later: a new folder gets in on its own.

## The risk the new list created

![IMAGE: an emptied disk passes the checks; the guard holds](/blog/images/home-backup-part-8/8a-trava.png)

With the old list, if the disk did not mount, `/mnt/hd-dados/RIZZO` did not exist and the backup stopped with an error. With the new list, the path is `/mnt/hd-dados`, which is only the mount point and always exists, even empty.

The job runner already checked that the disk was mounted, so that case was covered. What was not covered: the disk mounted, but with a lot deleted (a mistake of mine, a bad sync, malware through Windows). restic would take a small snapshot without complaining. Then the retention policy (7 daily, 4 weekly, 6 monthly) would replace the good snapshots with bad ones, one by one, without any warning.

So, with Claude, I built a **shrink guard**:

1. After each backup, the job runner reads how many files the new snapshot has.
2. It compares that number with the last accepted count, kept in a state file.
3. If the new snapshot has fewer than 90% of the files, it **does not run the snapshot clean-up**: the new one is kept and no old one is deleted.
4. A desktop warning shows up and **stays on screen until I close it**. It has the commands to check the folders and, only if I deleted a lot on purpose, the command to accept the new size.

A second bad day is blocked again, because the reference stays the good count, not the previous day's. I tested it with a fake restic in 13 cases (empty disk two days in a row, exactly 90%, 89.9%, manual release...). The first real backup with the guard passed: 485,796 files against a reference of 485,681.

(When I sent the script to the PC, an older version arrived, because I reused the same temporary folder. Checking the MD5 before installing caught it.)

## Cleaning up the cloud versions

In the cloud, whatever the sync would delete or overwrite goes to a folder named with the date, for example `_versoes/2026-11-28/`. The script that deletes the folders older than 6 months was still missing. A few choices:

- **The age comes from the folder name**, not from the file dates. A file moved into the versions keeps its original date, so a 2019 file deleted yesterday would look old.
- **It goes to the Drive trash**, which keeps it for 30 days. It uses quota for a month, but a mistake can be undone.
- **It refuses to delete** if any folder is dated in the future (wrong clock) or if it would delete every folder.
- It runs **once a month**, on day 1 at 4:30 a.m., with the same lock as the other jobs.

On its first real test it failed: the versions folder does not exist yet, because no sync has replaced a file so far. I fixed it to treat "folder not found" as "nothing to delete".

## What this does not do

- The guard does not catch small losses. Deleting 5% at once goes through without a warning.
- It counts files, it does not check their content. A corrupted file with the same name counts the same.
- After a large clean-up on purpose, the night backup stays blocked until I release it by hand.
- The versions clean-up depends on the PC's clock. It refuses future dates, but a clock running behind would only delete less.

## Next step

The copies are in place: the main disk, restic at home and the Windows folder in the cloud. The photos already live in Google Photos, and the old work stays in the cold archive. What is missing is proof that all of it comes back. So far, my only restore test was small: 19 random files, compared by MD5. The next step is to restore a whole folder to a separate place and check it file by file against the original.
