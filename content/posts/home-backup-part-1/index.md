---
title: "Home backup, part 1: merging 5 old drives without losing a single file"
date: 2026-11-20T08:00:00-07:00
draft: false
translationKey: "home-backup-part-1"
tags:
  - backup
  - linux
  - python
  - sqlite
  - homelab
summary: "The start of a series on my new file organization and backup routine: a catalog of everything, and rescuing two failing drives."
comments: true
---

Hi everyone!

In this series of posts (yes, this won't be a single post, given how big the project is getting), I'll walk through my new file and folder organization, and the new backup routine I'm building for my files on my personal computer, in the cloud, on external drives and so on.

I don't have a dedicated NAS, so all my files live on my desktop's internal disk and on a few external drives. I also keep a lot of things in the cloud (Google Drive).

My main goals for this project are: a reliable and safe backup routine, and a "3-2-1 backup" (3 copies, 2 different media, 1 copy outside the house, the famous *offsite*) for my most important documents and files.

One rule applies to the whole series: **nothing is deleted because it "looks" like a duplicate. It only goes when a hash proves there is another identical copy.** (A hash, MD5 in my case, works like a fingerprint of the content: if two files have the same MD5, they are identical, whatever their names.)

Tools used: Ubuntu, Python (standard library only), SQLite, rsync, rclone and smartmontools.

## November 5: a catalog of everything

![IMAGE: catalog of the drives and Google Drive](/blog/images/home-backup-part-1/1a-catalogo.png)

The first step was knowing exactly what I have and where.

- **A content-based catalog.** More than 1.4 million files, from every drive and from Google Drive, each recorded in a SQLite database with path, size, modification date and MD5. Zero hashing errors. Whether two files are duplicates is decided by the hash, never by the name.
- **Import only what is unique.** The catalog builds the copy list (one copy of each content, no auxiliary files), rsync copies it, and every file is checked again by MD5 at the destination. Result: 13,886 of 13,886 matched.
- **Google Photos.** The Google Photos API no longer lets third-party tools access a user's library. So I use scheduled Google Takeout exports delivered to Drive, downloaded with checksum verification and cataloged: 358 GiB, 179,195 files.
- Before deleting an old export, I compared it with the new one by hash. **6.9 GiB existed only in the old one** and were rescued first.

And what it doesn't do: the system never deletes anything on its own. Every removal goes through a report, my review and a dated quarantine.

## November 7: rescuing two failing drives

![IMAGE: rescuing the two failing drives](/blog/images/home-backup-part-1/1b-resgate-hds.png)

Two of the old drives are dying. They hold old personal backups and years of 3D work: client projects, asset libraries, textures and renders.

- **Seagate ST2000DM001** (2 TB, 3.5"): 14,012 interface CRC errors and a history of overheating.
- **Seagate ST4000LM024** (4 TB, 2.5", SMR): failed its extended SMART self-test at 20%, with 8 pending sectors after only 178 power-on hours.

The rule here was: **read as little as possible, but prove everything.**

1. **Metadata first.** I read only the NTFS index (name, size, date) and matched it against the catalog, which already had the MD5 of everything on the healthy drives. 99% matched without opening the file contents.
2. **Copy what is at risk first.** Everything that didn't match, plus every text and config file: 21,580 files, 217 GiB, each one checked by hash at the destination.
3. **Measure the guess.** Random hash samples found 0.6% to 1.5% false matches: asset metadata, Substance files and even `.blend` scenes rewritten with the same size and date. Metadata is a clue, not proof.
4. **Full hash pass, read-only.** The 593,830 files that "matched" were read on the failing drives and compared with their reference copies, in 14.5 hours. In total, 1,228 were different and were copied.

Result: **22,805 unique files (222.4 GiB) rescued**, every other file proven identical by MD5 to a copy on another drive, 0 read errors, and SMART counters unchanged after the full read.

Same week, a healthy drive: 1,389.9 GiB (217,215 files) imported from a Seagate ST4000DM004 into an 8 TB IronWolf (ST8000VN004). The 3D folders were copied as whole projects, so the `.blend` files keep their textures. Checked by MD5: 217,215 of 217,215.

## What it doesn't do

- **It doesn't repair the drive.** A bad sector is a physical defect; remapping only hides it.
- **It doesn't delete anything.** The old drives stay untouched until a final check proves every file exists somewhere else.
- **Putting everything on one disk is not a backup.** The encrypted 3-2-1 backup with restic comes later.

In [part 2](/blog/posts/home-backup-part-2/), the cleanup: how to delete and deduplicate without relying on guesses.
