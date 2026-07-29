# Module 2: Filesystem Navigation & File Operations 🟢

**Difficulty:** 🟢 Beginner
**Estimated Time:** 2.5 hours
**Prerequisites:** Module 1: Linux & Shell Fundamentals

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain the Linux filesystem hierarchy (FHS) and what the major top-level directories (`/`, `/home`, `/etc`, `/var`, `/tmp`, `/usr`, `/bin`, `/root`) are used for
- [ ] Tell the difference between absolute and relative paths, and use `.`, `..`, and `~` confidently
- [ ] Navigate the filesystem with `pwd` and `cd`, including `cd -`, `cd ~`, and `cd ..`
- [ ] List directory contents with `ls`, `ls -l`, `ls -la`, and `ls -lh`, and read every column of `ls -l` output
- [ ] Create directories and empty files with `mkdir`, `mkdir -p`, and `touch`
- [ ] Copy, move, and rename files and directories with `cp`, `cp -r`, and `mv`
- [ ] Delete files and directories safely with `rm`, `rm -r`, and `rm -rf` — and explain why `rm -rf` is dangerous
- [ ] Use wildcards (`*`, `?`, `[...]`) to operate on multiple files at once, and describe the difference between a symbolic link and a hard link
- [ ] Check disk usage with `du` and `df` at a basic level

---

## Module Goal

By the end of this module, you'll be able to move around any Linux filesystem — server, container, or laptop — like it's your own home, and manage files and folders without hesitation.

🎯 **On the job:** This is the single most-used skill set in any Linux role. Deploying an app, digging through log folders, setting up a project structure, cleaning up disk space on a server that's about to run out of room — all of it starts with knowing where you are, how to get somewhere else, and how to create, copy, move, and delete things without breaking anything. If Module 1 taught you to sit in the driver's seat, this module teaches you to actually drive.

---

## Core Concepts

### 1. What is a filesystem?

A **filesystem** is the way an operating system organizes and keeps track of files and directories on a storage device (like a hard drive or SSD). It's the system of rules that lets the OS answer questions like "where is this file physically stored?" and "what's inside this folder?"

### 2. What is a directory?

A **directory** is what most people call a "folder" — a container that holds files and other directories. In Linux, "directory" and "folder" mean the same thing; Linux documentation almost always says "directory," so that's the term we'll use.

### 3. The filesystem hierarchy — everything starts at `/`

