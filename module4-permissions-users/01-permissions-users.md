# Module 4: Permissions, Users & Ownership 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-3

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain the owner/group/others permission model, and describe what read, write, and execute actually mean for a **file** versus what they mean for a **directory**
- [ ] Read every character of an `ls -l` permission string like `-rwxr-xr--` without hesitating
- [ ] Change permissions with `chmod` using both **symbolic mode** (`u+x`, `g-w`, `o=r`, `a+x`) and **octal/numeric mode** (`755`, `644`), and do the r=4/w=2/x=1 math in your head
- [ ] Change ownership with `chown` and `chgrp`, including changing the user and group in a single command
- [ ] Explain what `umask` is and predict the default permissions of a newly created file or directory on your system
- [ ] Recognize **setuid**, **setgid**, and the **sticky bit** in `ls -l` output, and explain what each one does and why it matters for security
- [ ] Read and explain the structure of `/etc/passwd`, `/etc/group`, and `/etc/shadow`, and why passwords moved out of `/etc/passwd` into `/etc/shadow`
- [ ] Create and manage users and groups with `useradd`/`adduser`, `usermod`, `groupadd`, and `passwd`, and check your own identity with `whoami`, `id`, and `groups`
- [ ] Explain the difference between `su` and `sudo`, why `sudo` is preferred on modern Ubuntu/Debian, and why the root account is disabled by default

---

## Module Goal

By the end of this module, you'll be able to look at any file or directory, know instantly who can do what to it, and confidently fix or lock down access without guessing.

🎯 **On the job:** Nearly every "it worked on my machine" bug and every real security incident on Linux traces back to permissions and ownership. A deploy script that runs fine for you but throws `Permission denied` in CI. A web server that returns a 500 error because it can't read its own config file. A shared team server where anyone can read (or worse, edit) anyone else's files. A new hire who needs an account with just enough access to do their job — no more, no less. This module is what turns "I don't know why this doesn't work" into "I checked `ls -l`, the owner is wrong, here's the fix" in about ten seconds.

---

## Core Concepts

### 1. What is a permission?

A **permission** is a rule attached to a file or directory that controls who is allowed to do what with it. Every single file and directory on a Linux system carries this information at all times — there's no such thing as a file with "no permissions set."

### 2. The three "who" categories: owner, group, others

Linux splits *who* can access a file into exactly three categories:

- **Owner** (also called "user" in some docs) — the one specific user account that owns the file. Usually whoever created it.
- **Group** — one specific group of users assigned to the file. Anyone who belongs to that group gets the group's level of access.
- **Others** — everyone else on the system who isn't the owner and isn't in the owning group.

💡 **Analogy — the office keycard system:** Think of a file like a locked room in an office building. **You** (the owner) have your own personal keycard with full access. **Your team** (the group) shares a team keycard that opens the room to a certain level — maybe they can look around but not rearrange the furniture. **Everyone else** in the building (others) only gets whatever access the building manager decided to leave for random visitors — often nothing at all. Three different keycards, three different access levels, one door.

### 3. The three "what" actions: read, write, execute

For each of those three "who" categories, Linux tracks three possible permissions:

- **r (read)** — allowed to view the contents.
- **w (write)** — allowed to modify or delete the contents.
- **x (execute)** — allowed to run it as a program/script, **or** (for directories) allowed to enter/traverse it.

That's the part that trips almost everyone up at first, so let's slow down.

### 4. Read/write/execute on a FILE

For a regular file, the three permissions mean exactly what you'd guess:

| Permission | Meaning on a file |
|---|---|
| `r` | Open and read the file's contents (`cat`, `less`, a text editor) |
| `w` | Modify the file's contents, or delete/overwrite it |
| `x` | Run the file as a program or script (e.g. `./deploy.sh`) |

⚠️ **Note:** `w` on a file lets you *edit* it. Whether you can *delete* it is actually governed by the **directory** it lives in, not the file itself — more on that below, and again in Module 8 when we cover file ownership edge cases.

### 5. Read/write/execute on a DIRECTORY — the part that trips people up

A directory isn't a document you "read" the way you read a text file — it's a list of filenames. So the three permissions mean something different:

| Permission | Meaning on a directory |
|---|---|
| `r` | List the names of files inside it (`ls` works) |
| `w` | Create, delete, or rename files/directories inside it |
| `x` | **Enter/traverse** the directory (`cd` into it, or access a file inside it by path) — sometimes called the "search" bit |

