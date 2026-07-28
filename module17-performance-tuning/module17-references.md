# 📚 Module 17 References — Performance Tuning & Profiling Scripts

Curated resources for this module's scope: the `time` command (`real`/`user`/`sys`, `TIMEFORMAT`), section timing (`date +%s%N`, `$SECONDS`), the "Useless Use of Cat" and per-iteration-process anti-patterns, builtins vs. external commands, single-pass `awk`/`sed` rewrites, `xargs -P`, GNU `parallel`, `/usr/bin/time -v`, and `strace -c`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢🟡 — Practical, energetic walkthroughs of Linux performance and scripting topics, framed around real sysadmin tasks.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡🔴 — Dedicated Bash scripting videos covering efficiency patterns and shell internals for viewers past the basics.
- **[Luke Smith](https://www.youtube.com/@LukeSmithxyz)** 🟡 — Practical shell-scripting content, including efficient text-processing habits with `awk`/`sed`.

## 📖 Official Documentation

- **[GNU Bash Reference Manual — "Bash Builtin Commands"](https://www.gnu.org/software/bash/manual/bash.html#Bash-Builtins)** 🟡 — The authoritative list of every Bash builtin, confirming which operations avoid process-spawning entirely.
- **[GNU Coreutils Manual — `time`](https://www.gnu.org/software/coreutils/manual/html_node/time-invocation.html)** 🟡 — Reference for the standalone `time` utility and its options.
- **[GNU `parallel` Official Documentation](https://www.gnu.org/software/parallel/)** 🟢🟡 — The authoritative source for GNU `parallel`, including the official tutorial and man pages.
- **[GNU `findutils` Manual — `xargs`](https://www.gnu.org/software/findutils/manual/html_node/find_html/xargs-options.html)** 🟡 — Full option reference for `xargs`, including `-P` and `-I`.
- **`man time`, `man awk`, `man strace`** (local, run in your own terminal) 🟡🔴 — The exact behavior and every flag on your installed system's versions of these tools.

## 📝 Tutorials & Articles

- **["Useless Use of Cat" Award page](http://porkmail.org/era/unix/award.html)** 🟢🟡 — The original, widely-cited page that named and popularized the UUOC anti-pattern.
- **[Bash Hackers Wiki — Efficiency/Performance](https://web.archive.org/web/2021*/https://wiki.bash-hackers.org/scripting/performance)** 🟡🔴 — Deep-dive coverage of Bash performance characteristics, builtins vs. external commands, and where the real costs hide (archived; the original wiki has gone offline at times, hence the archive link).
- **[BashPitfalls — Greg's Wiki](https://mywiki.wooledge.org/BashPitfalls)** 🟡🔴 — A large catalog of common Bash mistakes, several directly related to unnecessary subshells and process spawning.
- **[GNU Parallel vs. `xargs` — comparison examples (GNU project)](https://www.gnu.org/software/parallel/parallel_alternatives.html)** 🟡 — A direct, official side-by-side of `parallel` against `xargs` and other alternatives.
- **[Julia Evans — "strace zine / articles"](https://jvns.ca/blog/2015/04/14/strace-zine/)** 🟡🔴 — An approachable, well-illustrated introduction to `strace` for readers who've never used a syscall tracer before.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash scripting content](https://www.freecodecamp.org/news/tag/bash/)** 🟢🟡 — Free, text-based articles that build toward efficient, production-grade scripting habits.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟡 — Free, self-paced course touching on system resource usage and performance fundamentals.
- **[Udemy — "Linux Performance Tuning" / "Advanced Bash Scripting"](https://www.udemy.com/)** 🟡🔴 — Popular, affordable paid courses with dedicated sections on profiling and optimization; search the current catalog since specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `awk`, `xargs`, or `time`-formatted command and see every part broken down individually.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to generate large test files and practice benchmarking without risking a real machine.
- **[GNU Parallel's own interactive tutorial](https://www.gnu.org/software/parallel/parallel_tutorial.html)** 🟡 — Official, hands-on walkthrough covering real parallelization scenarios step by step.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its scripting chapters build directly toward the efficiency habits this module covers.
- **"Efficient Linux at the Command Line" by Daniel J. Barrett (O'Reilly)** 🟡🔴 — A book specifically focused on working efficiently at the shell, including avoiding wasteful command patterns.
- **"Systems Performance" by Brendan Gregg (Addison-Wesley/Pearson)** 🔴 — The definitive, deep reference on performance analysis and profiling methodology on Linux, useful once you want to go far beyond script-level tuning into full system profiling (`strace`, and much more).

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why is my bash loop so slow" or "xargs -P vs parallel" already have excellent, heavily upvoted answers here.
- **[r/bash](https://www.reddit.com/r/bash/)** 🟡🔴 — Active community for Bash scripting questions, including real-world performance war stories and benchmark comparisons.
- **[GNU Parallel Mailing List / Google Group](https://lists.gnu.org/mailman/listinfo/parallel)** 🟡🔴 — The project's own community, useful for questions specific to complex parallel job configurations.
