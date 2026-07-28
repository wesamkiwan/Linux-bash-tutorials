# 🎤 Module 13 Interview Prep — Terminal Productivity (tmux & Dotfiles)

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a terminal multiplexer, and what problem does it solve?**

> "A terminal multiplexer is a program that lets you run multiple independent terminal sessions inside a single connection, and — critically — keeps those sessions running on the server even if your connection to them drops. The core problem it solves is that a plain terminal or SSH session ties everything you run to that one specific connection; close it or lose it, and everything running inside dies with it. `tmux` runs as its own process on the server, completely decoupled from whatever local terminal or SSH connection you're currently viewing it through, so the actual work survives independently of that connection."

**Q2: What's the difference between a `tmux` session, window, and pane?**

> "They're nested levels of organization. A session is the outermost container — a whole named working environment that keeps running independently. A window is like a tab within that session — one full-screen view; a session can have several windows, but you look at one at a time. A pane is a split within a single window — dividing that one window's screen into multiple side-by-side or stacked terminals you can see simultaneously."

**Q3: How do you detach from a `tmux` session, and what happens to what's running inside it?**

> "`Ctrl+b d` — press and release Ctrl+b, then press d. It detaches your current view from the session, but everything running inside — any commands, any panes, any windows — keeps running exactly as before. Detaching only disconnects your local view; it doesn't stop or pause anything."

**Q4: What is an alias, and how long does one defined directly at the prompt last?**

> "An alias is a custom short name that expands to a longer command — `alias ll='ls -la'` means typing `ll` runs `ls -la`. One defined directly at the prompt only lasts for that specific running shell session; the moment you close the terminal, it's gone, because it was never written anywhere persistent. To make it permanent you have to add it to a startup file that gets read every time a new shell starts — `.bashrc`."

**Q5: What does `PS1` control?**

> "`PS1` is the shell variable that defines what your command prompt looks like — the text shown before your cursor every time Bash is ready for a new command. It's built from literal text plus backslash-escape codes like `\\u` for username, `\\h` for hostname, and `\\w` for the current working directory, and it gets re-evaluated fresh every single time the prompt is drawn, not just once."

### 🟡 Intermediate

**Q6: What's the practical difference between `.bashrc` and `.bash_profile`/`.profile` — when does each actually run?**

> "It comes down to how the shell was started, not which machine you're on. `.bash_profile` (or `.profile` if that doesn't exist) is read only for a login shell — the classic case being a brand-new SSH connection, or logging into a machine for the first time in a session. `.bashrc` is read for every interactive non-login shell — most commonly, opening a new terminal tab on a machine you're already logged into. So on a fresh SSH login, Bash reads `.bash_profile`, not `.bashrc`, unless `.bash_profile` explicitly sources `.bashrc` itself — which is exactly why almost every real-world `.bash_profile` contains a small snippet that does exactly that. The practical upshot: if you only put your aliases in `.bashrc` and forget to check that sourcing snippet exists on a given account, those aliases will work fine in every local terminal tab but silently not appear the moment you SSH in fresh."

**Q7: Why would you use `tmux` over a plain SSH session for a long-running job?**

> "A plain SSH session ties everything running inside it directly to that one connection — if the connection drops for any reason, whether you close the laptop lid, your Wi-Fi hiccups, or the VPN resets, whatever was running dies with it, mid-task, with no way to see what state it was in. `tmux` decouples the two: the job runs inside a `tmux` session on the server itself, which keeps running independently of any specific connection to it. If the SSH connection drops, the session survives regardless — you just reconnect and run `tmux attach` to see it exactly where it was, output and all. It's the difference between a job that depends on one fragile network connection staying up the whole time, and a job that only needs that connection at the moments you actually choose to look at it."

**Q8: Why doesn't editing `.bashrc` change the behavior of a shell you're already sitting in?**

> "Startup files like `.bashrc` are read once, at the moment a shell starts — not continuously monitored. Editing the file afterward only affects shells that start after that edit; the shell you're already in already finished reading the old version and has no reason to look at the file again on its own. To apply changes to your current shell without closing it, you re-run the file's contents manually with `source ~/.bashrc`, which reads and executes it directly in your current shell, the same as if you'd typed each line yourself."

**Q9: What does `HISTCONTROL=ignoredups:erasedups` actually do, and why combine both?**

