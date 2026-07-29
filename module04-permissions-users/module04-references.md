# 📚 Module 4 References — Permissions, Users & Ownership

Curated resources for this module's scope: the permission model, `chmod`/`chown`/`chgrp`, `umask`, setuid/setgid/sticky bit, users/groups, `/etc/passwd`/`/etc/group`/`/etc/shadow`, and `su` vs `sudo`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck — "Linux Permissions" videos](https://www.youtube.com/@NetworkChuck)** 🟢 — Visual, energetic walkthroughs of `chmod`, `chown`, and the octal math, good for cementing the r=4/w=2/x=1 pattern.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Dedicated videos on file permissions, `sudo` configuration, and user/group management aimed at people actually administering servers.
- **[freeCodeCamp.org — Linux Permissions Crash Course](https://www.youtube.com/c/Freecodecamp)** 🟢 — Long-form, example-heavy coverage of the ownership/permission model as part of its broader Linux courses.

## 📖 Official Documentation

- **[GNU Coreutils Manual — `chmod`](https://www.gnu.org/software/coreutils/manual/html_node/chmod-invocation.html)** 🟢🟡 — The authoritative reference for every `chmod` flag and mode syntax, symbolic and octal.
- **[GNU Coreutils Manual — `chown`](https://www.gnu.org/software/coreutils/manual/html_node/chown-invocation.html)** 🟢🟡 — Authoritative reference for `chown`'s `user:group` syntax and recursive options.
- **[`man` pages (local)](https://man7.org/linux/man-pages/)** 🟢🟡🔴 — Run `man chmod`, `man chown`, `man umask`, `man passwd`, `man useradd`, `man usermod`, `man sudoers` directly in your terminal, or browse the same content online at man7.org — always current for your exact installed version.
- **[Sudo Project — Official Documentation](https://www.sudo.ws/docs/)** 🟡 — The authoritative source on `sudo`, `sudoers` syntax, and `visudo`, maintained by the actual sudo project.
- **[Debian Wiki — sudo](https://wiki.debian.org/sudo)** 🟢🟡 — Debian/Ubuntu-specific guidance on configuring and using `sudo`, including the `sudo` group convention.

## 📝 Tutorials & Articles

- **[DigitalOcean — "An Introduction to Linux Permissions"](https://www.digitalocean.com/community/tutorials/an-introduction-to-linux-permissions)** 🟢🟡 — A widely-used, clearly written walkthrough of the exact permission model and `chmod`/`chown` syntax covered in this module.
- **[DigitalOcean — "How To Add and Delete Users on Ubuntu"](https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-ubuntu-20-04)** 🟢 — Practical, step-by-step coverage of `adduser`, `usermod`, and `deluser` on Ubuntu specifically.
- **[Linux Journey — "Permissions" and "Users & Groups" sections](https://linuxjourney.com/)** 🟢 — Free, bite-sized interactive lessons covering exactly this module's ground, good as a second explanation if something didn't click the first time.
- **[Red Hat — "SUID, SGID, and Sticky Bit" guide](https://www.redhat.com/sysadmin/suid-sgid-sticky-bit)** 🟡 — A well-regarded, example-driven explanation of the three special permission bits, including real-world security context.

## 🎓 Courses & Learning Portals

- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with dedicated units on permissions, ownership, and user/group administration.
- **[Codecademy — Learn the Command Line](https://www.codecademy.com/learn/learn-the-command-line)** 🟢 — Interactive, browser-based practice covering `chmod` and ownership basics hands-on.
- **[KodeKloud — Linux Basics for DevOps / Linux Administration](https://kodekloud.com/)** 🟡 — Practical, lab-based courses (some free content) that put heavy emphasis on permissions and user management the way sysadmins actually use them daily.

## 🌐 Websites & Interactive Platforms

- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — Paste any `chmod`, `chown`, or `usermod` command and see each flag broken down visually — very useful while the symbolic-mode syntax is still new.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Several levels specifically require reading permission strings, finding setuid binaries, or navigating directories with restrictive permissions — direct hands-on practice for this module.
- **[Linux Survival](https://linuxsurvival.com/)** 🟢 — A free, guided, interactive browser terminal that includes a dedicated permissions module, safe to experiment in without touching a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its chapter on permissions covers `chmod`, `chown`, `umask`, and the ownership model in excellent, example-rich depth.
- **"How Linux Works" by Brian Ward** (No Starch Press) 🟡🔴 — Goes deeper into how the kernel actually enforces permissions and how users/groups map to UIDs/GIDs under the hood — good once you want more than "what to type."
- **"UNIX and Linux System Administration Handbook" by Nemeth et al.** 🔴 — The classic, comprehensive sysadmin reference; its chapters on user/account management and access control are considered essential once you're managing real production systems.

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; nearly every "why can't I access this file" or "what does this `s`/`t` in `ls -l` mean" question already has a high-quality answer here.
- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Practitioner-focused community discussing real production permission and account-management scenarios, good for seeing how professionals actually reason about access control.
- **[r/linux4noobs](https://www.reddit.com/r/linux4noobs/)** 🟢 — Beginner-friendly home for questions like "what's the difference between `su` and `sudo`?" — no question here is too basic.
