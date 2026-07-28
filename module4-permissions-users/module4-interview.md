# 🎤 Module 4 Interview Prep — Permissions, Users & Ownership

## Conceptual Questions

### 🟢 Beginner

**Q1: Explain the difference between `755` and `644` permissions.**

> "Both are octal permission modes — three digits for owner, group, and others, where read is worth 4, write is worth 2, and execute is worth 1. `755` breaks down to `rwxr-xr-x`: the owner gets full read/write/execute, and both group and others get read plus execute, but not write. `644` breaks down to `rw-r--r--`: the owner can read and write, but nobody — not even the owner — gets execute, and group/others can only read. In practice, `755` is what you use for scripts, executables, and directories that need to be entered or run, while `644` is for plain data or config files that should never be executed, like a `.txt` or `.conf` file."

**Q2: What does execute permission mean on a directory? Why isn't it the same as executing a program?**

> "A directory isn't a program you run — it's really just a list of filenames. So execute on a directory means something different: it controls whether you can actually *enter* or *traverse* that directory, using `cd`, or reach any file inside it by its path. Without execute on a directory, you can't get into it at all, even if you have read and write permission on it, and even if a file inside is fully world-readable — the shell simply can't walk through the directory to find that file. This is exactly why someone can get a confusing 'permission denied' on a file that looks perfectly readable — the actual missing bit is execute on one of its parent directories, not on the file itself."

**Q3: What are the three categories of 'who' in the Linux permission model?**

> "Owner, group, and others. The owner is the single account tied to the file — usually whoever created it. The group is one specific group assigned to the file, and anyone in that group gets whatever access level is set for the group category. Others is everyone else on the system who is neither the owner nor a member of that group. Every file always has exactly these three categories defined, never more, never fewer."

**Q4: What's the difference between `whoami`, `id`, and `groups`?**

> "`whoami` just prints your current username, nothing else. `groups` prints the names of every group you belong to. `id` gives you the fullest picture in one shot — your UID, your primary GID, and the complete list of every group you're a member of, both by number and by name. If I only need my username I'd use `whoami`; if I need to debug a permissions issue, I reach for `id` because it shows everything relevant at once."

**Q5: Why can't you just directly edit `/etc/passwd` or `/etc/shadow` with a text editor to add a user?**

> "You technically can open them in an editor, but you shouldn't, because the dedicated tools like `useradd`, `adduser`, and `passwd` validate the file format, keep the numbering (UID/GID) consistent, lock the file against concurrent writes, and update all the related files together — home directory creation, shadow entries, and so on. A manual typo in `/etc/passwd` can break logins for every single user on the system, since it's read constantly by the OS."

### 🟡 Intermediate

**Q6: What's the purpose of setuid, and why does `/usr/bin/passwd` need it?**

> "Setuid makes an executable run with the privileges of the file's *owner*, rather than whoever launched it. `passwd` needs to update `/etc/shadow`, which is only writable by root, so that a regular user's password change can actually be saved. Without setuid, a normal user literally couldn't run `passwd` successfully, because they have no write access to `/etc/shadow` themselves. Setuid lets `passwd` briefly act as root, but only to do the one specific job it's designed for — it's a controlled, narrow way to grant a privileged operation to unprivileged users."

**Q7: What does the sticky bit do, and why does `/tmp` need it?**

> "The sticky bit, set on a directory, means that even though multiple users can create files there, each user can only delete or rename the files *they themselves* own — not anyone else's. `/tmp` has to be writable by literally every user and process on the system, because all sorts of programs create temporary files there. Without the sticky bit, that world-writable permission would also let any user delete or overwrite any other user's temp files, which would be a serious problem on any multi-user system. The sticky bit closes that specific hole while still letting everyone use the directory freely."

**Q8: How does `umask` affect newly created files versus directories?**

> "`umask` is subtracted from a theoretical maximum permission at creation time — `666` for a new file and `777` for a new directory. Files never default to executable no matter what the umask is, because the starting point for files is `666`, which has no execute bit to begin with — you always have to add execute yourself afterward if a new file needs it. Directories start from `777`, so with the common Ubuntu default umask of `022`, new files come out as `644` and new directories come out as `755`. It's important to remember `umask` only affects things created after it's set — it never retroactively touches existing files."

