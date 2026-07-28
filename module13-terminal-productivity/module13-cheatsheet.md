# 📋 Module 13 Cheat Sheet — Terminal Productivity (tmux & Dotfiles)

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Terminal multiplexer** | A program that hosts multiple persistent, detachable terminal sessions inside one connection |
| **Session** | A named `tmux` container that keeps running independently of any specific connection to it |
| **Window** | A tab within a session — one full-screen view; a session can have many |
| **Pane** | A split within a window — multiple terminals visible side-by-side or stacked |
| **Prefix key** | The key combo (`Ctrl+b` by default) pressed before every `tmux` command key |
| **Login shell** | Started on initial login (e.g. a fresh SSH connection); reads `.bash_profile`/`.profile` |
| **Interactive non-login shell** | Started from an already-logged-in session (e.g. a new terminal tab); reads `.bashrc` |
| **Alias** | A custom short name that expands to a longer command |
| **`PS1`** | The shell variable controlling what your command prompt displays |

## `tmux` Session Commands (run outside `tmux`)

| Command | Effect |
|---|---|
| `tmux new -s name` | Start a new named session |
| `tmux ls` | List all running sessions |
| `tmux attach -t name` | Reattach to a named session |
| `tmux kill-session -t name` | Kill a named session and everything inside it |
| `tmux kill-server` | Kill **every** session on the machine |

## `tmux` Keybindings (prefix = `Ctrl+b`, press then release, then the next key)

### Sessions

| Keys | Effect |
|---|---|
| `Ctrl+b d` | Detach from the current session (leaves it running) |
| `Ctrl+b s` | Show an interactive list of sessions to switch between |
| `Ctrl+b $` | Rename the current session |

### Windows

| Keys | Effect |
|---|---|
| `Ctrl+b c` | Create a new window |
| `Ctrl+b n` | Next window |
| `Ctrl+b p` | Previous window |
| `Ctrl+b 0`–`9` | Jump to window by number |
| `Ctrl+b w` | Interactive window list |
| `Ctrl+b ,` | Rename the current window |
| `Ctrl+b &` | Kill the current window (asks for confirmation) |

### Panes

| Keys | Effect |
|---|---|
| `Ctrl+b %` | Split pane vertically (side-by-side) |
| `Ctrl+b "` | Split pane horizontally (stacked) |
| `Ctrl+b <arrow>` | Move focus to pane in that direction |
| `Ctrl+b o` | Cycle focus to next pane |
| `Ctrl+b z` | Zoom current pane full-screen (toggle) |
| `Ctrl+b x` | Kill current pane (asks for confirmation) |
| `exit` | Cleanly close the pane's shell (no prefix needed) |

## Dotfile Purpose Reference

| File | Read when | Typical contents |
|---|---|---|
| `~/.bash_profile` | Login shell only (fresh SSH login, initial console login) | Usually just a snippet that sources `.bashrc`; env vars needed once per login |
| `~/.profile` | Login shell, only if `.bash_profile`/`.bash_login` don't exist | Same role as `.bash_profile`, shell-agnostic (used by `sh` too) |
| `~/.bashrc` | Every interactive non-login shell (new terminal tab, `bash` run manually) | Aliases, `PS1`, history settings, shell functions — put customizations here |
| `~/.bash_logout` | When a login shell exits | Cleanup commands to run on logout (e.g. clearing the screen) |

**Rule of thumb:** put your customizations in `.bashrc`, and make sure `.bash_profile`/`.profile` sources it:

```bash
# inside ~/.bash_profile or ~/.profile
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

## Useful Aliases

| Alias | Expands to | Why |
|---|---|---|
| `alias ll='ls -la'` | `ls -la` | Long listing, including hidden files |
| `alias la='ls -A'` | `ls -A` | List hidden files without `.`/`..` clutter |
| `alias ..='cd ..'` | `cd ..` | Faster directory-up navigation |
| `alias ...='cd ../..'` | `cd ../..` | Two levels up |
| `alias grep='grep --color=auto'` | `grep --color=auto` | Highlight matches by default |
| `alias df='df -h'` | `df -h` | Human-readable disk sizes by default |
| `alias free='free -h'` | `free -h` | Human-readable memory sizes by default |
| `alias gs='git status'` | `git status` | Fast status check |
| `alias ..h='history \| tail -20'` | recent history | Quick glance at recent commands |

```bash
alias name='command'      # define (current session only)
unalias name               # remove (current session only)
```

✅ To make permanent: append to `~/.bashrc`, then `source ~/.bashrc`.

## `PS1` Escape Codes

| Code | Shows |
|---|---|
| `\u` | Username |
| `\h` | Hostname (up to first dot) |
| `\w` | Full working directory (`~` shorthand for home) |
| `\W` | Just the last directory component |
| `\$` | `$` for normal user, `#` for root |
| `\t` | Current time, 24-hour `HH:MM:SS` |
| `\d` | Date |

```bash
PS1='\u@\h:\w\$ '
```

## History Variables

| Variable | Effect |
|---|---|
| `HISTSIZE=5000` | Commands kept in memory for the current session |
| `HISTFILESIZE=10000` | Commands kept in `~/.bash_history` on disk |
| `HISTCONTROL=ignoredups:erasedups` | Skip immediate duplicate entries; erase older duplicates too |
| `shopt -s histappend` | Append history on shell exit instead of overwriting the file |

## 🔁 The Survive-a-Disconnect Workflow

Do this every time you start a long-running remote task:

1. `tmux new -s task` — start a named session.
2. Run the long command directly inside it.
3. `Ctrl+b d` — detach (or just let the connection drop; the session survives either way).
4. Reconnect later (new SSH session if needed).
5. `tmux ls` — confirm the session is still there.
6. `tmux attach -t task` — reattach and check progress.
7. `tmux kill-session -t task` once the work is confirmed done.

## 🔁 The Dotfile Update Workflow

Do this every time you change your shell configuration:

1. Edit `~/.bashrc` (add the alias, prompt line, or history setting).
2. `source ~/.bashrc` to apply it to your **current** shell immediately.
3. Test the change right there (run the alias, check the prompt).
4. Open a **new** terminal tab to confirm it persists on its own, unassisted.
