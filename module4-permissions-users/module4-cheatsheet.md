# 📋 Module 4 Cheat Sheet — Permissions, Users & Ownership

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Owner (user)** | The single account that owns a file — usually its creator |
| **Group** | A set of users sharing one access level on a file |
| **Others** | Everyone who is neither the owner nor in the owning group |
| **Permission bit** | One of read (`r`), write (`w`), execute (`x`) for one "who" category |
| **`umask`** | A mask subtracted from max permissions (`666` files / `777` dirs) at creation time |
| **setuid** | Executable runs with the *owner's* privileges, regardless of who runs it |
| **setgid** | Executable runs with the *group's* privileges; on a directory, new files inherit its group |
| **Sticky bit** | On a directory, only the file's own owner can delete/rename it, even if writable by all |
| **UID / GID** | Numeric user ID / group ID — the real identity behind a username/group name |
| **root / superuser** | The account (UID 0) that bypasses nearly all permission checks |

## `rwx` Meaning: File vs. Directory

| Bit | On a **file** | On a **directory** |
|---|---|---|
| `r` | Read contents | List filenames inside (`ls`) |
| `w` | Modify/overwrite contents | Create/delete/rename entries inside |
| `x` | Run as a program/script | **Enter/traverse it** (`cd`, or reach a file by path) — missing `x` blocks access even to a fully-readable file inside |

## Reading an `ls -l` String

```
-rwxr-xr-- 1 weki devteam 8192 Jul 28 09:00 deploy.sh
```

| Chars | Meaning |
|---|---|
| `-` | Type: `-` file, `d` directory, `l` symlink |
| `rwx` | Owner's permissions |
| `r-x` | Group's permissions |
| `r--` | Others' permissions |

## Octal Permission Chart — All 8 Combinations

| Value | Symbolic | r? | w? | x? |
|---|---|---|---|---|
| 0 | `---` | no | no | no |
| 1 | `--x` | no | no | yes |
| 2 | `-w-` | no | yes | no |
| 3 | `-wx` | no | yes | yes |
| 4 | `r--` | yes | no | no |
| 5 | `r-x` | yes | no | yes |
| 6 | `rw-` | yes | yes | no |
| 7 | `rwx` | yes | yes | yes |

**Math:** r=4, w=2, x=1 — add what's granted, one digit per owner/group/others.

## Common Octal Modes

| Octal | Symbolic | Typical use |
|---|---|---|
| `755` | `rwxr-xr-x` | Scripts, executables, directories others need to enter/run |
| `644` | `rw-r--r--` | Regular data/config files — owner edits, everyone reads |
| `700` | `rwx------` | Fully private script/directory |
| `600` | `rw-------` | Fully private file (SSH keys, secrets) |
| `750` | `rwxr-x---` | Owner full, group read/run, others nothing |
| `664` | `rw-rw-r--` | Shared team file — owner & group edit, others read |
| `775` | `rwxrwxr-x` | Shared team directory |
| `777` | `rwxrwxrwx` | ⚠️ Everyone does everything — avoid |

## `chmod` Symbolic Mode

```
chmod [who][operator][perm] file
```

| Who | Meaning |
|---|---|
| `u` | Owner |
| `g` | Group |
| `o` | Others |
| `a` | All three |

| Operator | Meaning |
|---|---|
| `+` | Add permission, leave rest alone |
| `-` | Remove permission, leave rest alone |
| `=` | Set exactly this, clearing anything else in that category |

| Example | Effect |
|---|---|
| `chmod u+x file` | Owner gains execute |
| `chmod g-w file` | Group loses write |
| `chmod o=r file` | Others set to read-only exactly |
| `chmod a+x file` | Everyone gains execute |
| `chmod u+x,g-w file` | Multiple changes, comma-separated |

## `chown` / `chgrp` Syntax

| Command | Effect |
|---|---|
| `chown user file` | Change owner only |
| `chgrp group file` | Change group only |
| `chown user:group file` | Change owner **and** group in one command |
| `chown :group file` | Change only the group (chown shorthand) |
| `chown -R user:group dir/` | Recursive — directory + everything inside |

## `umask`

| Item | Max permission | Formula |
|---|---|---|
| New file | `666` | `666 − umask` |
| New directory | `777` | `777 − umask` |

Ubuntu default `umask` = `022` → new files `644`, new directories `755`.

