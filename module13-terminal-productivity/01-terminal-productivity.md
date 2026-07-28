# Module 13: Terminal Productivity (tmux & Dotfiles) 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-10 (Shell Fundamentals through Process Management & Job Control)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain why terminal multiplexers matter — surviving SSH disconnects and running multiple panes/windows inside one terminal session
- [ ] Start, detach from, and reattach to a `tmux` session (`tmux new -s`, `Ctrl+b d`, `tmux attach -t`), and list all running sessions with `tmux ls`
- [ ] Create and navigate `tmux` windows (`Ctrl+b c`, `Ctrl+b n`/`p`, `Ctrl+b 0-9`) and panes (`Ctrl+b %`, `Ctrl+b "`, `Ctrl+b <arrow>`)
- [ ] Kill `tmux` panes, windows, and sessions cleanly, and explain why `tmux` has replaced `screen` as the modern standard
- [ ] Explain the practical difference between `~/.bashrc` and `~/.bash_profile`/`~/.profile` — which one runs on a fresh SSH login versus a new terminal tab, and why that determines where you put things
- [ ] Create, use, and remove aliases (`alias`, `unalias`), and make them permanent by adding them to `.bashrc`
- [ ] Customize your shell prompt with a simple, readable `PS1` value
- [ ] Tune your command history with `HISTSIZE`, `HISTFILESIZE`, `HISTCONTROL`, and `shopt -s histappend`

---

## Module Goal

By the end of this module, you'll be able to build a terminal environment that works *for* you all day long, instead of one you have to fight — one where long jobs survive a flaky connection, your most common commands are one short word away, and your prompt and history are tuned to how you actually work.

🎯 **On the job:** Picture this: you SSH into a remote server to kick off a database migration that will take 40 minutes. Fifteen minutes in, your hotel Wi-Fi drops, or your laptop goes to sleep, or your VPN hiccups — and if you were running that migration directly in a plain SSH session, it's now dead, possibly halfway through, with no way to see what it was doing when the connection died. `tmux` (a **terminal multiplexer** — a tool that lets one terminal session host multiple independent, persistent workspaces) solves this completely: the migration keeps running inside its `tmux` session even after your connection drops, and you simply reconnect and look at it again, output and all, exactly where you left it. Meanwhile, every single day, you're also typing dozens of repetitive commands — `ls -la`, `git status`, `docker ps` — and a well-tuned `.bashrc` full of aliases, a clear prompt, and smart history settings can shave real, cumulative time off every single session you open. This module builds both muscles: keeping your work alive, and making your day-to-day faster.

---

## Core Concepts

### 1. What is a terminal multiplexer, and why do you need one?

A **terminal multiplexer** is a program that runs inside your terminal and lets you create multiple independent virtual terminals — called **sessions**, **windows**, and **panes** — that all keep running even if the actual terminal window or SSH connection that started them disappears. The multiplexer itself keeps running as a background process on the remote machine, completely independent of your local connection to it.

💡 **Analogy — a TV with channels you can leave on:** Think of `tmux` as a television that stays powered on in another room, playing several channels at once (your different windows/panes), each doing its own thing. You can walk out of the room (disconnect) and the TV keeps playing every channel exactly as it was. When you walk back in (reconnect), you look at the screen again and everything picked up right where it left off — nothing paused, nothing lost. Compare that to running a job directly over a plain SSH connection: that's like watching a show on a screen that switches off completely the instant you leave the room.

Without a multiplexer, everything you run in an SSH session lives and dies with that specific connection. With `tmux`, your work lives inside a **session** on the remote server itself — your local terminal (or SSH connection) is just a window you're currently looking through it with, and you can put that window down and pick it back up later, from anywhere.

### 2. `tmux`: sessions, windows, and panes

`tmux` organizes your terminal work into three nested levels:

- A **session** is the outermost container — a full working environment, given a name (like `deploy` or `main`), that keeps running on the server independently of any specific connection to it. You can have several sessions running at once for different purposes.
- A **window** is like a tab within a session — a single full-screen view, similar to a tab in a graphical terminal app. A session can have many windows, but you only look at one at a time.
- A **pane** is a rectangular split *within* a window — dividing that one window's screen space into two or more independent, side-by-side (or stacked) terminals you can see simultaneously.

