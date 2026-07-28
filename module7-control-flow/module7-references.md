# 📚 Module 7 References — Control Flow

Curated resources for this module's scope: `if`/`elif`/`else`, `test`/`[ ]`/`[[ ]]`, comparison and logical operators, `case`/`esac`, `for`/`while`/`until` loops, and `break`/`continue`. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[freeCodeCamp.org](https://www.youtube.com/c/Freecodecamp)** 🟢 — Their full Bash scripting course dedicates substantial time to `if`/`case`/loops with live, typed-out demos — a direct continuation from Module 6's material.
- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Practical, energetic videos that use real `if`/loop logic inside actual sysadmin and networking scripts, good for seeing control flow applied rather than taught in the abstract.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡 — Has dedicated videos on Bash conditionals and loop constructs aimed at users past the absolute basics, including `[[ ]]` specifically.

## 📖 Official Documentation

- **[GNU Bash Manual — Conditional Constructs](https://www.gnu.org/software/bash/manual/bash.html#Conditional-Constructs)** 🟡 — The authoritative, precise reference for `if`, `case`, `[[ ]]`, and every comparison operator covered in this module.
- **[GNU Bash Manual — Looping Constructs](https://www.gnu.org/software/bash/manual/bash.html#Looping-Constructs)** 🟡 — Covers `for`, `while`, `until`, and the C-style `for ((...))` form directly from the source.
- **[GNU Bash Manual — Bash Conditional Expressions](https://www.gnu.org/software/bash/manual/bash.html#Bash-Conditional-Expressions)** 🟡🔴 — The exhaustive list of every file test, string, and comparison operator `[[ ]]` supports, including ones this module doesn't cover in depth.
- **`man test` / `help [[` (local)** 🟢🟡 — Run these directly in your own terminal for exact, always-available documentation on operator syntax.

## 📝 Tutorials & Articles

- **[Google Shell Style Guide — Loops and Conditionals sections](https://google.github.io/styleguide/shellguide.html)** 🟡 — Production-grade conventions for exactly this module's content, including the explicit recommendation to prefer `[[ ]]` over `[ ]`.
- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — Paste any script with conditionals or loops in and get instant warnings about missing quotes, `[ ]` pitfalls, and unreachable `case` patterns — extremely high-value while practicing this module.
- **[Bash Guide for Beginners (The Linux Documentation Project) — Chapters on Conditionals & Loops](https://tldp.org/LDP/Bash-Beginners-Guide/html/)** 🟢🟡 — Free, thorough, classic coverage of exactly this module's syntax with extra worked examples.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `if`/`case`/loop line and see each token visually broken down — useful for double-checking unfamiliar operator combinations.
- **["Comparing Files/Strings/Numbers in Bash" — Bash Hackers Wiki (archived mirrors)](https://web.archive.org/web/2023/https://wiki.bash-hackers.org/syntax/ccmd/classictest)** 🟡 — A detailed comparison of `[ ]` vs `[[ ]]` behavior, side by side, from a resource long trusted in the Bash community.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash Scripting curriculum](https://www.freecodecamp.org/news/tag/bash/)** 🟢 — Free, text-based lessons continuing directly from Module 6 into conditionals and loops.
- **[Codecademy — Learn Bash Scripting](https://www.codecademy.com/learn/learn-the-command-line)** 🟢🟡 — Interactive, browser-based practice with dedicated exercises on `if` statements and loops.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course covering shell scripting control structures as part of its broader curriculum.
- **[Udemy — "Bash Scripting and Shell Programming (Linux)"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course with dedicated sections on conditionals, `case`, and loops — search the current catalog, as specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — (Also listed above.) Run every script from this module through it — it will flag `[ ]` quoting issues and missing `case` catch-alls in real time.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟢🟡 — Several levels require writing small scripts with conditionals and loops to find the next password — excellent low-stakes practice.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to intentionally trigger an infinite loop or a broken `[ ]` condition without risking a real machine.
- **[HackerRank — Shell scripting track](https://www.hackerrank.com/domains/shell)** 🟡 — Free practice problems specifically exercising conditionals and loops with automated grading.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; the "Writing Shell Scripts" chapters cover `if`, `case`, and all three loop types in detail, matching this module closely.
- **"Learning the bash Shell" by Cameron Newham (O'Reilly)** 🟡 — A classic, thorough treatment of Bash conditionals and loops, good for a deeper second pass after this module.
- **"Wicked Cool Shell Scripts" by Dave Taylor & Brandon Perry (No Starch Press)** 🟡🔴 — Real, practical scripts to read once you're comfortable with this module's basics — nearly every script in the book leans heavily on `if`/`case`/loops.

## 👥 Communities

- **[r/bash](https://www.reddit.com/r/bash/)** 🟢🟡 — Active community for Bash-specific questions, including frequent threads on `[ ]` vs `[[ ]]` and loop debugging.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Searchable archive with excellent, heavily-upvoted answers to questions like "why does my `case` statement not match" or "why is my `while` loop infinite."
- **[Stack Overflow — `bash` tag](https://stackoverflow.com/questions/tagged/bash)** 🟢🟡🔴 — The largest archive of concrete control-flow bugs and fixes on the internet — search any error message verbatim.