| Command | Effect |
|---|---|
| `umask` | Show current mask |
| `umask 022` | Set mask for this shell session |

⚠️ Only affects files/dirs created **after** the change — never retroactive.

## Special Permission Bits

| Symbol in `ls -l` | Bit | Octal prefix | Meaning |
|---|---|---|---|
| `s` in owner's `x` slot | setuid | `4000` | Runs as the file's **owner**, not the caller (e.g. `/usr/bin/passwd`) |
| `s` in group's `x` slot | setgid | `2000` | Runs as the file's group; on a dir, new files inherit the dir's group |
| `t` in others' `x` slot | sticky | `1000` | On a dir, only a file's own owner can delete/rename it (e.g. `/tmp`) |
| Uppercase `S` / `T` | Same bit, but base `x` is missing | — | Usually unintentional |

```bash
chmod 4755 file   # setuid + rwxr-xr-x
chmod 2775 dir    # setgid + rwxrwxr-x
chmod 1777 dir    # sticky + rwxrwxrwx  (this is /tmp's exact mode)
```

## Identity Commands

| Command | Shows |
|---|---|
| `whoami` | Just your username |
| `id` | UID, GID, and every group you're in |
| `id username` | Same, for another user |
| `groups` | Just your group names |
| `groups username` | Same, for another user |

## User & Group Management

| Command | Effect |
|---|---|
| `sudo adduser username` | (Ubuntu-friendly, interactive) Create user + home dir + password prompt |
| `sudo useradd -m username` | (Lower-level) Create user; `-m` makes the home directory |
| `sudo adduser --ingroup group username` | Create user with a specific primary group |
| `sudo usermod -aG group username` | **Append** user to a supplementary group (always use `-aG` together) |
| `sudo groupadd groupname` | Create a new group |
| `sudo passwd username` | Set/change a user's password |
| `sudo deluser username` / `userdel username` | Remove a user |

⚠️ `usermod -G group username` (no `-a`) **replaces** all supplementary groups — always pair with `-a`.

## Key Files

| File | Contains | Permissions |
|---|---|---|
| `/etc/passwd` | Username : x : UID : GID : GECOS : home : shell | World-readable |
| `/etc/group` | Groupname : x : GID : member list | World-readable |
| `/etc/shadow` | Username : hashed password : aging info | Root-only |

## `su` vs `sudo`

| | `su` | `sudo` |
|---|---|---|
| Password needed | Target account's | Your own |
| Scope | New shell as that user | One command at a time |
| Audit trail | Weak | Strong (logs who ran what) |
| Ubuntu default | Root has no usable password | Works for the first install user (in `sudo` group) |

- Ubuntu/Debian `sudo`-capable group: **`sudo`**
- RHEL/Fedora/CentOS equivalent group: **`wheel`**
- Edit sudo rules only via **`sudo visudo`** — never edit `/etc/sudoers` directly.

## 🔁 The "Permission Denied" Debugging Workflow

Do this every time a command, script, or app throws `Permission denied`:

1. **Run `ls -l` on the exact file.** Note owner, group, and the permission string.
2. **Run `id` (or `groups`) on yourself (or the account running the process).** Are you the owner, in the group, or neither?
3. **Check every directory in the path with `ls -ld`.** Missing `x` on *any* parent directory blocks access, even if the file itself is `644`.
4. **Identify exactly which bit is missing for exactly which "who".** Don't guess — match owner/group/others against what you found in step 2.
5. **Grant the minimum needed bit** — `chmod u+x`, `chmod g+r`, or a precise octal value. Never jump straight to `777`.
6. **If ownership itself is wrong** (e.g. a deployed file owned by the wrong service account), fix that with `chown user:group` instead of loosening permissions.
7. **Re-test the exact command that failed** to confirm the fix, rather than assuming it worked.

## 🔁 The New Shared-Team-Folder Workflow

1. `sudo groupadd teamname` — create the group once.
2. `sudo usermod -aG teamname user1 user2 ...` — add each member.
3. `mkdir /srv/teamname && sudo chgrp teamname /srv/teamname`
4. `sudo chmod 2775 /srv/teamname` — `775` for group read/write/enter, plus **setgid** so new files always inherit the `teamname` group automatically.
5. Team members log out/in (or run `newgrp teamname`) so their shell picks up the new group membership.