💡 **Analogy continued:** If a `tmux` session is the whole TV, a window is one channel on it, and a pane is picture-in-picture — splitting that one channel's screen so you can watch two things on it at once without switching away.

### 3. The prefix key

Almost every `tmux` keyboard shortcut starts with a **prefix key** — by default `Ctrl+b` — pressed and released *before* the actual command key. You don't hold all three keys down together; you press `Ctrl+b`, let go, then press the next key (like `d` or `%`) separately. This module writes prefix combinations as `Ctrl+b d`, meaning "press Ctrl+b, release, then press d."

⚠️ **Common confusion:** `Ctrl+b` alone does nothing visible by itself — it just tells `tmux` "the next key you see is a command for me, not for whatever program is running in this pane." If you wait too long between the prefix and the next key, `tmux` gives up waiting and you'll need to press `Ctrl+b` again.

### 4. Starting, detaching, and reattaching a session

Three commands cover the entire "keep it alive across a disconnect" workflow:

- **`tmux new -s name`** — starts a brand-new, named session (e.g. `tmux new -s deploy`). Naming your sessions is strongly recommended over leaving them numbered, since names are far easier to recognize and reattach to later.
- **`Ctrl+b d`** — **detaches** from the current session. This does *not* stop anything running inside it — it simply disconnects your current terminal's view from the session, leaving the session and everything inside it running exactly as it was, on the server, in the background.
- **`tmux attach -t name`** — **reattaches** to a named, still-running session, from any terminal or SSH connection (even a completely new one), restoring your view exactly where you left off.

This detach/reattach pair is precisely what makes `tmux` immune to a dropped connection: whether you detach on purpose with `Ctrl+b d`, or your connection simply drops without warning, the session itself keeps running either way — a dropped connection just means you never got the chance to detach politely, but the session survives regardless. You reattach the same way in both cases.

### 5. Listing sessions

**`tmux ls`** (run from a plain shell, outside of `tmux`) lists every session currently running on that machine, by name, along with how many windows each has and whether anything is currently attached to it. This is your "what do I already have running?" command — essential after a surprise disconnect, when you need to remember what you named the session you were just in.

### 6. Windows: multiple tabs within one session

Within a session, you can open additional windows and flip between them:

| Keys | Effect |
|---|---|
| `Ctrl+b c` | **c**reate a new window |
| `Ctrl+b n` | Move to the **n**ext window |
| `Ctrl+b p` | Move to the **p**revious window |
| `Ctrl+b 0`–`9` | Jump directly to window number 0 through 9 |
| `Ctrl+b w` | Show an interactive list of all windows to choose from |

Each window gets a number (starting at 0) shown in the green status bar at the bottom of the screen, along with the name of whatever's currently running in it.

🎯 **On the job:** A common real layout is one window for editing/browsing code, a second window for running your application or tests, and a third for tailing logs — all inside a single `tmux` session, switched between instantly with `Ctrl+b n`/`Ctrl+b p`, instead of juggling several separate terminal application windows.

### 7. Panes: splitting one window into multiple views

Within a single window, you can split the screen into panes so you can see more than one thing at once:

| Keys | Effect |
|---|---|
| `Ctrl+b %` | Split the current pane **vertically** (side-by-side, left/right) |
| `Ctrl+b "` | Split the current pane **horizontally** (stacked, top/bottom) |
| `Ctrl+b <arrow key>` | Move focus to the pane in that direction |
| `Ctrl+b o` | Cycle focus to the next pane |
| `Ctrl+b z` | **Zoom** the current pane to fill the whole window temporarily (press again to un-zoom) |

💡 **Tip for remembering the split keys:** `%` visually looks like it has a diagonal slash separating two halves side by side (vertical split); `"` has two separate marks stacked (horizontal split). It's an odd mnemonic, but it sticks.

### 8. Killing panes, windows, and sessions

You'll eventually want to clean things up rather than let sessions pile up indefinitely:

