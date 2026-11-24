---
title: "Home backup, part 6: the same video twice, with different bytes"
date: 2026-11-27T08:00:00-07:00
draft: false
translationKey: "home-backup-part-6"
tags:
  - backup
  - python
  - sqlite
  - linux
  - homelab
summary: "Two videos that look the same on screen, and an MD5 that says they are different. How I found 1,654 near-duplicates without deleting a single real photo, and why the file name never decides anything."
comments: true
---

In [part 5](/blog/posts/home-backup-part-5/), I fixed the backup scope and built the cold disk. This time, the question came up while I was looking at a folder of videos: why were `GOPR0006.MP4` and `GOPR0006(1).MP4` still there, if I had already run an MD5 deduplication over the whole disk?

## First: identical copies with a suffix

![IMAGE: the name does not decide, the content does](/blog/images/home-backup-part-6/6a-nome-nao-decide.png)

The deduplication I did in part 2 worked on whole folders: a folder only left if everything in it had a copy somewhere else. Repeated files inside the same folder (the classic `photo(1).jpg` next to `photo.jpg`) never counted.

With Claude's help, I wrote a script for that, with a simple rule:

1. The file has a copy suffix in its name: `(1)`, ` (2)`, ` - Cópia`, ` - Copy`. A 4-digit number, like `(2018)`, does not count (it is almost always a year).
2. There is a file in the **same folder** with the name without the suffix.
3. Both have the **same MD5** (the "fingerprint" of the content).

If all three hold, the copy goes to quarantine and the original name stays.

The result surprised me. Of 12,945 names with a suffix, only **1,226 (2.51 GiB)** were identical copies. Another **7,720** had an "original" with the same name, but with **different content**:

- cameras and phones restart their numbering (`GOPR`, `IMG_`, `DSC`) when the card changes;
- WhatsApp reuses names;
- phones name a photo by date and time, and two photos in the same second become `20181003_165933.jpg` and `20181003_165933(0).jpg`.

Had I deleted by name, I would have lost thousands of real photos and videos.

## And GOPR0006?

That pair was on the "different" list. But I had watched both, and they were the same.

I took a closer look:

- almost the same size: 9,552,522 and 9,565,817 bytes (0.14% apart);
- same recording time, to the second;
- same resolution (1920×1080) and codecs;
- duration 17.25 s against 17.18 s.

With `ffmpeg`, I computed a hash of the video and audio streams only (no header), and it still came out different. Neither file had the tags a GoPro writes. Conclusion: the same clip, uploaded twice to Google Photos and re-encoded on each upload. No file hash finds that, because the bytes really are different.

## Near-duplicates: the rule

![IMAGE: time, camera, duration and a visual check](/blog/images/home-backup-part-6/6b-quase-duplicatas.png)

Again with Claude, I put together a second script, report only. It reads the header of every photo and video with `exiftool` (recording time, camera, resolution, duration) and keeps it all in a separate SQLite database, without touching the catalog. Two files only form a group if **everything** matches:

- same type (video with video, JPEG with JPEG; RAW never mixes with JPEG);
- same recording time, to the second;
- same camera model (photos from two phones in the same second stay apart);
- same aspect ratio;
- for video, duration within 0.5 s.

That alone is not enough. In the first run, the script paired two consecutive photos from the same phone that the EXIF stamped with the same second. So a **visual check** came in: `ffmpeg` shrinks each image to 64×64 in grey and measures the mean pixel difference (0 to 255).

We (the AI and I) calibrated the limits on real cases:

- the same photo re-compressed scores below 1;
- consecutive time-lapse frames score around 2.5;
- two burst shots score 5 to 27.

For videos, the first version of the comparison got it wrong, and Claude fixed the logic after I showed it the numbers. Since the durations differ a little, "the frame at 50% of the video" landed on slightly different moments, and in an action scene that already changes the picture. Now, at each point, the script looks for the most similar frame within a ±0.35 s window. After that, every matching video pair scored below 1.

Final limits: photo below 1, video below 3. In each group, the best-quality file stays (bitrate for video, pixels for photos).

## What I decided to delete

Across the whole disk, the report found about 10 thousand "strong" pairs (same resolution) and 15 thousand "weak" ones (a reduced copy). But most of them were in **different folders**: the Google Photos version on one side, the organized folder on the other. And an earlier rule of mine says the same photo may live in two organized albums.

So I played it safe:

1. only inside the **same folder**;
2. only in the personal photo and video folders (no games, 3D projects or courses);
3. only the strong and weak groups (the "possible burst" ones stay).

Before running it, I asked Claude for a sample of 10 random pairs and opened both files of each pair side by side, through a folder of shortcuts with neutral names (`01-forte-fica.mp4`, `01-forte-sai.mp4`). All of them were right.

## Result

- **1,226** identical copies with a suffix (2.51 GiB).
- **1,654** near-duplicates in the same folder (8.39 GiB).
- Everything in quarantine, with a journal of every move and the catalog updated. Before moving each file, the script re-reads the MD5 of the one leaving and of the one staying.

That is 10.9 GiB on an 8 TB disk. The space gain is small, but I wanted to stop seeing the same photo twice, and now I know why MD5 did not catch these cases.

## What this does not do

- It does not compare photos without an EXIF date (screenshots, WhatsApp images). They stayed out: almost 89 thousand files.
- It does not merge the same photo kept in two different folders. That is on purpose.
- It does not catch edited versions (cropped or filtered). The aspect ratio or the picture changes, and the pair never forms.
- One catch: the next Google Photos Takeout will bring back the compressed versions I removed, and I will have to check them against the journal before importing.

## Next step

Now for real, the internal 1 TB drive with the second copy of the core set. It will be bigger than planned: the videos I recorded myself and the work folders that only exist here have to go in too.
