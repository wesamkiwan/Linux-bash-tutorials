# 📋 Module 2 Cheat Sheet — Filesystem Navigation & File Operations

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Filesystem** | The OS's system for organizing files/directories on disk |
| **Directory** | Linux word for "folder" |
| **Root directory** (`/`) | The single top of the entire filesystem tree |
| **Working directory** | The folder you're "standing in" right now (`pwd` shows it) |
| **Absolute path** | Full path starting with `/`, valid from anywhere |
| **Relative path** | Path starting from your current location, no leading `/` |
| `.` | This directory |
| `..` | Parent directory |
| `~` | Your home directory |
| **Hidden file** | Any file/directory whose name starts with `.` |
| **Symbolic link** | A pointer file containing a path to another file (breaks if target is deleted) |
| **Hard link** | A second name for the same data on disk (data survives as long as one name remains) |

## FHS — Key Top-Level Directories

| Path | Purpose |
|---|---|
| `/` | Root of the entire tree |
| `/home` | Personal folders per regular user (`/home/you`) |
| `/root` | Home directory of the **root user** (not the same as `/`!) |
| `/etc` | System-wide config files |
| `/var` | Variable data: logs, caches, spool (`/var/log` = troubleshooting first stop) |
| `/tmp` | Scratch space, often wiped on reboot — never store anything important here |
| `/usr` | Bulk of installed programs, libraries, docs |
| `/bin` | Essential command binaries (symlinked to `/usr/bin` on modern Ubuntu) |

## Navigation

| Command | Action |
|---|---|
| `pwd` | Print current working directory (absolute path) |
| `cd <path>` | Move into `<path>` (absolute or relative) |
| `cd` | Go to home directory (shortcut for `cd ~`) |
| `cd ~` | Go to home directory explicitly |
| `cd ..` | Move up one level |
| `cd -` | Toggle back to the previous directory |

## Listing

| Command | Shows |
|---|---|
| `ls` | Names only |
| `ls -l` | Long format: permissions, links, owner, group, size, date, name |
| `ls -a` | Includes hidden (dotfile) entries |
| `ls -la` | Long format + hidden entries |
| `ls -lh` | Long format, human-readable sizes (`4.0K`, `1.2M`) |
| `ls -R` | Recursive listing into subdirectories |

### `ls -l` Output Legend

```
-rw-r--r-- 1 weki weki 2048 Jul 28 10:15 notes.txt
```

| Field | Example | Meaning |
|---|---|---|
| 1 | `-rw-r--r--` | Type + permissions (`-`=file, `d`=dir, `l`=symlink; owner/group/other rwx) |
| 2 | `1` | Hard link count |
| 3 | `weki` | Owner |
| 4 | `weki` | Group |
| 5 | `2048` | Size in bytes (or human-readable with `-h`) |
| 6 | `Jul 28 10:15` | Last modified |
| 7 | `notes.txt` | Name |

⚠️ Full permissions/`chmod`/`chown` deep-dive is Module 4 — this is read-only literacy for now.

## Creating

| Command | Action |
|---|---|
| `mkdir name` | Create one empty directory |
| `mkdir -p a/b/c` | Create nested chain, no error if parents already exist/are missing |
| `touch file` | Create an empty file, or update its timestamp if it exists |

## Copying / Moving / Renaming

| Command | Action |
|---|---|
| `cp src dest` | Copy a file |
| `cp -r src dest` | Copy a directory (and everything inside it) |
| `mv src dest` | Move a file/directory to a new location |
| `mv old new` | Rename in place (same directory = rename, not a move) |

## Deleting

| Command | Action | Danger Level |
|---|---|---|
| `rm file` | Delete a file | ⚠️ No undo |
| `rm -r dir` | Delete a directory recursively | ⚠️⚠️ No undo |
| `rm -rf dir` | Force-delete recursively, no prompts | ⚠️⚠️⚠️ No undo, no confirmation, no trash bin |

## Wildcard (Glob) Reference

| Pattern | Matches | Example | Matches |
|---|---|---|---|
| `*` | Zero or more of any character | `*.txt` | `a.txt`, `report.txt`, `.txt` |
| `?` | Exactly one character | `file?.txt` | `file1.txt` (not `file10.txt`) |
| `[...]` | One character from the set/range | `file[123].txt` | `file1.txt`, `file2.txt`, `file3.txt` |
| `[a-z]` | One character in a range | `[a-z]*.sh` | any lowercase-starting `.sh` file |

## Links

| Command | Creates |
|---|---|
| `ln target linkname` | Hard link (same underlying data) |
| `ln -s target linkname` | Symbolic link (pointer to a path) |

`ls -l` shows symlinks with an arrow: `linkname -> target`.

## Disk Usage

| Command | Answers |
|---|---|
| `du -sh path` | "How big is this file/folder?" (looks inward) |
| `df -h` | "How full is this disk/partition?" (looks outward, all mounts) |
| `df -h /` | Free/used space on the root filesystem specifically |

## 🔁 The Safe Delete Workflow

Do this **every single time** before running `rm -r` or `rm -rf`:

1. **Run `ls` (or `ls -la`) on the exact path first.** Confirm out loud (even mentally) what you're about to destroy.
2. **Double-check the path has no typos or stray spaces.** A misplaced space can turn one target into two, or an absolute path into an unintended one.
3. **Prefer `rm -r` over `rm -rf` when possible.** Let it prompt you on protected files — that's a safety net, not an annoyance.
4. **If you must force with `-f`, pause for 3 seconds and re-read the full command line before pressing Enter.**
5. **Never run `rm -rf` with a wildcard and a variable in the same command** (e.g. `rm -rf $VAR/*`) unless you've confirmed `$VAR` is not empty — an empty variable can turn `$VAR/*` into `/*`.
6. **Consider `mv` to a `trash/` folder instead of `rm` for anything you're not 100% sure about** — you can always empty it later.

## 🔁 The New Project Scaffold Workflow

Do this every time you start a new project folder:

1. `mkdir -p ~/projects/<name>/{src,docs,data}` — create the whole skeleton in one shot (brace expansion, a shortcut you'll formalize in later modules).
2. `cd ~/projects/<name>`
3. `touch README.md` — leave a marker file so the folder isn't empty and has a starting point.
4. `ls -la` — confirm the structure looks right before you start working.
