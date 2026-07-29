# 🎤 Module 2 Interview Prep — Filesystem Navigation & File Operations

## Conceptual Questions

### 🟢 Beginner

**Q1: What does it mean that Linux has "a single filesystem tree," and how is that different from Windows?**

> "Windows organizes storage into separate drive letters like `C:\` and `D:\`, each with its own tree. Linux instead has one single tree that starts at the root directory, written as `/`. Every file and folder on the system — even if it physically lives on a second hard disk — is somewhere underneath that one root. A second disk gets attached, or 'mounted,' onto a folder within that same tree rather than getting its own separate letter."

**Q2: What's the difference between an absolute path and a relative path?**

> "An absolute path always starts with a forward slash and describes the complete route from the root of the filesystem, so it works no matter where you currently are — like `/home/weki/notes.txt`. A relative path doesn't start with a slash; it's interpreted starting from your current working directory, so the same file might be referred to as just `notes.txt` if you're already standing in `/home/weki`."

**Q3: What do `.`, `..`, and `~` mean?**

> "A single dot means 'this directory' — the one you're currently in. Two dots mean the parent directory, one level up. And a tilde is shorthand for your home directory, so `~/projects` means 'the projects folder inside my home directory,' no matter where I currently am."

**Q4: How do you create a nested directory structure in one command, and why would plain `mkdir` fail?**

> "`mkdir -p path/to/nested/dir` creates the whole chain at once, even if none of the intermediate folders exist yet. Plain `mkdir` without `-p` would fail with an error if any parent in that path is missing — it refuses to guess your intent, which is actually a safety feature, not a limitation."

**Q5: Why does `mv` handle both moving and renaming a file?**

> "Because they're conceptually the same operation. Renaming a file is just moving it to a new name within the same directory. There's no separate 'rename' command in Linux — `mv oldname.txt newname.txt` moves the file from one name to another in place, which is exactly what `mv path/to/oldname.txt path/to/newname.txt` does when the destination directory happens to match the source."

### 🟡 Intermediate

**Q6: Walk me through every column in a line of `ls -l` output.**

> "Take `-rw-r--r-- 1 weki weki 2048 Jul 28 10:15 notes.txt`. The first character tells you the type — a dash for a regular file, `d` for a directory, `l` for a symlink. The next nine characters are the permission bits in three groups of three — read, write, execute — for the owner, the group, and everyone else. The number after that is the hard link count. Then you get the owner name, the group name, the size in bytes, the last-modified timestamp, and finally the filename itself."

**Q7: What's the difference between a symbolic link and a hard link?**

> "A hard link is a second name pointing at the exact same underlying data on disk — there's no 'original' versus 'copy,' both names are equally real, and the data survives as long as at least one hard link to it still exists. A symbolic link, created with `ln -s`, is a small separate file that just stores a path to another file. If you delete the target that a symlink points at, the symlink itself still exists but is now broken — it points at nothing."

**Q8: Why is `du` different from `df`, and when would you use each?**

> "`du` looks inward at a specific file or folder and totals up how much space it's consuming — I'd use `du -sh` on a suspect directory like `/var/log` to find what's eating space. `df` looks outward at the whole disk or partition and reports free versus used space across every mounted filesystem. If a monitoring alert says a server's disk is almost full, I'd check `df -h` first to confirm which mount is the problem, then use `du -sh` on folders within it to hunt down the specific culprit."

**Q9: What actually happens when the shell sees a wildcard like `*.txt` in a command?**

> "The shell expands the wildcard into a list of matching filenames *before* the command ever runs — this is called globbing. So `ls *.txt` doesn't actually pass the literal string `*.txt` to `ls`; Bash first scans the current directory, replaces `*.txt` with every matching filename, and only then executes `ls` with that full list as its arguments. That's an important distinction that matters a lot once you start writing scripts, because if zero files match, some shells pass the literal, unexpanded pattern through instead of an empty list."

### 🔴 Advanced

**Q10: Why is `rm -rf` specifically more dangerous than `rm -r` alone, and what precautions should a production script take?**

> "`rm -r` recurses into a directory and deletes everything inside it, but it can still stop and prompt on certain protected files. Adding `-f` forces past every one of those prompts and confirmations — it just deletes, no questions asked, no undo, and there's no trash bin to recover from on the command line. In production scripting, I'd never hardcode a bare `rm -rf` with a variable path — I'd validate the variable isn't empty first, prefer absolute paths, and ideally test destructive logic with `echo` in place of `rm` before ever running the real command against production data."

**Q11: How does a hard link's behavior differ from a symlink's when the original file is moved to a different filesystem/partition?**

> "Hard links can only exist within the same filesystem/partition, because they work by pointing directly at the same underlying data block (technically the same inode) — you physically can't create a hard link across two different disks. A symbolic link has no such restriction, since it's just a text pointer to a path, so it can point to a target on a completely different filesystem, mounted drive, or even a path that doesn't exist yet. That's one of the key practical trade-offs between the two."

---

## Practical/Coding Questions

**Q1: Show how you'd create a directory structure `app/config`, `app/logs`, and `app/data` in one command, then confirm it exists.**

Solution:
```bash
mkdir -p app/{config,logs,data}
ls -la app
```
Explanation: `mkdir -p` with brace expansion creates all three subdirectories under `app` in a single command, and would also create `app` itself if it didn't already exist. `ls -la` confirms the result.

