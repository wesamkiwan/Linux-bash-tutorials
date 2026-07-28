# Module 10: Process Management & Job Control 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-9 (Shell Fundamentals through Text Processing Power Tools)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain what a process is, what a PID and PPID are, and how every process on a Linux system fits into a single process tree rooted at PID 1 (`systemd` on Ubuntu)
- [ ] Read the output of `ps`, `ps aux`, and `ps -ef` confidently, including the USER, PID, %CPU, %MEM, STAT, START, and TIME columns, and visualize relationships with `pstree`
- [ ] Monitor a live system with `top`, and install and use `htop` as a friendlier alternative, reading load average and interactive keys
- [ ] Identify the four common process states (Running, Sleeping, Stopped, Zombie) and explain why zombie processes occur
- [ ] Run commands in the background with `&`, and manage them with `jobs`, `fg`, `bg`, and job IDs like `%1`
- [ ] Detach a background job from the terminal with `nohup` and `disown`, and explain why `SIGHUP` kills background jobs when a terminal closes
- [ ] Explain what a signal is, name the key signals (`SIGTERM`, `SIGKILL`, `SIGHUP`, `SIGINT`) with their numbers, and send them with `kill`, `killall`, and `pkill`
- [ ] Look under the hood at `/proc/<pid>/status` and `/proc/<pid>/cmdline` to see where process information actually comes from

---

## Module Goal

By the end of this module, you'll be able to confidently take control of anything running on a Linux machine — start it, watch it, move it to the background, detach it safely from your terminal, and stop it the right way when it needs to stop.

🎯 **On the job:** Picture two very common production scenarios. First: you SSH into a server and notice it's sluggish. You run `top` and see a single process pinned at 100% CPU, quietly starving everything else — maybe a runaway script, maybe a stuck cron job. You need to identify it, decide whether to kill it, and do so without corrupting whatever it was working on. Second: you kick off a long-running deployment or data-migration script over SSH, and you need to walk away from your desk — but if you just close your laptop lid, the process dies with your session unless you've protected it first. This module teaches you both skills: reading a system's vital signs, and controlling the lifetime and shutdown of anything you run on it. These are baseline, every-single-day skills for anyone operating Linux servers professionally.

---

## Core Concepts

### 1. What is a process?

A **process** is a running instance of a program. The program itself — say, `/usr/bin/python3` — is just a file sitting on disk, inert. The moment you run it, the kernel loads it into memory, allocates it CPU time, and gives it its own private workspace (memory, open files, etc.). That running, active thing is a process. You can run the same program twice and get two completely separate processes, each with its own state.

Every process is issued a **PID** (Process ID) — a unique number the kernel assigns when the process is created. No two processes running at the same time share a PID.

Every process (except the very first one) also has a **PPID** (Parent Process ID) — the PID of the process that created it. When a running process starts a new process (for example, your shell starting `ls` when you type it), the new process is called a **child**, and the process that started it is its **parent**.

💡 **Analogy — an org chart:** Think of every process as an employee at a company. Each employee has an employee ID (PID) and a manager (PPID) — the person who hired them for this task. Employees can hire their own employees (child processes), who report up through them. Trace the "manager" chain far enough for anyone in the company, and you eventually reach the CEO. On Linux, that CEO is a single process with PID 1.

### 2. PID 1 and the process tree

On Ubuntu/Debian (and virtually every modern Linux distribution), **PID 1** is `systemd` — the very first process the kernel starts when the machine boots. Every other process on the system is either a direct child of `systemd` or a descendant of one of its children, forming a single branching **process tree** with `systemd` as the root.

💡 **Historical note:** Older systems (and some minimal containers) use `init` (specifically `sysvinit`) as PID 1 instead. You'll still hear people say "the init process" generically to mean "whatever owns PID 1," even on a `systemd` machine — the terminology stuck around after the tool changed.

If a parent process dies before its children do, those children become **orphans**. On Linux, an orphaned process gets "re-parented" — its PPID is reassigned, typically to PID 1 (`systemd`), which adopts it so every process always has *some* valid parent to be cleaned up by later.

### 3. Viewing processes: `ps`

`ps` ("process status") prints a snapshot of processes at the moment you run it — it does **not** update live, unlike `top` (Concept 5). The plain, bare `ps` command only shows processes attached to your *current terminal*, which is rarely enough. Two much more useful invocations:

- `ps aux` — the traditional BSD-style syntax, showing **every** process on the system, in a wide, information-rich format.
- `ps -ef` — the UNIX/POSIX-style syntax, showing largely the same information with slightly different columns and formatting.

Both are extremely common in the real world — different teams and different scripts prefer one or the other, so you should be comfortable reading both.

### 4. Reading `ps aux` columns

`ps aux` output has one process per line, with these columns:

| Column | Meaning |
|---|---|
| **USER** | Which user account owns/started this process |
| **PID** | The process ID |
| **%CPU** | Percentage of CPU time this process is currently consuming |
| **%MEM** | Percentage of physical RAM this process is currently using |
| **VSZ** | Virtual memory size (in KB) |
| **RSS** | Resident Set Size — actual physical RAM currently in use (in KB) |
| **TTY** | The terminal associated with the process, or `?` if none (e.g. a background service) |
| **STAT** | The process's current state code (see Concept 6) |
| **START** | The time or date the process started |
| **TIME** | Total accumulated CPU time actually used (not wall-clock/elapsed time!) |
| **COMMAND** | The command and arguments that launched this process |

⚠️ **Common confusion:** `TIME` is **not** "how long ago it started." A process that's been idle for three days but only ever used 2 seconds of actual CPU work will show `TIME` of `00:00:02`. Use `START` (or `etime` in custom `ps` formats) if you want elapsed wall-clock time.

### 5. Visualizing relationships: `pstree`

`ps` gives you a flat list. `pstree` gives you the actual **tree shape** — indenting children under their parents, visually, so you can see at a glance which process launched which. This is the fastest way to answer "what's the parent of this thing?" or "did my script actually spawn three worker processes like I expected?"

### 6. Process states

Every process is, at any given instant, in one of a small number of **states**, shown in the `STAT` column of `ps`:

| State code | Name | Meaning |
|---|---|---|
| `R` | **Running** | Currently executing on the CPU, or ready and waiting its turn |
| `S` | **Sleeping** (interruptible) | Waiting for something — input, a timer, a lock — and can be woken up early by a signal |
| `D` | **Uninterruptible sleep** | Waiting on something low-level, usually disk I/O, and *cannot* be interrupted by a signal until it finishes |
| `T` | **Stopped** | Execution is paused, usually because it received `SIGSTOP` or you pressed Ctrl+Z |
| `Z` | **Zombie** | Has finished running, but its exit status hasn't been collected yet by its parent |

**Zombie processes**, explained: when a process finishes, it doesn't vanish instantly. The kernel keeps a small record — its exit code and some accounting info — until the parent process calls `wait()` to formally collect that result (this is how a parent knows whether its child succeeded or failed, exactly like checking `$?` after a command, which you learned in Module 6). Until the parent collects it, the finished process sits in the process table as a **zombie**: it holds no real memory or CPU, just a placeholder entry. A zombie is usually harmless and short-lived — the parent typically collects it within a fraction of a second. Zombies become a genuine problem only when a badly written parent process *never* calls `wait()`, leaving zombies to accumulate indefinitely; the fix is almost always to fix (or restart) the parent, since a zombie itself cannot be killed — there's no living process left to signal.

💡 **Analogy:** A zombie is like an employee who has already left the building, but HR hasn't officially closed their file yet — the record still exists on paper, but there's no actual person consuming a desk or a paycheck anymore. Once HR (the parent) processes the paperwork, the record disappears.

### 7. Live monitoring: `top`

`top` is `ps`'s live, continuously-refreshing cousin — it repaints the screen (by default every few seconds) with the current busiest processes at the top. It comes preinstalled on virtually every Linux system, which makes it your reliable fallback everywhere.

The top of the `top` screen shows summary information, including the **load average** — three numbers representing the average number of processes wanting CPU time over the last 1, 5, and 15 minutes. Roughly speaking, a load average at or below your number of CPU cores means the system is keeping up; well above it means processes are queuing for CPU time.

Useful interactive keys inside `top`:

| Key | Effect |
|---|---|
| `k` | Kill a process (prompts for a PID, then a signal) |
| `q` | Quit `top` |
| `P` | Sort by %CPU (often the default) |
| `M` | Sort by %MEM |
| `1` | Toggle showing per-core CPU breakdown |

### 8. `htop` — the modern, friendlier alternative