| Command | Effect |
|---|---|
| `Ctrl+b x` | Kill the **current pane** (asks for confirmation) |
| `exit` (typed normally, no prefix) | Also closes the current pane/window cleanly, from inside the shell running in it |
| `Ctrl+b &` | Kill the **current window** (asks for confirmation) |
| `tmux kill-session -t name` | Kill an entire **named session**, from outside `tmux` (or from within any session) |

✅ **Best Practice:** Typing `exit` in a pane's shell is usually cleaner than `Ctrl+b x`, since it lets whatever's running there shut down normally rather than being forcibly torn down. Reach for `Ctrl+b x` mainly when a pane's program is stuck and won't respond to a normal exit.

### 9. `screen` — the legacy alternative (brief mention)

Before `tmux` existed, **`screen`** was the standard terminal multiplexer, and it solves the same basic problem (persistent, detachable sessions). You may still encounter it on older servers or in the habits of longtime sysadmins. However, `tmux` has effectively become the modern industry standard: it has a more intuitive default configuration, easier pane-splitting, an actively maintained codebase, and a far more extensive plugin/scripting ecosystem. Unless you're on a system where only `screen` is installed and you can't install anything else, `tmux` is what you should learn and reach for.

✅ **Best Practice:** If you already have muscle memory from `screen` (`Ctrl+a` as its prefix, `Ctrl+a d` to detach), it's worth deliberately re-training yourself on `tmux`'s `Ctrl+b` equivalents — mixing the two habits is a common source of "why isn't this key doing anything" confusion, covered again in Common Pitfalls below.

### 10. Shell configuration files: where your customizations live

Everything from here on — aliases, prompt, history settings — needs to live *somewhere* so it survives closing your terminal. Bash reads different configuration files depending on **how** the shell was started, and getting this right the first time saves a lot of "why doesn't my alias work" confusion later.

Two shell "flavors" matter here:

- A **login shell** is what you get when you first authenticate — classically, logging into a physical terminal, or opening a fresh SSH connection to a remote machine. Bash treats this as the start of a whole new session.
- An **interactive non-login shell** is what you get when you open a *new terminal tab or window* on a machine you're already logged into (for example, opening a new tab in your terminal application on your own already-logged-in Ubuntu desktop), or when you run `bash` by hand from an existing shell.

Bash reads different startup files for each case:

