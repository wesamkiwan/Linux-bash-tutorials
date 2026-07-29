# 📚 Module 5 References — I/O Redirection, Pipes & Filters

Curated resources for this module's scope: standard streams, redirection (`>`, `>>`, `<`, `2>`, `2>&1`), pipes, filter commands (`sort`, `uniq`, `wc`), `tee`, `xargs`, and basic command substitution. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck — "Linux pipes explained"](https://www.youtube.com/@NetworkChuck)** 🟢 — Visual, energetic walkthroughs of pipes and redirection that make the "conveyor belt" mental model click quickly.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Focused, no-nonsense videos covering redirection operators and building real pipelines on Ubuntu/Debian systems.
- **[freeCodeCamp.org — Linux/Bash crash courses](https://www.youtube.com/c/Freecodecamp)** 🟢 — Long-form crash courses that dedicate real time to `|`, `>`, `xargs`, and `tee` with live terminal demos.

## 📖 Official Documentation

- **[GNU Bash Reference Manual — Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections)** 🟡 — The authoritative, exhaustive reference on every redirection operator Bash supports, including the exact processing order that explains the `2>&1` gotcha.
- **[GNU Coreutils Manual — `sort`, `uniq`, `wc`, `tee`](https://www.gnu.org/software/coreutils/manual/coreutils.html)** 🟢🟡 — Official reference for every filter command in this module.
- **[GNU Findutils Manual — `xargs`](https://www.gnu.org/software/findutils/manual/html_mono/find.html)** 🟡 — Official documentation for `xargs`, including `-print0`/`-0` and `-I` in full detail.
- **`man bash` (local, "REDIRECTION" section)** 🟢🟡 — Run `man bash` and search for "REDIRECTION" for the same authoritative detail, available offline anytime.

## 📝 Tutorials & Articles

- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — Paste any pipeline with redirection operators and flags and see each part visually broken down — extremely useful while learning what `2>&1` or `xargs -I {}` actually does.
- **[Linux Journey — "Streams of Text" section](https://linuxjourney.com/)** 🟢 — Free, bite-sized lessons covering redirection and pipes at exactly this module's level.
- **[DigitalOcean — "Using Pipes and Redirection in Linux"](https://www.digitalocean.com/community/tutorials/an-introduction-to-linux-i-o-redirection)** 🟢🟡 — A well-regarded, practical write-up covering the same streams/redirection/pipes ground with server-admin framing.
- **[Baeldung on Linux — `xargs` tutorial](https://www.baeldung.com/linux/xargs-command)** 🟡 — Clear, example-driven coverage of why `xargs` exists and the `-print0`/`-0` safety pattern.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux curriculum](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based lessons continuing directly from earlier modules' resources, including redirection and pipelines.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with a dedicated unit on I/O redirection and pipes.
- **[Codecademy — Learn the Command Line](https://www.codecademy.com/learn/learn-the-command-line)** 🟢 — Interactive browser-based practice covering pipes and redirection hands-on.

## 🌐 Websites & Interactive Platforms

- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Several levels specifically require chaining pipes and filters (`sort`, `uniq`, `grep`) to extract a hidden password — direct, practical reinforcement of this module.
- **[ExplainShell.com](https://explainshell.com/)** 🟢🟡 — (Also listed above.) Use it interactively any time you build a multi-stage pipeline you're not 100% sure about.
- **[Linux Survival](https://linuxsurvival.com/)** 🟢 — A free, interactive browser terminal that includes lessons on redirecting output and chaining commands, ideal for risk-free practice.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; the chapters on redirection and pipelines cover this exact module's content in excellent, example-heavy depth.
- **"How Linux Works" by Brian Ward** (No Starch Press) 🟡🔴 — Goes beneath the commands into how the kernel implements pipes and file descriptors — useful once you want to understand *why* this all works, not just *how* to type it.

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Searchable, high-quality Q&A; the `2>&1` ordering question and `xargs` spaces-in-filenames problem both have excellent, heavily-upvoted answers here already.
- **[r/bash](https://www.reddit.com/r/bash/)** 🟢🟡 — Active community specifically for Bash usage questions, including plenty of real-world pipeline and redirection discussions.
- **[Linux Questions Forums](https://www.linuxquestions.org/)** 🟢🟡 — Long-running, beginner-friendly forum with dedicated sections for shell scripting and command-line questions.
