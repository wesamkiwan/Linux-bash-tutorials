# 📋 Module 10 Cheat Sheet — Process Management & Job Control

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Process** | A running instance of a program, tracked by the kernel via a PID |
| **PID** | Process ID — a unique number identifying a running process |
| **PPID** | Parent Process ID — the PID of the process that started this one |
| **PID 1** | The root of the process tree; `systemd` on Ubuntu/Debian |
| **Job** | A shell-level concept — a command you started from your current interactive shell |
| **Signal** | A standardized kernel notification sent to a process, asking it to do something (often: stop) |
| **Zombie** | A finished process whose exit status hasn't been collected by its parent yet |
| **Orphan** | A process whose parent died first; re-parented, usually to `systemd` |
| **Foreground** | A process attached to your terminal, blocking your prompt until it finishes |
| **Background** | A process running detached from your prompt, started with `&` |

## `ps` Quick Reference

| Command | Shows |
|---|---|
| `ps` | Processes attached to your current terminal only |
| `ps aux` | Every process, BSD-style columns |
| `ps -ef` | Every process, UNIX-style columns |
| `ps aux --sort=-%cpu` | Sorted by CPU descending — worst offender on top |
| `ps aux --sort=-%mem` | Sorted by memory descending |
| `pstree -p <user>` | Visual parent/child tree, with PIDs |
| `pgrep -a <pattern>` | Read-only: list PIDs + full command line matching a name/pattern |

### `ps aux` Columns

| Column | Meaning |
|---|---|
| USER | Owner of the process |
| PID | Process ID |
| %CPU | Current CPU usage percentage |
| %MEM | Current RAM usage percentage |
| VSZ | Virtual memory size (KB) |
| RSS | Actual physical RAM in use (KB) |
| TTY | Attached terminal, or `?` if none |
| STAT | Process state code (see below) |
| START | Time/date the process started |
| TIME | **Total CPU time used** (not elapsed/wall-clock time) |
| COMMAND | The command + arguments |

### Process States (`STAT` column)

| Code | State | Meaning |
|---|---|---|
| `R` | Running | Executing, or ready and waiting for CPU |
| `S` | Sleeping | Waiting for an event; interruptible by a signal |
| `D` | Uninterruptible sleep | Waiting on I/O (usually disk); cannot be signaled until done |
| `T` | Stopped | Paused (e.g. via Ctrl+Z / `SIGSTOP`); still fully alive |
| `Z` | Zombie | Finished, but parent hasn't collected its exit status yet |

## Live Monitoring

| Tool | Notes |
|---|---|
| `top` | Preinstalled everywhere; live-refreshing snapshot |
| `htop` | Friendlier, colorized; install with `sudo apt install htop` |
| `k` (inside `top`) | Kill a process (prompts for PID + signal) |
| `q` (inside `top`) | Quit |
| `P` / `M` (inside `top`) | Sort by %CPU / %MEM |
| Load average | 1/5/15-min average of processes wanting CPU; compare against core count |

## Job Control

| Command | Effect |
|---|---|
| `command &` | Run in background; shell returns prompt immediately |
| `jobs` | List jobs known to *this* shell |
| `fg` / `fg %2` | Bring most recent job / job 2 to the foreground |
| `bg` / `bg %2` | Resume most recent stopped job / job 2 in the background |
| `Ctrl+Z` | Suspend (Stop) the current foreground job — pauses, doesn't kill |
| `Ctrl+C` | Sends `SIGINT` — typically terminates the foreground job |
| `%1`, `%2`, ... | Job ID syntax, used with `fg`/`bg`/`kill` |

## Detaching From the Terminal

| Command | Effect |
|---|---|
| `nohup command &` | Ignores `SIGHUP` from the start; output goes to `nohup.out` unless redirected |
| `disown %1` | Removes an already-backgrounded job from the shell's job table (won't be sent `SIGHUP` on exit) |
| `nohup command > out.log 2>&1 &` | `nohup` + explicit output redirection (recommended over relying on `nohup.out`) |

## Common Signals

| Number | Name | Default action | Catchable? | Sent by / used for |
|---|---|---|---|---|
| 1 | `SIGHUP` | Terminate | Yes | Terminal closing; daemons often reuse it as "reload config" |
| 2 | `SIGINT` | Terminate | Yes | Ctrl+C |
| 9 | `SIGKILL` | Terminate | **No** | Force-kill; last resort only |
| 15 | `SIGTERM` | Terminate | Yes | **Default** signal from plain `kill` — graceful shutdown request |
| 18 | `SIGCONT` | Continue | Yes | Resumes a stopped process |
| 19 | `SIGSTOP` | Stop | **No** | Force-pause; used internally by Ctrl+Z's `SIGTSTP` cousin |
| 20 | `SIGTSTP` | Stop | Yes | Ctrl+Z |

## Sending Signals

| Command | Effect |
|---|---|
| `kill <pid>` | Sends `SIGTERM` (default) to a specific PID |
| `kill -9 <pid>` / `kill -SIGKILL <pid>` | Sends `SIGKILL` to a specific PID |
| `kill -l` | Lists all signal names/numbers |
| `killall <name>` | Signals every process with an **exact** matching command name |
| `pkill <pattern>` | Signals every process matching a **pattern** (more flexible than `killall`) |
| `pgrep -a <pattern>` | **Read-only** — preview exactly what `pkill`/`killall` would match, before you signal anything |

## `/proc` Quick Reference

| Path | Contains |
|---|---|
| `/proc/<pid>/status` | Human-readable process state, PPID, memory usage |
| `/proc/<pid>/cmdline` | Exact command + args the process was launched with |
| `/proc/<pid>/environ` | The process's environment variables (null-separated) |

## 🔁 The Graceful Shutdown Workflow

Do this every time you need to stop a process:

1. **Identify it precisely** — `ps aux | grep <name>` or `pgrep -a <pattern>`. Confirm the exact PID before doing anything else.
2. **Try `SIGTERM` first** — plain `kill <pid>`. This asks it to clean up and exit on its own terms.
3. **Wait a few seconds** and check again (`ps -p <pid>` or `pgrep -a <pattern>`) to see if it actually exited.
4. **Escalate to `SIGKILL` only if it's still there** — `kill -9 <pid>`. This is the last resort, not the first move.

## 🔁 The Safe Background Deploy Workflow

Do this any time you're about to start a long job you'll walk away from:

1. Start it with `nohup` from the very beginning — `nohup ./script.sh > script.log 2>&1 &`.
2. Note the PID the shell prints (or capture it: `echo $!`).
3. Check on progress later with `tail -f script.log` and `ps -p <pid>`.
4. When it finishes (or if it needs to be stopped), follow the Graceful Shutdown Workflow above — don't jump to `-9`.
