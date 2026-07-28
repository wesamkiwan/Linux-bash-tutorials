# 📚 Module 13 References — Terminal Productivity (tmux & Dotfiles)

Curated resources for this module's scope: terminal multiplexers, `tmux` sessions/windows/panes, `screen` as the legacy alternative, `.bashrc` vs. `.bash_profile`/`.profile`, aliases, `PS1` prompt customization, and history tuning. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[The Primeagen — tmux videos](https://www.youtube.com/@ThePrimeagen)** 🟡 — Fast-paced, practical `tmux` workflows from a developer who lives inside it daily; good for seeing real muscle-memory usage beyond the basics.
- **[DistroTube — tmux and dotfiles content](https://www.youtube.com/@DistroTube)** 🟡 — Dedicated videos on `tmux` configuration and managing dotfiles, aimed at users past absolute basics.
- **[LearnLinuxTV](https://www.youtube.com/@LearnLinuxTV)** 🟢🟡 — Clear, methodical Ubuntu/Debian-focused walkthroughs, including shell customization basics that build directly on this module.

## 📖 Official Documentation

- **[tmux GitHub Wiki](https://github.com/tmux/tmux/wiki)** 🟡 — The official `tmux` project wiki, including a getting-started guide and links to further docs.
- **`man tmux` (local)** 🟡🔴 — The complete, authoritative reference for every `tmux` command and configuration option on your exact installed version; run it directly in your terminal.
- **[GNU Bash Reference Manual — "Bash Startup Files"](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)** 🟡 — The definitive, precise explanation of exactly which file Bash reads in which situation (login vs. non-login, interactive vs. non-interactive) — the ground truth behind the `.bashrc`/`.bash_profile` distinction taught in this module.
- **[GNU Bash Reference Manual — "Controlling the Prompt"](https://www.gnu.org/software/bash/manual/html_node/Controlling-the-Prompt.html)** 🟡 — The full, official list of every `PS1` backslash-escape code, beyond the handful covered in this module.
- **[GNU Bash Reference Manual — "Bash History Facilities"](https://www.gnu.org/software/bash/manual/html_node/Bash-History-Facilities.html)** 🟡 — Authoritative reference for `HISTSIZE`, `HISTFILESIZE`, `HISTCONTROL`, and related history variables.
- **[GNU `screen` manual](https://www.gnu.org/software/screen/manual/screen.html)** 🟢 — Official documentation for `screen`, useful if you ever need to work on a system where only it is installed.

## 📝 Tutorials & Articles

- **[DigitalOcean — "How To Use tmux on Ubuntu"](https://www.digitalocean.com/community/tutorials/tmux-open-source-tool-for-terminal-a)** 🟢🟡 — A clear, example-driven `tmux` walkthrough on Ubuntu, covering sessions, windows, and panes in the same order as this module.
- **[Red Hat Sysadmin — "How to use tmux to enhance your development environment"](https://www.redhat.com/sysadmin)** 🟡 — Practical framing of `tmux` for real day-to-day development and operations work.
- **[dotfiles.github.io](https://dotfiles.github.io/)** 🟢🟡 — A community-curated hub explaining what dotfiles are, why (and how) people keep them in git, and links to hundreds of real example dotfiles repositories for inspiration.
- **[Baeldung on Linux — "The Difference Between .bashrc and .bash_profile"](https://www.baeldung.com/linux/bashrc-vs-bash-profile)** 🟡 — A focused article walking through exactly the login-vs-non-login distinction this module covers, with additional edge cases.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any `alias`, `shopt`, or `tmux` invocation and see every flag broken down individually.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — tmux and Linux customization articles](https://www.freecodecamp.org/news/tag/linux/)** 🟢 — Free, text-based articles covering `tmux` fundamentals and shell customization.
- **[Linux Foundation — Introduction to Linux (LFS101x on edX)](https://www.edx.org/)** 🟢🟡 — Free, self-paced course with units touching shell configuration and terminal tooling.
- **[Udemy — "tmux: The Terminal Multiplexer" / general Linux productivity courses](https://www.udemy.com/)** 🟡 — Affordable paid courses dedicated specifically to `tmux`; search the current catalog since specific titles rotate.

## 🌐 Websites & Interactive Platforms

- **[tmuxcheatsheet.com](https://tmuxcheatsheet.com/)** 🟢🟡 — A dense, single-page visual cheat sheet of nearly every `tmux` keybinding — excellent as a bookmark for quick lookups on the job.
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to practice `tmux` sessions and dotfile edits without risking a real machine.
- **[dotfiles.github.io](https://dotfiles.github.io/)** 🟢🟡 — (Also listed above.) Doubles as a browsable directory of real-world example `.bashrc`/dotfiles setups from experienced engineers.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its chapters on customizing the shell environment and prompt map directly onto this module.
- **"tmux 2: Productive Mouse-Free Development" by Brian P. Hogan (Pragmatic Bookshelf)** 🟡🔴 — A short, focused, and widely recommended book dedicated entirely to `tmux`, going well beyond this module's basics into scripting and advanced layouts.
- **"How Linux Works" by Brian Ward (No Starch Press)** 🟡 — Covers shell startup behavior and environment configuration in more depth, for a strong second pass after this module.

## 👥 Communities

- **[r/tmux](https://www.reddit.com/r/tmux/)** 🟡 — Community focused specifically on `tmux` configurations, plugins, and workflows.
- **[r/unixporn](https://www.reddit.com/r/unixporn/)** 🟡 — Terminal and dotfiles customization showcases; a great source of inspiration for prompt and `tmux` configuration ideas, even if the aesthetics go far beyond this module's scope.
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "bashrc vs bash_profile" or "tmux detach not working" already have excellent, heavily upvoted answers here.
- **[Stack Overflow — `tmux` and `bashrc` tags](https://stackoverflow.com/questions/tagged/tmux)** 🟢🟡🔴 — Large archive of concrete `tmux` and shell-configuration problems and solutions.