**Q9: What's the difference between `chown` and `chgrp`, and how do you change both owner and group in one step?**

> "`chown` changes the owning user of a file, and `chgrp` changes the owning group. You don't actually need two separate commands to change both, though — `chown` itself supports a combined syntax, `chown user:group file`, which sets the owner and group in a single command. I'd use that combined form any time I'm handing a file over to a service account, like `chown www-data:www-data` after deploying a website, rather than running two separate commands."

**Q10: What does setgid do differently on a directory compared to on an executable file?**

> "On an executable file, setgid works like setuid but with the group instead of the owner — the program runs with the privileges of the file's group. On a directory, though, setgid means something more commonly used in practice: any new file or subdirectory created inside that directory automatically inherits the *directory's* group, rather than the creating user's own primary group. That's the standard trick for setting up a shared team folder — combine setgid with group write access so that every file dropped in there automatically belongs to the team, no matter who created it."

### 🔴 Advanced

**Q11: Why does `/etc/shadow` exist at all — why not just keep password hashes in `/etc/passwd`?**

> "`/etc/passwd` has to be world-readable, because all sorts of tools and system calls need to look up basic account information — usernames, UIDs, home directories, shells — for every user on the system, constantly. If password hashes lived in that same world-readable file, any local user could copy them out and run an offline brute-force or dictionary attack against them with no rate limiting at all. `/etc/shadow` was split out specifically to be readable only by root, so the actual hashes — plus password aging and expiration data — are locked down to a file that ordinary users can't even open, while `/etc/passwd` stays readable for everything else that legitimately needs it."

**Q12: A setuid-root binary has a bug that lets an attacker spawn a shell. Why is this worse than a bug in an equivalent non-setuid binary?**

> "Because the setuid bit means the binary always executes with root's privileges, regardless of who launched it. If an attacker can exploit a bug to make that binary do something unintended — like spawn an interactive shell — that shell inherits the *root* privileges the binary was running with, not the attacker's own limited account. In a non-setuid binary, the same bug would only ever hand the attacker whatever privileges they already had. That's exactly why setuid-root programs are treated as a high-value, carefully audited attack surface, and why you should never add setuid to a script or program unless you specifically understand and need that behavior."

---

## Practical/Coding Questions

**Q1: You need `deploy.sh` to be runnable by its owner and readable-but-not-runnable by everyone else. Show both a symbolic and an octal command that achieve this, and explain the octal math.**

Solution:
```bash
# Symbolic
chmod u=rwx,go=r deploy.sh

# Octal
chmod 744 deploy.sh
```
Explanation: Owner needs `rwx` = 4+2+1 = 7. Group and others each need read-only, `r--` = 4+0+0 = 4. So the full octal mode is `744`. The symbolic version sets owner to exactly `rwx` and both group and others to exactly `r` using `=`, which clears anything else that might already be set on those categories.

**Q2: A shared upload directory needs: owner full access, group can read/write/enter, others get nothing, and any new file created inside should automatically belong to the group. Show the command.**

Solution:
```bash
chmod 2770 uploads/
```
Explanation: `770` gives owner and group full `rwx` (7 = 4+2+1 each) and others nothing (0). The leading `2` sets the **setgid** bit, so any file or subdirectory created inside `uploads/` inherits the directory's group automatically, instead of the creating user's own primary group — exactly what's needed for a shared team folder.

**Q3: You deploy a website and need to hand every file under `/var/www/mysite` to the `www-data` user and group, recursively. Show the command.**

Solution:
```bash
sudo chown -R www-data:www-data /var/www/mysite
```
Explanation: `-R` makes the change recursive, walking into every subdirectory and file. The combined `user:group` syntax sets both the owner and group to `www-data` in a single pass, which is the standard step after deploying files as your own user so the web server process (which typically runs as `www-data`) can actually read — and if needed, write — its own application files.

**Q4: Create a new group called `ops`, create a user called `deployer` with `ops` as their primary group, and also add them to the existing `sudo` group — without wiping any of their other group memberships.**

Solution:
```bash
sudo groupadd ops
sudo adduser --ingroup ops deployer
sudo usermod -aG sudo deployer
groups deployer
```
Explanation: `groupadd` creates the group first since `adduser --ingroup` needs it to already exist. `adduser --ingroup ops deployer` creates the user with `ops` as the *primary* group. Then `usermod -aG sudo deployer` **appends** `sudo` as a supplementary group — the `-a` flag is essential here, because plain `-G sudo` would replace all of `deployer`'s existing groups with just `sudo`, silently removing them from `ops`.

