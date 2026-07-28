# 📚 Module 11 References — Package Management & System Monitoring

Curated resources for this module's scope: `apt` fundamentals (update/upgrade/install/remove/purge/autoremove), `dpkg`, PPAs and Snap, system identification (`uname`, `lsb_release`, `/etc/os-release`, `hostnamectl`), and system monitoring (`uptime`/load average, `free`, `df`, `du`, `vmstat`/`iostat`). Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Energetic, hands-on videos covering `apt`, system monitoring, and general Ubuntu server administration aimed at making package management feel approachable.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Methodical Ubuntu/Debian-focused sysadmin content, including dedicated videos on `apt`, disk management, and reading system health.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡 — Covers package management across multiple distro families (useful for the `apt` vs. `yum`/`dnf` awareness this module briefly touches), plus opinions on Snap/Flatpak trade-offs worth hearing a second perspective on.

## 📖 Official Documentation

- **[Ubuntu Server Guide — Package Management](https://ubuntu.com/server/docs/package-management)** 🟢🟡 — Canonical's own official guide to `apt`, `dpkg`, and package management concepts on Ubuntu — the authoritative source for this module's core tool.
- **[Debian Wiki — Apt](https://wiki.debian.org/apt)** 🟢🟡 — Debian's own reference for `apt`, since Ubuntu is Debian-based and shares the same underlying tooling and philosophy.
- **[Debian Wiki — dpkg](https://wiki.debian.org/dpkg)** 🟡 — Focused documentation on `dpkg` itself, the lower-level tool `apt` is built on.
- **[Ubuntu Community Help Wiki — PPA](https://help.ubuntu.com/community/PPA)** 🟡 — Official-adjacent community documentation explaining what PPAs are, how to add them, and the trust considerations involved.
- **[Snapcraft Documentation](https://snapcraft.io/docs)** 🟢🟡 — The official documentation for Snap, straight from the team that builds and maintains it.
- **`man apt` / `man dpkg` / `man uptime` / `man free` / `man df` / `man du` (local)** 🟢🟡 — Run these directly in your own terminal for the exact, authoritative flag reference for your installed version.

## 📝 Tutorials & Articles

- **[DigitalOcean — "How To Manage Packages in Ubuntu and Debian with Apt-Get and Apt"](https://www.digitalocean.com/community/tutorials/how-to-manage-packages-in-ubuntu-and-debian-with-apt-get-apt)** 🟢🟡 — A clear, example-driven walkthrough covering exactly this module's core `apt` commands.
- **[DigitalOcean — "How To Monitor System Metrics on Ubuntu 22.04 With Netdata"](https://www.digitalocean.com/community/tutorials/how-to-monitor-system-metrics-with-netdata-on-ubuntu-22-04)** 🟡 — Goes beyond the manual commands in this module into a real dashboarding tool, useful once you outgrow checking `free`/`df` by hand.
- **[Baeldung on Linux — "Understanding Linux Load Averages"](https://www.baeldung.com/linux/load-average)** 🟡 — A focused, well-explained deep dive specifically on interpreting load average relative to core count — exactly this module's trickiest concept.
- **[Linuxize — "Linux free Command"](https://linuxize.com/post/free-command-in-linux/)** 🟢🟡 — A clear breakdown of every `free` column, including the `buff/cache` vs. `available` distinction this module emphasizes.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `apt`, `dpkg`, `df`, or `du` invocation and see every flag broken down individually.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux system administration content](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based articles covering package management and system monitoring fundamentals.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with dedicated units on package management and system monitoring tools.
- **[Udemy — "Linux Administration Bootcamp"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course with hands-on labs covering `apt`, disk management, and monitoring; search the current catalog since specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Excellent for decoding unfamiliar `apt`/`dpkg`/`df`/`du` flag combinations you encounter in the wild.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice `apt` operations and monitoring commands without risking a real machine.
- **[Ubuntu Packages Search](https://packages.ubuntu.com/)** 🟢🟡 — Search the actual Ubuntu package repositories online before installing, to confirm a package name, version, and description ahead of time.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its chapters on package management and system monitoring map directly onto this module.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡🔴 — Strong deeper coverage of package management internals and system resource monitoring, a good second pass after this module.
- **"UNIX and Linux System Administration Handbook" by Nemeth et al. (Addison-Wesley)** 🟡🔴 — The industry-standard sysadmin reference, with thorough coverage of package management and performance monitoring across distro families.

## 👥 Communities

- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Active community for real-world Linux administration, including patch-management strategy and production monitoring war stories.
- **[Ask Ubuntu](https://askubuntu.com/)** 🟢🟡 — Enormous, searchable Q&A archive specifically for Ubuntu — questions like "apt vs apt-get" or "why is my disk full" already have excellent, heavily upvoted answers here.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Broader Linux/Unix Q&A archive, strong for cross-distro package management and system monitoring questions.
