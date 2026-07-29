# 📚 Module 2 References — Filesystem Navigation & File Operations

Curated resources for this module's scope: the filesystem hierarchy, navigation, listing, creating/copying/moving/deleting files, wildcards, links, and disk usage. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[freeCodeCamp.org — "Linux Crash Course" (filesystem/navigation sections)](https://www.youtube.com/c/Freecodecamp)** 🟢 — Long-form crash courses that dedicate real time to `cd`/`ls`/`cp`/`mv`/`rm` with live demos, great for reinforcing this module's commands.
- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Practical, visual walkthroughs of the Linux filesystem and everyday file commands, good for seeing the FHS "in the wild" on a real server/VM.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Focused, no-nonsense Linux tutorials, including dedicated videos on file management and the FHS.

## 📖 Official Documentation

- **[GNU Coreutils Manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)** 🟢🟡 — The authoritative reference for `ls`, `cp`, `mv`, `rm`, `mkdir`, `touch`, `ln`, `du`, `df`, and nearly every command in this module — bookmark this alongside the Bash manual from Module 1.
- **[Filesystem Hierarchy Standard (FHS) — pathname.com](https://refspecs.linuxfoundation.org/fhs.shtml)** 🟡 — The actual published standard defining what `/etc`, `/var`, `/usr`, etc. are supposed to contain; more detail than a beginner needs day one, but the definitive source once you're curious.
- **`man hier` (local)** 🟢🟡 — Run `man hier` in your own terminal for a concise manual page describing the Linux filesystem hierarchy, always available offline.

## 📝 Tutorials & Articles

- **[Linux Journey — "Grep'ing" and "Filesystem" sections](https://linuxjourney.com/)** 🟢 — Free, bite-sized lessons that cover exactly the ground in this module: the filesystem tree, navigation, and permissions basics.
- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — Paste any `cp`, `mv`, `rm`, or `ls` command with flags and see each part visually broken down — extremely useful while you're still learning what each flag does.
- **[DigitalOcean — "An Introduction to Linux Basics" tutorial series](https://www.digitalocean.com/community/tutorial-series/getting-started-with-linux)** 🟢 — Well-regarded, practical tutorials covering filesystem navigation and file management aimed at people who'll actually manage servers.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux curriculum](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based lessons covering file management fundamentals, a direct continuation of Module 1's resources.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with dedicated units on the filesystem and file management commands.
- **[Codecademy — Learn the Command Line](https://www.codecademy.com/learn/learn-the-command-line)** 🟢 — Interactive browser-based practice for exactly this module's commands (`ls`, `cd`, `cp`, `mv`, `rm`, wildcards).

## 🌐 Websites & Interactive Platforms

- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Continue from Module 1 — many early Bandit levels specifically require navigating directories, finding hidden files, and using wildcards to locate the next password.
- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Use it interactively any time you build a `cp`/`mv`/`rm` command with flags you're unsure about.
- **[Linux Survival](https://linuxsurvival.com/)** 🟢 — A free, interactive browser terminal that walks through filesystem navigation and file commands step by step, ideal for a beginner practicing without risk to a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; Part I ("Learning the Shell") covers this exact module's content — navigation, `ls`, `cp`, `mv`, `rm`, wildcards, and links — in excellent depth.
- **"How Linux Works" by Brian Ward** (No Starch Press) 🟡🔴 — Chapter on the filesystem hierarchy explains *why* the FHS looks the way it does under the hood; good once you want more than "what to type."

## 👥 Communities

- **[r/linux4noobs](https://www.reddit.com/r/linux4noobs/)** 🟢 — Still a great home for beginner questions like "what's the difference between `/root` and `/`?" — no question here is too basic.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Searchable, high-quality Q&A; a huge fraction of file-management questions ("why did `rm -rf` behave this way," "hard link vs symlink") already have excellent answers here.
- **[Linux Questions Forums](https://www.linuxquestions.org/)** 🟢🟡 — Long-running, beginner-friendly forum with dedicated sections for general Linux file/directory questions.
