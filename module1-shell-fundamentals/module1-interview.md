# 🎤 Module 1 Interview Prep — Linux & Shell Fundamentals

## Conceptual Questions

### 🟢 Beginner

**Q1: What's the difference between Linux, a distro, and the kernel?**

> "Linux, strictly speaking, is just the kernel — the core piece of software that manages hardware: CPU scheduling, memory, disk I/O. On its own, a kernel isn't something you can install and use. A distro, like Ubuntu or Debian, takes that kernel and bundles it with a package manager, system tools, and default configuration so you get a complete, installable operating system. When people say 'I use Linux,' they really mean they use a Linux distro."

**Q2: What is a shell, and what is Bash specifically?**

> "A shell is a program that sits between the user and the kernel — it reads the commands you type, interprets them, and asks the operating system to carry them out. Bash, which stands for 'Bourne Again SHell,' is the most common shell implementation and is the default on Ubuntu, Debian, and most Linux distributions. It's what almost every server, CI/CD pipeline, and Docker image assumes you're using, which is why it's the standard to learn first."

**Q3: What's the difference between a terminal and a shell?**

> "The terminal, or terminal emulator, is just the application window — it displays text and captures your keystrokes, but it doesn't understand commands at all. The shell is the program running inside that window, like Bash, that actually interprets what you type and executes it. You could open different terminal emulators — GNOME Terminal, iTerm2, Windows Terminal — and still be running the exact same Bash shell inside any of them."

**Q4: How do you read a prompt like `user@host:~$`?**

> "`user` is the account I'm logged in as, `host` is the machine's hostname, and `~` is my current working directory — the tilde is shorthand for my home directory. The trailing symbol tells me my privilege level: a `$` means I'm a regular user, while a `#` would mean I'm logged in as root, the superuser with unrestricted access."

**Q5: What's the structure of a typical Linux command?**

> "Most commands follow the pattern `command [options] [arguments]`. The command is the program to run, options modify how it behaves, and arguments are what it acts on — usually a file or directory. Options come in short form, like `-l`, using a single dash and letter, or long form, like `--long`, using a double dash and a full word. Short options can often be combined, so `-la` is the same as `-l -a`."

### 🟡 Intermediate

**Q6: When would you use `man` versus `--help` versus `tldr`?**

> "I reach for `--help` first when I just need a quick reminder of a command's flags — it's fast and stays right in my terminal. If I need the full, authoritative detail — edge cases, exit codes, every option — I go to `man`, which is the complete manual page. `tldr` is a newer, community-maintained tool that skips the formal documentation style and just shows practical, real-world examples, which is great when I want to see a command used correctly rather than read a spec."

**Q7: What's the practical difference between `clear` and `exit`?**

> "`clear` only wipes what's visible on the screen — your shell session, environment variables, and command history are completely untouched, it's purely cosmetic. `exit` actually terminates the current shell session. If it's a local terminal, the window may close; if I'm connected over SSH, `exit` logs me out of the remote machine entirely."

**Q8: Why does this course focus specifically on Bash rather than zsh or sh?**

> "Because Bash is the one shell you're guaranteed to find almost everywhere in production — it's the default on Ubuntu and Debian, it's what most Docker base images ship with, and it's the assumed interpreter for the vast majority of CI/CD scripts. zsh is popular for interactive daily use, especially on macOS, and sh — which on Ubuntu is actually a lightweight shell called dash — is used for maximum portability. But the core concepts, like variables and control flow, transfer between all of them, so mastering Bash first gives the best return on investment."

### 🔴 Advanced

**Q9: If a script needs to run on multiple Linux distros, why might `#!/bin/sh` behave differently than `#!/bin/bash`?**

> "Because `/bin/sh` isn't guaranteed to be Bash — on Ubuntu and Debian it's actually symlinked to `dash`, a minimal POSIX-compliant shell that doesn't support several Bash-only features like arrays, `[[ ]]` conditionals, or certain string manipulation syntax. A script that uses Bash-specific syntax but declares `#!/bin/sh` as its shebang can fail or behave subtly differently depending on the distro, because it's actually being executed by dash instead of Bash. That's why it matters to match the shebang to the actual features you're using."

**Q10: What's actually happening when you press `Ctrl+R` and Bash finds a matching command?**

