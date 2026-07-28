# 📚 Module 10 References — Process Management & Job Control

Curated resources for this module's scope: processes and the process tree, `ps`/`pstree`, `top`/`htop`, process states and zombies, job control (`&`, `jobs`, `fg`, `bg`), `nohup`/`disown`, signals and `kill`/`killall`/`pkill`, and `/proc`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Practical, energetic walkthroughs of `top`, `kill`, and background jobs on real Linux servers, aimed at making process management feel concrete rather than abstract.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡 — Several dedicated videos on job control, signals, and comparing `top`/`htop`/`btop`, aimed at users past absolute basics.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Clear, methodical Ubuntu/Debian-focused sysadmin content, including process monitoring and systemd basics that tie directly into "what is PID 1."

## 📖 Official Documentation

- **[GNU coreutils manual — `kill` and process utilities](https://www.gnu.org/software/coreutils/manual/coreutils.html)** 🟢🟡 — Authoritative reference for the core utilities that ship with every Linux system.
- **[procps-ng documentation (`ps`, `top`, `kill`, `pkill`, `pgrep`)](https://gitlab.com/procps-ng/procps)** 🟡 — The actual upstream project behind `ps`, `top`, `kill`, `pkill`, and `pgrep` on Ubuntu/Debian; its README and man page sources are the ground truth for exact flag behavior.
- **[htop official site](https://htop.dev/)** 🟢🟡 — The official homepage and documentation for `htop`, including its interactive key reference and FAQ.
- **`man ps` / `man kill` / `man signal` / `man proc` (local)** 🟢🟡🔴 — Run these directly in your own terminal; `man 7 signal` in particular gives the full, authoritative signal-number table for your exact system.
- **[Linux kernel documentation — `/proc` filesystem](https://www.kernel.org/doc/html/latest/filesystems/proc.html)** 🔴 — The definitive, deep-dive reference for exactly what every file under `/proc/<pid>/` means, straight from the kernel docs.

## 📝 Tutorials & Articles

- **[DigitalOcean — "How To Use `ps`, `kill`, and `nice` to Manage Processes in Linux"](https://www.digitalocean.com/community/tutorials/process-management-in-linux)** 🟢🟡 — A clear, example-driven walkthrough covering exactly this module's core commands on an Ubuntu-flavored setup.
- **[DigitalOcean — "How To Use Bash's Job Control to Manage Foreground and Background Processes"](https://www.digitalocean.com/community/tutorials/how-to-use-bash-s-job-control-to-manage-foreground-and-background-processes)** 🟡 — Focused specifically on `jobs`, `fg`, `bg`, and `nohup`, with runnable examples.
- **[Baeldung on Linux — "Zombie Processes"](https://www.baeldung.com/linux/kill-zombie-processes)** 🟡 — A focused explanation of why zombies occur and why they can't be killed directly, with practical detection commands.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `ps`, `kill`, or `pkill` invocation and see every flag broken down individually — useful once you start combining flags like `ps aux --sort=-%cpu`.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux process management content](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based articles that build directly on earlier modules into process and job control.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with a dedicated unit on process management, signals, and monitoring.
- **[Udemy — "Linux Administration Bootcamp"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course with hands-on process-management labs; search the current catalog since specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Excellent for decoding unfamiliar `ps`/`kill`/`pkill` flag combinations you encounter in the wild.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice killing runaway processes, testing `nohup`, and experimenting with signals without risking a real machine.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Continue from earlier modules — several levels require reasoning about running processes and background jobs to find the next password.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its chapter on job control and process management maps closely onto this module and goes into further depth on signals.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡🔴 — Excellent deeper coverage of the process model, `/proc`, and how the kernel manages processes under the hood, for a strong second pass after this module.
- **"The Linux Programming Interface" by Michael Kerrisk (No Starch Press)** 🔴 — The definitive, exhaustive reference on process creation, signals, and `/proc` internals at the systems-programming level — recommended once you're ready to go far beyond day-to-day usage.

## 👥 Communities

- **[r/linuxadmin](https://www.reddit.com/r/linuxadmin/)** 🟡🔴 — Active community for real-world Linux administration questions, including process troubleshooting and production incident war stories.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why can't I kill a zombie process" or "SIGTERM vs SIGKILL" already have excellent, heavily upvoted answers here.
- **[Stack Overflow — `process` and `signals` tags](https://stackoverflow.com/questions/tagged/process)** 🟢🟡🔴 — Large archive of concrete process-management and signal-handling problems and solutions.