🎯 **On the job — the single most common permissions surprise:** People assume `x` on a directory means "execute the directory," which makes no sense, so they ignore it. In reality, **without `x` on a directory, you cannot `cd` into it, and you cannot open any file inside it even if that file itself is fully readable** — not through `cat`, not through a script, not through anything, because the shell can't even *traverse* into the folder to find the file. This is exactly why a deploy script sometimes fails with `Permission denied` on a file that looks perfectly readable with `644` — the problem isn't the file, it's a missing `x` on one of the parent directories in the path.

💡 **Tip:** A directory needs `r` to *list* what's inside it and `x` to *enter/use* it. You'll very often see directories set to `rwx` for the owner and `r-x` for group/others — readable and enterable, but not writable, so other people can look around and use files but not create or delete anything.

### 6. Users and groups — the bigger picture

Every process and every file on Linux is tied to a **user account**. Every user belongs to at least one group (their **primary group**) and optionally more (**secondary/supplementary groups**). Permissions only make sense once you know which user and which group a file belongs to — that's what `chown` and `chgrp` control, and what we'll dig into after `chmod`.

### 7. The superuser: `root`

Linux has one special account, **`root`** (also called the **superuser**), that bypasses almost all permission checks — it can read, write, or execute anything, anywhere. This is powerful and dangerous in equal measure, which is why modern Ubuntu doesn't let you log in as root directly by default — you'll see exactly how that works near the end of this module.

---

## Detailed Explanations

### Reading the `ls -l` permission string in full depth

You got a first taste of this in Module 2. Now let's read every character. Here's a real example:

```
-rwxr-xr-- 1 weki devteam 8192 Jul 28 09:00 deploy.sh
```

Break the first 10 characters into this shape:

```
-  rwx  r-x  r--
1   2    3    4
```

