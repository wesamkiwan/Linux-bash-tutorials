# 📚 Module 8 References — Functions, Arrays & String Manipulation

Curated resources for this module's scope: defining and calling functions, `local` scope, `return` vs. `echo`+`$()`, recursion, indexed arrays, associative arrays, and parameter-expansion string manipulation. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[freeCodeCamp.org — Bash Scripting Tutorial for Beginners](https://www.youtube.com/c/Freecodecamp)** 🟡 — Covers functions and arrays as part of its broader scripting curriculum, a natural continuation from Module 6's coverage.
- **[DistroTube — Bash scripting playlist](https://www.youtube.com/@DistroTube)** 🟡 — Includes dedicated videos on Bash arrays and functions aimed at users past the absolute basics, with practical, real-terminal demonstrations.
- **[Learn Linux TV](https://www.youtube.com/@LearnLinuxTV)** 🟡 — Practical Linux administration content that frequently shows functions and arrays used in real automation scripts, not just toy examples.

## 📖 Official Documentation

- **[GNU Bash Manual — Shell Parameters (Arrays)](https://www.gnu.org/software/bash/manual/bash.html#Arrays)** 🟡 — The authoritative reference for indexed and associative array syntax, including edge cases this module doesn't have room to cover.
- **[GNU Bash Manual — Shell Parameter Expansion](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion)** 🟡🔴 — The single most important reference link in this module: the exhaustive, authoritative list of every `${...}` string-manipulation form, including a few rare ones beyond this module's scope. Bookmark this permanently.
- **[GNU Bash Manual — Shell Functions](https://www.gnu.org/software/bash/manual/bash.html#Shell-Functions)** 🟡 — Precise, official semantics for function definition, `local`, and `return`.
- **`help local`, `help return`, `help declare` (local)** 🟢🟡 — Run these directly in your terminal for exact, version-matched documentation on these built-ins.

## 📝 Tutorials & Articles

- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — Paste in a script using functions and arrays and get instant warnings about missing `local`, unquoted array expansions, and other issues this module covers — indispensable from this point forward.
- **[Bash Hackers Wiki — Arrays](https://web.archive.org/web/2023*/https://wiki.bash-hackers.org/syntax/arrays)** 🟡 — A frequently-cited, example-heavy community reference specifically on Bash array mechanics (archived link — the original site has had uptime issues; search "bash hackers wiki arrays" if this doesn't resolve).
- **[Google Shell Style Guide — Functions section](https://google.github.io/styleguide/shellguide.html#s5-functions-and-declarations)** 🟡 — Production-grade conventions for function naming, structure, and `local` usage from a widely respected style guide.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste a parameter-expansion snippet like `${file%%.*}` and see it broken down visually — very useful while you're still building intuition for `#`/`##`/`%`/`%%`.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash Scripting curriculum](https://www.freecodecamp.org/news/tag/bash/)** 🟢🟡 — Free, text-based lessons that build on earlier modules into functions and data structures.
- **[Codecademy — Learn Bash Scripting](https://www.codecademy.com/learn/learn-the-command-line)** 🟡 — Interactive, browser-based practice that includes functions and basic array usage.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟡 — Free, self-paced course covering scripting fundamentals including functions.
- **[Udemy — "Bash Scripting and Shell Programming (Linux)"](https://www.udemy.com/)** 🟡 — A popular, affordable paid course with dedicated sections on functions and arrays — search the current catalog, as specific course titles/instructors rotate over time.

## 🌐 Websites & Interactive Platforms

- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — (Also listed above.) Run every function-and-array-heavy script through it — it's excellent at catching the exact pitfalls covered in this module.
- **[Killercoda — Linux/Bash scenarios](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to experiment with `declare -A` and array slicing without risking a real machine.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟡 — Later levels sometimes require reading scripts that use functions and parameter expansion to find the next password — good applied practice.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; the "Writing Shell Scripts" chapters cover functions and arrays with the same plain-language approach as this module.
- **"Learning the bash Shell" by Cameron Newham (O'Reilly)** 🟡 — Thorough coverage of Bash functions, arrays, and parameter expansion, good for a deeper second pass.
- **"Wicked Cool Shell Scripts" by Dave Taylor & Brandon Perry (No Starch Press)** 🟡🔴 — A large collection of real scripts that make heavy, practical use of functions and string manipulation — excellent for seeing these concepts applied to genuine tasks.
- **"Pro Bash Programming" / "Beginning the Linux Command Line" by James Kerr** 🟡🔴 — Deeper treatments of Bash scripting patterns including function design and data structures, for readers wanting to go further after this module.

## 👥 Communities

- **[r/bash](https://www.reddit.com/r/bash/)** 🟢🟡🔴 — Active community for Bash-specific questions, including plenty of threads on the exact `return`-string trap and array-quoting gotchas covered here.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟡🔴 — Enormous, searchable archive; questions like "how to return a string from a bash function" or "bash associative array vs indexed array" already have excellent, heavily upvoted answers.
- **[Stack Overflow — `bash` tag](https://stackoverflow.com/questions/tagged/bash)** 🟡🔴 — The largest archive of concrete Bash problems and solutions, including deep dives on every parameter-expansion operator in this module's cheat sheet.
