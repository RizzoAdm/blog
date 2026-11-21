---
title: "Home backup, part 5: the hole a folder list opened, and a cold disk that checks itself"
date: 2026-11-26T08:00:00-07:00
draft: false
translationKey: "home-backup-part-5"
tags:
  - backup
  - restic
  - linux
  - sqlite
  - homelab
summary: "My backup ran every night without errors, and an important folder was still outside it. The fix, and a 906 GiB cold disk checked file by file."
comments: true
---

In [part 4](/blog/posts/home-backup-part-4/), I took 326 GiB out of Google Drive, always with a verified copy on the archive disk. In this part, I looked at the backup again and found a problem I had created myself.

My backup ran every night, without errors. And an important folder was still outside it.

## How it happened

![IMAGE: backup by rule, not by list](/blog/images/home-backup-part-5/5a-escopo-por-regra.png)

When I set up the restic job (the backup program I use), I wrote a list of my archive folders: media, work and studies. A few days later, I merged the folders that were split between the Google Drive mirror and the local archive, and that created a new documents folder (finance included). I did not add that folder to the list. Then, when I took out of the cloud everything that already had a local copy, the finance folder existed only on the main drive and in older snapshots (a snapshot is the "picture" the backup takes each night).

I found the problem by chance, while measuring folder sizes for something else.

## What I changed

1. **A rule instead of a list.** The backup now saves the whole archive root, minus what I exclude on purpose (today, only the 3D asset library). A new folder goes in on its own.
2. **Snapshots grouped by machine and tag.** By default, restic looks for the previous snapshot by the folder list. If the list changes, it does not find the previous one and reads everything again (hours), and the rule for how many snapshots to keep treats the old ones as another group, which is never trimmed. With `--group-by host,tags`, the previous snapshot was found and the new backup took 4 minutes.
3. **I checked inside the backup.** I counted the files of each folder inside the snapshot and compared them with my catalog. They matched, except for 1,472 `node_modules` files (dependencies of an old web project), which stay out on purpose.

As a bonus, I saw the cleanup of old snapshots (the `prune`) work for the first time, the way I had configured it in part 3.

## The cold disk

![IMAGE: the 2 TB cold disk, read back and checked](/blog/images/home-backup-part-5/5b-disco-frio.png)

The 3D asset library (905.9 GiB, 146,734 files) is not in restic. It is downloaded material, mostly recoverable. Even so, I wanted a second copy.

I used an old 2 TB drive. It has had communication errors and overheating in the past, but its surface is clean. That is exactly why I chose it: the weaker disk holds the least valuable data, and the healthy 1 TB disk keeps what matters most.

1. I formatted it with a script that finds the disk by serial number and refuses to touch a mounted disk or the protected ones (the main drive, the backup drive and the SSDs).
2. I copied with `rsync` to NTFS (so any Windows machine can read it): two and a half hours, 0 errors.
3. I read everything back: I computed the MD5 (a "fingerprint" of the content) of every file **by reading the cold disk** and compared it with the original, path by path. Result: 0 missing, 0 different, 0 extra.
4. I made the disk **describe itself**. At its root there is a folder with an instructions file (the check commands right at the top), the disk's identity, a SMART history (the drive's own health self-assessment) that only grows, and the MD5 list of every file. Three months from now, I plug in only this disk and check everything, without depending on anything else.

A bonus lesson: putting the disk in the dual dock reset the dock and unmounted my backup disk, which was in the other slot. No backup was running, but now it is a rule: never touch one slot while the other is in use.

## What this does and does not do

It does: any new archive folder goes into the backup without me having to remember it, and the cold disk can be checked on its own, on any Linux machine.

It does not:

- The cold disk does not follow changes. It is a dated copy, refreshed by hand.
- It does not leave the house. The off-site copy is the next disk.
- Only plain-copy disks have the MD5 list. The restic disks are checked by restic itself, so file names stay encrypted.

## Next step

The internal 1 TB drive will get the second copy of the core set (documents, finance and photos) in its own restic repository.
