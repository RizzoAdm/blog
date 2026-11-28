---
title: "Home backup, part 9: I restored a whole folder and checked it file by file"
date: 2026-12-02T08:00:00-06:00
draft: false
translationKey: "home-backup-part-9"
tags:
  - backup
  - restic
  - homelab
summary: "A backup is only worth something if it comes back. I restored 17 GiB of documents to a separate place and compared the MD5 of every file with the original: 7,883 of 7,883 identical, and one warning that needed an explanation."
comments: true
---

In [part 8](/blog/posts/home-backup-part-8/), I ended by saying it was still missing proof that everything comes back. So far, the only restore test I had done was a small one: 19 random files. This time, I restored a whole folder.

## The folder and the place

I picked the documents folder of my archive (the Acervo): 17 GiB and 9,355 files, almost all small and many with accents or odd characters in the name. That is where a restore usually goes wrong.

The target was a new temporary folder on the SSD, inside my home. I did not use `/tmp` because on Ubuntu it is sometimes kept in RAM, and then the test would not really write to disk. At the end, the script deletes the folder by itself, but only if there is no problem.

## What the test checks

With Claude, I extended the old test script to answer three questions:

1. **Did everything come back?** The script lists what the snapshot (the backup's "picture") has in that folder and checks that every file was restored.
2. **Is the content the same?** It compares the MD5 (a fingerprint of the file) of each restored file with the original.
3. **What changed after the backup?** The snapshot is from a point in time. If I edited a file later, it will differ, and that is not a backup error. The script sorts these cases by the modification date.

The result falls into classes. Problem: different content while the original did not change, or a file that did not come back. Informative: changed later, left the folder, is new. Warning: an old file that is in the folder but not in the backup.

Before the real run, I tested it with a fake restic and 10,008 files, including one with a line break in its name. Every class showed up where it should.

## Result

![IMAGEM: 7,883 of 7,883 files identical](/blog/images/home-backup-part-9/9a-resultado.png)

- Restore: 7,883 files in 2 min 28 s (about 110 MiB/s, reading from the backup disk in the USB dock).
- Comparison: 2 min 16 s.
- **7,883 identical (16.28 GiB). No differences, nothing missing, no errors.**

And the warning: **1,472 files** were in the folder and not in the backup. It looked like a hole. But they all came from a single folder of old projects, all inside `node_modules` (the libraries npm downloads for a JavaScript project). That name is in my exceptions list on purpose. I thought about adding an exception for the archive and decided to keep the rule: the projects' source code is in the backup, only the libraries stay out.

(The same night, the first scheduled runs happened: restic at 2:00 and the cloud sync at 3:30. Both finished without errors.)

## What this does not do

- It tests one folder and one snapshot, not the whole backup.
- It is not a disaster test: I did not restore on a new PC, without the main disk.
- It does not test the cloud copy.
- The split between "changed later" and "problem" depends on the modification date. A file changed without changing its date would show up as a problem (which is better than the other way around).

## Next step

The test can now be repeated on any folder. What is left is deciding how often to run it, and the bigger test: restoring everything to an empty disk. Before that, I want to go folder by folder through what is duplicated between the Acervo and the cloud.
