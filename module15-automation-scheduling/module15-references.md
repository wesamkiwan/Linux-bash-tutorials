# 📚 Module 15 References — Automation & Scheduling (cron/systemd)

Curated resources for this module's scope: `cron`/`crontab`, the 5-field time syntax, `/etc/crontab`/`/etc/cron.d/`, the cron `PATH`/environment gotcha, output redirection, `at`, and `systemd` timers (`OnCalendar=`, `systemctl`, `journalctl`). Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢🟡 — Energetic, practical walkthroughs of cron and Linux automation aimed at real sysadmin tasks.
- **[Learn Linux TV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Dedicated videos on cron scheduling and systemd service/timer management, well-suited to hands-on learners.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡🔴 — Deeper dives into systemd internals, including timers, for viewers past the absolute basics.

## 📖 Official Documentation

- **[`man crontab` / `man 5 crontab`](https://man7.org/linux/man-pages/man5/crontab.5.html)** 🟡 — The authoritative specification of the crontab file format, all five fields, and every special string.
- **[`man cron`](https://man7.org/linux/man-pages/man8/cron.8.html)** 🟡 — Documents the cron daemon itself, including how `/etc/crontab` and `/etc/cron.d/` are read.
- **[`man at`](https://man7.org/linux/man-pages/man1/at.1.html)** 🟢🟡 — The official reference for `at`, `atq`, and `atrm` usage and time-specification syntax.
- **[systemd.timer(5) man page](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)** 🟡🔴 — The authoritative reference for every `.timer` directive, including `OnCalendar=` and `Persistent=`.
- **[systemd.time(7) man page — calendar event syntax](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html)** 🟡🔴 — The full, precise grammar for `OnCalendar=` expressions, with many worked examples.
- **[systemctl(1) man page](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)** 🟡 — Covers `enable --now`, `list-timers`, `daemon-reload`, and every other subcommand used in this module.
- **[journalctl(1) man page](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)** 🟡 — Covers filtering logs by unit (`-u`), time range (`--since`), and more.

## 📝 Tutorials & Articles

- **[crontab.guru](https://crontab.guru/)** 🟢 — An interactive tool that translates a cron expression into plain English (and back) — the fastest way to build or verify a schedule without mentally decoding five fields.
- **[DigitalOcean — "How To Use Cron to Automate Tasks on Ubuntu"](https://www.digitalocean.com/community/tutorials/how-to-use-cron-to-automate-tasks-ubuntu-1804)** 🟢🟡 — A clear, example-driven walkthrough covering user crontabs, `/etc/cron.d/`, and common gotchas.
- **[DigitalOcean — "Understanding Systemd Units and Unit Files"](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)** 🟡 — Explains `.service` files and unit-file structure in depth, useful groundwork before writing `.timer` files.
- **[Red Hat — "Automating system tasks using systemd timers"](https://www.redhat.com/sysadmin/setting-systemd-timers)** 🟡🔴 — A practical, production-oriented guide to `.timer`/`.service` pairs, including `Persistent=` and troubleshooting.
- **[Ubuntu Server Guide — "Cron"](https://ubuntu.com/server/docs)** 🟢🟡 — Debian/Ubuntu-specific notes on cron configuration and file locations for this module's target platform.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux/Bash automation articles](https://www.freecodecamp.org/news/tag/linux/)** 🟢🟡 — Free, text-based content covering cron and task automation for production use.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟡 — Free, self-paced course touching on job scheduling and service management fundamentals.
- **[Udemy — "Linux System Administration" courses](https://www.udemy.com/)** 🟡🔴 — Affordable paid courses with dedicated sections on cron and systemd timers; search the current catalog since specific titles rotate.

## 🌐 Websites & Interactive Platforms

- **[crontab.guru](https://crontab.guru/)** 🟢 — (Also listed above.) Build and sanity-check any cron schedule instantly in the browser.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste a cron command line or `systemctl` invocation and see every flag broken down individually.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice writing crontabs and systemd timers without risking a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; includes coverage of `cron` and job scheduling fundamentals as part of its broader shell curriculum.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡🔴 — Includes a solid chapter on `systemd`, unit files, and how services and timers fit into overall system startup and management.
- **"Linux Administration: A Beginner's Guide" by Wale Soyinka (McGraw Hill)** 🟡 — Broad sysadmin coverage including task scheduling with `cron` and `at` in a production context.

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable archive; questions like "cron job works manually but not scheduled" or "systemd timer not firing" already have excellent, heavily upvoted answers.
- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Active community for production Linux administration questions, including real-world cron and systemd timer war stories.
- **[r/systemd](https://www.reddit.com/r/systemd/)** 🟡🔴 — Focused community specifically for `systemd`-related questions, including timers and unit-file troubleshooting.