> "`ignoredups` stops immediately-repeated commands from being recorded again — if you run the exact same command twice in a row, the second one isn't added as a fresh duplicate entry. `erasedups` goes further: any time a command is run, it removes all *earlier* occurrences of that same command anywhere else in your history, not just an immediately preceding one. Combining them keeps your history genuinely useful and searchable instead of cluttered with dozens of repeats of common commands like `ls` or `git status` scattered throughout it."

**Q10: What does `shopt -s histappend` do, and what actually goes wrong without it?**

> "By default, Bash overwrites the entire `~/.bash_history` file with the current session's history when a shell exits. If you have multiple terminal tabs open at once, each one only knows about its own commands — so whichever tab happens to close last completely overwrites the file, silently discarding the history that all the other tabs would otherwise have contributed. `shopt -s histappend` changes shell-exit behavior to append that session's history onto the existing file instead of replacing it, so multiple concurrent shells don't stomp on each other's history."

### 🔴 Advanced

**Q11: You SSH into a server, start a multi-hour data migration inside `tmux`, and need to hand off monitoring it to a teammate on a different machine. How would you actually do that?**

> "As long as we're both authenticating as the same user (or the session is set up with shared permissions), whoever's connected to that same server can run `tmux attach -t <session-name>` to see the exact same live session — same panes, same scrollback, same running processes. If we need to view it simultaneously rather than handing it off, `tmux` also supports multiple clients attached to the same session at once, so we could both be looking at it live. The key requirement is that the session lives on the shared server itself, not on either of our individual laptops — that's exactly why it survives and is reachable by anyone with access to that machine, regardless of who originally started it."

**Q12: A junior engineer says they don't need `tmux` because they just use `nohup` for long jobs. What's the tradeoff they're missing?**

> "`nohup` genuinely solves the 'my job dies when my terminal closes' problem, but it only gives you a log file to check afterward — you can't interactively watch it, send it input, or work alongside it in real time the way you can with a live `tmux` pane. `tmux` covers the same disconnect-survival case, plus lets you actively watch and interact with the job's live output at any point, run multiple related things side-by-side in other panes (monitoring tools, log tails, a second shell for other work), and reattach from a genuinely different machine or connection entirely. `nohup` is fine for a true fire-and-forget batch job where you only care about the final log; anything you want to actively babysit or interact with mid-run is a better fit for `tmux`."

**Q13: Why might putting a slow or blocking command directly in `.bashrc` cause problems across an entire team's environment?**

> "`.bashrc` runs every single time a new interactive shell starts — every new terminal tab, every `bash` invocation. If it contains something slow (a network call, a large `find` over a big directory) or something that can hang waiting on input, every single new shell anyone opens pays that cost, or worse, can freeze waiting on it. This is especially painful on shared or automated systems where scripts spawn many new shells programmatically — a slow `.bashrc` multiplies out badly. The practical fix is keeping `.bashrc` to genuinely fast, non-blocking operations (aliases, `PS1`, variable assignments) and pushing anything slower into something explicitly invoked on demand instead of run unconditionally on every shell startup."

---

## Practical/Coding Questions

**Q1: Write the commands to start a named `tmux` session called `backup`, and confirm afterward that it's running, without attaching to it.**

Solution:
```bash
tmux new -s backup
```
*(inside the session, detach)*
```
Ctrl+b d
```
```bash
tmux ls
```
Explanation: `tmux new -s backup` creates and immediately attaches to a named session. `Ctrl+b d` detaches without stopping anything inside it. `tmux ls`, run from a plain shell, lists every running session by name — confirming `backup` exists without needing to reattach and look inside it.

**Q2: Write the `.bashrc` snippet that adds three aliases and applies them without closing the terminal.**

Solution:
```bash
cat >> ~/.bashrc << 'EOF'
alias ll='ls -la'
alias ..='cd ..'
alias gs='git status'
EOF
source ~/.bashrc
```
Explanation: `cat >> ~/.bashrc` **appends** to the file (never `>`, which would erase everything already in it). The heredoc (`<< 'EOF' ... EOF`) writes multiple lines in one command. `source ~/.bashrc` re-reads the file into the current shell immediately, so the new aliases work right away instead of only in future terminals.

**Q3: Write the `.bash_profile` snippet that ensures `.bashrc` is also read on a fresh login shell.**

