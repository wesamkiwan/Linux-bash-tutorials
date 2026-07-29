# Module 1: Linux & Shell Fundamentals 🟢

**Difficulty:** 🟢 Beginner
**Estimated Time:** 2 hours
**Prerequisites:** None — this is the starting point

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain what Linux, a distro, the kernel, and the shell are — and how they relate to each other
- [ ] Tell the difference between a terminal emulator and a shell, and explain what Bash is
- [ ] Open a terminal and read the prompt to identify your user, host, current directory, and privilege level
- [ ] Break any command down into its command, options, and arguments
- [ ] Get help on any command using `man`, `--help`, `whatis`, `apropos`, and `tldr`
- [ ] Run basic commands (`whoami`, `pwd`, `date`, `echo`, `clear`, `history`, `exit`) confidently
- [ ] Use command history (arrow keys, `Ctrl+R`) and Tab completion to work faster
- [ ] Name the major shell types (bash, zsh, sh) and explain why this course focuses on Bash

---

## Module Goal

By the end of this module, you'll be comfortable just **sitting inside a terminal** — reading the prompt, running simple commands, getting help without panicking, and recovering commands you typed five minutes ago. That sounds small. It isn't.

🎯 **On the job:** Every single DevOps, sysadmin, backend, or SRE role starts here. When you SSH into a production server at 2 a.m. because something's on fire, there's no GUI, no mouse, no "click here." There's a prompt blinking at you. Everything in this course — and everything you'll do professionally on Linux — is built on the muscle memory you form in this module. Companies don't hire people who are scared of the terminal.

---

## Core Concepts

Let's build up your mental model one brick at a time. Nothing here assumes you know anything already.

### 1. What is an operating system?

An **operating system (OS)** is the software that manages a computer's hardware (CPU, memory, disk, network card) and provides a common way for other programs to use that hardware. Windows, macOS, and Linux are all operating systems.

### 2. What is the kernel?

The **kernel** is the core part of the operating system. It's the piece of software that talks directly to the hardware — it decides which program gets CPU time, manages memory, and handles reading/writing to disk. **Linux**, strictly speaking, is just the kernel — it was created by Linus Torvalds in 1991.

### 3. What is a distro?

A **distribution** (or **distro**) is the Linux kernel bundled together with a set of software: a package manager, default applications, configuration tools, and often a desktop environment. The kernel alone isn't usable — a distro turns it into a complete, installable operating system.

Popular distros include Ubuntu, Debian, Fedora, and Arch. **This course focuses on Ubuntu/Debian**, because they're the most common distros in professional environments (cloud servers, Docker images, CI/CD runners) and they share the same package manager (`apt`) and command set for everything we'll cover. We'll flag it clearly whenever another distro (like Fedora/RHEL, which uses `dnf`/`yum`) would do something differently.

### 4. What is a shell?

A **shell** is a program that reads the commands you type and asks the kernel to carry them out. It's the layer between you (the human) and the kernel. When you type `pwd` and hit Enter, the shell interprets that text, figures out what you mean, and requests the corresponding action from the operating system.

### 5. What is Bash?

**Bash** (short for "Bourne Again SHell") is the most common shell on Linux systems. It's the default shell on Ubuntu, Debian, and most other distros when you open a terminal. It's mature, stable, installed everywhere, and what almost every server and CI/CD pipeline in the world assumes you're using. That's exactly why this course teaches Bash — it's the one skill that transfers to every Linux environment you'll ever touch professionally.

### 6. What is a terminal (emulator)?

Here's where beginners often get confused, so let's use an analogy.

💡 **Analogy — Car dashboard:**
- The **kernel** is the car's engine — it does the actual work (burning fuel, turning the wheels) but you never touch it directly.
- The **shell** is the steering wheel and pedals — the control interface that lets you tell the engine what to do ("go forward," "turn left").
- The **terminal (emulator)** is the dashboard/windshield — the window through which you see what's happening and where you physically interact with the controls.

A **terminal emulator** (often just called "a terminal") is an application that opens a window, displays text, and lets you type. Examples: GNOME Terminal, Windows Terminal, iTerm2, the VS Code integrated terminal. The terminal emulator's job is just to show text input/output — it doesn't understand commands at all.

The **shell** is the program running *inside* that terminal window that actually interprets what you type. So: you open a terminal (the window), and inside it a shell (usually Bash) is running, waiting for your commands.

⚠️ **Warning:** People say "the terminal crashed" when they usually mean the shell (or a program running in it) crashed. They're related but different things — the terminal is the container, the shell is what's running inside it.

### 7. The console