**Q5: You want every new file your current shell session creates to default to `660` (owner and group read/write, others nothing) instead of the usual `644`. What do you run, and why?**

Solution:
```bash
umask 006
```
Explanation: Files start from a max of `666`. To land on `660`, the umask needs to remove write-only from others (subtract 6 from the last digit: 6−6=0) and leave owner/group untouched (subtract 0 each). That's a umask of `006`. This only applies to files/directories created after running this command, for the rest of the current shell session — it would need to go in a startup file like `~/.bashrc` to persist across sessions.

---

## Gotcha Questions

**Q1: "I ran `chmod 777` on a file to fix a 'Permission denied' error and it worked — so it was the right fix, right?"**

> Trap: The candidate should recognize that `777` "working" only proves the *symptom* went away, not that the fix was correct. `777` grants read, write, and execute to literally every account and process on the system, including any attacker who gets even minimal access. The right response is to have diagnosed *which specific bit, for which specific category* was actually missing — usually a single `chmod u+x` or a specific octal value — rather than opening every door at once. A strong candidate calls out `777` as a red flag in a code review or server audit, not a valid solution.

**Q2: "I set `umask 000` expecting more permissive files, but new files still aren't executable. What's going on?"**

> Trap: This tests whether the candidate understands that umask is subtracted from a *ceiling* that's different for files versus directories — `666` for files, `777` for directories. Since `666` has no execute bit in it to begin with, no umask value can ever make a newly created plain file executable; you always have to add execute explicitly afterward with `chmod +x`. A umask of `000` on a fresh file gives `666` (`rw-rw-rw-`), not `777` — that misconception, that umask can "add" execute where the base ceiling never had it, is the trap.

**Q3: "A production file is owned by `root`, and now my own script running as a regular user can't touch it — but the file shows permission `644`, which includes read for others. Why can't I even delete it?"**

> Trap: This tests the deeper point that deleting a file is actually controlled by the **directory** it lives in, not the file's own permission bits. `644` on the file lets anyone read it, but whether you can remove or rename it depends on whether *you* have write permission on the containing directory — if that directory isn't writable to you (or doesn't have the sticky bit protecting your own files specifically), root ownership of the file itself has nothing to do with why you can't delete it; the directory's permissions are the actual blocker.

---

## Quick-Fire Rapid Review

- **Q: What does `r` mean on a directory?** A: You can list the filenames inside it.
- **Q: What does `x` mean on a directory?** A: You can enter/traverse it (`cd` into it or reach files inside by path).
- **Q: Octal value for `rwx`?** A: `7` (4+2+1).
- **Q: Octal value for `r--`?** A: `4`.
- **Q: What does `chmod 755` set?** A: `rwxr-xr-x` — owner full, group/others read+execute.
- **Q: What does `chmod 644` set?** A: `rw-r--r--` — owner read/write, group/others read-only.
- **Q: Command to change both owner and group at once?** A: `chown user:group file`.
- **Q: What subtracts from `666`/`777` to set default new-file/directory permissions?** A: `umask`.
- **Q: What does setuid do on an executable?** A: Runs it with the file owner's privileges, not the caller's.
- **Q: What does the sticky bit do on `/tmp`?** A: Lets everyone create files, but only the file's own owner can delete/rename it.
- **Q: Which file stores hashed passwords?** A: `/etc/shadow` (not `/etc/passwd`).
- **Q: Why is `/etc/shadow` locked to root-only?** A: So password hashes can't be copied out and cracked offline by any local user.
- **Q: Flag to append a user to a group without wiping existing memberships?** A: `usermod -aG group user`.
- **Q: `su` asks for whose password? `sudo` asks for whose?** A: `su` asks for the target account's password; `sudo` asks for your own.
- **Q: Ubuntu/Debian sudo-capable group name?** A: `sudo` (RHEL/Fedora equivalent: `wheel`).
- **Q: Correct way to edit `/etc/sudoers`?** A: `sudo visudo`, never a plain text editor.
- **Q: Is the root account usable by default on Ubuntu?** A: No — it has no set password by default; `sudo` is the intended path.
