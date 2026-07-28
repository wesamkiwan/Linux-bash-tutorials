# 📋 Module 1 Cheat Sheet — Linux & Shell Fundamentals

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Kernel** | The core OS program that talks directly to hardware (CPU, memory, disk) |
| **Distro** | Kernel + package manager + tools bundled into an installable OS (e.g. Ubuntu) |
| **Shell** | A program (e.g. Bash) that interprets the commands you type |
| **Terminal (emulator)** | The window/app that displays text input/output (e.g. GNOME Terminal, WSL window) |
| **Bash** | "Bourne Again SHell" — the default shell on Ubuntu/Debian and most Linux systems |
| **Prompt** | The line waiting for your input, e.g. `user@host:~$` |
| `~` | Shorthand for your home directory (e.g. `/home/yourusername`) |
| `$` vs `#` | `$` = regular user · `#` = root (superuser — be careful) |

## Command Anatomy

```
command [options] [arguments]
```

| Piece | Example | Notes |
|---|---|---|
| Command | `ls` | The program to run |
| Short option | `-l` | Single dash, single letter; combinable (`-la`) |
| Long option | `--all` | Double dash, full word; more readable |
| Argument | `/home` | What the command acts on |

## Your First Commands

| Command | Purpose |
|---|---|
| `whoami` | Print current username |
| `pwd` | Print current working directory |
| `date` | Print current date/time |
| `echo "text"` | Print text to screen |
| `clear` | Clear the visible screen (history untouched) |
| `history` | List recently run commands, numbered |
| `exit` | Close the current shell session |

## Getting Help — Decision Table

| Tool | Use it when... | Example |
|---|---|---|
| `man <cmd>` | You want the full, authoritative reference | `man ls` |
| `<cmd> --help` | You just need a quick flag reminder | `ls --help` |
| `whatis <cmd>` | You forgot what a command does, in one line | `whatis ls` |
| `apropos <keyword>` | You know the task, not the command name | `apropos "list directory"` |
| `tldr <cmd>` | You want practical examples fast, no jargon (not installed by default) | `tldr ls` |

💡 **Inside `man`:** press `/pattern` to search, `n` for next match, **`q` to quit**.

## Command History & Editing

| Key | Action |
|---|---|
| `↑` / `↓` | Cycle through previous / next commands |
| `Ctrl+R` | Reverse search — type to find a past command live |
| `Ctrl+R` again | Cycle to older matches during a search |
| `Enter` (during search) | Run the found command |
| `Esc` / `→` (during search) | Drop into edit mode on the found command instead of running it |
| `Ctrl+C` | Cancel the reverse search |
| `Tab` | Auto-complete command/filename; press twice to list all matches |

## Shell Types Quick Reference

| Shell | Where it's the default | Notes |
|---|---|---|
| `bash` | Ubuntu/Debian, most Linux distros | What this course teaches |
| `sh` | `/bin/sh` on Ubuntu is actually `dash` | Minimal, POSIX-focused, used for portable scripts |
| `zsh` | macOS default | Feature-rich, popular with Oh My Zsh customization |

## 🔁 The Getting Unstuck Workflow

Do this every time a command's behavior confuses you or a screen won't respond to normal typing:

1. **Try `q`.** If you're inside `man` or any pager, `q` almost always gets you out.
2. **Check `--help`.** Run `<command> --help` for a fast summary of what it does and its flags.
3. **Go deeper with `man`.** If `--help` wasn't enough, `man <command>` has the full detail — use `/keyword` to search inside it.
4. **Search by task, not name.** If you don't even know the command name, `apropos <keyword>` searches descriptions for you.
5. **Grab a quick example.** If you just want to see it used correctly, `tldr <command>` gives copy-pasteable examples.
6. **Still stuck?** Press `Ctrl+C` to cancel whatever's running and get back to a clean prompt, then start over from step 1.

## 🔁 The "Don't Retype It" Workflow

Do this every time you need to run a command you've run recently:

1. Press **`Ctrl+R`**.
2. Type a few distinctive characters from the command.
3. Press **Enter** to run it immediately, or **→ / Esc** to edit it first.
4. If the wrong match appears, press **Ctrl+R** again to cycle further back.