You may also hear the word **console**. Historically, "the console" meant the physical keyboard/screen directly attached to a machine (as opposed to connecting remotely). Today, people often use "console," "terminal," and "terminal emulator" loosely to mean the same thing — a window where you type commands. Don't stress over the distinction; just know they all refer to roughly the same idea: your text-based interface to the shell.

---

## Detailed Explanations

### Opening a terminal

This course assumes you already have one Linux environment set up, per the course [README](../README.md) — WSL2, a VM, a cloud instance, or a Docker container. Here's how you'd get to a Bash prompt in each:

| Environment | How to open it |
|---|---|
| **WSL2 (Windows)** | Open the Start Menu → type "Ubuntu" → press Enter. This drops you straight into a Bash shell. |
| **VM (VirtualBox/VMware)** | Boot the VM, log in, then open "Terminal" from the applications menu. |
| **Cloud VM** | `ssh your_user@your_server_ip` from your local terminal/PowerShell. |
| **Docker container** | `docker run -it --rm ubuntu:24.04 bash` from your local terminal. |

Whichever route you took, you should now see a blinking cursor next to some text — that's your **prompt**, and Bash is waiting for you.

### Reading the prompt

A typical Ubuntu/Debian Bash prompt looks like this:

```
user@host:~$
```

Let's break it down piece by piece:

| Part | Meaning |
|---|---|
| `user` | Your username — who you're logged in as |
| `@` | Separator, just means "at" |
| `host` | The hostname — the name of the machine you're on |
| `:` | Separator |
| `~` | Your **current working directory** — see below |
| `$` or `#` | The prompt symbol — tells you your privilege level |

**What does `~` mean?** The tilde is shorthand for your **home directory** — the personal folder assigned to your user account (on Ubuntu, typically `/home/yourusername`). Whenever you see `~` in a path, mentally substitute it with "my home directory."

**`$` vs `#` — this one really matters:**
- `$` means you're a **regular (non-root) user** — the normal, limited-privilege account you should use for almost everything.
- `#` means you're **root** — the superuser with unrestricted power to do *anything* on the system, including breaking it permanently.

⚠️ **Warning:** If you ever see `#` in your prompt and you didn't deliberately switch to root (e.g., via `sudo -i`), stop and figure out why before running anything. Commands that would ask for confirmation or fail safely as a regular user can silently destroy the whole system as root.

✅ **Best Practice:** Do your daily work as a regular user. Only use `sudo` (which temporarily elevates a *single command* to root) when a command specifically requires it, like installing packages.

### Command anatomy

Almost every Linux command follows the same shape:

```
command [options] [arguments]
```

- **command** — the program you want to run, e.g. `ls`, `cp`, `grep`.
- **options** (a.k.a. "flags" or "switches") — modify *how* the command behaves. Optional.
- **arguments** — the "things" the command acts on, e.g. a filename. Optional, depending on the command.

**Short vs. long options:**
- **Short options** use a single dash and a single letter, e.g. `-l`. Several can often be combined: `-la` means `-l` and `-a` together.
- **Long options** use two dashes and a full word, e.g. `--long`. They're more readable and self-documenting, which matters when you (or a teammate) reread a script six months later.

Example: `ls -l --all` combines a short option and a long option — both are valid, and most commands support both forms for the same behavior.

🎯 **On the job:** Reading someone else's shell script or a command from a Stack Overflow answer is far easier once you can mentally parse `command [options] [arguments]` on sight, instead of seeing a wall of cryptic dashes.

### Getting help

You will **never** memorize every option for every command — nobody does. What separates a confident Linux user from a stuck one is knowing how to look things up *fast*, without leaving the terminal.

| Tool | What it does | When to use it |
|---|---|---|
| `man <command>` | Opens the full manual page — comprehensive, technical, the "textbook" | You want the complete, authoritative reference |
| `<command> --help` | Prints a short usage summary right in your terminal | You just need a quick reminder of the flags |
| `whatis <command>` | Prints a one-line description of the command | You forgot what a command does, in one sentence |
| `apropos <keyword>` | Searches command descriptions for a keyword | You know what you want to *do* but not which command does it |
| `tldr <command>` | Community-maintained, example-driven cheat sheet (not installed by default) | You want practical examples fast, skip the jargon |

💡 **Tip:** `man` opens in a pager (usually `less`). Beginners often panic because they don't know how to leave it. Press **`q`** to quit. That's it. Also try `/searchterm` inside `man` to search the page, then `n` to jump to the next match.

⚠️ **Warning:** `tldr` isn't installed by default on Ubuntu. Install it with `sudo apt update && sudo apt install tldr` (or `npm install -g tldr`, or `pip install tldr` — several implementations exist). It's a modern, friendlier companion to `man`, not a replacement — use `tldr` for quick examples and `man` when you need the full, authoritative detail.

### Your very first commands

