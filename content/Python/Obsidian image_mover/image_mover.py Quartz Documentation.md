---
tags:
  - Python
title: Obsidian image_mover.py
created: 2026-02-02T15:07:00
modified: 2026-02-02
---
Commit: https://github.com/hectorproko/Lab/commit/177885d4653a817c0c9f8fba5fe85553a3bd5851

# Problem it solves

Obsidian vaults often start as flat structures: hundreds or thousands of .md notes and embedded images (.png, .jpg, etc.) all in one root folder. While Obsidian's [[link]] syntax keeps everything connected beautifully during daily use, this becomes painful when you want to:

- Compile a set of related notes + visuals into a clean, self-contained folder (e.g., for portfolio articles, blog posts, GitHub repos, or sharing).
- Avoid breaking links in other notes that reference the **same shared images**.

Manually copying files risks orphaning links or duplicating assets. This script automates safe bundling.

# What the script does

Given a single target note (by path or name), the tool:

1. Scans your entire Obsidian vault to collect **every image reference** across **all**.md files.
2. Identifies which images are referenced **only** in the target note (unique → safe to move).
3. Prompts for a destination folder name (defaults to note title with underscores).
4. Creates the folder (if needed) in the note's parent directory.
5. Moves:
    - The target .md note itself
    - Only its **uniquely referenced** images
6. Leaves shared images untouched to prevent breaking links elsewhere.

Result: A tidy, portable sub-folder containing the note + its exclusive assets, ready for exporting or publishing — without side effects.

# Key technical highlights

- **File system handling** — Uses pathlib.Path for cross-platform safety (rglob, resolve, expanduser).
- Safe moves — shutil.move only on verified unique files + the note itself.

# Demo
![[obsidianmoverdemo.gif]]


