# 🎤 Module 10 Interview Prep — Process Management & Job Control

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a process, and what's the difference between a PID and a PPID?**

> "A process is a running instance of a program — the program file on disk is inert, but the moment the kernel loads it and starts running it, that active, running thing is a process. Every process gets a PID, a unique number the kernel assigns it. Every process except the very first one also has a PPID — the PID of whatever process created it, its parent. So PID identifies the process itself, PPID tells you who launched it."

**Q2: What is PID 1, and why does it matter?**

> "PID 1 is the very first process the kernel starts at boot, and every other process on the system is either a direct child of it or a descendant of one of its children — it's the root of the entire process tree. On Ubuntu and most modern distributions that's `systemd`. It matters practically because if a process's parent dies before it does, it gets orphaned and re-parented, almost always to PID 1, so every process always has some valid parent looking after it."

**Q3: What's the difference between `ps aux` and `ps -ef`?**

> "They're two different historical option styles for the same underlying tool — `ps aux` is the BSD-style syntax, `ps -ef` is the UNIX/POSIX style. Both list every process on the system by walking the same process table, they just show slightly different columns by default — `aux` gives you %CPU, %MEM, VSZ, RSS; `-ef` gives you PPID and a more precise start time instead. In practice you'll see both used interchangeably depending on the team or script, so it's worth being comfortable reading either."

**Q4: What does running a command with `&` at the end actually do?**

> "It starts the command in the background instead of the foreground. Normally the shell waits for a command to finish before giving you the prompt back; `&` tells it not to wait — it starts the command, immediately hands the prompt back to you, and prints a job ID and PID so you can track it, while the command keeps running independently."

**Q5: What key does Ctrl+C send, and what does it do by default?**

> "Ctrl+C sends `SIGINT` to the foreground process. By default that terminates the process, though a well-behaved program can catch that signal and decide to do something else with it — like ask 'are you sure?' before actually exiting."

### 🟡 Intermediate

**Q6: Explain SIGTERM vs. SIGKILL — what's actually different between them, and when would you use each?**

> "SIGTERM, signal 15, is the default signal plain `kill` sends — it's a polite request asking the process to terminate, and crucially, the process can catch it and run its own cleanup code first: closing files, finishing a write, releasing a lock, then exiting on its own terms. SIGKILL, signal 9, cannot be caught, blocked, or ignored at all — the kernel destroys the process immediately, with zero opportunity to clean up anything mid-operation. My rule is always try SIGTERM first and give it a few seconds; only escalate to SIGKILL if the process genuinely isn't responding. Reaching for `-9` immediately risks corrupting whatever that process was in the middle of — a half-written file, an unreleased lock, an inconsistent database state."

**Q7: What is a zombie process, and why does it occur?**

> "When a process finishes running, it doesn't disappear instantly — the kernel keeps a small record of it, mainly its exit status, until its parent process calls `wait()` to formally collect that result. Until the parent does that, the finished process shows up in the process table as a zombie — state `Z` in `ps` — holding no real memory or CPU, just a placeholder entry waiting to be collected. It's usually harmless and lasts a fraction of a second, since well-written parents call `wait()` almost immediately. It only becomes a real problem when a parent process has a bug and never calls `wait()` at all, letting zombies pile up indefinitely — and you can't kill a zombie directly, because there's no living process left to signal; you have to fix or restart the parent instead."

**Q8: What's the difference between a process and a job?**

> "A process is a kernel-level concept — any running program on the system, tracked by a PID, regardless of who started it. A job is a shell-level concept that only exists within your current interactive shell session — it's specifically a command you started from that shell, tracked in that shell's own job table so `jobs`, `fg`, and `bg` can reference it. Every job corresponds to one or more processes, but most processes on a system — background services, other users' processes, things from other terminal sessions — aren't jobs of your shell at all; your shell simply has no bookkeeping entry for them."

**Q9: Why does closing your terminal kill a background job, and how do you prevent that?**

> "When the shell itself exits, it sends `SIGHUP` — 'hang up,' named after literal telephone modems — to every job still in its job table. Most programs respond to `SIGHUP` by terminating by default, so a plain background job (`command &`) dies right along with the terminal session. To prevent it, I either start the command with `nohup command &` from the beginning, which makes the process ignore `SIGHUP` entirely, or, if it's already running, use `disown %1` to remove it from the shell's job table so the shell won't send it `SIGHUP` when it exits at all. `nohup` from the start is the safer habit, since `disown` only helps if you remember to run it before the terminal actually closes."

