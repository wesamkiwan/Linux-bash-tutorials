# 📚 Module 3 References — Viewing & Finding Files

Curated resources for this module's scope: viewing file contents (`cat`, `less`, `head`, `tail`), counting (`wc`), searching (`grep` fundamentals), finding files (`find`, `locate`, `which`/`whereis`), and a first look at `sort`/`uniq`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck — "grep" and "find" tutorials](https://www.youtube.com/@NetworkChuck)** 🟢 — Visual, practical walkthroughs of `grep` and `find` with real terminal demos, a good reinforcement of this module's core searching skills.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Focused, no-nonsense videos covering `less`, `tail -f`, and log-watching workflows exactly as used on the job.
- **[freeCodeCamp.org — Linux Crash Course (searching/finding sections)](https://www.youtube.com/c/Freecodecamp)** 🟢 — Long-form crash course content with dedicated time on `grep`, `find`, and viewing files, good for beginners wanting a single continuous walkthrough.

## 📖 Official Documentation

- **[GNU Grep Manual](https://www.gnu.org/software/grep/manual/grep.html)** 🟢🟡 — The authoritative reference for every `grep` flag covered here (and the regex depth that's coming in Module 9).
- **[GNU Findutils Manual (`find`, `locate`, `updatedb`)](https://www.gnu.org/software/findutils/manual/html_mono/find.html)** 🟢🟡 — The authoritative reference for `find`'s predicates (`-name`, `-type`, `-size`, `-mtime`, `-exec`) and how `locate`/`updatedb` work together.
- **[GNU Coreutils Manual (`cat`, `head`, `tail`, `wc`, `sort`, `uniq`)](https://www.gnu.org/software/coreutils/manual/coreutils.html)** 🟢🟡 — Covers every viewing/counting command in this module in full detail — the same manual referenced in Module 2.
- **`man less` (local)** 🟢 — Run `man less` in your own terminal for the complete list of navigation keys and search options inside the pager.

## 📝 Tutorials & Articles

- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — Paste any `grep`, `find`, or `tail` command with flags and see each part visually broken down — extremely useful while learning what each flag actually does.
- **[Linux Journey — "Grep'ing" section](https://linuxjourney.com/)** 🟢 — Free, bite-sized lesson specifically on `grep` fundamentals, matching this module's beginner scope closely.
- **[DigitalOcean — "How To Use Find to Search for Files" tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-find-and-locate-to-search-for-files-on-linux)** 🟢🟡 — A well-regarded, practical walkthrough of `find` and `locate` side by side, aimed at people who'll actually manage servers.
- **[DigitalOcean — "grep Command in Linux/Unix with Examples" tutorial](https://www.digitalocean.com/community/tutorials/grep-command-in-linux-unix-with-examples)** 🟢 — Clear, example-driven coverage of the exact `grep` flags taught in this module.

## 🎓 Courses & Learning Portals

- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with dedicated units on searching, finding, and viewing files, continuing directly from Modules 1-2.
- **[Codecademy — Learn the Command Line](https://www.codecademy.com/learn/learn-the-command-line)** 🟢 — Interactive browser-based practice covering `grep`, `find`, and file-viewing commands hands-on.
- **[freeCodeCamp — Linux curriculum](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based lessons that continue naturally into searching and finding files after the filesystem-navigation basics.

## 🌐 Websites & Interactive Platforms

- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Continue from Modules 1-2 — many Bandit levels specifically require `grep`, `find`, and reading files with `cat`/`less` to locate the next password.
- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Use it interactively any time you build a `grep` or `find` command with flags you're unsure about.
- **[Linux Survival](https://linuxsurvival.com/)** 🟢 — A free, interactive browser terminal with dedicated lessons on viewing and searching files, ideal for risk-free beginner practice.
- **[Regex101](https://regex101.com/)** 🟡🔴 — Not needed for this module's simple literal patterns, but bookmark it now — you'll use it heavily once Module 9 goes deep on `grep` regex.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; the chapters on "Redirection" and "Seeing the World as the Shell Sees It" cover `cat`, `less`, `head`, `tail`, `grep`, and `find` in excellent, example-rich depth.
- **"Efficient Linux at the Command Line" by Daniel J. Barrett** (O'Reilly) 🟡 — A modern, workflow-focused book with strong coverage of searching and filtering files quickly — good once you're comfortable with the basics and want speed habits.

## 👥 Communities

- **[r/linux4noobs](https://www.reddit.com/r/linux4noobs/)** 🟢 — A welcoming home for beginner questions like "why didn't my `grep` pattern match?" — no question is too basic.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Searchable, high-quality Q&A; nearly every "why does `find -name` behave this way" or "`grep` vs `egrep`" question already has a well-explained answer here.
- **[Linux Questions Forums](https://www.linuxquestions.org/)** 🟢🟡 — Long-running, beginner-friendly forum with dedicated sections for shell scripting and command-line troubleshooting.
