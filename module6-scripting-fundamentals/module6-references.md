# 📚 Module 6 References — Bash Scripting Fundamentals

Curated resources for this module's scope: what a script is, shebangs, execution methods, variables and quoting, environment vs. shell variables, command substitution, arithmetic, positional parameters, `read`, and exit codes. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[freeCodeCamp.org — "Bash Scripting Tutorial for Beginners"](https://www.youtube.com/c/Freecodecamp)** 🟢 — A dedicated, long-form crash course that walks through shebangs, variables, arguments, and exit codes with live demos, a natural continuation from earlier modules.
- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Energetic, practical walkthroughs that include real bash-scripting projects, good for seeing scripts applied to actual sysadmin tasks.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡 — Linux-focused channel with several videos specifically on Bash scripting fundamentals and shell customization, aimed at users past the absolute basics.

## 📖 Official Documentation

- **[GNU Bash Manual](https://www.gnu.org/software/bash/manual/bash.html)** 🟢🟡🔴 — The authoritative reference for every feature in this module — shell variables, parameter expansion, `$(...)`, arithmetic expansion, and positional parameters. Bookmark this permanently; you'll return to it constantly.
- **[Bash Reference Manual — Shell Parameters section](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameters)** 🟡 — The specific section covering positional parameters, `$@`, `$*`, `$#`, and special variables like `$?`, `$$`, `$!` in full precise detail.
- **`man bash` / `help read` (local)** 🟢🟡 — Run these directly in your own terminal for exact, always-available documentation on `read`'s flags and Bash's built-in behavior.

## 📝 Tutorials & Articles

- **[Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)** 🟡 — A widely respected, production-grade style guide covering exactly the conventions this module introduces (quoting, `readonly`, exit codes, script headers) — essential reading once you start writing scripts other people will read.
- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — Paste any script in and get instant, specific warnings about unquoted variables, legacy syntax, and common mistakes covered in this module — one of the single highest-value tools for a new script writer.
- **[Bash Guide for Beginners (The Linux Documentation Project)](https://tldp.org/LDP/Bash-Beginners-Guide/html/)** 🟢🟡 — A free, thorough, classic guide covering variables, quoting, and script structure in accessible depth.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste a command or script snippet and see each part visually broken down; useful again here for understanding unfamiliar parameter expansions.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash Scripting curriculum](https://www.freecodecamp.org/news/tag/bash/)** 🟢 — Free, text-based lessons that build directly on the navigation and permissions modules into scripting.
- **[Codecademy — Learn Bash Scripting](https://www.codecademy.com/learn/learn-the-command-line)** 🟢🟡 — Interactive, browser-based practice covering variables, arguments, and control basics.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with a unit specifically on shell scripting fundamentals.
- **[Udemy — "Bash Scripting and Shell Programming (Linux)"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course dedicated entirely to scripting — search the current catalog, as specific course titles/instructors rotate over time.

## 🌐 Websites & Interactive Platforms

- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — (Also listed above.) Use it on every script you write from this module onward — it will catch unquoted variables and other issues in real time.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Continue from earlier modules — several levels require reasoning about scripts, variables, and command substitution to find the next password.
- **[Katacoda-style sandboxes / Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to test destructive or experimental script logic without risking a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; Part II ("Configuration and the Environment") and Part III ("Writing Shell Scripts") map almost exactly onto this module's content, going into excellent additional depth.
- **"Learning the bash Shell" by Cameron Newham (O'Reilly)** 🟡 — A classic, thorough treatment of Bash including variables, quoting, and scripting fundamentals, good for a deeper second pass after this module.
- **"Wicked Cool Shell Scripts" by Dave Taylor & Brandon Perry (No Starch Press)** 🟡🔴 — A large collection of real, practical scripts to read and adapt once you're comfortable with this module's basics — great for seeing these concepts applied.

## 👥 Communities

- **[r/bash](https://www.reddit.com/r/bash/)** 🟢🟡 — Active community specifically for Bash scripting questions, from beginner quoting confusion to advanced scripting patterns.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why do I need to quote variables" or "\$@ vs \$*" already have excellent, heavily upvoted answers here.
- **[Stack Overflow — `bash` tag](https://stackoverflow.com/questions/tagged/bash)** 🟢🟡🔴 — The single largest archive of concrete bash scripting problems and solutions on the internet.