> "Bash keeps an in-memory (and eventually on-disk, in `~/.bash_history`) history of commands you've entered. `Ctrl+R` triggers an incremental reverse search through that history buffer — as I type characters, Bash searches backward for the most recent line containing that substring and displays it live, without executing it. That's an important distinction from the up arrow, which just steps through history sequentially — reverse search does an actual substring match, which scales much better once your history has hundreds of entries."

---

## Practical/Coding Questions

**Q1: Show me how you'd find out what user you are and what directory you're in, and prove it.**

Solution:
```bash
whoami
pwd
```
Explanation: `whoami` prints the logged-in username with no options needed; `pwd` prints the absolute path of the current working directory. Together they answer "who am I and where am I," which is the first thing any experienced Linux user checks when they land on an unfamiliar system.

**Q2: You want to know what the `-a` flag does for the `ls` command, but you're not sure if it's already installed or documented. Walk through how you'd find out, in order.**

Solution:
```bash
ls --help
man ls
tldr ls
```
Explanation: I'd start with `ls --help` since it's the fastest way to see a flag summary right in the terminal. If I need more depth — for instance, exact behavior around dotfiles — I'd check `man ls` for the full description. If I just want to see it used in a realistic example, `tldr ls` shows practical usage patterns.

**Q3: You just ran a long command five minutes ago and don't want to retype it. Show the exact keystrokes/commands.**

Solution: Press `Ctrl+R`, type a distinctive fragment of the command (e.g., a few characters from the middle of it), then press `Enter` to execute the match Bash finds, or press the right arrow key to drop into edit mode first if you want to tweak it before running.
Explanation: This demonstrates reverse-incremental search, which is dramatically faster than scrolling with the up arrow through dozens of history entries, especially in a long session.

**Q4: How would you confirm your current shell is actually Bash?**

Solution:
```bash
echo $0
bash --version
```
Explanation: `echo $0` prints the name of the currently running shell process (it should show `bash` or `-bash`). `bash --version` confirms Bash is installed and shows the version, which matters since some Bash features (like associative arrays) require Bash 4+.

---

## Gotcha Questions

**Q1: "I typed `man ls` and now my terminal is frozen and won't accept any commands — what's wrong?"**

> Trap: The candidate might think something crashed. In reality, `man` opens the manual inside a pager (`less`), which captures the full screen and waits for pager-specific keys. Nothing is broken — the fix is simply pressing `q` to quit back to the shell prompt.

**Q2: "Why did `LS -L` fail with 'command not found' when `ls -l` works fine?"**

> Trap: Linux commands and filenames are case-sensitive. `LS` is not the same as `ls` — there is no command literally named `LS`, so Bash reports it as not found. This trips up people coming from Windows, where the filesystem is case-insensitive.

**Q3: "The prompt suddenly shows `#` instead of `$` — is that a problem?"**

> Trap: It signals the user is now operating as root, the superuser with no built-in guardrails. It isn't inherently a "bug," but it should raise a flag — every command run in that state has full power to modify or delete anything on the system, so the candidate should stop and confirm whether escalating to root was actually intended before proceeding.

**Q4: "`clear` didn't actually delete anything, so why can't I find that command I ran ten minutes ago anymore?"**

> Trap: `clear` only wipes the visible screen — it has nothing to do with the command history. If a command seems to have vanished, the issue is elsewhere (e.g., a new shell session was started, since in-memory history is per-session until it's written to `~/.bash_history` on exit). `history` and `Ctrl+R` should still find it within the same session.

---

## Quick-Fire Rapid Review

- **Q: What does `pwd` stand for?** A: Print Working Directory.
- **Q: What key exits `man`?** A: `q`.
- **Q: `$` vs `#` in the prompt?** A: `$` = regular user, `#` = root.
- **Q: What does `~` represent?** A: The current user's home directory.
- **Q: Short option vs long option syntax?** A: Short: single dash + letter (`-l`); long: double dash + word (`--long`).
- **Q: Command to see a one-line description of a command?** A: `whatis <command>`.
- **Q: Command to search for a command by what it does?** A: `apropos <keyword>`.
- **Q: Keyboard shortcut for reverse history search?** A: `Ctrl+R`.
- **Q: Default shell on Ubuntu/Debian?** A: Bash.
- **Q: What does Tab do when typing a filename?** A: Auto-completes it; pressing twice lists all possible matches.
- **Q: Does `clear` erase command history?** A: No, it only clears the visible screen.
- **Q: What is `/bin/sh` on Ubuntu actually pointing to?** A: `dash`, a minimal POSIX shell, not Bash.
