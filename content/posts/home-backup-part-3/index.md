---
title: "Home backup, part 3: the first restic backup (from 19 to 93 MB/s, and the prune trap)"
date: 2026-11-24T08:00:00-07:00
draft: false
translationKey: "home-backup-part-3"
tags:
  - backup
  - restic
  - linux
  - systemd
  - homelab
summary: "The second copy of the 3-2-1: 2.75 TiB onto an SMR disk, five times faster after one change, and a retention bug caught a week before it would have fired."
comments: true
---

With everything organized on a single disk (see [part 1](/blog/posts/home-backup-part-1/) and [part 2](/blog/posts/home-backup-part-2/)), it was time for the real backup. This post is about the **second copy** of the 3-2-1.

Tools for this part: Ubuntu, restic 0.18, systemd (service, timers and sleep inhibitor), ext4 and Bash.

## November 14: the first full backup

![IMAGE: first restic backup on the SMR disk](/blog/images/home-backup-part-3/3-restic-smr.png)

- **Source:** 458,458 files on the 8 TB main disk, plus my home folder.
- **Target:** a reused 4 TB Seagate ST4000DM004 in a USB 3 dock, formatted ext4, holding a restic repository (format 2, compression on).

One important detail: this disk is **SMR** (drive-managed). SMR is a technology that makes disks cheaper and bigger, but very slow when writes are not sequential.

### 1. Find the bottleneck before tuning anything

After a fast first half hour, the backup settled at 23 MB/s, with an ETA of 30 hours. A 20-second sample of `/sys/block/*/stat` answered the question:

- the target disk was writing 19 MB/s at 97% busy;
- the source disk was reading at only 11% busy;
- the bus was running at 5 Gbps and the CPU was 96% idle.

The SMR disk was choking on out-of-order writes: two *packs* written in parallel, plus the ext4 journal.

### 2. Make the writes sequential

I stopped the backup and restarted it with `-o local.connections=1 --pack-size 64`. Restarting cost little: restic is content-addressed, so the 468 GiB already stored were re-read in 40 minutes and nothing was written twice.

After the change: **93.5 MB/s written, at 87% busy**. The whole backup took 15 h 45 min, with fast stretches (105–126 MB/s) and long slower ones (31–41 MB/s).

### 3. The prune trap

While cleaning up the 600 orphaned *packs* left by the interrupted run (10 GiB), a manual `restic prune` refused to start: **prune requires at least two connections**.

The scheduled job runs `forget --prune` with the same options as the backup. But `forget` only prunes when it actually forgets a snapshot. In other words: the job had passed today and would have failed about a week later, on the first day the retention policy kicked in.

The fix: the single connection now applies to the backup command only, and the job refuses at start-up any configuration that would break prune. Better to find out before hours of backup than after.

### 4. Proof, not assumption

- 2.749 TiB of source stored in 2.162 TiB (thanks to restic's compression and deduplication).
- Random files from every area, plus a 1.4 GB database, were extracted with `restic dump` and compared by MD5: **19 of 19 identical**.
- `restic check` read 1% of the data *packs* (538) with 0 errors.

### 5. Automation that fits how I use the machine

The dock is only switched on for backups. So:

- **mounting the disk starts the backup** (a systemd service tied to the mount unit);
- a 02:00 timer catches up on a missed backup when the computer boots;
- a separate daily check, which doesn't need the backup disk, warns after 7 days without a successful backup, showing how many files are still not backed up;
- root holds the sleep inhibitor (so the computer doesn't sleep halfway through), and restic runs as my user, never as root.

## What it doesn't do

- **It doesn't make SMR a good backup target.** Long runs still slow down, and I didn't measure the cause of the slow stretches (small files or the drive's cache filling up). It's acceptable here because the daily backups are incremental.
- **It doesn't generalize.** One connection and 64 MiB *packs* made *this* disk, in *this* dock, faster. The numbers apply to this setup only.
- **It doesn't prove the whole repository.** Reading 1% of the data is a sample; full reads need to be part of the routine.
- **It's not 3-2-1 yet.** This is copy two. The second local copy and the cloud copy are the next phases.

In [part 4](/blog/posts/home-backup-part-4/), Google Drive: merging split folders, a duplicate-name bug and 326 GiB out of the cloud.
