# 📋 Module 17 Cheat Sheet — Performance Tuning & Profiling Scripts

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **`real` time** | Wall-clock time — the actual time that passed, what everyone waited for |
| **`user` time** | CPU time spent in your program's own code, in normal (non-kernel) mode |
| **`sys` time** | CPU time spent inside the kernel on your program's behalf (system calls) |
| **UUOC** | "Useless Use of Cat" — piping `cat file \| cmd` instead of `cmd file` |
| **Builtin** | A command implemented inside the shell itself (`[[`, `((...))`, `read`) — no process spawned |
| **fork/exec** | The two-step OS process-creation mechanism every external command pays for, every time it runs |
| **I/O-bound** | Bottlenecked on waiting for disk/network, not on CPU computation |
| **CPU-bound** | Bottlenecked on actual computation, not on waiting |
| **`xargs -P`** | Run multiple copies of a command in parallel, reading args from stdin |
| **GNU `parallel`** | A more feature-rich alternative to `xargs -P` for running jobs in parallel |
| **Page fault** | The OS loading memory a program just tried to access; frequent faults can signal memory pressure |

## `time` Output Fields

| Field | Meaning | High value hints at... |
|---|---|---|
| `real` | Total wall-clock elapsed time | The number everyone actually cares about |
| `user` | CPU time in user-mode code | Actual computation being CPU-heavy |
| `sys` | CPU time in kernel/system calls | Heavy I/O, or too many process spawns (fork/exec is a syscall) |

| Pattern observed | Likely cause |
|---|---|
| `real` ≫ `user + sys` | I/O-bound — waiting on disk/network, CPU mostly idle |
| `user + sys` ≈ `real` | CPU-bound — single-core computation dominates |
| `user + sys` > `real` | Multiple cores worked simultaneously (parallelism happening) |
| High `sys`, low `user` | Likely too many process spawns or excessive small I/O calls |

## Customizing `time`

```bash
TIMEFORMAT='Elapsed: %R sec (user: %U, sys: %S, cpu: %P%%)'
time ./script.sh
```

| `%`-code | Meaning |
|---|---|
| `%R` | Real time (seconds) |
| `%U` | User time (seconds) |
| `%S` | Sys time (seconds) |
| `%P` | Percent CPU (`(user+sys)/real × 100`) |

## Timing a Section

```bash
start=$(date +%s%N)
# ... code ...
end=$(date +%s%N)
echo "$(( (end - start) / 1000000 )) ms"
```

```bash
SECONDS=0
# ... code ...
echo "$SECONDS s"
```

## Anti-Patterns: Before → After

| Anti-pattern (slow) | Fix (fast) | Why |
|---|---|---|
| `cat file \| grep x` | `grep x file` | Avoids spawning `cat`; `grep` reads files directly |
| `cat file \| wc -l` | `wc -l file` | Same UUOC fix |
| `while read line; do echo "$line" \| grep -q x; done < file` | `awk '/x/ { ... }' file` | One process for the whole file vs. two per line |
| `while read a b; do total=$(echo "$total+$b"\|bc); done < file` | `awk '{ total += $2 } END { print total }' file` | `awk` has native arithmetic; no `bc`/`echo` per line |
| `expr $x + 1` in a loop | `(( x++ ))` or `x=$(( x + 1 ))` | Arithmetic builtin, no process spawned |
| `test $x -eq 5` in a loop | `[[ $x -eq 5 ]]` | `[[` is a shell keyword, not an external test |
| `echo "$file" \| sed 's/\.txt$//'` | `${file%.txt}` | Parameter expansion, no `sed` process |
| `echo "$str" \| tr 'a-z' 'A-Z'` | `${str^^}` | Bash 4+ builtin case conversion |

## Builtin vs. External Command Equivalents

| Task | External (spawns a process) | Builtin (no process) |
|---|---|---|
| Condition test | `test`, `[ ]` | `[[ ]]` |
| Arithmetic | `expr` | `$(( ))`, `(( ))` |
| String length | `expr length "$s"` | `${#s}` |
| Strip suffix | `sed 's/pattern$//'` | `${var%pattern}` |
| Strip prefix | `sed 's/^pattern//'` | `${var#pattern}` |
| Substring | `cut`/`expr substr` | `${var:offset:length}` |
| Uppercase/lowercase | `tr` | `${var^^}` / `${var,,}` |
| Read a line | (n/a — always builtin) | `read` |

## `xargs -P` / GNU `parallel` Quick Reference

```bash
# xargs -P — run up to N jobs at once
find . -name "*.log" | xargs -P 4 -I {} gzip {}

# GNU parallel — install first
sudo apt install parallel
find . -name "*.log" | parallel -j 4 gzip {}

# Check how many cores you have before choosing -P/-j
nproc
```

| Tool | Strengths | Use when |
|---|---|---|
| `xargs -P` | Already installed everywhere, simple | Quick, simple parallel fan-out |
| GNU `parallel` | Progress bars, load balancing, richer templating, result logging | Complex jobs, uneven job sizes, need more control |

| Parallelizing helps when... | Parallelizing does NOT help when... |
|---|---|
| Tasks are independent (no shared state/order) | Tasks depend on each other's output |
| I/O-bound (waiting overlaps across jobs) | Bottlenecked on one shared resource (single disk, rate-limited API, DB connection limit) |
| CPU-bound with multiple cores available | `-P`/`-j` set far above `nproc` for CPU-bound work |

## Profiling Tools

| Tool | Shows | Notes |
|---|---|---|
| `time cmd` | `real`/`user`/`sys` | Shell keyword; fast, always available |
| `/usr/bin/time -v cmd` | Full resource report: max memory, page faults, context switches | Different program — needs full path |
| `strace -c cmd` | Syscall counts and time spent per syscall | Advanced; high `fork` count confirms process-spawning problems |
| `nproc` | Number of CPU cores available | Check before picking `-P`/`-j` values |

## 🔁 The Script Performance Triage Workflow

Do this every time a script feels slow:

1. **Time it** — `time ./script.sh` — get a baseline `real`/`user`/`sys` reading.
2. **Read the ratio** — `real` ≫ `user+sys` means I/O-bound; `user+sys` ≈ `real` means CPU-bound. This decides whether parallelizing can even help.
3. **Find the loop** — almost every slow script has one dominant loop or one dominant pass over a large file. Section-time it (`$SECONDS` or `date +%s%N`) to confirm which part is actually slow.
4. **Check for process-per-iteration calls** — does the loop body call `grep`, `sed`, `awk`, `expr`, `test`, `bc`, or `echo | something` once per line/item? That's almost always the bottleneck.
5. **Check for UUOC** — any `cat file | cmd` that could be `cmd file`?
6. **Rewrite the hot loop as a single `awk`/`sed` pass** if it's calling external tools per line, or switch to builtins (`[[`, `(( ))`, parameter expansion) if it's small per-iteration checks.
7. **Re-time it** — confirm the fix actually helped, and by how much.
8. **If still slow and the work is independent**, consider `xargs -P` or GNU `parallel` — but only after confirming with step 2 that the workload can actually benefit.
9. **If you need more than a duration**, profile with `/usr/bin/time -v` (memory/page faults) or `strace -c` (syscall breakdown) to dig deeper.
