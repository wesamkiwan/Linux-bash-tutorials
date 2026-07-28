# 📚 Module 14 References — Error Handling, Traps & Debugging

Curated resources for this module's scope: `set -e`/`set -u`/`set -o pipefail`, the `set -euo pipefail` idiom, `set -x`/`PS4` tracing, `trap` (`EXIT`/`ERR`/`INT`), the `die()` pattern, `bash -n`, and `shellcheck`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢🟡 — Practical, energetic walkthroughs of Bash scripting hardening topics, framed around real sysadmin/DevOps tasks.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡🔴 — Dedicated videos on Bash scripting best practices, `trap`, and error handling for viewers past the absolute basics.
- **[Luke Smith](https://www.youtube.com/@LukeSmithxyz)** 🟡 — Practical shell-scripting content with a focus on idiomatic, defensive Bash.

## 📖 Official Documentation

- **[Bash Reference Manual — "The Set Builtin"](https://www.gnu.org/software/bash/manual/bash.html#The-Set-Builtin)** 🟡 — The authoritative, exact specification of every `set` option including `-e`, `-u`, and `pipefail`, straight from GNU.
- **[Bash Reference Manual — "Bourne Shell Builtins" (`trap`)](https://www.gnu.org/software/bash/manual/bash.html#Bourne-Shell-Builtins)** 🟡 — The precise, authoritative behavior of `trap`, including which pseudo-signals like `EXIT` and `ERR` are supported.
- **[ShellCheck — official wiki (every warning code explained)](https://github.com/koalaman/shellcheck/wiki)** 🟢🟡🔴 — The canonical reference for every ShellCheck code (SC1000-SCxxxx), each with a rationale and a fixed example.
- **`man bash`** (local, run `man bash` in your own terminal) 🟡🔴 — Search for `set -e`, `trap`, and `PS4` directly in your installed Bash's own manual for the exact behavior on your system/version.

## 📝 Tutorials & Articles

- **["Use the Unofficial Bash Strict Mode (Unless You Looove Debugging)" by Aaron Maxwell](http://redsymbol.net/articles/unofficial-bash-strict-mode/)** 🟡 — The widely-cited article that popularized the `set -euo pipefail` idiom, explaining the reasoning and caveats in depth.
- **[BashPitfalls — Greg's Wiki](https://mywiki.wooledge.org/BashPitfalls)** 🟡🔴 — An enormous, battle-tested catalog of common Bash mistakes (many directly related to `set -e`'s gotchas and quoting) with corrected versions of each.
- **[Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)** 🟡🔴 — A widely-referenced professional style guide covering error handling conventions, `set -e` usage recommendations, and general defensive-scripting practices used at scale.
- **[DigitalOcean — "Using Trap to Exit Bash Scripts Cleanly"](https://www.digitalocean.com/community/tutorials/using-trap-to-exit-bash-scripts-cleanly)** 🟡 — A focused, example-driven walkthrough of `trap EXIT`/`ERR` cleanup patterns.
- **[ShellCheck.net](https://www.shellcheck.net/)** 🟢🟡 — Paste any script directly into the browser and get instant `shellcheck` output with no installation — the fastest way to try it before installing locally.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash scripting content](https://www.freecodecamp.org/news/tag/bash/)** 🟢🟡 — Free, text-based articles that build directly toward defensive, production-grade scripting habits.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟡 — Free, self-paced course with material touching on scripting robustness and system reliability practices.
- **[Udemy — "Bash Scripting and Shell Programming"](https://www.udemy.com/)** 🟡🔴 — A popular, affordable paid course with dedicated sections on error handling and traps; search the current catalog since specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[ShellCheck.net](https://www.shellcheck.net/)** 🟢🟡 — (Also listed above.) The fastest way to check a snippet without installing anything.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste a `trap`, `set`, or `shellcheck`-flagged line and see every part broken down individually.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice hardening scripts and deliberately triggering failures without risking a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its chapters on scripting build directly toward the error-handling discipline this module covers.
- **"Pro Bash Programming" by Chris Johnson & Jayant Varma (Apress)** 🟡🔴 — Covers `trap`, signal handling, and defensive scripting patterns in more depth than an introductory text.
- **"The Linux Programming Interface" by Michael Kerrisk (No Starch Press)** 🔴 — The definitive, exhaustive reference on signals and process behavior at the systems-programming level, useful once you want to understand exactly *why* `SIGKILL` can't be trapped, from the kernel's perspective.

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why doesn't set -e catch this" or "trap not firing" already have excellent, heavily upvoted answers here.
- **[r/bash](https://www.reddit.com/r/bash/)** 🟡🔴 — Active community specifically for Bash scripting questions, including real production war stories about error handling gone wrong.
- **[ShellCheck GitHub Issues/Discussions](https://github.com/koalaman/shellcheck)** 🟡🔴 — The project's own repository; useful for understanding the reasoning behind specific warnings or reporting an edge case.