| Command | What it does |
|---|---|
| `whoami` | Prints the username you're currently logged in as |
| `pwd` | "Print Working Directory" — shows the full path of the folder you're currently in |
| `date` | Prints the current date and time |
| `echo` | Prints text back to the screen — useful for testing and for printing variable values later |
| `clear` | Clears the terminal screen (doesn't affect history, just the visible screen) |
| `history` | Lists the commands you've typed recently, numbered |
| `exit` | Closes the current shell session |

### Command history

Bash remembers the commands you type. You don't have to retype something you ran two minutes — or two days — ago.

- **Up arrow / Down arrow** — cycle backward/forward through previously typed commands.
- **`Ctrl+R`** — **reverse search**: start typing any part of a past command, and Bash finds the most recent match live as you type. Press `Ctrl+R` again to cycle to older matches. Press Enter to run it, or Right Arrow/Esc to edit it first.

💡 **Tip:** `Ctrl+R` is one of the single highest-leverage habits you can build. Professionals rarely retype long commands — they search for them.

### Tab completion

Press **Tab** while typing a command or a filename, and Bash tries to auto-complete it for you. If there's only one match, it completes instantly. If there are multiple possible matches, pressing Tab twice lists them all.

✅ **Best Practice:** Use Tab completion constantly — it's faster than typing, and it prevents typos in filenames, which is one of the most common sources of "file not found" errors.

### Shell types overview

Bash isn't the only shell in existence:

| Shell | Notes |
|---|---|
| **bash** | The default on Ubuntu/Debian and most Linux distros. What this course teaches. Widely used in scripts, servers, and CI/CD. |
| **sh** | The original Bourne shell (or a minimal POSIX-compliant shell — on Ubuntu, `/bin/sh` is actually a lightweight shell called `dash`). Scripts written for `sh` are more portable but have fewer features than Bash. |
| **zsh** | A popular, more feature-rich shell, especially loved on macOS (it's the default there) and by developers who customize their prompt heavily with frameworks like Oh My Zsh. |

We teach **Bash** in this course because it's the universal default you'll find on virtually every Linux server, Docker container, and CI/CD runner in production. Once you know Bash, reading a `zsh` or `sh` script is a small leap — the core concepts (commands, options, arguments, variables, control flow) are shared across all of them.

---

## Practical Examples

### Example 1 — Confirming who and where you are

```bash
whoami
```

Expected output:
```
weki
```

Line-by-line:
- `whoami` takes no options or arguments here — it just prints the username of the currently logged-in user.

```bash
pwd
```

Expected output:
```
/home/weki
```

Line-by-line:
- `pwd` prints the absolute path of your current directory. Right after login, this is almost always your home directory.

### Example 2 — Basic output and the date

```bash
echo "Hello, Linux!"
```

Expected output:
```
Hello, Linux!
```

Line-by-line:
- `echo` is the command.
- `"Hello, Linux!"` is the argument — the text to print. Quotes keep the whole phrase together as one argument (we'll cover quoting rules in depth in a later module).

```bash
date
```

Expected output:
```
Tue Jul 28 14:32:07 UTC 2026
```

Line-by-line:
- `date` with no options/arguments prints the current system date and time in a default format.

### Example 3 — Options and arguments together

```bash
ls -l /home
```

Expected output:
```
drwxr-xr-x 5 weki weki 4096 Jul 28 09:10 weki
```

Line-by-line:
- `ls` is the command — it lists directory contents.
- `-l` is a short option meaning "long format" — show permissions, owner, size, and modification date instead of just names.
- `/home` is the argument — the directory to list.

```bash
ls --all --long /home
```

Line-by-line:
- Same command, but this time using the **long forms** of both options (`--all` instead of `-a`, `--long` instead of `-l`). Both styles are valid Bash — long options are more readable, short options are faster to type.

💡 **Tip:** Combine short options: `ls -la` is identical to `ls -l -a`.

### Example 4 — Getting help without leaving the terminal

```bash
whatis ls
```

Expected output:
```
ls (1)               - list directory contents
```

```bash
apropos "list directory"
```

Expected output (abbreviated):
```
ls (1)               - list directory contents
dir (1)              - list directory contents
```

```bash
ls --help
```

Expected output (abbreviated):
```
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).

  -a, --all                  do not ignore entries starting with .
  -l                         use a long listing format
  ...
```

```bash
man ls
```

This opens the full manual page in a pager. Scroll with the arrow keys or Space, search with `/pattern`, and **press `q` to exit**.

⚠️ **Warning:** New users often get "stuck" in `man` because they don't know `q` quits it. If you ever feel trapped in a terminal screen that won't respond to normal typing, try `q` first.

### Example 5 — History and reverse search

```bash
history
```

Expected output (abbreviated):
```
  501  whoami
  502  pwd
  503  ls -l /home
  504  man ls
```

Now try:
1. Press **Ctrl+R**.
2. Type `who` — Bash will show the most recent command containing "who" (`whoami`).
3. Press **Enter** to run it, or **Ctrl+C** to cancel out of the search.

🎯 **On the job:** Imagine you ran a long, complicated command to restart a service 20 minutes ago and need to run it again. `Ctrl+R restart` finds it in under a second — no retyping, no scrolling through terminal output.

### Example 6 — Leaving the shell

```bash
exit
```

This closes the current shell session. If you're in a local terminal window, the window itself may close too. If you're connected via SSH, this logs you out of the remote server.

---

## Common Pitfalls & Best Practices

- **Confusing "terminal" with "shell."** The terminal is the window; the shell (Bash) is the program running inside it interpreting your commands. If your shell crashes, the terminal window itself is usually fine — you'd just start a new shell.
- **Getting stuck inside `man`.** Press `q` to quit — it's the #1 "why is my terminal frozen" question from beginners.
- **Case sensitivity surprises.** Linux commands and filenames are **case-sensitive**. `LS`, `Ls`, and `ls` are three different things to Bash (only `ls` exists as a command), and `Report.txt` and `report.txt` are two different files. Windows users especially trip on this at first.
- **Assuming `#` in the prompt is normal.** It means you're root. Don't run unfamiliar commands as root just because a tutorial told you to — understand what a command does first.
- **Typing commands from memory instead of searching history.** If you ran something similar recently, `Ctrl+R` is almost always faster and less error-prone than retyping it.
- **Not using Tab completion.** Beginners often type full filenames by hand and introduce typos. Get in the habit of typing a few characters and pressing Tab.
- **Confusing `exit` with `clear`.** `clear` just wipes the visible screen; your history and session are untouched. `exit` ends the session entirely.

---

## Hands-on Exercise

**Task:**

1. Use **three different help tools** (`man`, `--help`, and `tldr` if installed) to learn **three things about the `ls` command** that you didn't already know (e.g., an option you've never used).
2. Run at least **five different commands** from this module (`whoami`, `pwd`, `date`, `echo`, `ls -l`, etc.).
3. Use `history` to view what you just ran.
4. Use **`Ctrl+R`** to reverse-search for one of those commands and re-run it without retyping it.

Take a few minutes to actually do this in your terminal before reading the solution below.

### Solution

Here's a worked walkthrough of one way to complete this exercise:

**Step 1 — Explore `ls` with three help tools:**

```bash
man ls
```
Inside the manual, I search for the word "sort" by typing `/sort` and pressing Enter. I discover `-t` sorts by modification time, newest first. I press `q` to exit.

```bash
ls --help
```
Scanning the output, I notice `-h` (used with `-l`) prints sizes like `1.2K`, `3.4M` instead of raw byte counts — much easier to read.

```bash
tldr ls
```
This shows a friendly example: `ls -lhS` — lists files in long format, human-readable sizes, sorted by size largest-first. I hadn't combined those three flags together before.

So my three new things learned: `-t` (sort by time), `-h` (human-readable sizes), and the combo `-lhS` (long + human-readable + sorted by size).

**Step 2 — Run five commands:**

```bash
whoami
pwd
date
echo "learning bash"
ls -l
```

**Step 3 — Check history:**

```bash
history
```

Output shows all five commands, numbered, e.g.:
```
  510  whoami
  511  pwd
  512  date
  513  echo "learning bash"
  514  ls -l
```

**Step 4 — Reverse search and re-run:**

I press **Ctrl+R**, type `echo`, and Bash shows `echo "learning bash"` from my history. I press **Enter**, and it runs again, printing:
```
learning bash
```

✅ Exercise complete — I've now practiced getting help three different ways and used history search instead of retyping a command.

---

## ✅ Module Completion Checklist

- [ ] I can explain what Linux, a distro, the kernel, and the shell are — and how they relate to each other
- [ ] I can tell the difference between a terminal emulator and a shell, and explain what Bash is
- [ ] I can open a terminal and read the prompt to identify my user, host, current directory, and privilege level
- [ ] I can break any command down into its command, options, and arguments
- [ ] I can get help on any command using `man`, `--help`, `whatis`, `apropos`, and `tldr`
- [ ] I can run basic commands (`whoami`, `pwd`, `date`, `echo`, `clear`, `history`, `exit`) confidently
- [ ] I can use command history (arrow keys, `Ctrl+R`) and Tab completion to work faster
- [ ] I can name the major shell types (bash, zsh, sh) and explain why this course focuses on Bash

## Next Step

Continue to [Module 2: Filesystem Navigation & File Operations](../module02-filesystem-navigation/)