| Position | Characters | Meaning |
|---|---|---|
| 1 | `-` | **File type**: `-` = regular file, `d` = directory, `l` = symbolic link (others exist, like `c`/`b` for device files, but these three cover almost everything you'll see) |
| 2 | `rwx` | **Owner's** permissions: read, write, execute — all three granted |
| 3 | `r-x` | **Group's** permissions: read and execute granted, write denied (the dash means "not granted") |
| 4 | `r--` | **Others'** permissions: read only, no write, no execute |

So `-rwxr-xr--` reads as: "This is a regular file. The owner can read, write, and execute it. Anyone in the owning group can read and execute it, but not modify it. Everyone else can only read it."

💡 **Tip:** A `-` in any position always means "this permission is absent," never "unknown" or "inherited." Every position is always explicitly on or off.

### `chmod` — symbolic mode

`chmod` ("change mode") changes a file or directory's permissions. **Symbolic mode** lets you add, remove, or set specific bits without touching the others, using this pattern:

```
chmod  [who][operator][permission]  file
```

| Who | Meaning |
|---|---|
| `u` | User (the owner) |
| `g` | Group |
| `o` | Others |
| `a` | All (owner + group + others) |

| Operator | Meaning |
|---|---|
| `+` | Add this permission, leave everything else as-is |
| `-` | Remove this permission, leave everything else as-is |
| `=` | Set *exactly* this permission, clearing anything else in that "who" category |

| Example | Effect |
|---|---|
| `chmod u+x script.sh` | Add execute for the owner only |
| `chmod g-w file.txt` | Remove write from the group only |
| `chmod o=r file.txt` | Set others to read-only, exactly — removes write/execute for others if they were set |
| `chmod a+x script.sh` | Add execute for everyone (owner, group, others) |
| `chmod u+x,g+x script.sh` | Combine multiple changes in one command with a comma, no space |

✅ **Best Practice:** Reach for symbolic mode when you want to change *one specific bit* without having to know or recompute the entire permission string — e.g. "just make this executable" is `chmod u+x file`, full stop.

### `chmod` — octal (numeric) mode

**Octal mode** sets the *entire* permission string at once using numbers. This is the mode you'll see constantly in real production configs (`644`, `755`, `600`...), so the math is worth memorizing cold.

Each of the three permissions has a fixed point value:

| Permission | Value |
|---|---|
| `r` (read) | 4 |
| `w` (write) | 2 |
| `x` (execute) | 1 |
| (none) | 0 |

You **add** the values together for each "who" category to get one digit, and you write three digits in order: owner, group, others.

| Combination | Sum | Meaning |
|---|---|---|
| `rwx` | 4+2+1 = **7** | read + write + execute |
| `rw-` | 4+2+0 = **6** | read + write |
| `r-x` | 4+0+1 = **5** | read + execute |
| `r--` | 4+0+0 = **4** | read only |
| `-wx` | 0+2+1 = **3** | write + execute |
| `-w-` | 0+2+0 = **2** | write only |
| `--x` | 0+0+1 = **1** | execute only |
| `---` | 0+0+0 = **0** | nothing |

So a permission string is just three of these digits stacked, one per "who": **owner, group, others.**

| Octal | Symbolic | Common use |
|---|---|---|
| `755` | `rwxr-xr-x` | Scripts and directories — owner full control, everyone else can read/run/enter |
| `644` | `rw-r--r--` | Regular files/config — owner can edit, everyone else can only read |
| `700` | `rwx------` | Private script/directory — owner only, nobody else gets anything |
| `600` | `rw-------` | Private file (e.g. SSH private keys) — owner can read/write, nobody else can touch it |
| `777` | `rwxrwxrwx` | Everyone can do everything — almost always a mistake (see Pitfalls) |
| `664` | `rw-rw-r--` | Owner and group can edit, others can only read — common for shared team files |

💡 **Tip — the memorization trick:** Read each digit left to right as "owner, group, others," and read each digit's math as "read is worth 4, write is worth 2, execute is worth 1, add what's granted." `7` always means "everything" because 4+2+1=7 is the maximum for one category.

### `chown` and `chgrp` — changing ownership

`chown` ("change owner") changes which user owns a file. `chgrp` ("change group") changes which group owns it. You need appropriate privileges (usually `sudo`) to give a file to a *different* user than yourself.

| Command | Effect |
|---|---|
| `chown newuser file` | Change the owner to `newuser`, leave the group alone |
| `chgrp newgroup file` | Change the group to `newgroup`, leave the owner alone |
| `chown newuser:newgroup file` | Change **both** owner and group in a single command |
| `chown :newgroup file` | Change only the group, using `chown`'s combined syntax (equivalent to `chgrp`) |
| `chown -R user:group dir/` | Recursively change ownership of a directory and everything inside it |

🎯 **On the job:** `chown -R www-data:www-data /var/www/mysite` is one of the most common commands you'll run when deploying a web app — it hands the site's files over to the user account the web server actually runs as, so the server process can read (and sometimes write) them.

### `umask` — the default permission filter

**`umask`** is a value that gets subtracted from a maximum "default" permission every time a new file or directory is created, to decide its starting permissions. It doesn't change existing files — only what happens at creation time.

The starting maximums are:
- **Files** start from a theoretical max of `666` (`rw-rw-rw-`) — note files never default to executable, even with a permissive umask; you always add `x` yourself afterward if needed.
- **Directories** start from a theoretical max of `777` (`rwxrwxrwx`).

The umask then **removes** permission bits from that maximum. On most Ubuntu systems, the default umask for a regular user is `022`.

| Category | Max for files | Minus umask `022` | Result |
|---|---|---|---|
| Owner | 6 | -0 | 6 (`rw-`) |
| Group | 6 | -2 | 4 (`r--`) |
| Others | 6 | -2 | 4 (`r--`) |

Result: new files default to `644` (`rw-r--r--`). For new directories, the same umask against a max of `777` gives `755` (`rwxr-xr-x`).

💡 **Tip:** Think of `umask` as a stencil with holes cut out — the holes are what gets *blocked*, not what gets *granted*. A umask of `002` only blocks write for others, giving new files `664` and new directories `775` — common on systems where a shared group needs to collaborate on files.

⚠️ **Warning:** `umask` only affects files and directories **created after** you set it, for the remainder of that shell session (or permanently if set in a startup file like `~/.bashrc`). It never retroactively changes anything that already exists.

### Special permissions: setuid, setgid, and the sticky bit

Beyond the normal `rwx` bits, Linux has three special permission flags. You don't need to master every use case yet — just recognize them and know what they're for.

**Setuid (`s` in the owner's execute slot):** When set on an *executable file*, the program runs with the permissions of the file's **owner**, not the person running it. Classic example: `/usr/bin/passwd` is setuid root, so any regular user can run it to change their own password — a task that requires briefly writing to the normally root-only `/etc/shadow` file — without you ever needing root access yourself.

**Setgid (`s` in the group's execute slot):** On an executable, similar idea but with the file's **group**. On a *directory*, it means something extra useful: any new file created inside inherits the **directory's group**, instead of the creating user's own primary group — commonly used for shared team folders so every file automatically belongs to the team, regardless of who created it.

**Sticky bit (`t` in the others' execute slot):** On a directory, it means users can only delete or rename files **they themselves own**, even if the directory's permissions would otherwise let anyone write to it. The classic example is `/tmp` — every user needs to be able to create files there (`rwxrwxrwt`), but without the sticky bit, anyone could delete anyone else's temp files. The sticky bit closes that hole.

| Symbol in `ls -l` | Special bit | Where you see it |
|---|---|---|
| `s` in owner's `x` slot (e.g. `rws`) | setuid | `/usr/bin/passwd` |
| `s` in group's `x` slot (e.g. `rwxrws`) | setgid | shared team directories |
| `t` in others' `x` slot (e.g. `rwxrwxrwt`) | sticky bit | `/tmp` |
| Uppercase `S` or `T` | Same special bit, but the underlying execute permission is **missing** (unusual/likely a mistake) | rare |

In octal, these are a **fourth leading digit** in front of the usual three: setuid = 4000, setgid = 2000, sticky = 1000. So `chmod 4755 file` sets setuid plus `rwxr-xr-x`, and `chmod 1777 dir` is exactly what `/tmp` uses.

⚠️ **Security note:** Setuid/setgid programs are a well-known attack surface — if a setuid-root program has a bug, an attacker can potentially exploit it to gain root-level access. Never set setuid on a script or program yourself unless you fully understand why it needs it.

### Users and groups: `/etc/passwd`, `/etc/group`, `/etc/shadow`

Three plain-text files (readable with `cat` or `less`, which you already know from Module 3) define every user and group on the system.

**`/etc/passwd`** — one line per user account, colon-separated fields:

```
weki:x:1000:1000:Weki:/home/weki:/bin/bash
```

| Field # | Example | Meaning |
|---|---|---|
| 1 | `weki` | Username |
| 2 | `x` | Password placeholder — the real (hashed) password lives in `/etc/shadow`, never here |
| 3 | `1000` | UID (user ID number) — `0` is always root; regular users on Ubuntu usually start at `1000` |
| 4 | `1000` | GID (primary group ID) |
| 5 | `Weki` | GECOS field — a comment, usually the full name |
| 6 | `/home/weki` | Home directory |
| 7 | `/bin/bash` | Login shell — the program that runs when this user logs in |

**`/etc/group`** — one line per group:

```
devteam:x:1002:weki,alex
```

| Field # | Example | Meaning |
|---|---|---|
| 1 | `devteam` | Group name |
| 2 | `x` | Password placeholder (group passwords are rarely used) |
| 3 | `1002` | GID |
| 4 | `weki,alex` | Comma-separated list of members whose *primary* group is something else but who belong to this group as a supplementary group |

**`/etc/shadow`** — why it exists: `/etc/passwd` must be readable by *everyone* (all sorts of tools need to look up usernames), but you never want everyone able to read password hashes and try to crack them offline. So the actual hashed passwords, and password-aging rules, were split out into `/etc/shadow`, which is readable **only by root** (`chmod 640` or `600` with root ownership). A typical line looks like:

```
weki:$6$randomsalt$longhashvalue:19500:0:99999:7:::
```

Key fields: username, the hashed password (or `!`/`*` if the account is locked/has no password login), the date of last password change, minimum/maximum days between changes, a warning period, and account expiration info.

🎯 **On the job:** If you ever `cat /etc/shadow` and get `Permission denied`, that's correct behavior, not a bug — it's one of the most deliberately locked-down files on the whole system.

### Checking identity: `whoami`, `id`, `groups`

| Command | Shows |
|---|---|
| `whoami` | Just your current username |
| `id` | Your UID, primary GID, and every group you belong to, all at once |
| `groups` | Just the list of group names you belong to |

### Managing users and groups

| Command | Effect |
|---|---|
| `sudo adduser newuser` | (Debian/Ubuntu-friendly, interactive) Creates a new user, home directory, and prompts for a password and details |
| `sudo useradd -m newuser` | (Lower-level, more manual) Creates a new user; `-m` creates their home directory too — without it, no home directory is made |
| `sudo usermod -aG groupname username` | Adds `username` to `groupname` as a **supplementary** group — always use `-aG` together (append to group), never just `-G` alone (which *replaces* all supplementary groups!) |
| `sudo groupadd teamname` | Creates a new group |
| `sudo passwd username` | Sets or changes a user's password |
| `sudo deluser username` / `sudo userdel username` | Removes a user account |

⚠️ **Warning:** `usermod -G groupname username` (no `-a`) **replaces** every supplementary group that user was already in with just this one. Forgetting the `-a` is a classic way to accidentally strip someone's existing group memberships. Always use `-aG`.

💡 **Tip:** On Ubuntu/Debian, `adduser` is the friendlier, interactive front-end script; `useradd` is the lower-level, more "scriptable" command with fewer built-in conveniences. Both exist; `adduser` is generally recommended for manual/interactive use.

### `su` vs `sudo`

- **`su`** ("switch user") starts a new shell logged in as another user — typically root — and asks for **that target user's password**. Once inside, you stay "as" that user until you exit.
- **`sudo`** ("superuser do") runs a **single command** with root (or another user's) privileges, and asks for **your own password** (not root's), assuming your account is authorized to use `sudo`.

| | `su` | `sudo` |
|---|---|---|
| Password required | The target account's password | Your own account's password |
| Scope | Opens a whole new shell session as that user | Runs one command, then you're back to yourself |
| Audit trail | Weaker — less detail on what was done as root | Stronger — logs exactly who ran what command and when |
| Ubuntu default | Root password isn't even set by default, so plain `su` to root doesn't work out of the box | Works immediately for the first user created during install, who's added to the `sudo` group |

✅ **Best Practice:** On modern Ubuntu/Debian, `sudo` is the standard, recommended approach. The root account is **disabled by default** — it has no usable password — precisely to force administrative actions through `sudo`, which keeps a clear, per-command audit log of exactly who did what.

Which users are allowed to use `sudo` is controlled by group membership: on **Debian/Ubuntu**, that's the **`sudo`** group; on **RHEL/Fedora/CentOS**, the equivalent is traditionally the **`wheel`** group. Run `groups` on yourself to check.

The detailed rules for who can run what via `sudo` live in `/etc/sudoers`, and its accompanying `/etc/sudoers.d/` directory. You should never edit `/etc/sudoers` directly with a normal text editor — always use `sudo visudo`, which locks the file and validates the syntax before saving, preventing a typo from locking every admin out of `sudo` entirely. This module won't go deep into sudoers syntax; just know the command's name and why it exists.

---

## Practical Examples

### Example 1 — Reading a permission string end to end

```bash
ls -l /usr/bin/passwd
```

Expected output:
```
-rwsr-xr-x 1 root root 68208 Mar  1  2024 /usr/bin/passwd
```

Line-by-line:
- `-` — a regular file.
- `rws` — owner (root) can read, write, and execute; the `s` replacing the usual `x` means **setuid is set** — the program runs *as root* no matter who launches it.
- `r-x` — group can read and execute, not write.
- `r-x` — others can read and execute, not write.
- This is exactly the setuid example from Core Concepts: it's how an ordinary user can run `passwd` to change their own password, even though updating `/etc/shadow` normally requires root.

### Example 2 — `chmod` in symbolic mode

```bash
touch deploy.sh
ls -l deploy.sh
```

Expected output:
```
-rw-r--r-- 1 weki weki 0 Jul 28 10:00 deploy.sh
```

```bash
chmod u+x deploy.sh
ls -l deploy.sh
```

Expected output:
```
-rwxr--r-- 1 weki weki 0 Jul 28 10:00 deploy.sh
```

Line-by-line:
- A brand-new file created with `touch` gets `rw-r--r--` by default (that's `umask` at work — more on this below).
- `chmod u+x` adds execute for the **owner only** — group and others are untouched, still `r--`.
- 💡 **Tip:** This is the fix for the single most common beginner error: `bash: ./deploy.sh: Permission denied` when trying to run a script. Nine times out of ten, the fix is `chmod u+x scriptname`.

### Example 3 — `chmod` in octal mode, with the math shown

Let's set `deploy.sh` so the owner has full control, the group can read and run it, and others get nothing at all.

Desired permissions: owner `rwx`, group `r-x`, others `---`.

```
Owner:  r(4) + w(2) + x(1) = 7
Group:  r(4) + w(0) + x(1) = 5
Others: r(0) + w(0) + x(0) = 0
```

```bash
chmod 750 deploy.sh
ls -l deploy.sh
```

Expected output:
```
-rwxr-x--- 1 weki weki 0 Jul 28 10:00 deploy.sh
```

Line-by-line:
- `7` → owner gets `rwx` (4+2+1).
- `5` → group gets `r-x` (4+0+1).
- `0` → others get `---` (nothing).
- ✅ **Best Practice:** This exact pattern (`750`) is a great default for a private team script: the owner has full control, teammates can run it, and random other accounts on the box get nothing.

### Example 4 — `chown` and `chgrp`, including both at once

```bash
sudo chown www-data /var/www/mysite/index.html
ls -l /var/www/mysite/index.html
```

Expected output:
```
-rw-r--r-- 1 www-data weki 512 Jul 28 10:05 /var/www/mysite/index.html
```

```bash
sudo chown www-data:www-data /var/www/mysite/index.html
ls -l /var/www/mysite/index.html
```

Expected output:
```
-rw-r--r-- 1 www-data www-data 512 Jul 28 10:05 /var/www/mysite/index.html
```

Line-by-line:
- The first command changes only the **owner** to `www-data` (a common account web servers run as); notice the group is still `weki`.
- The second command uses the combined `user:group` syntax to set **both** the owner and group to `www-data` in one shot — this is the standard move right after deploying a website so the web server process actually owns its own files.

### Example 5 — `umask` in action

```bash
umask
```

Expected output:
```
0022
```

```bash
touch newfile.txt
mkdir newdir
ls -l newfile.txt
ls -ld newdir
```

Expected output:
```
-rw-r--r-- 1 weki weki    0 Jul 28 10:10 newfile.txt
drwxr-xr-x 2 weki weki 4096 Jul 28 10:10 newdir
```

Line-by-line:
- `umask` alone prints the current mask — `022` here (the leading `0` is just the special-bits placeholder, always `0` for a normal umask).
- The new **file** landed on `644`: max `666` minus `022` = `644`.
- The new **directory** landed on `755`: max `777` minus `022` = `755`.
- 💡 **Tip:** `ls -ld newdir` (note the `-d`) lists the directory entry *itself*, not its contents — without `-d` you'd see what's inside `newdir`, which is nothing yet.

### Example 6 — The sticky bit on `/tmp`

```bash
ls -ld /tmp
```

Expected output:
```
drwxrwxrwt 12 root root 4096 Jul 28 09:00 /tmp
```

Line-by-line:
- `rwxrwxrwx` would normally mean literally anyone can create *and delete* anything in here.
- The trailing `t` (instead of `x`) in the others' slot is the **sticky bit**: everyone can still create files, but each user can only delete or rename **their own** files — not anyone else's — even though the directory is world-writable. That's exactly why `/tmp` is safe to share across every user and process on the system.

### Example 7 — Setting the sticky bit and setgid yourself

```bash
mkdir teamshare
chmod 2775 teamshare
ls -ld teamshare
```

Expected output:
```
drwxrwsr-x 2 weki devteam 4096 Jul 28 10:20 teamshare
```

Line-by-line:
- The leading `2` in `2775` is the **setgid** bit; `775` is the normal owner/group/others permissions (`rwx`, `rwx`, `r-x`).
- Notice the `s` in the group's execute slot in the output (`rws` for group) — any file *created inside* `teamshare` from now on will automatically belong to the `devteam` group, regardless of which user creates it. That keeps a shared folder consistently owned by the team, without everyone remembering to `chgrp` manually every time.

### Example 8 — Users and groups: checking your own identity

```bash
whoami
```
Expected output:
```
weki
```

```bash
id
```
Expected output:
```
uid=1000(weki) gid=1000(weki) groups=1000(weki),27(sudo),1002(devteam)
```

```bash
groups
```
Expected output:
```
weki sudo devteam
```

Line-by-line:
- `whoami` just confirms the username.
- `id` shows the full picture: UID `1000`, primary group `weki` (GID `1000`), and every group membership — here including `sudo` (so this user can run `sudo`) and `devteam` (a custom team group).
- `groups` is a shorter version showing just the group names.

### Example 9 — Creating a user and a group

```bash
sudo groupadd devteam
sudo adduser --ingroup devteam alex
```

Expected output (abbreviated, `adduser` is interactive):
```
Adding user `alex' ...
Adding new group `alex' (1003) ...
Adding new user `alex' (1004) with group `devteam' ...
Creating home directory `/home/alex' ...
...
Enter new UNIX password:
```

```bash
sudo usermod -aG sudo alex
groups alex
```

Expected output:
```
alex : devteam sudo
```

Line-by-line:
- `groupadd devteam` creates the shared team group first.
- `adduser --ingroup devteam alex` creates a new user `alex` with `devteam` as their **primary** group, plus a home directory — `adduser` walks you through setting a password interactively.
- `usermod -aG sudo alex` **adds** (`-a` = append) `alex` to the `sudo` group as a supplementary group, without removing them from `devteam`. Skipping `-a` here would have *replaced* their group list with just `sudo`.

### Example 10 — ⚠️ Why `chmod 777` is dangerous

```bash
chmod 777 config.php
ls -l config.php
```

Expected output:
```
-rwxrwxrwx 1 weki weki 2048 Jul 28 10:30 config.php
```

⚠️⚠️ **Warning:** This grants **every single account on the system** — and every process running as any other account — full read, write, *and execute* access to this file. On a shared or internet-facing server, this is one of the most common real-world ways sites get compromised: an attacker who gets even a low-privileged foothold can now edit application config files, inject code, or overwrite anything protected only by `777`. It's almost never the actual fix to a permissions problem — it's a way of making the *symptom* (an error message) disappear while creating a much bigger problem.

✅ **Best Practice:** When you hit `Permission denied`, resist the urge to reach for `777`. Instead: check `ls -l` to see the actual owner/group/permissions, figure out *which* category needs *which* specific bit, and grant exactly that — usually `644` for data files or `755` for scripts/directories is all you need.

---

## Common Pitfalls & Best Practices

- **Reaching for `chmod 777` to "make an error go away."** It suppresses the symptom by granting universal access — including to attackers — instead of fixing the actual permission that was missing. Diagnose first, grant the minimum specific bit second.
- **Forgetting execute (`x`) on a parent directory.** A file can be `644` and perfectly readable, but if any directory in its path is missing `x` for your user/group, you still get `Permission denied` — the shell can't traverse in to reach it. Check every directory in the path, not just the file.
- **Using `usermod -G` instead of `usermod -aG`.** Without `-a`, you silently wipe out every supplementary group the user was already in and replace it with only the group(s) you just listed. Always pair `-a` with `-G`.
- **Overusing `sudo` for things that don't need it.** If you find yourself typing `sudo` in front of nearly every command out of habit, stop and ask why a normal file in your own home directory needs root privileges to touch — it usually doesn't, and it's a sign ownership somewhere is wrong.
- **Editing `/etc/passwd`, `/etc/group`, or `/etc/shadow` directly with a text editor.** Use the dedicated tools (`usermod`, `groupmod`, `passwd`, `vipw`/`vigr` if you must edit by hand) — they validate the file format and lock it against concurrent edits. A malformed `/etc/passwd` can break logins for every user on the box.
- **Editing `/etc/sudoers` with a plain text editor instead of `visudo`.** A syntax error in `sudoers` can lock every admin out of `sudo` at once. `visudo` validates the file before it's saved, specifically to prevent this.
- **Assuming a setuid/setgid program is automatically dangerous, or automatically safe.** They're a legitimate, necessary mechanism (`passwd` couldn't work without setuid) — the risk is specifically in *poorly written* setuid programs. Don't add setuid to your own scripts casually; only do it when you understand exactly why it's needed.
- **Forgetting that `umask` only affects new files/directories.** Changing your `umask` does nothing to files that already exist — you'd need a separate `chmod` pass for those.

---

## Hands-on Exercise

**Task:**

1. Create a directory called `project-vault` in your home directory.
2. Inside it, create a file called `secrets.txt`.
3. Using **symbolic mode**, make `secrets.txt` readable and writable by the owner only (no access for group or others).
4. Using **octal mode**, set `project-vault` itself to `750` (owner full access, group can read/enter, others nothing).
5. Create a new group called `auditors`.
6. Create a new user called `reviewer` whose primary group is `auditors`.
7. Add `reviewer` to the `auditors` group as a supplementary group too (yes, redundant here — just for practice with `-aG`), and confirm with `groups reviewer`.
8. Verify all your work with `id`, `ls -l`, and `ls -ld`.

Try this yourself before reading the solution.

### Solution

```bash
# 1. Create the vault directory
mkdir ~/project-vault
cd ~/project-vault

# 2. Create the file
touch secrets.txt
ls -l secrets.txt
```
Expected output:
```
-rw-r--r-- 1 weki weki 0 Jul 28 11:00 secrets.txt
```

```bash
# 3. Symbolic mode: owner rw only, remove everything from group/others
chmod u=rw,g=,o= secrets.txt
ls -l secrets.txt
```
Expected output:
```
-rw------- 1 weki weki 0 Jul 28 11:00 secrets.txt
```

```bash
# 4. Octal mode on the directory itself
cd ~
chmod 750 project-vault
ls -ld project-vault
```
Expected output:
```
drwxr-x--- 2 weki weki 4096 Jul 28 11:00 project-vault
```

```bash
# 5. Create the group
sudo groupadd auditors

# 6. Create the user with that primary group
sudo adduser --ingroup auditors reviewer
```
Expected output (abbreviated):
```
Adding user `reviewer' ...
Adding new user `reviewer' (1005) with group `auditors' ...
Creating home directory `/home/reviewer' ...
```

```bash
# 7. Add as a supplementary group too, then confirm
sudo usermod -aG auditors reviewer
groups reviewer
```
Expected output:
```
reviewer : auditors
```

```bash
# 8. Verify everything
id reviewer
ls -l ~/project-vault/secrets.txt
ls -ld ~/project-vault
```
Expected output:
```
uid=1005(reviewer) gid=1006(auditors) groups=1006(auditors)
-rw------- 1 weki weki 0 Jul 28 11:00 secrets.txt
drwxr-x--- 2 weki weki 4096 Jul 28 11:00 project-vault
```

Explanation: `secrets.txt` ended up at `600` (owner-only, no group/others access at all) using symbolic `=` to set each category exactly, clearing anything that might have been there before. `project-vault` is `750` — I can go in and do anything, my group can list and enter it (but not create/delete anything, since group only has `r-x`), and others get nothing. The `reviewer` user was created with `auditors` as its primary group in one step via `adduser --ingroup`, then explicitly appended to the same group with `-aG` for practice — `id`/`groups` confirms membership either way.

✅ Exercise complete — you've now set permissions both symbolically and numerically, and created a real user/group pair from scratch.

---

## ✅ Module Completion Checklist

- [ ] I can explain the owner/group/others permission model, and describe what read, write, and execute actually mean for a file versus what they mean for a directory
- [ ] I can read every character of an `ls -l` permission string like `-rwxr-xr--` without hesitating
- [ ] I can change permissions with `chmod` using both symbolic mode (`u+x`, `g-w`, `o=r`, `a+x`) and octal/numeric mode (`755`, `644`), and do the r=4/w=2/x=1 math in my head
- [ ] I can change ownership with `chown` and `chgrp`, including changing the user and group in a single command
- [ ] I can explain what `umask` is and predict the default permissions of a newly created file or directory on my system
- [ ] I can recognize setuid, setgid, and the sticky bit in `ls -l` output, and explain what each one does and why it matters for security
- [ ] I can read and explain the structure of `/etc/passwd`, `/etc/group`, and `/etc/shadow`, and why passwords moved out of `/etc/passwd` into `/etc/shadow`
- [ ] I can create and manage users and groups with `useradd`/`adduser`, `usermod`, `groupadd`, and `passwd`, and check my own identity with `whoami`, `id`, and `groups`
- [ ] I can explain the difference between `su` and `sudo`, why `sudo` is preferred on modern Ubuntu/Debian, and why the root account is disabled by default

## Next Step

Continue to [Module 5: I/O Redirection, Pipes & Filters](../module5-io-redirection-pipes/)
