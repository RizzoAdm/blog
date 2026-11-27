---
title: "Home backup, part 7: the copy that leaves the house (and a rule I should have made on day one)"
date: 2026-11-30T08:00:00-06:00
draft: false
translationKey: "home-backup-part-7"
tags:
  - backup
  - rclone
  - google-drive
  - systemd
  - homelab
summary: "I gave up on a third disk at home and sent my Windows profile folder to Google Drive: a plain copy, checked file by file, every day at 3:30 a.m., with a permission that keeps the script away from the rest of the Drive."
comments: true
---

In [part 6](/blog/posts/home-backup-part-6/), I found the near-duplicates and sent them to quarantine. The plan now was the internal 1 TB drive, with the second copy of the "core set". That is not what happened.

## First, the quarantine

Before deleting, I checked on disk, against the journal of each batch, that:

- every quarantined file was there;
- every original that stays still existed;
- nothing had gone back to its old place.

I also asked for one more sample of 20 near-duplicate pairs: the 10 with the largest visual difference (the riskiest) and 10 random ones. All of them were the same photo. I deleted 2,880 files (about 11 GiB), and the catalog was updated with them.

(The sample script stopped silently the first time. It worked with a few test lines, but with 1,200 lines `sort | head` dies from SIGPIPE, the "nobody is reading anymore" signal. The lesson: test with real volume.)

## The 1 TB that didn't fit

![IMAGE: what fits on the 1 TB](/blog/images/home-backup-part-7/7a-nucleo-nao-cabe.png)

I used the catalog to measure the **unique** size of each combination of folders (each MD5 counted once, the way restic does). The basic core set (documents, the Windows folder, the files from another computer and the Linux home folder) came to 464 GiB, half the disk. Anything more blew past it:

- with the 3D work: 1,208 GiB;
- with photos and videos, without Google Photos: 971 GiB.

So I changed my mind. Two copies at home (the main disk and the restic disk) are enough. What 3-2-1 is still missing is **the copy outside the house**, and I already pay for 2 TB of Google Drive, with 1.1 TiB free.

## The Windows folder in the cloud

I started with my Windows profile folder (Documents, Pictures, Videos): 52.6 GiB. A few choices:

1. **A plain copy, readable in Drive.** I can open any file from my phone without restoring anything. The price is no encryption.
2. **`rclone sync` with a versions folder.** The cloud mirrors the disk, but whatever would be deleted or overwritten goes to a dated folder (`_versoes/2026-11-27/`) instead of disappearing. Versions older than 6 months are removed.
3. **A new, narrow permission.** The rclone access that reads Drive for the local mirror is read-only. To upload, I created a second access with the `drive.file` permission: it only sees the files it created itself. To check, I listed Drive through this access before creating the destination folder: it came back empty. A wrong command has no way to delete the shared folders.
4. **Careful with the mirror.** The local mirror downloads the whole Drive, so it would bring the copy back to the disk. I added an exclusion to its command.

The first upload took about 5 hours (35,494 files, 0 errors). Then `rclone check` recomputed the MD5 of every local file and compared it with the MD5 Google stores: **35,494 matching, 0 differences**.

## A rule I should have made on day one

In [part 5](/blog/posts/home-backup-part-5/), an important folder was left out of the backup because the restic list said what goes **in**, and the folder was created after the list. Now the rule applies to everything: **the backup takes the whole folder, and the only list is the list of exceptions** (what stays out). The cloud copy already works this way: the exceptions file has `Downloads`, an old copy of documents and the Windows auxiliary files. Any new folder is included on its own.

## Every day, without me remembering

![IMAGE: restic at 2 a.m., cloud at 3:30, the same lock](/blog/images/home-backup-part-7/7b-agenda.png)

With Claude, I built a job runner on the same model as the restic one:

- a systemd timer at 3:30 a.m., which runs when I turn the PC on if it was off at that time;
- **the same lock as restic**: if both coincide, one waits for the other, and the disk is never read twice at once;
- it **refuses to run if the source folder is empty** (a sync from an empty folder would move the whole cloud copy into the versions);
- a desktop warning if 7 days go by without a successful upload.

I tested everything with a fake rclone before the real thing. The first run through systemd, with nothing new to send, took 49 seconds.

## What this does not do

- It does not encrypt. Anyone with access to the Google account can read the documents.
- It does not keep versions forever: after 6 months, an old mistake can only be fixed from restic, at home. (I still have to write the script that deletes old versions. It goes by the date in the folder name, because a moved file keeps its original date.)
- It does not cover the rest of the disk. Photos and work have two copies, both at home.
- restic still uses the old list of what goes in. It breaks the new rule.

## Next step

Replace the restic list with a list of exceptions: the whole disk, minus what I choose to leave out.