**Q10: What's the practical difference between `killall` and `pkill`?**

> "Both send a signal to multiple processes at once based on matching, rather than one specific PID, but they differ in how they match. `killall` matches by the exact command name. `pkill` matches against a pattern, which is more flexible — it can match partial names or more complex expressions, not just an exact command name. Both are more dangerous than targeting a verified single PID, because on a busy or shared system it's easy to match more processes than you intended — I always run the read-only equivalent, `pgrep -a <pattern>`, first to see exactly what would be hit before actually sending a signal."

### 🔴 Advanced

**Q11: A production database process needs to be stopped for a planned maintenance window. Walk me through exactly how you'd do it.**

> "First I'd identify the exact process with `ps aux | grep` or `pgrep -a` and confirm I have the right PID — not just matching on a guessed name. Then I'd send it a plain `kill <pid>`, which is `SIGTERM` — that gives the database process a chance to finish any in-flight write, flush buffers, and shut down its own way, which for a database specifically matters a lot, since an abrupt kill mid-write risks a corrupted or inconsistent data file. I'd wait a reasonable amount of time — how long depends on the specific database and how large its shutdown/checkpoint process typically is — and check with `ps -p <pid>` whether it's actually exited. Only if it's still there after a fair wait would I escalate to `kill -9`, and even then, I'd expect to follow up afterward with an integrity check or crash-recovery process on that database, precisely because SIGKILL gave it no chance to shut down cleanly."

**Q12: You SSH into a server and find dozens of zombie processes accumulating over time. What's actually going on, and how do you fix it?**

> "Zombies by themselves are just unread exit-status records — accumulating zombies specifically means some parent process is spawning children and never calling `wait()` to collect their results, which is a bug in that parent, not in the children themselves. I can't fix it by touching the zombie PIDs directly, since there's no living process there to signal. The real fix is identifying the parent — `ps -o ppid= -p <zombie-pid>` or reading `pstree` — and either fixing that parent's code to properly reap its children, or, more immediately in production, restarting the parent process itself, which causes the kernel to re-parent (and then clean up) its zombie children."

**Q13: Why might `nohup ./script.sh &` still not survive a closed SSH session in every case, and what's a more robust alternative?**

> "`nohup` protects specifically against `SIGHUP` — it doesn't protect against other things that can still end a session's processes, like the whole machine losing power, the SSH client-side network genuinely severing in a way the server-side process group still gets cleaned up on, or certain SSH server configurations/PAM settings that more aggressively tear down a user's entire process group on logout regardless of `SIGHUP` handling. A more robust alternative for anything long-running and important is a terminal multiplexer like `tmux` or `screen` — you start the job inside a `tmux` session, detach from that session (the job keeps running exactly as if you were still there), and reattach to it later from any new SSH connection to see its live output, rather than only relying on captured log output the way `nohup` requires."

---

## Practical/Coding Questions

**Q1: You suspect one process is consuming excessive CPU. Write the commands to confirm it and find its exact PID.**

Solution:
```bash
ps aux --sort=-%cpu | head -5
```
Explanation: `--sort=-%cpu` sorts descending by CPU usage, so the worst offender is on the first data line. `head -5` keeps the header plus the top four processes, which is enough to spot the culprit and read its exact PID from the second column without scrolling through the entire process table.

**Q2: Write the commands to start a script in the background so it survives you closing your terminal, redirecting its output to a specific log file (not the default `nohup.out`).**

Solution:
```bash
nohup ./backup.sh > /var/log/backup-run.log 2>&1 &
```
Explanation: `nohup` makes the process ignore `SIGHUP`. `> /var/log/backup-run.log` redirects standard output to a chosen file instead of the default `nohup.out`, and `2>&1` redirects standard error into that same file so error messages aren't lost or left going to the (soon-to-be-closed) terminal. The trailing `&` backgrounds it.

**Q3: Write a one-liner to gracefully stop every process matching the name `worker-node`, waiting to confirm before considering escalating.**

Solution:
```bash
pgrep -a worker-node        # verify exactly what would be matched
pkill worker-node           # sends SIGTERM (default) to all matches
sleep 3
pgrep -a worker-node        # confirm they actually stopped
```
Explanation: `pgrep -a` first is the safety check — it shows exactly which PIDs and full command lines would be hit before anything is actually signaled. `pkill worker-node` (no `-9`) sends the default, graceful `SIGTERM`. The `sleep 3` and second `pgrep -a` confirm whether they actually exited before deciding whether `pkill -9 worker-node` is genuinely needed — never assume it worked without checking.