`htop` is a widely used, more visual replacement for `top`: colorized bars for CPU/memory usage per core, mouse support, easier scrolling, and a friendlier way to select and signal a process without memorizing a PID first. It is **not** installed by default on most Ubuntu/Debian systems, so you install it with:

```bash
sudo apt update
sudo apt install htop
```

✅ **Best Practice:** Learn `top` first and thoroughly, since it's guaranteed to be present on any Linux box you SSH into, even minimal servers where you can't (or shouldn't) install extra packages. Reach for `htop` when it's available and you want a faster, more visual read of the same information.

### 9. Jobs vs. processes

A quick but important distinction: a **process** is a kernel-level concept — any running program, tracked by PID, regardless of who started it or how. A **job**, by contrast, is a shell-level concept — it exists only within *your current interactive shell session*, and refers specifically to a command (or pipeline of commands) *you* started from that shell. Every job corresponds to one or more processes, but plenty of processes on the system (system services, other users' processes, processes from other terminal sessions) are not jobs of your shell at all — your shell has no job-table entry for them, and `jobs`, `fg`, `bg` know nothing about them.

### 10. Running in the background with `&`

Normally, when you run a command, your shell waits for it to finish before giving you the prompt back — this is running it in the **foreground**. Appending `&` to the end of a command instead starts it in the **background**: the shell immediately hands you back the prompt, while the command keeps running independently.

```bash
long-task.sh &
```

The shell prints a **job ID** in brackets and a PID right away, e.g. `[1] 12345`, and that job now exists in your shell's job table.

### 11. Managing jobs: `jobs`, `fg`, `bg`

| Command | Effect |
|---|---|
| `jobs` | Lists all jobs known to the *current shell*, with their job IDs and status (Running/Stopped) |
| `fg` | Brings the most recent job to the **foreground** (waits for it, connects it to your keyboard again) |
| `fg %2` | Brings job number 2 specifically to the foreground |
| `bg` | Resumes the most recently **stopped** job, running it in the background |
| `bg %2` | Resumes job number 2 in the background specifically |
| `Ctrl+Z` | Suspends (pauses) the current foreground job — it becomes **Stopped**, not terminated |

Job IDs are referenced with a percent sign — `%1`, `%2`, and so on — matching the numbers `jobs` shows in brackets.

💡 **Analogy:** If signals are instructions you shout at an employee (Concept 13), `&`, `bg`, and `fg` are about where that employee is physically working — at their desk in full view (foreground, blocking your own attention) or quietly in a back room (background, freeing you to do something else) — while `jobs` is just you checking who's currently on the clock for you specifically.

### 12. Detaching from the terminal: `nohup` and `disown`

Here's the trap that catches almost everyone at least once: a background job (`&`) is still tied to the terminal session that launched it. If that terminal closes — you close the window, your SSH connection drops, your laptop sleeps — the shell sends a signal called **`SIGHUP`** ("hang up," a name inherited from the days of literal telephone modems hanging up) to every job still attached to it. By default, most programs respond to `SIGHUP` by terminating. That means your background job dies the instant your terminal session ends, unless you've explicitly protected it.

Two tools solve this:

- **`nohup`** — literally "no hang up." Launch a command with `nohup command &`, and it will **ignore** `SIGHUP` entirely, so it survives your terminal closing. By default, `nohup` also redirects the command's output to a file called `nohup.out` in the current directory (since there's no longer a terminal to print to).
- **`disown`** — run *after* a job is already backgrounded (`command &` then `disown %1`), it removes the job from your shell's job table entirely. Once disowned, your shell no longer considers it "yours," and critically, it will not send that process a `SIGHUP` when the shell exits — because as far as the shell is concerned, it's no longer tracking it at all.

✅ **Best Practice:** For anything you're about to start knowing you'll want to walk away from (a long deploy, a multi-hour data job), start it with `nohup command &` from the very beginning. Don't rely on remembering to `disown` it after the fact — that only works if you remember before the terminal closes.

🎯 **On the job:** This is precisely the scenario in the Module Goal above — the vanishing deploy script. `./deploy.sh &` looks safe because the prompt comes back immediately, but close that SSH session and the deploy dies partway through, possibly leaving the target system in a half-updated state. `nohup ./deploy.sh &` (or running it inside a terminal multiplexer like `tmux`, previewed in a later module) is what actually survives.

### 13. Signals — what they are

A **signal** is a small, standardized notification the kernel (or another process, or you) can send to a running process, asking it to do something — most commonly, to stop. It's not data, and it's not a request the process can inspect and decide to answer later — it interrupts the process immediately with a simple, numbered message.

💡 **Analogy — instructions you shout at a worker:** If a process is an employee at their desk, a signal is a specific word you shout across the room, and different words carry different weight. "Hey, wrap up when you get a chance" (a polite request they can finish their current sentence before responding to) is very different from "the fire alarm is going off, walk out right now" (something the building enforces regardless of what they're doing) versus "leave when convenient" (barely urgent at all). Signals work the same way — some can be caught and handled gracefully by the program, and one absolutely cannot be ignored no matter what.

### 14. Key signals to know

| Number | Name | Default action | Can be caught/ignored? | Typical use |
|---|---|---|---|---|
| 1 | `SIGHUP` | Terminate | Yes | Sent when a controlling terminal closes; also traditionally used to tell a daemon "reload your config" |
| 2 | `SIGINT` | Terminate | Yes | What Ctrl+C sends — an interactive interrupt request |
| 9 | `SIGKILL` | Terminate | **No — never** | Force-kill; the kernel destroys the process immediately, no cleanup |
| 15 | `SIGTERM` | Terminate | Yes | The **default** signal `kill` sends — a polite "please shut down" request |

`SIGTERM` (15) and `SIGKILL` (9) are the two you'll use constantly, and the difference between them matters enormously:

- **`SIGTERM`** asks the process to terminate, but lets the process **catch** that signal and run its own cleanup code first — closing open files, finishing writing a database record, releasing a lock — before actually exiting. This is the graceful, default option, and it's what plain `kill <pid>` sends.
- **`SIGKILL`** cannot be caught, blocked, or ignored by the target process at all — the kernel terminates it immediately, with **zero** opportunity for the process to clean up anything. Anything mid-write when `SIGKILL` arrives is simply abandoned exactly where it stood.

✅ **Best Practice:** Always try `SIGTERM` first (plain `kill <pid>`) and give the process a few seconds to exit on its own. Reach for `SIGKILL` (`kill -9 <pid>`) only if it refuses to respond — it should be your last resort, not your first move.

🎯 **On the job:** Imagine a database process mid-write when you kill it. `kill <pid>` (SIGTERM) gives it a chance to finish or roll back that write safely before exiting. `kill -9 <pid>` (SIGKILL) yanks it out instantly, potentially leaving a half-written record, a corrupted file, or a lock that never gets released — the kind of production incident that turns a two-minute restart into a multi-hour data-recovery exercise.

### 15. Sending signals: `kill`, `killall`, `pkill`, `kill -l`

- **`kill <pid>`** — sends `SIGTERM` (the default) to the process with that specific PID.
- **`kill -9 <pid>`** or **`kill -SIGKILL <pid>`** — sends `SIGKILL` to that PID instead. Both the numeric form and the name form work identically.
- **`kill -l`** — lists every signal name and number the system supports, a quick reference when you forget one.
- **`killall <name>`** — sends a signal (`SIGTERM` by default) to **every** process whose command name matches exactly.
- **`pkill <pattern>`** — sends a signal to every process whose command line matches a given pattern (more flexible matching than `killall`, since it can match partial names/patterns, not just an exact command name).

⚠️ **Warning:** `killall` and `pkill` match by **name/pattern**, not by a specific PID you've verified — this is powerful but dangerous. It's alarmingly easy to match more processes than you intended (imagine `pkill python` on a shared server running several unrelated Python services). Always confirm exactly what you're about to signal with `ps`/`pgrep` first (Common Pitfalls, below, covers this in depth).

### 16. Where this all comes from: `/proc`

You met the filesystem in earlier modules; `/proc` is a special, virtual filesystem that doesn't represent real files on disk at all — it's the kernel exposing its own live internal state as if it were a directory of files, generated on the fly whenever you read them. Every running process gets its own subdirectory, named after its PID: `/proc/1234/`.

Two files in there are directly useful for what this module covers:

- **`/proc/<pid>/status`** — a human-readable breakdown of that process's state, including its name, PID, PPID, current state, and memory usage — the exact raw data `ps` and `top` are built on top of.
- **`/proc/<pid>/cmdline`** — the exact command and arguments that process was launched with (note: the bytes are separated by null characters instead of spaces, so it can look a little jumbled printed directly).

🎯 **On the job:** Every process-inspection tool you've learned in this module — `ps`, `top`, `htop`, `pstree` — is, under the hood, just reading and parsing files inside `/proc`. Knowing this ties directly back to the "everything is a file" philosophy from earlier modules, and it means that if a tool is ever missing on a minimal system, you can fall back to reading `/proc` directly yourself.

---

## Detailed Explanations

### Why `ps aux` and `ps -ef` look different but overlap so much

`ps` predates POSIX standardization, and two option "dialects" survived: the BSD style (`ps aux`, no dash before the letters) and the UNIX/POSIX style (`ps -ef`, with a dash). Both walk the exact same underlying process table (ultimately reading from `/proc`); they just differ in which columns they show by default and how you spell the flags. `ps -ef` swaps `%CPU`/`%MEM`/`VSZ`/`RSS` for `PPID`, `C`, and a more precise `STIME`, and its `CMD` column is the equivalent of `aux`'s `COMMAND`. In real jobs you'll see both used interchangeably depending on which team or script you're reading — recognize both rather than picking a favorite.

### Why `SIGKILL` can't be caught, and what that costs you

A process normally installs its own **signal handlers** — small pieces of its own code that run when a particular signal arrives, letting it decide how to respond (for example, "on `SIGTERM`, flush my write buffer to disk, close my network connections, then exit"). `SIGKILL` is special-cased at the kernel level specifically so it *cannot* be intercepted this way — by design, there must always be a way to guarantee a process stops, even a malfunctioning or malicious one that's deliberately ignoring every other signal. The price of that guarantee is that the process gets absolutely no chance to run any of its own shutdown logic — nothing it would have done in a `SIGTERM` handler happens at all.

### Why closing a terminal doesn't kill a `nohup`-launched job

When your shell process itself is about to exit (because you closed the terminal, or your SSH session dropped), it sends `SIGHUP` to every job still listed in its job table — this is standard, longstanding shell behavior, not something unique to any one terminal program. A plain background job (`command &`) is still in that table and still uses the default `SIGHUP` behavior (terminate), so it dies right along with the shell. `nohup` works by explicitly telling the *process itself* to ignore `SIGHUP` before it even runs — so when that signal arrives, the process simply shrugs it off and keeps going, orphaned but alive, re-parented to `systemd` as described in Concept 2.

### Stopped vs. killed — `Ctrl+Z` isn't `Ctrl+C`

It's worth being very precise about a common beginner mix-up: `Ctrl+C` sends `SIGINT`, which by default **terminates** the foreground process (though a program can choose to catch and ignore it). `Ctrl+Z` sends `SIGTSTP`, which by default **suspends** the process — it's paused, marked `T` (Stopped) in `ps`, still fully present in memory, and can be resumed later exactly where it left off with `fg` or `bg`. Nothing is lost with Ctrl+Z; the process is just frozen, not destroyed.

---

## Practical Examples

### Example 1 — Reading a real `ps aux` snapshot

```bash
ps aux | head -6
```

Realistic output:
```
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1   0.0  0.1 168420 11284 ?        Ss   09:02   0:03 /sbin/init
root       842   0.0  0.2  22536 10112 ?        Ss   09:02   0:00 /usr/sbin/sshd -D
weki      1103   0.0  0.3  21968 12500 pts/0    Ss   09:04   0:00 -bash
weki      2087  97.5  4.1 912340 84212 pts/0    R+   10:31   3:12 python3 process_orders.py
weki      2140   0.0  0.1  19212  3312 pts/0    R+   10:33   0:00 ps aux
```

Line-by-line:
- `USER root`, `PID 1`, `COMMAND /sbin/init` — this is `systemd`, PID 1, the root of the whole process tree (Concept 2). Note: on many Ubuntu systems the binary is genuinely `systemd`; some environments symlink or reference it as `init` for compatibility — either way, PID 1 is what matters.
- `sshd -D` (PID 842) is a system service, no controlling terminal (`TTY` is `?`), state `Ss` (Sleeping, session leader) — normal for a background service just waiting for connections.
- `-bash` (PID 1103) is your own interactive shell, attached to `pts/0` (a pseudo-terminal, i.e. your SSH/terminal session).
- `python3 process_orders.py` (PID 2087) is the one to worry about: **97.5% CPU**, state `R+` (Running, foreground). This is the "runaway process eating CPU" from the Module Goal — exactly what you'd investigate further before deciding whether to signal it.
- `ps aux` itself (PID 2140) shows up in its own output — a process inspecting the process table is, itself, a process.

💡 **Tip:** `ps aux --sort=-%cpu | head` sorts every process by CPU descending, so the worst offender is right at the top instead of buried in a long list.

### Example 2 — `pstree`: seeing the actual parent/child shape

```bash
pstree -p weki
```

Realistic output:
```
bash(1103)───python3(2087)─┬─python3(2091)
                            └─python3(2092)
```

Line-by-line:
- `bash(1103)` is the parent shell.
- `python3(2087)` is the main script you started from that shell — its PPID is `1103`.
- `python3(2091)` and `python3(2092)` are two **worker** processes that `2087` itself spawned — their PPID is `2087`, not `1103`. `ps aux` alone would show all three as a flat list; `pstree` immediately shows you the actual hierarchy, which is exactly what you need before deciding "do I kill just the top one, or does it leave orphaned workers behind?"

### Example 3 — Background jobs with `jobs`, `fg`, and `bg`

```bash
sleep 300 &
```
```
[1] 15820
```

```bash
jobs
```
```
[1]+  Running                 sleep 300 &
```

```bash
fg %1
```
```
sleep 300
```
*(the prompt now hangs, waiting — press Ctrl+Z)*
```
^Z
[1]+  Stopped                 sleep 300
```

```bash
bg %1
```
```
[1]+ sleep 300 &
```

```bash
jobs
```
```
[1]+  Running                 sleep 300 &
```

Line-by-line:
- `sleep 300 &` starts a 300-second sleep in the background; the shell immediately reports job `[1]` with PID `15820` and gives the prompt back.
- `jobs` confirms it's job `1`, currently `Running`.
- `fg %1` brings it into the foreground — now your terminal is attached to it and waiting, just like if you'd typed `sleep 300` directly without `&`.
- `Ctrl+Z` sends `SIGTSTP`, suspending it — it's now `Stopped` (paused, not terminated), and you get your prompt back immediately.
- `bg %1` resumes the stopped job, but in the background this time — it starts running again, counting down, without blocking your terminal.

🎯 **On the job:** This exact sequence — background it, check on it, bring it up briefly to see live output, push it back to the background — is a completely normal way to babysit a long-running task without dedicating your whole terminal session to it.

### Example 4 — `nohup` and surviving a closed terminal

```bash
nohup ./long-deploy.sh &
```
```
[1] 16044
nohup: ignoring input and appending output to 'nohup.out'
```

```bash
jobs
disown %1
exit
```

Then, from a **new** terminal/SSH session:

```bash
ps aux | grep long-deploy
```
```
weki      16044  0.4  0.2  21212  8100 ?        S    11:02   0:01 /bin/bash ./long-deploy.sh
```

```bash
tail -f nohup.out
```

Line-by-line:
- `nohup ./long-deploy.sh &` launches the script backgrounded, with `SIGHUP` immunity baked in from the start. `nohup` immediately tells you it's redirecting output to `nohup.out` since there's no terminal to print to anymore once you disconnect.
- `disown %1` additionally removes it from this shell's own job table — belt-and-suspenders alongside `nohup`, so this shell won't even attempt to signal it on exit.
- After `exit` closes this terminal entirely (simulating a dropped SSH session or closed window), the script is still alive — `ps aux` from a completely fresh session finds it running under PID `16044`, now with `TTY ?` and reparented (its original parent shell is gone).
- `tail -f nohup.out` lets you check on its progress after the fact, since its output has been quietly accumulating in that file the whole time.

⚠️ **Warning:** If you skip `nohup` and just run `./long-deploy.sh &` then close the terminal, `SIGHUP` gets sent and the script dies immediately, likely mid-step — the "vanishing deploy script" problem from this module's opening scenario.

### Example 5 — `kill` (SIGTERM) vs. `kill -9` (SIGKILL) and why it matters

Imagine a script that's mid-way through writing an important file:

```bash
cat > careful-writer.sh << 'EOF'
#!/bin/bash
trap 'echo "Caught SIGTERM — finishing write and cleaning up..."; echo "done" > /tmp/writer-status.txt; exit 0' TERM

echo "Writer started, PID $$"
while true; do
    sleep 1
done
EOF
chmod +x careful-writer.sh
./careful-writer.sh &
```
```
Writer started, PID 16210
[1] 16210
```

```bash
kill 16210
```
```
Caught SIGTERM — finishing write and cleaning up...
[1]+  Done                    ./careful-writer.sh
```

```bash
cat /tmp/writer-status.txt
```
```
done
```

Now compare with `SIGKILL` against a fresh copy:

```bash
./careful-writer.sh &
```
```
Writer started, PID 16333
[1] 16333
```

```bash
rm -f /tmp/writer-status.txt
kill -9 16333
```
```
[1]+  Killed                  ./careful-writer.sh
```

```bash
cat /tmp/writer-status.txt
```
```
cat: /tmp/writer-status.txt: No such file or directory
```

Line-by-line:
- `trap '...' TERM` (a preview of trap handling, covered fully in a later module) registers a handler that runs when `SIGTERM` arrives — here, it writes a "done" status file before actually exiting.
- `kill 16210` sends the default signal, `SIGTERM`. The handler runs, writes its file, and exits cleanly — `Done` in the job-status line confirms a normal exit.
- `kill -9 16333` sends `SIGKILL`. The kernel destroys the process **immediately** — the `trap` handler never runs at all, no matter what it was written to do, because `SIGKILL` cannot be caught. The job-status line says `Killed`, and the status file that would have proven a graceful shutdown never gets created.

🎯 **On the job:** Swap `careful-writer.sh` for a real database engine, a payment processor, or a file-transfer job, and the stakes become obvious: `SIGTERM` gives production software a chance to finish its own critical cleanup; `SIGKILL` guarantees it never gets that chance. This is exactly why `SIGTERM` first, `SIGKILL` only as a last resort, is the standing rule.

### Example 6 — Finding and signaling by name safely with `pgrep`/`pkill`

```bash
pgrep -a python3
```
```
2087 python3 process_orders.py
2091 python3 process_orders.py --worker
2092 python3 process_orders.py --worker
```

```bash
kill 2087
```

```bash
pgrep -a python3
```
```
(no output — nothing left running)
```

Line-by-line:
- `pgrep -a python3` (a read-only "preview" of `pkill`'s exact matching, with `-a` showing the full command line) confirms exactly which PIDs match before touching anything — this is the safety check the Common Pitfalls section below insists on.
- `kill 2087` targets just the main process by its specific, verified PID, using default `SIGTERM`. Because it was the parent, its two worker children exit along with it once it shuts down cleanly (a well-behaved parent stops its own children as part of its cleanup).
- A follow-up `pgrep -a python3` with no output confirms the whole family is gone — cheaper and safer than blindly trusting `pkill python3` would have matched only what you intended.

---

## Common Pitfalls & Best Practices

- **Jumping straight to `kill -9` instead of trying `SIGTERM` first.** `kill -9` feels satisfying because it's instant and never fails to remove the process from the table, but it gives the target zero chance to clean up — see Example 5. Always try plain `kill <pid>` (SIGTERM) first, wait a few seconds, and escalate to `-9` only if it's still there.
- **Forgetting `nohup` and losing a long job.** A background job (`&` alone) is still tied to your terminal session — close it, and `SIGHUP` takes the job down with it, possibly mid-task. Start anything you intend to walk away from with `nohup command &` from the very first moment, not as an afterthought.
- **Killing by name with `pkill`/`killall` without checking first.** These match by pattern or exact name across the **entire system**, not just your own processes — on a shared server, `pkill python` might take down someone else's unrelated service too. Always run the read-only equivalent first (`pgrep -a <pattern>` or `ps aux | grep <name>`) to see exactly what would be matched before you actually signal anything.
- **Confusing `TIME` (CPU time used) with how long a process has been running.** Check `START` (or `ps -o etime` for elapsed time) if you actually want "how long ago did this start," not `TIME`.
- **Assuming a zombie process can be killed directly.** A zombie has already finished executing — there's nothing left to signal. The only fix is dealing with its parent (which isn't calling `wait()`), not signaling the zombie's own PID.
- **Mixing up `Ctrl+C` and `Ctrl+Z`.** `Ctrl+C` (`SIGINT`) generally terminates; `Ctrl+Z` (`SIGTSTP`) only suspends, and the job is still fully alive until you `fg`/`bg` it or explicitly kill it — an accidentally `Ctrl+Z`'d process left forgotten is a classic "why is this still using memory" surprise later.
- **Piping `ps aux` output straight into `kill` from a one-liner without checking it first.** Building automated `kill $(ps aux | grep pattern | awk '{print $2}')`-style one-liners is fragile — the `grep` itself often matches its own process line unless carefully excluded, and blind automation of a destructive command deserves a manual double-check first, especially in production.

✅ **Best Practice — The graceful-shutdown mindset:** Before signaling anything, ask "do I know exactly which PID this is, and have I given it a chance to shut down on its own terms?" If the answer to either is no, slow down and verify with `ps`/`pgrep` before you `kill`.

---

## Hands-on Exercise

**Task:**

1. Start a long-running background command that simulates real work: `sleep 600 &` (10 minutes).
2. Use `jobs` to confirm it's running, and note its job ID and PID.
3. Detach it safely from your terminal so it would survive you closing the session — even though it's already running, use `disown` to remove it from the shell's job table (in a real scenario, you'd instead have started it with `nohup` from the beginning; this step shows the after-the-fact alternative).
4. Confirm, using `ps` or `pgrep`, that the process is still there and find its exact PID.
5. Terminate it **gracefully** — try `SIGTERM` first via plain `kill`, confirm it's gone, and only reach for `kill -9` if (hypothetically) it hadn't responded.

Try this yourself in a real terminal before reading the solution below.

### Solution

```bash
# 1. Start a long-running background job
sleep 600 &
```
```
[1] 18402
```

```bash
# 2. Confirm it's running and note the job ID / PID
jobs
```
```
[1]+  Running                 sleep 600 &
```
*(Job ID: `1`; PID: `18402`, from the earlier `[1] 18402` line)*

```bash
# 3. Detach it from the shell's job table
disown %1
jobs
```
```
(no output — the shell no longer tracks it as a job)
```

```bash
# 4. Confirm it's still alive and find its PID independently
pgrep -a sleep
```
```
18402 sleep 600
```

```bash
# 5a. Terminate gracefully with SIGTERM first
kill 18402
```

```bash
# Confirm it's gone
pgrep -a sleep
```
```
(no output — process 18402 is gone)
```

```bash
# 5b. (Hypothetical) if it had NOT responded to SIGTERM, only then escalate:
kill -9 18402
```

Explanation: I never guessed at a PID — every step confirmed it independently, first with `jobs` (while it was still a tracked job), and again with `pgrep -a sleep` after disowning it, which matches by process name and prints the full command line so I can be certain I'm targeting the right process before signaling anything. `disown` removed it from *this shell's* bookkeeping without stopping it — it kept running the entire time, exactly mimicking what would happen if I'd closed this terminal after using `nohup` from the start. I sent plain `kill` (SIGTERM, the graceful default) first rather than jumping to `-9`, giving the process every chance to exit cleanly — since `sleep` has no cleanup work to do, it exits immediately either way, but the *habit* of trying SIGTERM first is what this exercise is really building. The `kill -9` step is shown only as the escalation path you'd use if the graceful signal had been ignored — in this exercise it's never actually needed.

✅ Exercise complete — you've backgrounded a job, detached it from your terminal, verified it independently through the process table, and shut it down using the graceful-first, force-as-last-resort signal discipline.

---

## ✅ Module Completion Checklist

- [ ] I can explain what a process is, what a PID and PPID are, and how every process on a Linux system fits into a single process tree rooted at PID 1 (`systemd` on Ubuntu)
- [ ] I can read the output of `ps`, `ps aux`, and `ps -ef` confidently, including the USER, PID, %CPU, %MEM, STAT, START, and TIME columns, and visualize relationships with `pstree`
- [ ] I can monitor a live system with `top`, and install and use `htop` as a friendlier alternative, reading load average and interactive keys
- [ ] I can identify the four common process states (Running, Sleeping, Stopped, Zombie) and explain why zombie processes occur
- [ ] I can run commands in the background with `&`, and manage them with `jobs`, `fg`, `bg`, and job IDs like `%1`
- [ ] I can detach a background job from the terminal with `nohup` and `disown`, and explain why `SIGHUP` kills background jobs when a terminal closes
- [ ] I can explain what a signal is, name the key signals (`SIGTERM`, `SIGKILL`, `SIGHUP`, `SIGINT`) with their numbers, and send them with `kill`, `killall`, and `pkill`
- [ ] I can look under the hood at `/proc/<pid>/status` and `/proc/<pid>/cmdline` to see where process information actually comes from

## Next Step

Continue to [Module 11: Package Management & System Monitoring](../module11-package-mgmt-monitoring/)
