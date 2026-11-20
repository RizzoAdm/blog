---
title: "Home backup, part 4: cleaning up Google Drive and taking 326 GiB out of the cloud"
date: 2026-11-25T08:00:00-07:00
draft: false
translationKey: "home-backup-part-4"
tags:
  - backup
  - google-drive
  - rclone
  - python
  - sqlite
  - homelab
summary: "Folders split between the disk and the cloud, a Drive mirror that kept 'deleting' documents on its own, and 326 GiB taken out of the cloud without losing a byte."
comments: true
---

In the previous parts, I organized the drives ([part 1](/blog/posts/home-backup-part-1/) and [part 2](/blog/posts/home-backup-part-2/)) and made the first restic backup ([part 3](/blog/posts/home-backup-part-3/)). Google Drive was what was left.

First, how my Drive mirror works:

- My main archive lives on the 8 TB disk. Google Drive is a working area, not the "official" home of the files.
- Drive is copied one way only, from the cloud to the disk, with `rclone sync`.
- Any file the sync would delete on the disk is not deleted: the `--backup-dir` option moves it to a dated **set-aside folder**, and I review it later.
- The SQLite catalog has the path, size, date and MD5 of every file on the disk. So "do I still have this somewhere?" is a query, not a search.

Tools for this part: Ubuntu, Python (standard library only), SQLite, rclone and Bash.

## November 15: every folder whole, in one place

![IMAGE: merging folders split between the archive and the cloud](/blog/images/home-backup-part-4/4a-uma-pasta-uma-casa.png)

When I imported the old drives, every file that already had an identical copy in the Drive mirror was skipped as "already there". The result: the archive ended up with **skeleton folders**. Project folders holding only the files that existed nowhere else, while the rest lived only in the mirror. Nothing was lost, but no single place had the complete project.

The rule now is simple: **every folder lives whole in one place, the archive.** The cloud gets what I choose to upload.

1. **Measure before planning.** The mirror held 37,766 files (748 GiB). I synced what was pending and re-cataloged the disk before planning anything.
2. **Find where each cloud folder lives in the archive.** For every mirror file with a copy in the archive, the most similar path produces a rule such as "cloud folder X → Archive/Work/3D/X". A rule only counts if it is backed by at least 3 files and explains at least half of the files in that folder. Comparing by path, not only by content, is what exposes a skeleton: the folder exists on both sides but shares almost no files.
3. **Content is not an address.** On the first pass, travel folders were paired with the Google Photos export, and course material with the 3D asset library, because those areas contain the same files. Now they count as copies, never as the "home" of a folder.
4. **I choose the home, and the folder goes there whole.** Twelve cloud folders got a home in the archive, including new areas for documents. Each project is copied with its own structure instead of being scattered.
5. **Two copies of the same business folder become one.** My folder and a shared one held mostly the same content, reorganized: 737 identical paths, about 2,800 files at different paths and 1,408 only in the shared one. The shared folder was merged into mine: 2,315 files (21 GiB) skipped because they were already there, the unique ones went in at the same relative path, and 4 same-name conflicts were held for review.
6. **Native Google files.** The sync already exports Docs, Sheets and Slides to docx, xlsx and pptx. All 88 native files were matched to their exported copies; the 4 that can't be exported (forms and a map) were flagged for manual export.
7. **Copy with proof.** The plan: 28,341 files (690 GiB). The copy tool checks every line against the catalog and the disk, writes to the journal before copying, copies to a temporary file while hashing it, reads it back from disk and hashes it again, keeps the original date and only gives it its final name if the name is still free. It never overwrites and is safe to re-run.

## November 16: one name, two files

![IMAGE: the Google Drive mirror that kept "deleting" documents](/blog/images/home-backup-part-4/4b-um-nome-dois-arquivos.png)

The merge was finished: 35,378 files, 727 GiB, every byte read back and checked. The last step before cleaning the cloud was routine: run the sync, look at the dry-run, go.

**The dry-run said 28 documents had been deleted in the cloud. Nobody had deleted anything.**

Half an hour later, another dry-run wanted to copy 16 files back into the same folder, with nothing changed in the cloud. The log had the answer, 274 times:

```
Duplicate object found in source - ignoring
```

Google Drive is not a file system: **two files with the same name can sit in the same folder.** rclone has to turn the cloud into a path tree, so for each duplicated name it keeps one and ignores the other. Which one it keeps depends on the order the API returns them, and that order isn't stable. If the two have different content, every sync replaces one with the other. Nothing is lost, but the set-aside folder fills up with ghosts.

**I measured first.** `rclone dedupe --dedupe-mode list` only reads and lists the groups of repeated names: 272 groups, 186 identical (repeated uploads, mostly in two photo folders) and 86 with different content.

**Then, the mistake.** The `--dedupe-mode skip` mode deletes only identical twins and sends them to the Drive trash (30 days to undo). Since my mirror is read-only, I created a second access with write permission just for this operation. Dry-run: 178. Real run: 178, no errors.

Then, checking one of the pairs: the file left in the folder was a **shortcut**. The real file was in the trash.

rclone follows Drive shortcuts, and a shortcut has the same size and MD5 as the file it points to. To dedupe, "a file plus a shortcut to it" looks the same as "two identical files", and it had no reason to prefer the real one. I restored all 178 from the trash on the web. Nothing was lost (the trash did its job), but it had to be redone.

**The fix.** With `--drive-skip-shortcuts`, rclone ignores shortcuts. The same list now showed 253 groups and 174 identical pairs of real files. The dedupe ran again: 174. Then a script checked every deleted file: is the identical twin still in the cloud? Still in the mirror? Is there a copy with the same MD5 in the archive? **174 / 174 / 174.** The write access was deleted the same hour.

**The second cause.** One folder of 28 files kept flipping. This time, the same folder ID showed up under two paths: the real one and a **folder shortcut** inside my own Drive. rclone put the files under one path or the other on each run. I removed the shortcut on the web and that was it.

**Closing the loop.** With the mirror stable, I cleaned up the set-aside folder: for each file, a script looks for a copy with the same MD5 in the archive, **re-reads both files from disk**, and only then deletes. 198 went. The 78 left were unique (old versions of documents still in use in the cloud, and a few work files deleted from it years ago). I decided to keep them all in the archive. When a different file with the same name already existed (the current version), the old version was copied next to it with the suffix `(Drive <date>)`. Before running on the real disk, I rehearsed the whole sequence on a test disk and catalog, and the rehearsal failed twice, right at the hand-offs between scripts. Better in a test than in the archive.

**Four rules that came out of this:**

1. **Never run `rclone dedupe` on Google Drive without `--drive-skip-shortcuts`.** A shortcut carries the hash of the file it points to and will always look identical to it.
2. **Write access to the cloud is temporary.** The mirror stays read-only; write access exists for one operation and is deleted as soon as it's verified.
3. **A deletion is verified after it happens, not only before.** The dry-run count matching the real count is necessary, but not enough. For every file removed, prove where its copies are.
4. **Rehearse the whole sequence, not just one step.** Each script was right on its own; the failures were in the hand-off from one to the next.

The 174 removed duplicates freed only 0.38 GiB. The point was never space: a mirror that keeps flipping files can't tell you what was actually deleted in the cloud, and that is the one question a backup has to answer.

## November 17: 326 GiB out of Google Drive

![IMAGE: 326 GiB taken out of Google Drive](/blog/images/home-backup-part-4/4c-326-gib-fora-da-nuvem.png)

With the mirror stable, it was time for the step the whole merge was for: **making the cloud smaller**.

The rule was written before starting: **nothing leaves the 8 TB disk except junk.** "Cold archive" means *off the cloud*, not *gone*. A folder leaves Google Drive only when every file in it is proven to be in the archive, byte for byte (and the archive is in the backup).

### The twins you can't pick from the web

Six groups were left that no script could settle: same name, same folder, different content. They were 2010 spreadsheets and a 2018 project plan. In four of them, the twins had **the same name, the same size and the same date**. The only difference was the MD5, and the Drive web interface doesn't show the MD5. There was no way to "pick the right one" by looking.