Unlike Windows, which has separate drive letters (`C:\`, `D:\`), Linux has **one single tree** that starts at a single point called the **root directory**, written as just a forward slash: `/`. Every single file and directory on the entire system — no matter which physical disk it actually lives on — is somewhere underneath `/`.

💡 **Analogy — the upside-down tree:** Picture a tree growing upside down, with its trunk at the very top. That trunk is `/`. Every branch growing down from it is a directory, and every leaf is a file. No matter how deep you go, you can always trace a path back up to that one trunk. There's no "second tree" the way Windows has a separate `D:` drive — even a second hard disk gets grafted onto a branch of this same tree (this is called "mounting," which you'll meet again in a later module).

This standardized layout is called the **FHS** (Filesystem Hierarchy Standard) — a convention that most Linux distros follow so that software (and humans) can predict where things live.

### 4. Paths — how you describe "where something is"

A **path** is the "address" of a file or directory in the tree — a string like `/home/weki/notes.txt` that tells the shell exactly how to walk down from the trunk (or from where you currently stand) to reach that file.

### 5. Your current position — the working directory

At any moment, your shell has a **current working directory** (also just called "the working directory" or "cwd") — the folder you're "standing in" right now. Every relative path you type is interpreted starting from this location. You already met the command that shows it in Module 1: `pwd`.

### 6. Absolute vs. relative paths

- An **absolute path** always starts with `/` and describes the full route from the root of the tree, no matter where you currently are. Example: `/home/weki/projects/app`.
- A **relative path** does *not* start with `/`. It describes a route starting from wherever you currently stand. Example: if you're in `/home/weki`, then `projects/app` (no leading slash) means the same thing as the absolute path above.

💡 **Analogy:** An absolute path is like giving someone GPS coordinates — it works no matter where they're standing. A relative path is like saying "turn left, then it's the second door" — it only makes sense if you know where the person is starting from.

### 7. The special shortcuts: `.`, `..`, and `~`

- `.` (a single dot) means **"this directory"** — the one you're currently in.
- `..` (two dots) means **"the parent directory"** — one level up the tree from where you are.
- `~` (tilde) means **your home directory** — you saw this in Module 1's prompt (`user@host:~$`). On Ubuntu, this is typically `/home/yourusername`.

🎯 **On the job:** You'll type `cd ..` and `../` constantly to move up out of a folder, and `~/` to jump straight back to your home directory from anywhere — these are some of the highest-frequency keystrokes in daily Linux work.

---

## Detailed Explanations

### The major top-level directories (FHS essentials)

Here's what lives directly under `/` and why each one matters, at a beginner level. You don't need to memorize every corner of this — just build a rough mental map.

| Directory | What it's for | Why it matters on the job |
|---|---|---|
| `/` | The root of the entire filesystem tree — everything else lives under it | Every path traces back here |
| `/home` | Contains a personal subfolder for each regular user (e.g. `/home/weki`) | Where your own files, projects, and configs live |
| `/root` | The **home directory of the root user** (the superuser), separate from `/home` | Don't confuse this with `/` (root of the tree) — `/root` is a specific user's home folder |
| `/etc` | System-wide **configuration files** (mostly plain text) | You'll edit files here constantly to configure services (e.g. `/etc/ssh/sshd_config`) |
| `/var` | "Variable" data that changes while the system runs — logs, caches, spool files | `/var/log` is where you go first when troubleshooting a broken service |
| `/tmp` | **Temporary** files that any user/program can write to; usually wiped on reboot | Safe scratch space — never store anything important here |
| `/usr` | The bulk of installed user-facing programs, libraries, and documentation | Most non-essential software you install ends up somewhere under here |
| `/bin` | Essential command **bin**aries (programs) needed even in single-user/recovery mode, e.g. `ls`, `cp` | On modern Ubuntu, `/bin` is actually a symlink to `/usr/bin` — same content, historical split |

⚠️ **Warning:** Don't confuse `/root` (the root *user's* home directory) with `/` (the root of the whole filesystem). They're two completely different things that happen to share the word "root."

💡 **Tip:** You don't need to know every FHS directory by heart today. Just recognize these names when you see them — you'll build intuition for what lives where as you gain experience.

### Navigating: `pwd` and `cd`

You already know `pwd` ("print working directory") from Module 1 — it prints your current absolute path. The command to actually *move* is `cd` ("change directory").

| Command | What it does |
|---|---|
| `cd <path>` | Move into `<path>` (absolute or relative) |
| `cd` (no argument) | Go straight to your home directory — shortcut for `cd ~` |
| `cd ~` | Go to your home directory explicitly |
| `cd ..` | Move up one level to the parent directory |
| `cd -` | Jump back to the **previous** directory you were in (toggles back and forth) |

💡 **Tip:** `cd -` is a lifesaver when you're bouncing between two directories repeatedly — no need to type the full path each time.

### Listing contents: `ls` and its flags

`ls` lists what's inside a directory. On its own, it just prints names. Add flags to see more:

| Command | What it adds |
|---|---|
| `ls` | Plain list of names in the current directory |
| `ls -l` | **Long format** — one entry per line with permissions, owner, group, size, and date |
| `ls -a` | **All** — includes hidden files (names starting with `.`), which `ls` hides by default |
| `ls -la` | Combines both: long format, including hidden files |
| `ls -lh` | Long format with **human-readable** sizes (`4.0K`, `1.2M`) instead of raw byte counts |

A **hidden file** in Linux is simply any file or directory whose name starts with a dot, e.g. `.bashrc` or `.config`. It isn't hidden by permissions or magic — `ls` just filters dot-names out by default to reduce clutter. Configuration files are very often hidden this way.

### Reading `ls -l` output

This is the part beginners find intimidating at first — but it's just seven columns in a fixed order. Here's an example line:

```
-rw-r--r-- 1 weki weki 2048 Jul 28 10:15 notes.txt
```

| Column | Value | Meaning |
|---|---|---|
| 1 | `-rw-r--r--` | The permissions string (see below) |
| 2 | `1` | Number of **hard links** pointing to this file (more on this later in the module) |
| 3 | `weki` | The **owner** — which user owns this file |
| 4 | `weki` | The **group** that owns this file |
| 5 | `2048` | Size in bytes (or human-readable with `-h`) |
| 6 | `Jul 28 10:15` | Last modified date/time |
| 7 | `notes.txt` | The file or directory name |

**The permissions string, briefly:** The first character tells you the type: `-` for a regular file, `d` for a directory, `l` for a symbolic link. The remaining nine characters are three groups of three (`rwx` = read/write/execute) for **owner**, **group**, and **everyone else**, in that order. A dash means that permission is absent.

⚠️ **Note:** This module only teaches you to *read* these columns so `ls -l` isn't a wall of noise. The full deep-dive on permissions — `chmod`, `chown`, numeric modes like `755` — is coming in Module 4. Don't worry about mastering permissions yet.

🎯 **On the job:** The very first thing experienced engineers do when a script "won't run" or a file "won't open" is run `ls -l` on it — nine times out of ten, the answer is sitting right there in that permissions string or the owner column.

### Creating directories and files

| Command | What it does |
|---|---|
| `mkdir <name>` | Creates a new, empty directory |
| `mkdir -p <path/to/nested/dir>` | Creates a full chain of nested directories in one shot, even if the parents don't exist yet |
| `touch <filename>` | Creates a new, empty file if it doesn't exist; if it already exists, just updates its "last modified" timestamp |

💡 **Tip:** Without `-p`, `mkdir project/src` fails with an error if `project` doesn't already exist — `mkdir` normally refuses to guess. `-p` ("parents") tells it "create every missing directory along this path, don't complain."

### Copying and moving: `cp` and `mv`

| Command | What it does |
|---|---|
| `cp source dest` | Copies a **file** from `source` to `dest` |
| `cp -r source dest` | Copies a **directory** recursively (copies the folder and everything inside it) |
| `mv source dest` | Moves a file or directory to a new location — **or renames it** if `dest` is in the same directory |

**Why does `mv` also rename things?** Linux doesn't really have a separate "rename" command. Renaming a file is conceptually identical to moving it to a new name in the same folder — `mv oldname.txt newname.txt` moves the file from "oldname.txt" to "newname.txt" in place. There's no distinct concept to learn here; it's the same operation.

⚠️ **Warning:** Plain `cp` on a directory fails with an error like `cp: -r not specified; omitting directory`. You must add `-r` ("recursive") to copy a directory and everything inside it. Forgetting `-r` is one of the most common beginner mistakes.

### Deleting: `rm`

| Command | What it does |
|---|---|
| `rm <file>` | Deletes a file |
| `rm -r <directory>` | Deletes a directory and everything inside it, recursively |
| `rm -rf <directory>` | Same, but **forces** deletion without asking for confirmation, even for write-protected files |

⚠️⚠️ **CRITICAL WARNING — read this twice:** There is **no Recycle Bin, no Trash, no "Are you sure?" undo** on the Linux command line. When `rm` deletes something, it is gone. Not "gone until you empty the trash" — gone. `rm -rf` is especially dangerous because `-f` ("force") suppresses the confirmation prompts that would normally give you a chance to catch a mistake, and `-r` means it will happily tear through an entire directory tree, including everything nested inside it, without stopping. A single typo — an extra space, a wrong variable, a stray `/` — in an `rm -rf` command has ended careers and taken down production systems. **Always double-check the exact path before pressing Enter on any `rm -rf` command.** We'll cover a concrete safety workflow for this later in the module.

### Wildcards (globbing)

A **wildcard** (or **glob pattern**) is a special character that the shell expands into matching filenames *before* the command even runs. This lets you operate on many files at once without naming each one individually.

| Wildcard | Matches | Example |
|---|---|---|
| `*` | Zero or more of **any** characters | `*.txt` matches every file ending in `.txt` |
| `?` | Exactly **one** of any character | `file?.txt` matches `file1.txt` but not `file10.txt` |
| `[...]` | Any **one** character from the set/range inside the brackets | `file[123].txt` matches `file1.txt`, `file2.txt`, `file3.txt` |

💡 **Analogy:** Think of `*` as a wildcard playing card that stands in for "anything, any length" and `?` as a wildcard that stands in for exactly one letter — like a crossword blank.

🎯 **On the job:** Wildcards are how you clean up log directories, bulk-rename batches of files, or copy "every `.conf` file" in one command instead of typing each filename by hand.

### Symbolic vs. hard links

A **link** is a filesystem entry that points to another file, sort of like a shortcut. There are two kinds:

- A **hard link** is essentially a second name for the *exact same data* on disk. Both names are equally "real" — there's no original vs. copy. Deleting one name leaves the data intact as long as another hard link still points to it. (Remember that "links" column in `ls -l`? That's counting hard links.)
- A **symbolic link** (or **symlink**, created with `ln -s`) is a small special file that just contains the *path* to another file — like a signpost pointing elsewhere. If you delete the original file, the symlink becomes "broken" and points at nothing.

| Command | Creates |
|---|---|
| `ln target linkname` | A hard link named `linkname` pointing to the same data as `target` |
| `ln -s target linkname` | A symbolic link named `linkname` pointing at the path `target` |

💡 **Analogy:** A hard link is like two people having identical house keys to the same physical house — either key works, and the house doesn't care which key you used. A symbolic link is like a sticky note on your desk that just says "the house is at 123 Main St." — useful, but if the house is demolished, the note is now pointing at nothing.

🎯 **On the job:** Symlinks are extremely common — e.g. `/bin` being a symlink to `/usr/bin`, or a `current` symlink pointing at whichever deployed release folder is "live" right now, so you can flip a deployment by just repointing one symlink.

### Checking disk usage: `du` and `df`

| Command | What it tells you | Typical use |
|---|---|---|
| `du -sh <path>` | The total size of a specific file or directory ("disk usage"), human-readable | "How big is this folder?" |
| `df -h` | Free/used space on each mounted filesystem/disk ("disk free"), human-readable | "Is this server about to run out of disk space?" |

💡 **Tip:** `du` looks *inward* — it sums up what's inside a folder. `df` looks *outward* — it reports on the whole disk/partition. If a server alert says "disk almost full," you check `df -h` first to confirm it, then use `du -sh` on suspect folders (like `/var/log`) to find what's eating the space.

---

## Practical Examples

### Example 1 — Where am I, and what's around me?

```bash
pwd
```

Expected output:
```
/home/weki
```

```bash
ls -la
```

Expected output (abbreviated):
```
drwxr-xr-x  6 weki weki 4096 Jul 28 09:00 .
drwxr-xr-x  3 root root 4096 Jul 20 12:00 ..
-rw-r--r--  1 weki weki  220 Jul 20 12:00 .bashrc
drwxr-xr-x  2 weki weki 4096 Jul 28 09:00 projects
```

Line-by-line:
- `pwd` confirms we're standing in `/home/weki`.
- `ls -la` lists everything, including hidden dotfiles like `.bashrc`.
- Notice the first two entries are always `.` (this directory) and `..` (its parent) — `ls -a` shows these explicitly because they're real directory entries too.

💡 **Tip:** `.bashrc` is a hidden configuration file that runs every time you open a new Bash shell — you'll edit files like this in later modules.

### Example 2 — Moving around with `cd`

```bash
cd /etc
pwd
```

Expected output:
```
/etc
```

```bash
cd ..
pwd
```

Expected output:
```
/
```

```bash
cd ~
pwd
```

Expected output:
```
/home/weki
```

```bash
cd /var/log
cd -
```

Expected output:
```
/var/log
/home/weki
```

Line-by-line:
- `cd /etc` uses an **absolute path** to jump straight to `/etc`.
- `cd ..` moves up one level from `/etc` to its parent, which is `/`.
- `cd ~` returns to the home directory, no matter where we were.
- `cd /var/log` then `cd -` shows how `cd -` prints the directory it's jumping *to* and toggles you back — here it printed `/var/log` (confirming where we jumped from when toggling back) then returned us home.

### Example 3 — Reading `ls -l` output

```bash
ls -l /etc/hostname
```

Expected output:
```
-rw-r--r-- 1 root root 10 Jul 20 08:00 /etc/hostname
```

Line-by-line:
- `-` (first character) — this is a regular file, not a directory or link.
- `rw-r--r--` — owner can read/write, group can only read, everyone else can only read.
- `1` — one hard link to this file's data.
- `root root` — both owner and group are `root`.
- `10` — the file is 10 bytes.
- `Jul 20 08:00` — last modified timestamp.
- `/etc/hostname` — the path we asked about.

### Example 4 — Creating a project structure

```bash
mkdir -p ~/projects/demo/src
cd ~/projects/demo
touch README.md src/app.py
ls -R
```

Expected output:
```
.:
README.md  src

./src:
app.py
```

Line-by-line:
- `mkdir -p ~/projects/demo/src` creates `projects`, `demo`, and `src` all at once, even though none of them existed before — `-p` skips complaining about missing parents.
- `touch README.md src/app.py` creates two empty files in one command: one in the current folder, one inside `src/` using a relative path.
- `ls -R` (recursive listing, not covered elsewhere in this module but handy here) shows the structure we just built.

### Example 5 — Copying, moving, and renaming

```bash
cp README.md README.md.bak
cp -r src src_backup
mv README.md.bak old-readme.txt
ls
```

Expected output:
```
old-readme.txt  README.md  src  src_backup
```

Line-by-line:
- `cp README.md README.md.bak` makes a simple file copy.
- `cp -r src src_backup` copies the whole `src` directory (and anything inside it) — the `-r` is required for directories.
- `mv README.md.bak old-readme.txt` renames the backup file in place — same directory, new name.

✅ **Best Practice:** Before overwriting or deleting something important, make a quick `.bak` copy like this. It costs nothing and can save you from a bad day.

### Example 6 — Wildcards in action

```bash
touch report1.txt report2.txt report3.txt notes.md
ls *.txt
```

Expected output:
```
report1.txt  report2.txt  report3.txt
```

```bash
ls report?.txt
```

Expected output:
```
report1.txt  report2.txt  report3.txt
```

```bash
mv report[12].txt archive/
```

Line-by-line:
- `ls *.txt` matches every file ending in `.txt`, regardless of what comes before it.
- `ls report?.txt` matches exactly one character between "report" and ".txt" — it would *not* match `report10.txt` since `?` only stands for one character.
- `mv report[12].txt archive/` uses a character set to move only `report1.txt` and `report2.txt` (not `report3.txt`) into an `archive` folder — assuming `archive/` already exists.

### Example 7 — Symbolic links

```bash
ln -s ~/projects/demo current-project
ls -l current-project
```

Expected output:
```
lrwxrwxrwx 1 weki weki 24 Jul 28 11:00 current-project -> /home/weki/projects/demo
```

Line-by-line:
- `ln -s target linkname` creates a symlink named `current-project` pointing at `~/projects/demo`.
- Notice the permissions string starts with `l` for "link," and `ls -l` shows the arrow `->` pointing at the real target — that arrow only appears for symlinks.

### Example 8 — Checking disk usage

```bash
du -sh ~/projects
```

Expected output:
```
48K	/home/weki/projects
```

```bash
df -h /
```

Expected output:
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G   12G   26G  32% /
```

Line-by-line:
- `du -sh ~/projects` totals up everything inside `projects` and prints it in human-readable form (`48K` = 48 kilobytes).
- `df -h /` shows the root filesystem is 40G total, 12G used, 26G free, 32% full — the numbers you'd check first if a server warns about low disk space.

### Example 9 — ⚠️ The `rm -rf` danger zone

```bash
rm -r src_backup
```

This asks nothing and deletes `src_backup` and everything inside it — recursion means it walks the whole tree.

```bash
rm -rf archive
```

⚠️⚠️ **Warning:** This deletes `archive` and every file inside it **immediately, with zero confirmation, and zero undo.** Compare this to `rm` on its own, which at least errors out on directories and sometimes prompts before overwriting protected files. `-rf` removes every one of those safety nets at once.

✅ **Best Practice:** Before running any `rm -rf`, run `ls` on the *exact* same path first to see precisely what you're about to destroy. We'll turn this into a formal workflow in the cheat sheet.

---

## Common Pitfalls & Best Practices

- **Forgetting `-r` on `cp` or `rm` for directories.** Both commands refuse to touch a directory without it — that refusal is a safety feature, not a bug. Don't reach for `-f` to force past it without understanding why it stopped you.
- **Confusing `/root` with `/`.** `/root` is the root *user's* home folder; `/` is the top of the entire tree. They are not the same thing.
- **Typing `rm -rf` with a wrong or extra space in the path.** A stray space can turn `rm -rf ~/temp /oldfiles` (delete two things) into something that deletes far more than intended if a variable was empty. Always sanity-check the full command before hitting Enter.
- **Assuming hidden files are somehow protected.** "Hidden" just means the name starts with `.` — it has nothing to do with permissions or security. `ls -a` reveals them instantly.
- **Confusing symlinks with hard links.** If you delete the *original* file behind a symlink, the symlink breaks (points at nothing). A hard link, by contrast, keeps the data alive as long as at least one hard link to it still exists.
- **Relying on relative paths in scripts without checking the working directory.** A script that does `cd data && rm -rf *` behaves very differently depending on where it's run from. Prefer absolute paths in anything automated.
- **Not checking `ls` before `rm -rf`.** This is the single highest-value habit in this entire module — see the Safe Delete Workflow in the cheat sheet.

---

## Hands-on Exercise

**Task:**

1. Create a small nested project structure under your home directory: a folder called `sandbox` containing subfolders `docs` and `data`.
2. Inside `docs`, create three empty files: `intro.txt`, `chapter1.txt`, `chapter2.txt`.
3. Use a wildcard to list only the `chapter*.txt` files.
4. Copy `intro.txt` into `data` as a backup, then rename the original `intro.txt` to `introduction.txt`.
5. Create a symbolic link named `latest-docs` in your home directory that points at `sandbox/docs`.
6. Check the size of the whole `sandbox` folder with `du`.
7. Safely delete the entire `sandbox` structure using the Safe Delete Workflow (check with `ls` first, then `rm -r`).

Try this yourself before reading the solution.

### Solution

```bash
# 1. Build the structure
mkdir -p ~/sandbox/docs ~/sandbox/data
cd ~/sandbox

# 2. Create the files
touch docs/intro.txt docs/chapter1.txt docs/chapter2.txt

# 3. Wildcard listing
ls docs/chapter*.txt
```
Expected output:
```
docs/chapter1.txt  docs/chapter2.txt
```

```bash
# 4. Copy then rename
cp docs/intro.txt data/intro.txt.bak
mv docs/intro.txt docs/introduction.txt
ls docs
```
Expected output:
```
chapter1.txt  chapter2.txt  introduction.txt
```

```bash
# 5. Symlink
ln -s ~/sandbox/docs ~/latest-docs
ls -l ~/latest-docs
```
Expected output:
```
lrwxrwxrwx 1 weki weki 22 Jul 28 12:00 /home/weki/latest-docs -> /home/weki/sandbox/docs
```

```bash
# 6. Check size
du -sh ~/sandbox
```
Expected output:
```
20K	/home/weki/sandbox
```

```bash
# 7. Safe delete — check first, then delete
ls -la ~/sandbox
rm -r ~/sandbox
rm ~/latest-docs
```

Explanation: I ran `ls -la ~/sandbox` right before deleting to confirm exactly what I was about to remove and that the path was correct. I used `rm -r` (no `-f`) since I wanted to see any prompts along the way, not `rm -rf`, because nothing here required forcing past protections. Finally I removed the now-broken `latest-docs` symlink separately with plain `rm` — since it's just a link, not a directory, no `-r` needed.

✅ Exercise complete — you've now created, listed, copied, renamed, linked, measured, and safely deleted a real directory structure.

---

## ✅ Module Completion Checklist

- [ ] I can explain the Linux filesystem hierarchy (FHS) and what the major top-level directories (`/`, `/home`, `/etc`, `/var`, `/tmp`, `/usr`, `/bin`, `/root`) are used for
- [ ] I can tell the difference between absolute and relative paths, and use `.`, `..`, and `~` confidently
- [ ] I can navigate the filesystem with `pwd` and `cd`, including `cd -`, `cd ~`, and `cd ..`
- [ ] I can list directory contents with `ls`, `ls -l`, `ls -la`, and `ls -lh`, and read every column of `ls -l` output
- [ ] I can create directories and empty files with `mkdir`, `mkdir -p`, and `touch`
- [ ] I can copy, move, and rename files and directories with `cp`, `cp -r`, and `mv`
- [ ] I can delete files and directories safely with `rm`, `rm -r`, and `rm -rf` — and explain why `rm -rf` is dangerous
- [ ] I can use wildcards (`*`, `?`, `[...]`) to operate on multiple files at once, and describe the difference between a symbolic link and a hard link
- [ ] I can check disk usage with `du` and `df` at a basic level

## Next Step

Continue to [Module 3: Viewing & Finding Files](../module03-viewing-finding-files/)