**Q2: You have a directory `old-site` and want a full copy named `old-site-backup` before making risky changes. Show the command, and explain what happens if you forget a flag.**

Solution:
```bash
cp -r old-site old-site-backup
```
Explanation: `-r` (recursive) is required because `old-site` is a directory, not a single file. Without `-r`, `cp` refuses and prints an error like `cp: -r not specified; omitting directory 'old-site'` — it won't silently do a partial or wrong copy, it just stops.

**Q3: You need to move every `.log` file from the current directory into an `archive/logs` folder that may not exist yet. Show the full command sequence.**

Solution:
```bash
mkdir -p archive/logs
mv *.log archive/logs/
```
Explanation: `mkdir -p` guarantees the destination exists (creating both `archive` and `logs` if needed) without erroring if it's already there. The wildcard `*.log` expands to every matching filename in the current directory before `mv` even runs, so all of them move in one shot.

**Q4: Show how you'd safely delete a directory called `temp-build`, following good practice, rather than just blindly running `rm -rf temp-build`.**

Solution:
```bash
ls -la temp-build
rm -r temp-build
```
Explanation: I run `ls -la` on the exact target first to visually confirm what's about to be deleted and that the path is correct. I use `rm -r` instead of `rm -rf` unless I have a specific reason to force past prompts — letting `rm` stop and ask on protected files is a safety net I don't want to throw away without cause.

**Q5: How would you find out how much space your home directory is using, human-readable, without listing every single file inside it?**

Solution:
```bash
du -sh ~
```
Explanation: `-s` gives a single summarized total instead of a line per file/subdirectory, and `-h` converts the raw byte count into a human-readable unit like `1.2G`. Without `-s`, `du` would recursively print a size for every subdirectory, which is far more output than needed for a quick "how big is this" check.

---

## Gotcha Questions

**Q1: "I ran `rm -rf /` (or a variant like `rm -rf $DIR/` where `$DIR` was accidentally empty) — what actually goes wrong?"**

> Trap: Candidates sometimes think Linux has a built-in guardrail that stops you from deleting the whole system. Modern `rm` on Ubuntu does refuse a bare `rm -rf /` by default (it requires an extra `--no-preserve-root` flag to override that specific safeguard) — but that protection does **not** extend to something like `rm -rf $DIR/` where `$DIR` is an unset or empty variable, because that silently collapses to `rm -rf /` in effect once the shell expands it, and the guard only checks for the literal `/` argument in simple cases, not always every expanded path. The real lesson: always validate variables before using them in a destructive command, and never assume forgiveness where you haven't verified it.

**Q2: "`cp mydir newdir` failed but `cp -r mydir newdir` worked — was the first one broken?"**

> Trap: Nothing is broken. Plain `cp` refuses to copy a directory at all — it explicitly requires `-r` (recursive) to walk into a directory and copy its contents. The error is a deliberate safeguard, not a bug, so the candidate should recognize this as expected, correct behavior, not something to "work around" carelessly.

**Q3: "I typed `ls report?.txt` expecting it to match `report10.txt` too, but it only matched `report1.txt`—`report9.txt`. What happened?"**

> Trap: `?` in a glob matches **exactly one** character, not "one or more." `report10.txt` has two characters after "report" (`1` and `0`), so it doesn't match `report?.txt` at all — only `*` would match a variable number of characters. This is a classic wildcard mix-up between `?` and `*`.

**Q4: "I deleted a file that a symlink pointed to, but `ls -l` still shows the symlink — did the delete not work?"**

> Trap: The delete worked fine — the symlink itself is a separate, tiny file that just stores a path string. Removing the target it points to doesn't remove the symlink; it just leaves the symlink "dangling" or "broken," now pointing at a path that no longer resolves to anything. Many systems will highlight a broken symlink in a different color when you run `ls -l`, but the symlink file itself persists until you explicitly remove it.

---

## Quick-Fire Rapid Review

- **Q: What character does an absolute path always start with?** A: `/`.
- **Q: What does `..` mean?** A: The parent directory.
- **Q: What does `~` expand to?** A: Your home directory.
- **Q: Command to jump back to the previous directory?** A: `cd -`.
- **Q: Flag to see hidden files with `ls`?** A: `-a`.
- **Q: Flag to see human-readable sizes with `ls` or `du`?** A: `-h`.
- **Q: Flag required to copy a directory with `cp`?** A: `-r`.
- **Q: Is there a separate "rename" command in Linux?** A: No — `mv` handles renaming by moving a file to a new name in the same directory.
- **Q: What does `mkdir -p` do differently from plain `mkdir`?** A: Creates all missing parent directories along the path instead of erroring.
- **Q: What's the most dangerous combination of flags on `rm`?** A: `-rf` — recursive and forced, no prompts, no undo.
- **Q: What does `*` match in a glob pattern?** A: Zero or more of any character.
- **Q: What does `?` match in a glob pattern?** A: Exactly one character.
- **Q: What happens to a symlink if its target is deleted?** A: It becomes a broken/dangling link, still present but pointing at nothing.
- **Q: Which command reports free space on a whole disk/partition?** A: `df -h`.
- **Q: Which command reports the size of a specific folder?** A: `du -sh <path>`.