- **`~/.bash_profile`** (or `~/.profile` if `.bash_profile` doesn't exist) — read once, only for a **login shell**.
- **`~/.bashrc`** — read for every **interactive non-login shell** (like a new terminal tab).

⚠️ **The practical trap:** on a fresh SSH login, Bash reads `~/.bash_profile` (a login shell) — and *not* `~/.bashrc` — unless `~/.bash_profile` itself explicitly tells it to also load `~/.bashrc`. This is why nearly every real-world `.bash_profile`/`.profile` you'll encounter contains a small snippet that sources `.bashrc` from inside it (Detailed Explanations, below, shows exactly what that looks like). Ubuntu's default new-user setup already includes this snippet for you, which is precisely why beginners rarely notice the distinction at all — until they set up a fresh server or user account without it and their aliases mysteriously don't appear over SSH.

### 11. Aliases: shortcuts for commands you type constantly

An **alias** is a custom name you define for a longer command, so you can type the short version instead. Once defined, typing the alias runs the full command it stands for, exactly as if you'd typed the whole thing out.

```bash
alias ll='ls -la'
```

After running that, typing `ll` runs `ls -la`. Remove an alias with **`unalias`**:

```bash
unalias ll
```

An alias defined this way only lasts for your *current* shell session — close the terminal, and it's gone. To make an alias permanent, it needs to live in `~/.bashrc` (Practical Examples, below, shows this end-to-end).

### 12. Customizing your prompt with `PS1`

**`PS1`** ("prompt string 1") is the shell variable that controls what your command prompt looks like — the text printed before your cursor, every time Bash is ready for a new command. Bash builds it from a mix of literal characters and special **backslash-escape codes** that get replaced with live information each time the prompt is drawn.

A few of the most common escape codes:

| Code | Shows |
|---|---|
| `\u` | Current username |
| `\h` | Hostname (up to the first dot) |
| `\w` | Current working directory (full path, `~` shorthand for home) |
| `\W` | Just the final component of the current directory (not the full path) |
| `\$` | A `$` for a normal user, or `#` if you're root |

A simple, genuinely useful custom prompt:

```bash
PS1='\u@\h:\w\$ '
```

This isn't a deep dive into every possible `PS1` feature (colors, git-branch integration, multi-line prompts) — just enough to build one clear, informative prompt of your own. Practical Examples, below, shows this running with real output.

### 13. Tuning your command history

Bash keeps a record of commands you've run — your **history** — and several variables control how much of it is kept and how it behaves:

- **`HISTSIZE`** — how many commands are kept in memory during your *current* session.
- **`HISTFILESIZE`** — how many commands are kept in the history *file* on disk (`~/.bash_history`) across sessions.
- **`HISTCONTROL`** — controls *which* commands get recorded at all. Setting it to `ignoredups:erasedups` means: don't record a command if it's identical to the one immediately before it (`ignoredups`), and additionally erase *all* earlier duplicates of a command elsewhere in your history when it's run again (`erasedups`) — so your history stays free of repetitive clutter.
- **`shopt -s histappend`** — `shopt` (short for "shell option") toggles various Bash behaviors on and off; `histappend` specifically makes Bash **append** your session's history to the history file when the shell exits, instead of overwriting the whole file. Without it, if you have multiple terminal tabs open at once, the last one to close silently wipes out history from all the others.

---

## Detailed Explanations

### `.bashrc` vs. `.bash_profile`/`.profile` — the practical difference, spelled out

This is the single most confusing thing for people new to shell configuration, so let's be very concrete about it.

**When you open a brand-new SSH connection to a Ubuntu server**, Bash starts as a **login shell**. It looks for startup files in this order and reads the *first one it finds*, stopping there: `~/.bash_profile`, then `~/.bash_login`, then `~/.profile`. It does **not** automatically also read `~/.bashrc` on its own.

**When you open a new terminal tab on a machine you're already logged into** (or run `bash` again from inside an existing shell), Bash starts as an **interactive non-login shell**, and it reads `~/.bashrc` directly — never `.bash_profile` at all.

So if you put an alias only in `~/.bashrc`:
- A brand-new terminal tab on your desktop → sees the alias immediately (reads `.bashrc` directly).
- A fresh SSH login to a remote server → does **not** see the alias, *unless* that server's `~/.bash_profile` (or `~/.profile`) explicitly sources `.bashrc` too.

This is exactly why the standard, near-universal fix — and what Ubuntu's default `~/.bash_profile`/`~/.profile` setup already does for you — is to put a small snippet in `~/.bash_profile` that says "also go read `.bashrc`":

```bash
# inside ~/.bash_profile (or ~/.profile)
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

`. ~/.bashrc` (the leading dot is a synonym for the `source` command) reads and executes `.bashrc`'s contents right there inside the login shell, as if it had been typed directly. With this snippet in place, `.bashrc` effectively runs in *both* scenarios, and you get one single, reliable place to put all your customizations.

✅ **Best Practice:** Put aliases, `PS1` customization, and history settings in **`~/.bashrc`** — not `.bash_profile` — and rely on `.bash_profile`/`.profile` sourcing it (as shown above) to cover the login-shell case too. Reserve `.bash_profile`/`.profile` itself for things that genuinely only make sense to run once per login (like setting environment variables that child processes should inherit, e.g. `PATH` additions) — though in practice, many people keep everything in `.bashrc` alone and never touch `.bash_profile` beyond that one sourcing snippet, which is a perfectly reasonable simplification for a single-user machine.

🎯 **On the job:** You'll hit this directly the first time you provision a brand-new server or user account and your aliases "don't work" over SSH, even though they worked fine in every local terminal tab. The fix is never "aliases are broken" — it's always "check whether `.bash_profile` actually sources `.bashrc` on this particular account."

### Why edits to `.bashrc` don't take effect immediately

`.bashrc` is read **once**, at the moment a new interactive shell starts. Editing the file afterward changes what a *future* new shell will read — it does nothing to a shell that's already running, because that shell already finished reading the file before you made your edit. To apply changes to your *current* shell without closing it, you re-run the file's contents manually with:

```bash
source ~/.bashrc
```

`source` (or its shorthand, a leading dot: `. ~/.bashrc`) reads and executes a file's commands directly inside your current shell, exactly as if you'd typed each line yourself at the prompt — as opposed to running it as a separate script in a subprocess, which would apply the changes there and then immediately throw them away when that subprocess exits.

---

## Practical Examples

### Example 1 — Full `tmux` workflow: sessions, windows, panes, detach, and reattach after a simulated disconnect

```bash
tmux new -s deploy
```

Realistic output — a new, mostly-blank terminal appears, with a green status bar at the bottom:

```
[0] 0:bash*                                                    "myserver" 14:02 28-Jul-26
```

Now, inside this session, start a long-running task and split off a second pane to watch it:

```bash
./run-migration.sh
```
```
Starting migration...
Migrating table 1 of 12...
```

Without stopping it, split the window to open a second pane:

```
Ctrl+b %
```

The screen splits vertically; the new (right-hand) pane gets a fresh prompt while the migration keeps printing progress in the left pane. In the new pane:

```bash
htop
```

Now simulate a disconnect on purpose — detach cleanly:

```
Ctrl+b d
```
```
[detached (from session deploy)]
```

You're back at a plain shell prompt, as if nothing were running at all. Confirm the session is still alive:

```bash
tmux ls
```
```
deploy: 1 windows (created Tue Jul 28 14:02:00 2026)
```

Now simulate reconnecting later — from this same terminal, or a completely new SSH connection to the same machine:

```bash
tmux attach -t deploy
```

You're dropped right back into the same session — the migration's output has kept accumulating in the left pane exactly as if you'd never left, and `htop` is still running live in the right pane.

Line-by-line:
- `tmux new -s deploy` creates a **named** session called `deploy` — naming it means you can find and reattach to it by a memorable word instead of a number later.
- `./run-migration.sh` starts the actual long-running work, directly inside the session.
- `Ctrl+b %` splits the current window vertically, so you can watch a second thing (here, `htop`) alongside the migration without losing sight of either.
- `Ctrl+b d` detaches — the session and everything in it (the migration, `htop`, both panes) keeps running on the server; only your local view of it goes away.
- `tmux ls`, run from a plain shell outside `tmux`, confirms `deploy` is still alive and lists it by name.
- `tmux attach -t deploy` reattaches to that exact session by name, from anywhere, restoring the full view — this is the step that would also be what you'd run after a genuine, unplanned SSH drop, not just a deliberate detach.

💡 **Tip:** This works identically whether you detached on purpose with `Ctrl+b d` or your SSH connection simply died without warning — either way, the session survives, and `tmux attach -t deploy` brings it back.

### Example 2 — Windows and navigating between them

```
Ctrl+b c
```

A brand-new window opens (window `1`), with its own fresh shell — the migration and `htop` from window `0` keep running untouched, you're just looking at something else now.

```bash
tail -f /var/log/syslog
```

Switch back to the original window:

```
Ctrl+b p
```

You're back in window `0`, exactly where you left it. Switch directly by number:

```
Ctrl+b 1
```

Back in window `1`, still tailing the log.

Line-by-line:
- `Ctrl+b c` creates window `1` — a completely separate full-screen workspace from window `0`, within the *same* session.
- `Ctrl+b p` moves to the previous window (`0`); `Ctrl+b n` would move forward instead.
- `Ctrl+b 1` jumps directly to window `1` by its number, shown in the status bar — faster than cycling through `n`/`p` once you have several windows open.

### Example 3 — Killing a session you no longer need

```bash
tmux kill-session -t deploy
```

No output on success — the session, and everything running inside every one of its windows and panes, is terminated immediately.

⚠️ **Warning:** `tmux kill-session` does **not** ask "are you sure" — anything still running inside that session (an unfinished migration, an open editor with unsaved changes) is torn down immediately, the same as closing the terminal on a plain background job. Confirm the work inside it is actually done, or that you're fine losing it, before running this.

### Example 4 — Aliases, made permanent in `.bashrc`

First, try an alias in your current session only:

```bash
alias ll='ls -la'
ll
```
```
total 32
drwxr-xr-x  4 weki weki 4096 Jul 28 14:10 .
drwxr-xr-x 18 weki weki 4096 Jul 28 09:02 ..
-rw-r--r--  1 weki weki  220 Jul 28 09:02 .bash_logout
-rw-r--r--  1 weki weki 3771 Jul 28 09:02 .bashrc
```

That works right now, but it disappears the moment this terminal closes, because it was only ever defined in this one running shell. To make it (and a few other genuinely useful daily aliases) permanent, add them to `~/.bashrc`:

```bash
cat >> ~/.bashrc << 'EOF'

# --- Custom aliases ---
alias ll='ls -la'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias df='df -h'
alias free='free -h'
EOF
```

Apply the change to your current shell immediately, without closing it:

```bash
source ~/.bashrc
```

Now open a brand-new terminal tab (or SSH connection, assuming `.bash_profile` sources `.bashrc` as covered above) and confirm it persisted:

```bash
ll
```
```
total 32
drwxr-xr-x  4 weki weki 4096 Jul 28 14:10 .
drwxr-xr-x 18 weki weki 4096 Jul 28 09:02 ..
-rw-r--r--  1 weki weki  220 Jul 28 09:02 .bash_logout
-rw-r--r--  1 weki weki 3946 Jul 28 14:12 .bashrc
```

Remove one you decide you don't want, for the current session:

```bash
unalias grep
```

Line-by-line:
- `cat >> ~/.bashrc << 'EOF' ... EOF` **appends** (`>>`, never `>`, which would erase the whole file) a block of alias definitions to the end of `.bashrc`, using a heredoc so multiple lines get written in one command.
- `..` and `...` are aliases for quick directory navigation — genuinely used constantly once you have them.
- `alias grep='grep --color=auto'` is a common pattern: aliasing a real command's name to itself plus extra default flags. `grep` still works exactly as `grep`, just with color highlighting on by default now.
- `source ~/.bashrc` re-reads the file into the *current* shell immediately — without this, you'd have to close and reopen your terminal to see the new aliases here.
- A brand-new terminal tab shows `ll` working without you doing anything else, because it reads `.bashrc` fresh, on its own, every time it starts.
- `unalias grep` removes just that one alias for this session; permanently removing it would mean deleting or commenting out that specific line in `.bashrc` and sourcing it again.

✅ **Best Practice:** Keep your custom aliases together under one clearly commented block (`# --- Custom aliases ---`) in `.bashrc`, rather than scattering them — it makes the file far easier to scan and edit later.

### Example 5 — Customizing `PS1`

Default prompt on a typical Ubuntu system often looks like this already:

```
weki@myserver:~$
```

Set a clear, explicit version of essentially the same thing, permanently, in `.bashrc`:

```bash
cat >> ~/.bashrc << 'EOF'

# --- Custom prompt ---
PS1='\u@\h:\w\$ '
EOF
source ~/.bashrc
```

```
weki@myserver:~$ 
```

Now `cd` somewhere and watch the prompt update live:

```bash
cd /var/log
```
```
weki@myserver:/var/log$ 
```

Line-by-line:
- `\u@\h:\w\$ ` builds the prompt from username (`\u`), an `@`, hostname (`\h`), a colon, full working directory (`\w`), then `\$` (a plain `$` for a normal user), followed by a literal trailing space so your typed command doesn't visually run into the prompt.
- Moving to `/var/log` immediately updates the `\w` portion, since `PS1` is re-evaluated fresh every single time the prompt is drawn, not just once when it's set.

💡 **Tip:** Once this feels comfortable, common next steps (not required for this module) include adding ANSI color codes around each piece, or showing the current git branch — both are popular, well-documented extensions once you're ready to go further than this basic version.

### Example 6 — History customization

Add these settings to `.bashrc`:

```bash
cat >> ~/.bashrc << 'EOF'

# --- History tuning ---
HISTSIZE=5000
HISTFILESIZE=10000
HISTCONTROL=ignoredups:erasedups
shopt -s histappend
EOF
source ~/.bashrc
```

Demonstrate `ignoredups:erasedups` in action:

```bash
echo "first"
echo "second"
echo "second"
history | tail -3
```
```
  501  echo "first"
  502  echo "second"
  503  history | tail -3
```

Line-by-line:
- `HISTSIZE=5000` keeps up to 5,000 commands in memory for the current session.
- `HISTFILESIZE=10000` allows up to 10,000 commands to be kept in the `~/.bash_history` file on disk, across all past sessions.
- `HISTCONTROL=ignoredups:erasedups` means the second, identical `echo "second"` was never recorded at all as a fresh duplicate entry, and any *older* copies of that same exact command elsewhere in history get erased too — notice it appears only once in the `history` output above, even though it was typed twice in a row.
- `shopt -s histappend` ensures that when this shell exits, its history is **appended** to `~/.bash_history` rather than overwriting the whole file — critical if you regularly have more than one terminal tab open at once, since without it, whichever tab closes last would silently wipe out history the other tabs had already written.

---

## Common Pitfalls & Best Practices

- **Editing `.bashrc` and expecting the change immediately in the shell you're already sitting in.** `.bashrc` is only read once, when a shell starts — editing it later affects future shells, not your current one. Run `source ~/.bashrc` to apply changes to the shell you're already in, or just open a new terminal tab.
- **Putting aliases in `.bash_profile`/`.profile` and being confused when they don't show up in new terminal tabs.** New tabs on an already-logged-in machine read `.bashrc`, not `.bash_profile` — that file is only read for login shells (fresh SSH connections, initial logins). Keep aliases, prompt settings, and history tuning in `.bashrc`.
- **Alias name collisions with real commands.** Defining `alias ls='ls --color=auto -la'` (aliasing a command's name to a variant of itself) is a common and fine pattern, but blindly aliasing a common name to something unrelated — say, accidentally shadowing `cp` with a personal script — can cause deeply confusing behavior for yourself, or anyone else who later logs in as you and expects `cp` to behave like plain `cp`. Keep aliases either genuine shortcuts (`ll`, `..`) or small variations of the real command's own behavior, and document anything less obvious with a comment in `.bashrc`.
- **Losing `tmux` muscle memory to old `screen` habits, or vice versa.** If you've used `screen` before, its prefix is `Ctrl+a`, not `Ctrl+b` — muddling the two means pressing a prefix that does nothing in whichever tool you're actually in. If you regularly touch both, it's worth deliberately committing to `tmux` as your default and treating `screen` purely as "the thing I fall back to if `tmux` genuinely isn't installed and I can't install it."
- **Forgetting `Ctrl+b d` detaches, it doesn't exit.** Beginners sometimes look for a way to "close" `tmux` and reach for `Ctrl+b d`, then get confused when their long-running job is still going (correctly!) the next time they reattach. Detaching is not the same as stopping anything — that's the entire point of it.
- **Running `tmux kill-session` on the wrong name.** Session names can look similar (`deploy` vs `deploy2`) — always run `tmux ls` first to double check the exact name before killing anything, the same "verify before you act" discipline from signal-handling in Module 10.

✅ **Best Practice — Name every `tmux` session deliberately.** Unnamed, numbered sessions (`0`, `1`, `2`...) are easy to lose track of, especially if you open several over the course of a day. A quick `tmux new -s <purpose>` (`deploy`, `logs`, `scratch`) pays for itself the first time you need to reattach from a different machine hours later.

---

## Hands-on Exercise

**Task:**

1. Start a new named `tmux` session called `practice`.
2. Split the window into two panes. In one pane, start a long-running placeholder command (`sleep 300` works fine, standing in for a real long-running job). In the other pane, run something you can watch, like `date` on a loop, or just leave it idle.
3. Detach from the session.
4. Confirm the session is still listed and running, without reattaching yet.
5. Reattach to the session and confirm the `sleep` is still counting down (or has finished) exactly as expected.
6. Kill the session cleanly once you're done.
7. Separately: add at least three custom aliases and a history-customization block (`HISTCONTROL=ignoredups:erasedups` and `shopt -s histappend`) to your `~/.bashrc`, then `source` it and confirm one alias works.

Try this yourself in a real terminal before reading the solution below.

### Solution

```bash
# 1. Start a new named session
tmux new -s practice
```
```
[0] 0:bash*                                                    "myserver" 15:00 28-Jul-26
```

```bash
# 2. Start a long-running placeholder in this pane
sleep 300
```

Split off a second pane without interrupting it:

```
Ctrl+b %
```

In the new pane:

```bash
date
```
```
Mon Jul 28 15:00:12 UTC 2026
```

```bash
# 3. Detach
```
```
Ctrl+b d
```
```
[detached (from session practice)]
```

```bash
# 4. Confirm it's still running, without reattaching
tmux ls
```
```
practice: 1 windows (created Mon Jul 28 15:00:00 2026)
```

```bash
# 5. Reattach
tmux attach -t practice
```

The left pane still shows `sleep 300` counting down (or already finished, depending on how much time passed) and the right pane still shows the `date` output from earlier, exactly as left.

```bash
# 6. Kill the session when done
tmux kill-session -t practice
```

```bash
# 7. Aliases and history settings in .bashrc
cat >> ~/.bashrc << 'EOF'

# --- Custom aliases ---
alias ll='ls -la'
alias ..='cd ..'
alias gs='git status'

# --- History tuning ---
HISTCONTROL=ignoredups:erasedups
shopt -s histappend
EOF
source ~/.bashrc
```

```bash
ll
```
```
total 24
drwxr-xr-x  4 weki weki 4096 Jul 28 15:05 .
drwxr-xr-x 18 weki weki 4096 Jul 28 09:02 ..
-rw-r--r--  1 weki weki  220 Jul 28 09:02 .bash_logout
-rw-r--r--  1 weki weki 4102 Jul 28 15:05 .bashrc
```

Explanation: Splitting the window with `Ctrl+b %` let the placeholder long-running job (`sleep 300`, standing in for something like the migration script from earlier in this module) keep running visibly in one pane while a second pane stayed free for other work — exactly the layout you'd use to babysit a real job while doing something else alongside it. Detaching with `Ctrl+b d` proved the session doesn't stop just because your view of it goes away; `tmux ls`, run from a completely plain shell with no active session, confirmed `practice` was still alive before I ever reattached, which is the same check you'd run after a genuine unplanned disconnect. Reattaching showed both panes exactly as left. `tmux kill-session -t practice` cleanly tore the whole thing down once there was nothing left worth keeping. Separately, appending to `.bashrc` (never overwriting it with `>`) and then `source`-ing it applied the new aliases and history behavior to the current shell immediately, and `ll` confirmed one of them works without needing to open a new terminal at all.

✅ Exercise complete — you've run a full `tmux` survive-a-disconnect cycle with multiple panes, and permanently customized your own shell environment with aliases and smarter history behavior.

---

## ✅ Module Completion Checklist

- [ ] I can explain why terminal multiplexers matter — surviving SSH disconnects and running multiple panes/windows inside one terminal session
- [ ] I can start, detach from, and reattach to a `tmux` session (`tmux new -s`, `Ctrl+b d`, `tmux attach -t`), and list all running sessions with `tmux ls`
- [ ] I can create and navigate `tmux` windows (`Ctrl+b c`, `Ctrl+b n`/`p`, `Ctrl+b 0-9`) and panes (`Ctrl+b %`, `Ctrl+b "`, `Ctrl+b <arrow>`)
- [ ] I can kill `tmux` panes, windows, and sessions cleanly, and explain why `tmux` has replaced `screen` as the modern standard
- [ ] I can explain the practical difference between `~/.bashrc` and `~/.bash_profile`/`~/.profile` — which one runs on a fresh SSH login versus a new terminal tab, and why that determines where you put things
- [ ] I can create, use, and remove aliases (`alias`, `unalias`), and make them permanent by adding them to `.bashrc`
- [ ] I can customize my shell prompt with a simple, readable `PS1` value
- [ ] I can tune my command history with `HISTSIZE`, `HISTFILESIZE`, `HISTCONTROL`, and `shopt -s histappend`

## Next Step

Continue to [Module 14: Error Handling, Traps & Debugging](../module14-error-handling-debugging/)