**Q4: Given only a PID, write the commands to find (a) what command actually launched that process and (b) its parent's PID, without using `ps`.**

Solution:
```bash
cat /proc/1234/cmdline | tr '\0' ' '; echo
cat /proc/1234/status | grep PPid
```
Explanation: `/proc/<pid>/cmdline` holds the exact command and arguments the process was launched with, but its fields are null-separated rather than space-separated, so `tr '\0' ' '` makes it readable on one line. `/proc/<pid>/status` is a human-readable breakdown of that process's state, and grepping it for `PPid` pulls out the parent PID directly — the same underlying data `ps` itself is built from.

---

## Gotcha Questions

**Q1: "I ran `kill -9` on a stuck process and it's gone from `ps`, but the disk is now showing corrupted data from what it was writing. Isn't `kill -9` supposed to guarantee the process stops cleanly?"**

> Trap: `kill -9` guarantees the process *stops* — immediately and unconditionally — but it explicitly does **not** guarantee it stops *cleanly*. SIGKILL cannot be caught by the target process, meaning any in-progress write, any buffer that hadn't been flushed to disk yet, any lock that hadn't been released, is simply abandoned exactly where it stood the instant the kernel tore the process down. "Stops" and "cleans up after itself" are two different guarantees, and SIGKILL only ever promised the first one. This is precisely why SIGTERM-first is the standing rule for anything doing real I/O.

**Q2: "I started a job with `command &`, and closed my laptop lid a minute later assuming it would keep running since it's a background job. Why did it die?"**

> Trap: The candidate needs to recognize that "background" (not blocking your terminal) and "detached" (surviving the terminal's own death) are two completely different properties, and `&` alone only gives you the first one. A plain background job is still fully attached to the shell's job table; when that shell session ends — laptop sleeping and dropping the SSH connection counts — it sends `SIGHUP` to every job it still knows about, and most programs terminate on `SIGHUP` by default. `&` alone was never protection against the terminal closing; only `nohup` (or `disown`, or running inside `tmux`/`screen`) actually protects against that.

**Q3: "`ps aux` shows this process has been running (`START`) since three days ago, so I assumed it was doing heavy, continuous work all that time — but its `TIME` column only shows a few seconds. Which column is lying?"**

> Trap: Neither column is wrong — they measure two completely different things, and conflating them is the actual mistake. `START` is when the process began (wall-clock/elapsed time). `TIME` is the total *CPU time actually consumed* since then — how much real work it's done, not how long it's existed. A process that's been alive for three days but spends nearly all of that time sleeping/waiting (state `S`) and only wakes up briefly now and then will show a `START` of three days ago and a `TIME` of just a few seconds, because it genuinely has only used a few seconds of actual CPU work in all that time. If you want elapsed running time specifically, you want `ps -o etime`, not `TIME`.

---

## Quick-Fire Rapid Review

- **Q: What is PID 1 on Ubuntu/Debian?** A: `systemd`.
- **Q: What identifies which process started another process?** A: Its PPID (Parent Process ID).
- **Q: Default signal sent by plain `kill <pid>`?** A: `SIGTERM` (15).
- **Q: Can `SIGKILL` be caught or ignored by a process?** A: No — never, by design.
- **Q: What signal does Ctrl+C send?** A: `SIGINT` (2).
- **Q: What signal does Ctrl+Z send, and does it terminate the process?** A: `SIGTSTP` — it suspends (Stops) it, doesn't kill it.
- **Q: What does the `Z` state mean in `ps`?** A: Zombie — finished, but the parent hasn't collected its exit status yet.
- **Q: Can you kill a zombie process directly?** A: No — there's nothing left to signal; fix or restart its parent instead.
- **Q: What does `&` at the end of a command do?** A: Runs it in the background, returning the prompt immediately.
- **Q: What command lists jobs known to the current shell?** A: `jobs`.
- **Q: What does `nohup` actually do?** A: Makes the process ignore `SIGHUP`, so it survives the terminal closing.
- **Q: What signal does a terminal closing send to its jobs?** A: `SIGHUP` (1).
- **Q: What's the read-only way to preview what `pkill`/`killall` would match?** A: `pgrep -a <pattern>`.
- **Q: Which `ps aux` column shows total CPU time used, not elapsed time?** A: `TIME`.
- **Q: Where does `ps`/`top` actually get its process data from?** A: The `/proc` virtual filesystem (e.g. `/proc/<pid>/status`).
