# 📚 Module 18 References — Security Auditing Scripts

Curated resources for this module's scope: world-writable files, unexpected UID-0/SUID/SGID findings, `sshd_config` hardening, cron auditing, listening-port review, package vulnerability checks (`debsecan`, `unattended-upgrades`), and failed-login log scanning (plus `fail2ban`). Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck — "Linux Server Hardening"](https://www.youtube.com/@NetworkChuck)** 🟢🟡 — Practical, energetic walkthroughs of SSH hardening, firewalls, and basic server security aimed at making the concepts concrete for newcomers.
- **[The Cyber Mentor](https://www.youtube.com/@TCMSecurityAcademy)** 🟡🔴 — Security-focused channel covering Linux privilege escalation techniques (SUID abuse, cron misconfigurations) from an offensive-security angle, which is genuinely useful for understanding exactly what an audit is trying to catch.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Clear, methodical Ubuntu/Debian-focused videos on SSH configuration, `fail2ban` setup, and general server hardening.

## 📖 Official Documentation

- **[`sshd_config` man page (Ubuntu manpages)](https://manpages.ubuntu.com/manpages/noble/en/man5/sshd_config.5.html)** 🟡 — The authoritative, complete reference for every `sshd_config` directive, including `PermitRootLogin` and `PasswordAuthentication`, straight from OpenSSH's own documentation.
- **[`find` man page (Ubuntu manpages)](https://manpages.ubuntu.com/manpages/noble/en/man1/find.1.html)** 🟢🟡 — The full reference for `find`'s `-perm`, `-xdev`, `-type`, and `-exec` options used throughout this module.
- **[fail2ban official documentation](https://fail2ban.readthedocs.io/en/latest/)** 🟡 — The project's own manual, covering installation, jail configuration, and how it actually bans offending IPs.
- **[Debian Security Team — debsecan](https://www.debian.org/security/audit/tools)** 🟡🔴 — Debian's own security-audit tooling page, including `debsecan`'s purpose and usage (Ubuntu is Debian-based and shares the same tool).
- **[Ubuntu Server Guide — Security](https://ubuntu.com/server/docs/security-introduction)** 🟢🟡 — Canonical's own official guidance on server security basics, including `unattended-upgrades` configuration.
- **`man sshd_config` / `man find` / `man crontab` / `man ss` (local)** 🟢🟡🔴 — Run these directly on your own system for the exact version installed — always the most accurate reference for your specific machine.

## 📝 Tutorials & Articles

- **[DigitalOcean — "Recommended Steps to Secure a VPS"](https://www.digitalocean.com/community/tutorials/recommended-security-measures-to-protect-your-servers)** 🟢🟡 — A clear, practical checklist covering SSH hardening, firewalls, and automatic updates on exactly the kind of server this module targets.
- **[DigitalOcean — "How To Protect SSH with fail2ban on Ubuntu"](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-20-04)** 🟡 — A step-by-step `fail2ban` installation and configuration guide, showing exactly how it automates what this module's manual log-scanning check only detects.
- **[Ubuntu Wiki — Automatic Security Updates](https://help.ubuntu.com/community/AutomaticSecurityUpdates)** 🟢🟡 — Covers configuring `unattended-upgrades` step by step.
- **[LinuxSecurity.com — "Understanding SUID, SGID, and Sticky Bit"](https://linuxsecurity.com/)** 🟡 — Focused explainer on the special permission bits and their security implications, going deeper on the exact risk this module's SUID check is built around.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `find`, `awk`, or `grep` invocation from this module and see every flag broken down individually.

## 🎓 Courses & Learning Portals

- **[TryHackMe — "Linux Fundamentals" and "Hardening Basics" rooms](https://tryhackme.com/)** 🟢🟡 — Free/freemium, hands-on, browser-based rooms that let you practice exactly the kind of privilege-escalation-hunting (SUID abuse, cron misconfig, weak permissions) this module's checks are designed to catch, from the attacker's own perspective.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course whose broader curriculum touches on basic system security and permissions.
- **[Udemy — "Linux Security and Hardening"](https://www.udemy.com/)** 🟡🔴 — A popular, affordable paid course with hands-on labs specifically on server hardening; search the current catalog since specific titles rotate.

## 🌐 Websites & Interactive Platforms

- **[CIS Benchmarks — Ubuntu Linux](https://www.cisecurity.org/benchmark/ubuntu_linux)** 🟡🔴 — Free-to-download, industry-standard hardening checklists for Ubuntu, covering everything this module touches (and much more) in exhaustive, audit-ready detail — the actual reference many compliance audits are measured against.
- **[Lynis (CISOfy)](https://cisofy.com/lynis/)** 🟡🔴 — A free, open-source, actively-maintained security auditing tool for Linux that automates a much deeper version of exactly what this module builds by hand; a natural "graduate to this next" tool once you've understood the underlying checks yourself.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Useful for decoding unfamiliar flag combinations in audit one-liners.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice running audit checks, intentionally misconfigure `sshd_config`, and see the findings change.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its permissions and `find` coverage underpins this module's core commands.
- **"UNIX and Linux System Administration Handbook" by Nemeth, Snyder, Hein, Whaley, Mackin (Pearson)** 🟡🔴 — A comprehensive, widely-respected reference with strong chapters on system security, auditing, and hardening practices used across the industry.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡🔴 — Deeper background on permissions, processes, and the boot/init/cron subsystems this module's checks reach into.

## 👥 Communities

- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Active discussion of real-world hardening decisions, incident war stories, and audit tooling trade-offs.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why is my SSH server getting brute-forced" or "what does this SUID binary do" already have excellent, heavily upvoted answers here.
- **[Server Fault](https://serverfault.com/)** 🟡🔴 — Stack Exchange's dedicated site for sysadmin/server questions, including extensive SSH-hardening and server-security-audit discussions.
- **[CIS Community](https://www.cisecurity.org/)** 🟡🔴 — The organization behind the CIS Benchmarks; their site and mailing lists are a good next step once you're operating at a compliance-driven organization.
