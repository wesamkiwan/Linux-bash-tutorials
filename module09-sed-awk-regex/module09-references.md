# 📚 Module 9 References — sed, awk & Regex

Curated resources for this module's scope: regex fundamentals (BRE vs. ERE, anchors, character classes, quantifiers, grouping/alternation, greedy matching), `grep -E`/`grep -P`, `sed` (substitution, flags, in-place editing, addresses, capture groups), `awk` (fields, `BEGIN`/`END`, patterns, aggregation), and combined pipelines. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[freeCodeCamp.org — "Sed and Awk 101 Hacks" / Linux text processing playlists](https://www.youtube.com/c/Freecodecamp)** 🟡 — Long-form, hands-on walkthroughs of sed and awk fundamentals with live terminal demos, a natural continuation from this course's earlier modules.
- **[DistroTube — sed/awk and regex videos](https://www.youtube.com/@DistroTube)** 🟡 — Focused Linux-power-user content that covers sed/awk one-liners and regex in a practical, terminal-first style.
- **[Corey Schafer — Regular Expressions videos](https://www.youtube.com/@coreyms)** 🟢🟡 — Clear, patient regex explanations; even though some are framed around Python, the regex concepts (anchors, classes, quantifiers, greediness) transfer directly.

## 📖 Official Documentation

- **[GNU sed Manual](https://www.gnu.org/software/sed/manual/sed.html)** 🟡🔴 — The authoritative reference for every sed feature in this module — substitution flags, addressing, `-i`, and beyond. The canonical source once you outgrow the cheat sheet.
- **[GNU awk (gawk) User's Guide](https://www.gnu.org/software/gawk/manual/gawk.html)** 🟡🔴 — The full, definitive awk reference — built-in variables, functions, and advanced features far beyond this module's scope.
- **[GNU grep Manual](https://www.gnu.org/software/grep/manual/grep.html)** 🟡 — Covers `-E`, `-P`, and every other flag precisely, including the exact regex dialects grep supports.
- **`man 7 regex` (local)** 🟡🔴 — Run this directly in your Ubuntu terminal — the POSIX regex man page covering BRE/ERE syntax formally and precisely.
- **`man sed`, `man awk`, `man grep` (local)** 🟢🟡 — Always-available, exact, version-matched documentation for whatever's actually installed on your machine.

## 📝 Tutorials & Articles

- **[regex101.com](https://regex101.com/)** 🟢🟡 — An interactive regex tester with a live explanation panel for every part of your pattern; set the flavor to "PCRE" or similar for closest behavior to `grep -P`, and note that POSIX BRE/ERE (plain grep/sed) behaves slightly differently in edge cases — invaluable for building and debugging patterns before running them on real data.
- **[Grymoire's sed & awk tutorials](https://www.grymoire.com/Unix/Sed.html)** 🟡🔴 — A long-standing, extremely thorough, free reference covering sed (and a companion awk guide) from basics through advanced scripting — one of the internet's most respected deep-dive resources on this exact topic.
- **[ShellCheck.net](https://www.shellcheck.net/)** 🟡 — Still useful here: paste a script using sed/awk/grep and catch common quoting or portability issues in the surrounding shell code.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste a sed/awk/grep one-liner and see each flag and argument broken down individually — great for reverse-engineering someone else's dense one-liner.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Linux text-processing curriculum](https://www.freecodecamp.org/news/tag/linux/)** 🟢🟡 — Free, text-based lessons continuing directly from earlier course modules into sed/awk/regex territory.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟡 — Free, self-paced course with units touching on text processing and shell tool usage.
- **[Udemy — "Sed and Awk" / regex-focused courses](https://www.udemy.com/)** 🟡🔴 — Several affordable, dedicated courses exist specifically on sed/awk and regex mastery — search the current catalog, as exact titles and instructors rotate over time.

## 🌐 Websites & Interactive Platforms

- **[regex101.com](https://regex101.com/)** 🟢🟡 — (Also listed above.) The single best place to build and test a pattern before pasting it into a live sed/grep/awk command.
- **[sed.js — Online sed playground](https://sed.js.org/)** 🟡 — A browser-based sed simulator for experimenting with substitutions and flags without touching a real file.
- **[Regex Crossword](https://regexcrossword.com/)** 🟢🟡 — A genuinely fun way to drill character classes, quantifiers, and anchors through puzzle-solving.
- **[OverTheWire: Bandit](https://overthewire.org/wargames/bandit/)** 🟡 — Continue from earlier modules — several later levels specifically require sed/awk/grep regex reasoning to extract the next password.

## 📚 Books

- **["sed & awk" by Dale Dougherty & Arnold Robbins (O'Reilly)](https://www.oreilly.com/library/view/sed-awk/1565922255/)** 🟡🔴 — The classic, definitive book on exactly this module's topic — still the gold standard reference decades after publication for anyone who wants to go deep on both tools.
- **["The AWK Programming Language" by Aho, Kernighan & Weinberger](https://www.oreilly.com/library/view/the-awk-programming/9780138269722/)** 🔴 — Written by awk's own creators; the authoritative, in-depth treatment of the language straight from the source, including material well beyond this module's scope.
- **["Mastering Regular Expressions" by Jeffrey Friedl (O'Reilly)](https://www.oreilly.com/library/view/mastering-regular-expressions/0596528124/)** 🔴 — A famously thorough deep-dive into regex engines and theory across many tools and languages; more depth than this module needs, but the definitive reference once you want to understand *why* regex engines behave the way they do.

## 👥 Communities

- **[r/bash](https://www.reddit.com/r/bash/)** 🟢🟡 — Active community for exactly the kind of sed/awk/regex one-liner questions this module covers.
- **[r/commandline](https://www.reddit.com/r/commandline/)** 🟡 — Broader command-line community where clever grep/sed/awk pipelines get shared and dissected regularly.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable archive; questions like "BRE vs ERE," "sed capture groups," or "awk NR vs FNR" already have excellent, heavily upvoted answers.
- **[Stack Overflow — `sed` / `awk` / `regex` tags](https://stackoverflow.com/questions/tagged/sed)** 🟢🟡🔴 — The largest searchable archive of concrete sed, awk, and regex problems and solutions on the internet.