Solution:
```bash
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```
Explanation: This checks that `~/.bashrc` exists (`-f` tests for a regular file), and if so, sources it (`.` is shorthand for `source`) directly inside the login shell. Without this, anything defined only in `.bashrc` would never appear on a fresh SSH login, only in already-logged-in new terminal tabs.

**Q4: Write the history-tuning block for `.bashrc` that keeps 5000 commands in-session, 10000 on disk, removes duplicate entries, and safely supports multiple concurrent terminal tabs.**

Solution:
```bash
HISTSIZE=5000
HISTFILESIZE=10000
HISTCONTROL=ignoredups:erasedups
shopt -s histappend
```
Explanation: `HISTSIZE`/`HISTFILESIZE` set the in-memory and on-disk history limits respectively. `HISTCONTROL=ignoredups:erasedups` removes duplicate clutter. `shopt -s histappend` is the piece that specifically makes multiple concurrent tabs safe — without it, whichever tab closes last would overwrite (not merge with) history from every other tab that already closed.

---

## Gotcha Questions

**Q1: "I put my alias in `~/.bashrc` and it works perfectly in every new terminal tab I open on my desktop. Why doesn't it show up the moment I SSH into my server?"**

> Trap: The candidate needs to recognize that a new local terminal tab and a fresh SSH connection start **different kinds** of shells. A new tab on an already-logged-in machine is an interactive non-login shell, which reads `.bashrc` directly — that's why it works there. A fresh SSH connection is a login shell, which reads `.bash_profile`/`.profile` instead, and only reaches `.bashrc` at all if that file explicitly sources it. The alias isn't broken — it's simply never being loaded on that account's login path at all.

**Q2: "I detached from my `tmux` session with `Ctrl+b d` before leaving for the day. The next morning it's gone. Doesn't `tmux` sessions survive forever?"**

> Trap: `tmux` sessions survive a **dropped connection or intentional detach**, not a server reboot, a crashed `tmux` server process, or someone (or an automated process) running `tmux kill-server`/`kill-session` against it. Detaching only decouples your local view — the session itself is still an ordinary process tree on that machine, subject to the same fate as any other process if the machine restarts or the process is killed outright. "Survives a disconnect" and "survives literally anything" are not the same guarantee.

**Q3: "I edited `.bashrc`, ran the script again in a new subprocess to test it, saw it work, but my actual current terminal still doesn't have the new alias. What's going on?"**

> Trap: Running `.bashrc` as a separate script (`bash ~/.bashrc` or `./bashrc`) executes it in a **new subprocess** — any variables or aliases it defines exist only in that subprocess and vanish the instant it exits, never touching the parent shell you're actually typing in. To apply changes to the shell you're currently sitting in, you must **source** it (`source ~/.bashrc` or `. ~/.bashrc`), which runs the commands directly inside your current shell rather than spawning a throwaway new one.

---

## Quick-Fire Rapid Review

- **Q: What key combo is the default `tmux` prefix?** A: `Ctrl+b`.
- **Q: What command starts a new named `tmux` session?** A: `tmux new -s name`.
- **Q: What detaches from a `tmux` session without stopping anything?** A: `Ctrl+b d`.
- **Q: What command reattaches to a named session?** A: `tmux attach -t name`.
- **Q: What command lists all running `tmux` sessions?** A: `tmux ls`.
- **Q: Keys to split a pane vertically? Horizontally?** A: `Ctrl+b %` (vertical), `Ctrl+b "` (horizontal).
- **Q: Which dotfile is read on a fresh SSH login?** A: `~/.bash_profile` (or `~/.profile`).
- **Q: Which dotfile is read when opening a new terminal tab on an already-logged-in machine?** A: `~/.bashrc`.
- **Q: What command applies `.bashrc` changes to your current shell without restarting it?** A: `source ~/.bashrc`.
- **Q: What does `HISTCONTROL=ignoredups:erasedups` do?** A: Skips immediate repeated commands and erases older duplicates too.
- **Q: What does `shopt -s histappend` fix?** A: History from multiple concurrent terminal tabs overwriting each other on exit.
- **Q: What variable controls the shell prompt's content?** A: `PS1`.
- **Q: What does the `\\w` escape code show in `PS1`?** A: The full current working directory.
- **Q: What's the modern standard multiplexer, `tmux` or `screen`?** A: `tmux`.
- **Q: How do you remove an alias for the current session?** A: `unalias name`.