So I changed the question: instead of *which one should stay?*, *does it matter which one stays?* A read-only script checked all 13 versions: **all 13 had a copy in the archive.** With every version safe, the choice on the web stopped being a risk.

And that mattered. In two pairs the dates were different, and the plan was to keep the newer one. The dry-run disagreed: the 2010 versions had stayed and the 2020 ones had gone to the trash. A date column in a web list is a weak instrument. Nothing was lost: the set-aside rule found the copies in the archive, checked both sides and took care of it.

**The lesson:** when someone has to make a choice they can't verify, first make the choice not matter.

### 326 GiB, folder by folder

Five areas left the cloud: the old 3D course folders (254 GiB), the part of the 3D business folder owned by my personal account (46 GiB), the finance folder, the downloads folder and a 2018 trip video project (25 GiB). Each one went through the same loop:

1. **Prove the copies.** A read-only report showing how many files have a copy with the same MD5 outside the mirror, and where. Every area came back at 100%.
2. **Check for shortcuts** inside the folders leaving. None.
3. **Move it to the trash on the web.** The Drive trash keeps everything for 30 days.
4. **Dry-run.** Only one kind of line may appear, "move into backup dir", and the count must match the report: 1,327 for the 3D courses, 4,312 for the business folder, 4,251 for the other three. It matched.
5. **Sync.** With `--backup-dir` on the same disk, rclone doesn't delete anything, it just renames: 254 GiB moved in 19.5 seconds.
6. **Register, verify and delete.** And this is where a hole in the method showed up.

### The hole: the catalog didn't know where the files went

The script that deletes files from the set-aside folder looks each one up in the catalog by path. But the sync had just moved the files, and the catalog still had them at the old path. Run as it was, the script would have said none of the 1,327 files were in the catalog and deleted nothing. Safe, but useless. The only way out would be re-cataloging 730,000 files, for every folder.

The fix was a new mode in the catalog repair script. Since `--backup-dir` keeps the relative path, every set-aside file has a matching catalog row with the same size and date. The script just moves that row to the new path and reuses the MD5 that was already there. That's safe because the old MD5 is never trusted on its own: the delete script still re-reads both files from disk before removing anything. The repair takes seconds; the verification still takes its full hour for 254 GiB, and that's how it should be.

Two special cases got their own explicit option, instead of a decision buried in a script:

- **Empty files.** A 0-byte file has no content to prove a copy of, so it can never pass the copy test. They are deleted with their own log line.
- **Someone else's files.** A shared course folder edited by another account leaves old versions in the set-aside folder after almost every sync. The current version is still in that account's cloud and in my mirror. I decided those versions can be deleted, knowing that in-between versions are lost, and that decision is on record.

### What stays in the cloud

The raw wedding and honeymoon videos, about 306 GiB, were the other candidate to leave. They stay. The 8 TB disk and the restic backup are in the same house; until the rotating offsite copy exists, the cloud is the only copy of those files outside the house. Space is not the main reason to keep data in two places, and irreplaceable files are the last thing to take out of the only offsite copy.

The Drive trash also stays full for now. Google only frees the space when the trash is emptied, so the 326 GiB will show up in the account in about 30 days. Those are the same 30 days that make every one of these removals reversible from the web.

### The numbers

| | Files | GiB |
|---|---:|---:|
| Old 3D course folders | 1,327 | 254.37 |
| 3D business folder (personal-account part) | 4,312 | 46.16 |
| Finance + downloads + 2018 video | 4,251 | 25.63 |
| **Out of the cloud** | **9,890** | **326.16** |
| Mirror before → after | 37,658 → ≈ 27,770 | 747.7 → ≈ 421.5 |

Sync errors: 0. Files deleted without a verified copy in the archive: 1 empty file and 3 versions of another account's files, both by explicit rule. Everything else: re-read on both sides, with a journal line before each deletion.

## What it does and what it doesn't

It guarantees that nothing personal leaves the cloud unless an identical copy is on the archive disk, read and compared on the day of the deletion.

It doesn't guarantee that the archive disk survives. That's the backup's job, and the reason the next step is a second copy on another disk. It also doesn't choose between versions: when two versions of a file differ, both stay in the archive, and I decide later.
