---
title: "Home backup, part 2: delete only with proof (cleanup, deduplication and reorganization)"
date: 2026-11-23T08:00:00-07:00
draft: false
translationKey: "home-backup-part-2"
tags:
  - backup
  - linux
  - python
  - sqlite
  - homelab
summary: "After merging the drives, the cleanup: re-catalog everything, decide which copy stays, and reorganize 2.75 TiB without losing track of a single file."
comments: true
---

In [part 1](/blog/posts/home-backup-part-1/), I merged the content of five old drives onto an 8 TB disk. Now comes the delicate part: cleaning up. Deleting is easy; deleting while knowing nothing was lost is another story.

The rule stays the same: nothing is deleted because it looks redundant. It is deleted because a hash says it is.

Tools for this part: Ubuntu, Python (standard library only), SQLite, rsync, rclone and 7-Zip.

## November 7: prove it before you delete it

![IMAGE: re-cataloging and cleanup with proof](/blog/images/home-backup-part-2/2a-prove-antes-de-apagar.png)

After rescuing the two failing drives and importing a third, the main disk held 2 TiB of new content. Before removing anything, the catalog had to describe the disk as it is now, not as it was a week earlier.

1. **Re-catalog everything.** A full metadata pass over the 8 TB IronWolf: 933,033 files, 4.88 TiB, 0 errors. The import area matched the six import manifests exactly: 437,239 files.
2. **Hash what is new.** 452,970 files (2.18 TiB) in 4 h 06 min, with 0 read errors, 0 files changed mid-read and none left without a hash.
3. **Explain every gap.** 79,910 catalog rows no longer existed on disk. All of them came from two recorded changes: a quarantine batch already deleted, and files the Google Drive sync had set aside. No unexplained loss.
4. **Delete with evidence.** 356 GiB of Google Takeout archives were removed only after confirming that all 179,195 extracted files were on the disk with the same MD5.
5. **Junk goes to quarantine, but checked first.** 22.8 GiB of Drive app upload fragments, `desktop.ini` files and cloud shortcuts went to quarantine, with an "is this unique?" check before every move. An old trash folder with 2,229 files had 19 unique items, which were left for me to decide.

Every removal follows the same path: **report → review → quarantine → delete**, logged and with a way back.

## November 9: which copy stays?

![IMAGE: deduplication with proof](/blog/images/home-backup-part-2/2b-deduplicacao.png)

Starting point: 4.69 TiB on the disk, with 1,038.8 GiB of exact duplicates mapped by MD5.

Finding duplicates is easy. The real problem is deciding **which copy stays**.

1. **Folders before files.** A folder leaves only if every file in it has an identical copy outside it, in a place that will keep existing. "Dump" folders (phone backups, "redo" folders) go first; the organized tree goes last. A level rule prevents trading an organized folder for a copy sitting in the middle of the mess. Result: 331.4 GiB in a single old backup tree.
2. **File by file only where folders can't decide.** Inside dump folders, a file leaves only if the same content stays in a better-organized folder, re-hashed on both sides before the move: 16.4 GiB.
3. **Respect how people organize.** In a Google Photos export, the album copies stay and the year-folder copies go: 30,610 files, 75.2 GiB.
4. **A 3D project needs a twin, not scattered copies.** A project folder leaves only if one other folder holds all its files. 84 exact twins and 26 content twins were removed (23.3 GiB); 6 projects whose files exist only scattered stayed. A first version without priorities would have deleted 10.1 GiB of my organized asset library to keep the messy copy. Review caught it, and the rule changed.
5. **Open archives before deleting them.** 530 ZIP, 7z and RAR files were hashed member by member. 89 (8.7 GiB) went because everything in them was already in the asset library. 35 whose content exists only inside client projects stayed, to be added to the library later. The ones that couldn't be opened stayed too.

Every step: **dry-run → my review → quarantine → check on disk → delete**. In the end, **456.6 GiB of duplicates removed**, and a fresh scan found 736,051 files, exactly the expected count, with 0 errors.

A few things became clear:

- **Only identical MD5 counts.** No "similar".
- **The script doesn't decide the organization.** Duplicates between two equally organized folders (20.7 GiB) wait for me, in the reorganization phase.
- **A name is not proof.** The import from one of the failing drives, done by name, size and date to avoid re-reading the drive, brought in 5,104 Windows `desktop.ini` files as "unique". They were removed by exact name, and future imports already exclude that kind of file.

## November 11: moving 2.75 TiB

![IMAGE: reorganizing the disk by subject](/blog/images/home-backup-part-2/2c-reorganizacao.png)

With the disk clean, it was time to organize: turn the content of five imported drives into a single layout, organized by subject. That's **537,515 files, 2.75 TiB**. The challenge was moving everything while keeping the catalog exact.

1. **Rename, don't copy.** Every move is a plan I approve: origin → destination. The dry-run classifies each file as move, duplicate or conflict, and execution is refused if a single conflict remains. Since it's all on the same disk, files are moved with `os.rename`: instant, with dates preserved. The catalog path is updated in the same step, and a line is written to a journal before each operation. A full scan afterwards didn't need to re-hash anything and matched the expected count exactly: 724,732 files, 0 errors.
2. **Let the drives vote.** Merging the trees from five drives produced 4,687 groups with the same name, size and date but a different MD5. Wherever the healthy drives had a copy, they decided: 1,437 groups were real edits (every version kept). Silent corruption showed up only on the failing drive; those copies went and the good ones stayed.
3. **Names lie.** In a Google Photos export, IMG_1234 repeats across cameras and years. Matching the photos' auxiliary files (*sidecars*) by name alone deleted 659 of them, from 8 albums. No photo was lost, and the gap showed up in review. Matching now requires the same capture time, to the second, or the same MD5. The 659 will be rebuilt from a fresh export, by MD5, not guessed from names.
4. **Archives go only when their content exists extracted.** Every member hashed and found by MD5 in an asset or study folder. Copies inside client projects don't count. 25 archives (7 GiB) went and 10 stayed.

One lesson: **don't trust a script that moves files without updating the catalog.** One of the early batches did that. A repair tool reconciled everything, and since then every move is either recorded by the tool itself or reconciled before the next step.

## What it doesn't do

- **It doesn't decide where things belong.** The folder layout and the conflict rules are my decisions, written down before execution.
- **It doesn't delete in one step.** Everything goes through report, review and quarantine.
- **It doesn't "fix" what it can't prove.** When there is no proof, the file stays and the decision is mine.

In [part 3](/blog/posts/home-backup-part-3/), the backup at last: the first full copy with restic.
