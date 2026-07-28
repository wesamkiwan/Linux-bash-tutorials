# 📚 Module 12 References — Networking Basics

Curated resources for this module's scope: `curl` and `wget`, SSH fundamentals and key-based authentication, `scp`/`rsync`, `ping`/ICMP, `ss`/`netstat`, `/etc/hosts`, and DNS lookups (`dig`/`nslookup`/`host`). Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Energetic, practical walkthroughs of `curl`, SSH keys, and basic networking concepts, aimed at making them feel concrete for beginners rather than abstract theory.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Clear, methodical Ubuntu/Debian-focused videos on SSH setup, `rsync` backups, and general server administration.
- **[freeCodeCamp.org](https://www.youtube.com/@freecodecamp)** 🟢 — Full-length free courses that include dedicated SSH and networking-fundamentals sections, good for a structured first pass.

## 📖 Official Documentation

- **[curl.se — Everything curl (free online book)](https://curl.se/docs/manual.html)** 🟡 — The authoritative, exhaustive reference for every `curl` flag and behavior, written by curl's own maintainer.
- **[GNU Wget Manual](https://www.gnu.org/software/wget/manual/wget.html)** 🟢🟡 — The official, complete reference for every `wget` option, including resuming and recursive downloads.
- **[OpenSSH official documentation](https://www.openssh.com/manual.html)** 🟡🔴 — The authoritative source for `ssh`, `ssh-keygen`, `scp`, and `sshd_config`, straight from the project that ships as the default SSH implementation on Ubuntu/Debian.
- **[rsync official documentation](https://rsync.samba.org/documentation.html)** 🟡 — The official manual and man pages for `rsync`, including the full flag reference behind `-avz`.
- **[iproute2 / `ss` man page (via Ubuntu manpages)](https://manpages.ubuntu.com/manpages/noble/en/man8/ss.8.html)** 🟡 — The definitive flag reference for `ss`, hosted on Ubuntu's own manpage archive.
- **`man ssh` / `man curl` / `man ss` / `man rsync` (local)** 🟢🟡🔴 — Run these directly in your own terminal for the exact version installed on your system — always the most accurate reference for your specific machine.

## 📝 Tutorials & Articles

- **[DigitalOcean — "How To Set Up SSH Keys on Ubuntu"](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)** 🟢🟡 — A clear, step-by-step walkthrough of exactly the `ssh-keygen`/`ssh-copy-id` flow covered in this module, on the exact distribution this course targets.
- **[DigitalOcean — "How To Use curl to Download Files"](https://www.digitalocean.com/community/tutorials/how-to-use-curl-to-download-files-from-the-command-line)** 🟢 — Practical, example-driven coverage of `curl`'s file-handling flags (`-o`, `-O`, `-L`).
- **[Baeldung on Linux — "rsync vs scp"](https://www.baeldung.com/linux/rsync-vs-scp)** 🟡 — A focused comparison explaining exactly why `rsync` is considered more robust, with concrete examples.
- **[DigitalOcean — "How To Use ss to Monitor Network Connections"](https://www.digitalocean.com/community/tutorials/how-to-use-ss-to-monitor-network-connections-on-a-linux-server)** 🟡 — Walks through reading `ss` output column by column on an Ubuntu server, matching this module's coverage closely.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `curl`, `ssh`, or `rsync` invocation and see every flag broken down individually — very useful once commands start stacking multiple flags together.

## 🎓 Courses & Learning Portals

- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course covering basic networking commands and SSH as part of its broader curriculum.
- **[freeCodeCamp — Linux/networking articles](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based guides that build naturally on earlier shell modules into SSH and basic networking.
- **[Udemy — "Linux Administration Bootcamp"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course with hands-on SSH and networking labs; search the current catalog since specific titles rotate.

## 🌐 Websites & Interactive Platforms

- **[httpbin.org](https://httpbin.org/)** 🟢 — A free, public "echo" API purpose-built for testing HTTP clients like `curl` — every request you send is reflected back so you can confirm your headers, body, and method were built correctly.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Excellent for decoding unfamiliar flag combinations on `curl`, `ssh`, and `rsync`.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice SSH key setup, `scp`/`rsync`, and `ss` without needing a real second machine.
- **[SSH.com — "SSH Academy"](https://www.ssh.com/academy/ssh)** 🟡 — A vendor-authored but genuinely thorough set of explainer articles specifically on SSH keys, `known_hosts`, and the protocol's security model.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; includes practical coverage of network commands that maps closely onto this module.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡🔴 — A strong deeper pass on the networking stack once you're comfortable with this module's basics.
- **"SSH Mastery" by Michael W. Lucas** 🟡🔴 — A short, focused, highly practical book entirely dedicated to SSH — key management, `~/.ssh/config` tricks, and server-side hardening, well beyond this module's scope for anyone going deeper.

## 👥 Communities

- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Active community for real-world networking and SSH troubleshooting discussions and war stories.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why did my host key change" or "curl vs wget" already have excellent, heavily upvoted answers here.
- **[Server Fault](https://serverfault.com/)** 🟡🔴 — Stack Exchange's dedicated site for sysadmin/server questions, including a large volume of SSH configuration and networking-diagnostics content.
