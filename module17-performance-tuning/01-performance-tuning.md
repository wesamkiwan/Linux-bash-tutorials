# Module 17: Performance Tuning & Profiling Scripts 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-14 (Shell Fundamentals through Error Handling, Traps & Debugging). Module 9's `sed`/`awk`/regex is used directly and extensively in this module — if single-pass `awk` feels unfamiliar, revisit Module 9 first. Module 10's process/subshell concepts and Module 5's pipes are also assumed.

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain why script performance matters on the job — a slow script isn't just an inconvenience, it's blocked pipelines, missed deadlines, and wasted compute cost
- [ ] Measure a script's runtime with `time`, and correctly explain what `real`, `user`, and `sys` each mean and why they can differ wildly
- [ ] Customize `time`'s output format with `TIMEFORMAT`, and time individual sections of a script using `date +%s%N` or `$SECONDS`
- [ ] Identify the "Useless Use of Cat" (UUOC) anti-pattern and other unnecessary-subshell patterns, and fix them
- [ ] Explain why spawning an external process (or subshell) once per loop iteration is slow, and prefer Bash builtins (`[[`, parameter expansion, `$(())`) instead
- [ ] Rewrite a bash loop that calls external commands line-by-line as a single `awk`/`sed` pass, and benchmark the before/after difference
- [ ] Parallelize independent work with `xargs -P`, know when GNU `parallel` is a better fit, and recognize when parallelizing does **not** help
- [ ] Profile a script's resource usage with `/usr/bin/time -v`, and know that `strace -c` exists for syscall-level profiling

---

## Module Goal

By the end of this module, you'll be able to take a script that's *correct* but *slow*, find out exactly where the time is going, and fix the specific bottleneck — without guessing, and without rewriting the whole thing from scratch.

🎯 **On the job:** Picture this: your team has a nightly script that processes the day's application log file — a few million lines — extracting error counts, computing summary statistics, and writing a report before the morning stand-up. For months it finished in under two minutes. Then traffic grew, the log file grew with it, and one morning the script is still running at 9 AM, blocking the report the on-call engineer needs and delaying a downstream billing job that depends on it. Nobody touched the script's logic — it still produces the right numbers, eventually — but "eventually" now costs real money in blocked pipelines and idle compute. This module is about diagnosing exactly *why* a script that used to be fast has become slow, and fixing the actual bottleneck instead of randomly tweaking things and hoping.

---

## Core Concepts

### 1. Why script performance matters on the job

A script that takes 2 hours instead of 2 minutes on a large file isn't merely "less snappy" — it has real consequences:

- **Blocked pipelines.** If a nightly job is supposed to finish before a downstream job starts, a slow script pushes everything after it later, potentially into business hours when people are waiting on the result.
- **Wasted compute cost.** Cloud compute is billed by the minute or the CPU-second. A script that runs 60x longer than necessary can burn through 60x the compute budget for the same output — real money, every single night it runs.
- **Hidden scaling cliffs.** A script that works fine on a 10,000-line test file can be secretly quadratic or process-per-line, and only reveal how slow it really is once the input grows to millions of lines in production — often discovered at the worst possible time.

💡 **Tip:** Performance work isn't about making every script maximally fast. It's about finding the *one* place a script spends 95% of its time and fixing *that*, because that's almost always where all the waste actually lives.

### 2. Measure first: the `time` command

**`time`** is a shell keyword (and also a standalone program) that runs a command and reports how long it took, broken into three numbers:

```bash
time ./process_logs.sh access.log
```

```
real    1m42.384s
user    1m18.216s
sys     0m22.940s
```

These three numbers mean very different things, and confusing them is one of the most common mistakes when reading `time` output:

- **`real`** — "wall-clock time." The actual time that passed on a clock on the wall, from when the command started to when it finished. This is what your users and your pipeline schedule actually care about — if `real` is 1m42s, that's how long everyone waited, full stop.
- **`user`** — CPU time spent executing your program's own code (and any libraries it called into) in "user mode" — the normal, non-kernel mode almost all application code runs in.
- **`sys`** — CPU time spent inside the kernel on your program's behalf — things like reading from a file, writing output, creating a process, or any other system call (a request your program makes to the operating system to do something it can't do by itself, like touching the filesystem or network).

💡 **Why `real` can be bigger *or* smaller than `user + sys`:** If a script spends time waiting on something outside the CPU entirely — a slow disk, a network call, another process — that waiting time counts toward `real` but not toward `user` or `sys`, because the CPU isn't actually busy during that wait; it's just idle, waiting. That's why `real` is often noticeably larger than `user + sys` for I/O-heavy scripts. On a multi-core machine running things in parallel, the reverse can happen: `user + sys` can add up to *more* than `real`, because multiple CPU cores were each burning CPU time simultaneously during the same wall-clock window.

🎯 **On the job:** A high `sys` time relative to `user` is a strong hint that a script is spending its time on system calls rather than actual computation — often a sign of spawning too many processes (each `fork`/`exec` is a system call) or doing too much small, repeated I/O. That's exactly the pattern this module spends most of its time fixing.

### 3. Customizing `time`'s output with `TIMEFORMAT`

The `time` keyword's default three-line output is fine, but you can reshape it with the **`TIMEFORMAT`** variable, which uses `%`-escapes similar to `printf`:

```bash
TIMEFORMAT='Elapsed: %R sec (user: %U, sys: %S)'
time ./process_logs.sh access.log
```

```
Elapsed: 102.384 sec (user: 78.216, sys: 22.940)
```

- `%R` — real time in seconds
- `%U` — user time in seconds
- `%S` — sys time in seconds
- `%P` — the percentage of CPU this job got (`user+sys` divided by `real`, as a percentage — useful for spotting how parallel or how I/O-bound a run was)

💡 **Tip:** `TIMEFORMAT` only affects the `time` shell keyword, not the standalone `/usr/bin/time` binary (Concept 11 covers that one separately, with its own formatting).

### 4. Timing individual sections of a script

`time`-ing the whole script tells you the total damage, but not *which part* of the script is slow. For that, capture timestamps around just the section you suspect:

```bash
start=$(date +%s%N)
# ... suspect section here ...
end=$(date +%s%N)
echo "Section took $(( (end - start) / 1000000 )) ms"
```

`date +%s%N` prints the current time as nanoseconds since the Unix epoch (`%s` is whole seconds, `%N` is nanoseconds within the current second) — subtracting two of these gives you an elapsed duration with sub-second precision, then dividing by 1,000,000 converts nanoseconds to milliseconds.

For coarser timing (whole seconds is enough), Bash's built-in **`$SECONDS`** variable is simpler — it starts at 0 when the shell begins and auto-increments once per second:

```bash
SECONDS=0
# ... suspect section here ...
echo "Section took $SECONDS seconds"
```

✅ **Best Practice:** When a script has several stages (download, parse, transform, upload), wrap each stage with its own timer and log the result. That turns "the whole script is slow" into "stage 3, the parse step, is 90% of the runtime" — the difference between guessing and knowing exactly where to focus.

### 5. The real cost of spawning a process

Every time Bash runs an external command — `grep`, `awk`, `cut`, even a simple `echo` run as `/bin/echo` instead of the builtin — the operating system has to **fork** (create a new process, a copy of the calling process) and then **exec** (replace that copy's memory with the new program's code) before that program can even start doing useful work. This setup cost is small in absolute terms — often well under a millisecond — but it is **not free**, and it is paid **every single time**, regardless of how trivial the work the command actually does.

💡 **Analogy — hiring a contractor per brick:** Imagine building a brick wall. One approach: hire a full construction crew once, hand them the entire pile of bricks and a blueprint, and let them lay every single brick as one continuous job. The other approach: for every individual brick, call an agency, hire a new contractor, have them drive to the site, lay exactly one brick, and then send them home — and repeat that entire hiring process for the next brick. Both approaches eventually produce the same wall. But the second one spends far more total effort on the *overhead of hiring* than on the *actual bricklaying* — and that overhead scales directly with the number of bricks, not the size of the wall. A bash loop that spawns `grep` or `awk` once per line of a million-line file is doing exactly this: paying the "hire a contractor" cost a million times over, for one brick's worth of work each time.

🎯 **On the job:** This single insight — process-spawning overhead multiplied by loop iterations — explains the overwhelming majority of "why is my script so slow on large files" tickets. The fix is almost always the same shape: replace "spawn a process per line" with "spawn one process for the whole file."

### 6. The "Useless Use of Cat" (UUOC) anti-pattern

**UUOC** is the single most famous named Bash anti-pattern: piping `cat` into a command that could just read the file itself.

```bash
# UUOC — spawns an extra process for no reason
cat access.log | grep "ERROR"
```
```bash
# Fixed — grep reads the file directly
grep "ERROR" access.log
```

Both produce identical output. The first version spawns `cat` just to stream the file's bytes into a pipe, when `grep` (like nearly every standard Unix text tool — `awk`, `sed`, `wc`, `head`, `tail`, `sort`) already accepts a filename argument directly and can read the file itself, with no `cat` involved at all.

⚠️ **Why this matters more than it looks like it should:** On a single small file, the extra `cat` process costs a few milliseconds — genuinely not worth worrying about on its own. The real damage happens when this pattern is repeated inside a loop, or applied to a very large file where the extra process and extra pipe add measurable overhead every single run.

✅ **Best Practice:** If a command's very first argument accepts a filename, give it the filename directly instead of piping `cat` into it. Save `cat` for what it's actually for: concatenating *multiple* files together, or genuinely needing stdin (some commands, like `tr`, don't accept a filename argument at all and legitimately need something piped in).

### 7. Builtins vs. external processes inside a loop

Bash has **builtins** — commands implemented directly inside the shell itself (`[[`, `((...))`, parameter expansion like `${var%pattern}`, `read`, arithmetic `$(())`) — that run with **no process-spawning cost at all**, because they execute as part of the already-running shell. Compare that to **external commands** (`test`, `expr`, `grep`, `sed`, `awk` run as standalone programs) which each require their own fork/exec.

For a single command run once, the difference is invisible. Inside a loop that runs thousands or millions of times, it is the difference between a script finishing in under a second and one that takes minutes:

| Task | External-process version | Builtin version |
|---|---|---|
| Test a condition | `test "$x" -eq 5` or `[ "$x" -eq 5 ]` | `[[ $x -eq 5 ]]` |
| Do arithmetic | `expr $x + 1` | `$(( x + 1 ))` |
| Strip a suffix | `echo "$file" \| sed 's/\.txt$//'` | `${file%.txt}` |
| Get string length | `expr length "$str"` | `${#str}` |
| Uppercase a string | `echo "$str" \| tr 'a-z' 'A-Z'` | `${str^^}` (Bash 4+) |

Every row on the right runs entirely inside the current shell process; every row on the left spawns a brand-new process, every single time it runs.

### 8. Reading files: `while read` vs. calling external tools per line

**`while read -r line; do ... done < file`** is the idiomatic, efficient way to process a file one line at a time in pure Bash — `read` is a builtin, so this loop itself spawns no extra processes just to *read* the file, no matter how many lines it has.

The performance trap isn't the `while read` loop itself — it's what you put **inside** it. If the loop body calls an external command (`grep`, `awk`, `cut`, `echo` as `/bin/echo`) once per line, you've reintroduced the exact "contractor per brick" cost from Concept 5, just wrapped in a builtin loop instead of a `for` loop. A million-line file means a million process spawns, regardless of which kind of loop is doing the iterating.

✅ **Best Practice:** `while read` is fine — even good — for line-by-line logic that stays entirely inside Bash builtins and parameter expansion. The moment the loop body needs to call `grep`, `sed`, or `awk` on every single line, that's the signal to stop and ask whether the *entire file* could be processed in one pass instead (Concept 9).

### 9. The big win: one `awk`/`sed` pass instead of a line-by-line loop

This is the single highest-leverage optimization in this whole module. Tools like `awk` and `sed` (Module 9) are built specifically to stream through an entire file, line by line, **inside a single process** — no forking, no re-invoking anything per line. Whatever a bash loop does by calling an external command once per line, `awk` can very often do in one pass, spawning exactly **one** process for the entire file, regardless of whether that file has a hundred lines or a hundred million.

```bash
# Bash loop calling external commands per line — one fork/exec pair per line
error_count=0
while read -r line; do
    if echo "$line" | grep -q "ERROR"; then
        error_count=$(( error_count + 1 ))
    fi
done < access.log
echo "Errors: $error_count"
```

```bash
# Single awk pass — one process, period
awk '/ERROR/ { count++ } END { print "Errors:", count+0 }' access.log
```

Both count lines containing `ERROR`. The first spawns `grep` (via `echo ... | grep`, which is *also* a UUOC-adjacent pattern — an unnecessary `echo`/pipe per line) once for every single line in the file. The second spawns nothing beyond the one `awk` process for the entire run. Concept 12 (Detailed Explanations) shows the realistic timing difference this produces on a large file.

### 10. Parallelizing independent work

Some tasks are naturally **independent** — processing file A doesn't depend on the result of processing file B — which means they can run **at the same time** instead of one after another, using multiple CPU cores (or overlapping I/O wait time) simultaneously.

**`xargs -P`** runs multiple copies of a command in parallel, reading arguments from stdin:

```bash
find . -name "*.log" | xargs -P 4 -I {} gzip {}
```

`-P 4` means "run up to 4 of these at once." Each `gzip {}` compresses one file independently of the others, so running four at a time genuinely uses up to four CPU cores simultaneously instead of one core doing all the files sequentially.

**GNU `parallel`** is a more powerful, purpose-built alternative, with far more features than `xargs -P` (progress bars, better handling of complex command templates, automatic load balancing across jobs of uneven size, per-job result logging, and more). Install it on Ubuntu/Debian:

```bash
sudo apt install parallel
```

```bash
find . -name "*.log" | parallel -j 4 gzip {}
```

`-j 4` is `parallel`'s equivalent of `xargs -P 4`'s job count.

⚠️ **When parallelizing helps, and when it doesn't:**
- **Helps:** Independent, **I/O-bound** tasks (each one spends time waiting on disk/network, so multiple can overlap their waiting) or independent **CPU-bound** tasks on a machine with multiple cores (each one gets its own core to actually compute on).
- **Doesn't help — and can actively hurt:** Tasks that aren't actually independent (each depends on the previous one's output — parallelizing this produces wrong results, not just no speedup). Also, running more parallel jobs than you have CPU cores for pure CPU-bound work just adds context-switching overhead without adding real throughput — `-P`/`-j` set far higher than your core count rarely helps and can make things slower. And parallelizing a task that's bottlenecked on a *single shared resource* (one disk, one database, one rate-limited API) can overwhelm that resource and make everything slower for everyone hitting it, not faster.

### 11. Profiling resource usage: `/usr/bin/time -v` and `strace -c`

Plain `time` tells you duration. **`/usr/bin/time -v`** (note: the full path — this is a *different program* from the `time` shell keyword, with much more detailed output) reports a full resource-usage breakdown: maximum memory used, number of page faults (a page fault happens when a program accesses memory that isn't where the CPU expects it, often triggering the OS to load it from disk — frequent page faults can signal a program is using more memory than physically available), context switches, and more.

```bash
/usr/bin/time -v ./process_logs.sh access.log
```

```
        Command being timed: "./process_logs.sh access.log"
        User time (seconds): 78.22
        System time (seconds): 22.94
        Percent of CPU this job got: 98%
        Elapsed (wall clock) time (h:mm:ss or m:ss): 1:42.38
        Maximum resident set size (kbytes): 184320
        Minor (reclaiming a frame) page faults: 41203
        Major (requiring I/O) page faults: 0
        Voluntary context switches: 15442
        Involuntary context switches: 892
```

🎯 **On the job:** "Maximum resident set size" is the peak memory the process actually used — invaluable when a script mysteriously gets killed on a smaller server, or you need to right-size a container's memory limit for it. A high count of "voluntary context switches" alongside a high `sys` time (Concept 2) is another strong signal of a process-spawning-heavy script, since each spawned process involves the kernel switching between processes.

**`strace -c`** goes one level deeper: it traces every **system call** (`read`, `write`, `open`, `fork`, and so on) a program makes and prints a summary table of how many times each was called and how much time was spent in each — genuinely useful for hunting down *why* `sys` time is high, but advanced enough that for this module, just knowing it exists and what it's for is enough:

```bash
strace -c ./process_logs.sh access.log
```

```
% time     seconds  usecs/call     calls    syscall
------ ----------- ----------- --------- ----------------
 41.20    0.089213           2     41880 read
 28.55    0.061840           1     41880 write
 18.90    0.040955          20      2010 fork
 ...
```

A syscall table dominated by `fork` (creating a new process) with tens of thousands of calls is a direct, hard confirmation of the "spawning a process per loop iteration" problem this whole module is built around.

---

## Detailed Explanations

### `real` vs. `user` vs. `sys`, worked through concretely

Take a script that spends most of its time waiting on a slow network download, then briefly does some local CPU-heavy parsing:

```bash
time ./download_and_parse.sh
```
```
real    0m45.203s
user    0m3.114s
sys     0m0.842s
```

Here `real` (45 seconds) is vastly larger than `user + sys` (about 4 seconds combined). That gap — about 41 seconds — is time the CPU spent essentially idle, waiting on the network. This is the signature of an **I/O-bound** script: the CPU isn't the bottleneck at all, the network (or disk, or another external system) is. Optimizing the *parsing* code here would be wasted effort — it's already only 4 seconds of the 45. The only way to meaningfully speed this up is to address the wait itself: maybe download multiple files in parallel (Concept 10), or find a faster network path.

Now compare a **CPU-bound** script — heavy in-memory computation, no waiting on anything external:

```bash
time ./compute_heavy_report.sh
```
```
real    0m12.401s
user    0m12.190s
sys     0m0.088s
```

Here `real` and `user` are nearly identical, and `sys` is tiny. The CPU was busy computing essentially the entire time the script ran — this is a genuinely CPU-bound workload. Parallelizing this across multiple cores (if the work can be split into independent chunks) would actually help, because there's real CPU work to spread across cores — unlike the I/O-bound example above, where there was barely any CPU work to parallelize in the first place.

✅ **Best Practice:** Before reaching for parallelization or "faster" algorithms, always look at the `real` vs. `user + sys` ratio first. A big gap points you toward I/O and waiting; a small gap points you toward actual computation — and the fix looks completely different depending on which one you're facing.

### A concrete before/after: bash loop vs. single `awk` pass

This is the benchmark every professional should be able to reproduce and explain. Take a realistic access log with 2 million lines, and count how many contain `"status":500` (a server error marker), two different ways.

**Before — bash loop, external command per line:**

```bash
#!/bin/bash
count=0
while IFS= read -r line; do
    if echo "$line" | grep -q '"status":500'; then
        count=$(( count + 1 ))
    fi
done < access.log
echo "500 errors: $count"
```

```bash
$ time ./count_errors_slow.sh access.log
500 errors: 8412

real    3m21.740s
user    2m48.902s
sys     0m29.115s
```

**After — single `awk` pass:**

```bash
#!/bin/bash
awk '/"status":500/ { count++ } END { print "500 errors:", count+0 }' access.log
```

```bash
$ time ./count_errors_fast.sh access.log
500 errors: 8412

real    0m1.184s
user    0m1.091s
sys     0m0.093s
```

Same file, same correct answer (`8412`), **~170x faster** (3m21s down to about 1.2 seconds). The slow version spawns two processes per line (`echo` and `grep`) — 4 million process spawns total for a 2-million-line file — each one costing a small but very real fork/exec overhead that adds up relentlessly. The fast version spawns exactly one process (`awk`) for the entire file, no matter how many lines it has.

🎯 **On the job:** This exact shape of rewrite — "loop that calls an external command per line" becomes "one `awk`/`sed` pass" — is very often the single biggest win available in a slow data-processing script, frequently bigger than any amount of clever algorithmic tuning elsewhere in the same script. Always look for this pattern first.

---

## Practical Examples

### Example 1 — Useless Use of Cat (UUOC), before and after

```bash
# Before — UUOC
time (cat /var/log/syslog | wc -l)
```
```
real    0m0.041s
user    0m0.012s
sys     0m0.029s
```

```bash
# After — wc reads the file directly
time (wc -l /var/log/syslog)
```
```
real    0m0.019s
user    0m0.008s
sys     0m0.011s
```

Line-by-line:
- `cat /var/log/syslog | wc -l` spawns two processes (`cat` and `wc`) connected by a pipe; `wc -l` then counts the lines `cat` streamed to it.
- `wc -l /var/log/syslog` spawns exactly one process; `wc` opens and reads the file itself.
- On one file, the difference is roughly 20 milliseconds — genuinely not worth a special trip to fix on its own.

⚠️ **Warning:** The reason UUOC is worth knowing isn't that one instance matters — it's that this exact pattern (`cat file | some_command`) shows up constantly, often nested inside loops, where the "not worth fixing" cost of a single instance gets multiplied by thousands of iterations into something that very much is worth fixing.

✅ **Best Practice:** `grep pattern file`, `wc -l file`, `sort file`, `head file`, `awk '...' file` — nearly every standard text tool takes a filename directly. Reach for `cat file |` only when you're genuinely concatenating multiple files or feeding a tool that has no filename argument of its own.

### Example 2 — Loop with subshells vs. single `awk` pass (full benchmark)

This restates the Detailed Explanations benchmark as a hands-on example with the full line-by-line breakdown.

```bash
#!/bin/bash
# slow_sum.sh — sums a numeric column using an external process per line
total=0
while IFS=, read -r name amount; do
    total=$(echo "$total + $amount" | bc)
done < transactions.csv
echo "Total: $total"
```

```bash
$ time ./slow_sum.sh transactions.csv    # 500,000 lines
Total: 12484910.55

real    2m03.481s
user    1m41.207s
sys     0m18.660s
```

```bash
#!/bin/bash
# fast_sum.sh — single awk pass
awk -F, '{ total += $2 } END { printf "Total: %.2f\n", total }' transactions.csv
```

```bash
$ time ./fast_sum.sh transactions.csv
Total: 12484910.55

real    0m0.512s
user    0m0.478s
sys     0m0.031s
```

Line-by-line:
- `slow_sum.sh` spawns a `bc` process (an external calculator, since Bash's own `$(())` arithmetic only handles integers, not the decimal amounts here) **once per transaction**, plus the `echo` needed to build `bc`'s input — two process spawns per line, 1 million total for 500,000 lines.
- `fast_sum.sh` uses `awk`'s own built-in floating-point arithmetic (`total += $2`) inside a single running process, reading and summing the entire file in one streaming pass.
- The wall-clock difference (over 2 minutes vs. about half a second) is entirely explained by process-spawning overhead — `awk`'s actual arithmetic work is trivially fast either way; it's the *fork/exec cost, repeated 500,000 times*, that dominates the slow version.

💡 **Tip:** Needing floating-point math is exactly the kind of case people reach for `bc` inside a loop for — and exactly the case `awk` (which has native floating-point arithmetic) usually eliminates the need for entirely.

### Example 3 — Builtins vs. external processes inside a loop

```bash
#!/bin/bash
# Slow: expr + test, both external processes, called 100,000 times
count=0
for i in $(seq 1 100000); do
    if test $(expr "$i" % 2) -eq 0; then
        count=$(expr "$count" + 1)
    fi
done
echo "Even numbers: $count"
```

```bash
$ time ./slow_evens.sh
Even numbers: 50000

real    0m41.902s
user    0m35.774s
sys     0m5.981s
```

```bash
#!/bin/bash
# Fast: [[ ]] and $(()) are builtins, no process spawned at all
count=0
for (( i=1; i<=100000; i++ )); do
    if (( i % 2 == 0 )); then
        (( count++ ))
    fi
done
echo "Even numbers: $count"
```

```bash
$ time ./fast_evens.sh
Even numbers: 50000

real    0m0.087s
user    0m0.081s
sys     0m0.006s
```

Line-by-line:
- The slow version calls `expr` twice per iteration (once for the modulo check, once for the increment) plus `test` once — three external process spawns per iteration, 300,000 total across 100,000 iterations.
- The fast version uses `(( ... ))` arithmetic evaluation and a C-style `for` loop — both Bash builtins — so the entire loop runs inside the one already-running shell process, with zero additional processes spawned.
- ~480x faster, purely from eliminating per-iteration process spawns — the actual arithmetic (`i % 2`) is identical in both versions.

✅ **Best Practice:** Inside any loop that runs more than a handful of times, default to `[[ ]]`, `(( ))`, and parameter expansion over `test`, `expr`, and piping into `sed`/`tr` for simple string operations. Save external tools for work that genuinely needs them — regex matching more complex than `[[ =~ ]]` can express, for instance.

### Example 4 — Parallelizing independent work with `xargs -P`

Suppose you need to gzip-compress 200 independent log files — a classic **independent, I/O- and CPU-mixed** task, since gzip does real CPU work but also reads/writes disk.

```bash
# Sequential — one file at a time
time (find /var/log/app -name "*.log" | xargs gzip)
```
```
real    0m48.302s
user    0m41.115s
sys     0m5.884s
```

```bash
# Parallel — up to 4 at once
time (find /var/log/app -name "*.log" | xargs -P 4 gzip)
```
```
real    0m13.741s
user    0m42.006s
sys     0m6.210s
```

Line-by-line:
- Sequential `xargs gzip` compresses each file one after another; total wall time is roughly the sum of every individual compression.
- `xargs -P 4` runs up to 4 `gzip` processes simultaneously; on a 4+ core machine, wall time drops to roughly a quarter, while `user` time (the *total* CPU-seconds spent across all cores) stays about the same — because the same total amount of work is being done, just spread across cores at the same time instead of one after another.
- Notice `user` didn't shrink — that's expected and correct. Parallelizing doesn't reduce the total amount of CPU work; it reduces the wall-clock time by doing that same total work on multiple cores simultaneously.

💡 **Tip:** `xargs -P 0` means "as many parallel jobs as arguments" (unbounded) — almost never what you want for CPU-bound work; pick a `-P` value at or below your machine's core count (check with `nproc`) instead.

---

## Common Pitfalls & Best Practices

- **Optimizing before measuring.** The single most common mistake in performance work: guessing which part of a script is slow and rewriting it, only to find the real bottleneck was somewhere else entirely. Always `time` first, then narrow down with section timers (Concept 4), *then* optimize the specific part that's actually slow.
- **Premature optimization on code that barely matters.** If a section of a script only ever runs for a few milliseconds total, rewriting it to be "faster" wastes engineering time for a gain nobody will ever notice. Spend optimization effort where the profiling data actually points — usually one specific loop or one specific pass over a large file.
- **Confusing I/O-bound with CPU-bound before parallelizing.** Concept 10 covers this directly: parallelizing genuinely independent work helps when there's real waiting (I/O-bound) or real computation to spread across cores (CPU-bound) — but slapping `-P 8` on a task that's bottlenecked on one shared, non-parallel resource (a single disk, a rate-limited API, a database with limited connections) can make things *worse*, not better, by overwhelming that one shared resource with simultaneous requests.
- **Over-parallelizing past your core count.** Setting `-P`/`-j` far higher than `nproc`'s output for pure CPU-bound work adds context-switching overhead without adding real throughput, since there aren't enough physical cores to actually run that many things simultaneously.
- **Assuming every loop with external commands is automatically catastrophic.** A loop that calls `date` once at the very top, or `mkdir` a handful of times, is completely fine — the anti-pattern is calling an external command *per iteration of a large loop*, not using external commands at all. Don't rewrite a 10-iteration setup loop into unreadable `awk` for no measurable benefit.
- **Forgetting that `user + sys` doesn't have to equal `real`.** A big gap either direction (I/O waiting making `real` bigger, or multi-core parallelism making `user + sys` bigger) is informative, not a bug in `time` itself.

✅ **Best Practice — the profiling mindset:** Treat every "this script is slow" report the same way you'd treat a bug report: reproduce it, measure it, form a specific hypothesis about *where* the time goes, and verify that hypothesis with data (`time`, section timers, `/usr/bin/time -v`, `strace -c`) before touching a single line of code.

---

## Hands-on Exercise

**Task:** Below is a script that processes a large CSV of order records, counting how many orders have `status=shipped`, using an external command per line. Benchmark it, then rewrite it as a single `awk` pass and benchmark the rewrite.

Starting point, `count_shipped_slow.sh` (assume `orders.csv` has ~1,000,000 lines, format `order_id,customer,status,amount`):

```bash
#!/bin/bash
count=0
while IFS=, read -r order_id customer status amount; do
    if echo "$status" | grep -q "^shipped$"; then
        count=$(( count + 1 ))
    fi
done < orders.csv
echo "Shipped orders: $count"
```

Your job:
1. Time the slow version with `time`.
2. Identify every unnecessary process spawn happening per loop iteration.
3. Rewrite it as a single `awk` pass.
4. Time the rewrite and compare.

Try it yourself before reading the solution below.

### Solution

**Step 1 — time the slow version:**

```bash
$ time ./count_shipped_slow.sh
Shipped orders: 412903

real    1m38.552s
user    1m22.740s
sys     0m14.201s
```

**Step 2 — identify the waste:** Each of the 1,000,000 iterations spawns `echo` (to produce `$status` as text) piped into `grep -q` (to test it) — two process spawns per line, 2 million total, purely to check whether one field equals the literal string `"shipped"`. Both of those are jobs `awk` handles natively, with zero process spawning, using its own field-splitting and string comparison.

**Step 3 — rewrite as a single `awk` pass**, `count_shipped_fast.sh`:

```bash
#!/bin/bash
awk -F, '$3 == "shipped" { count++ } END { print "Shipped orders:", count+0 }' orders.csv
```

**Step 4 — time the rewrite:**

```bash
$ time ./count_shipped_fast.sh
Shipped orders: 412903

real    0m0.623s
user    0m0.571s
sys     0m0.048s
```

Explanation of the changes: `-F,` tells `awk` the field separator is a comma, so `$3` refers directly to the `status` column without needing `IFS`, `read`, or manual splitting. `$3 == "shipped"` is a direct in-process string comparison — no `echo`, no `grep`, no pipe, no subshell. The `END` block runs once, after the entire file has streamed through, printing the final count. Both versions produce the identical answer (`412903`), but the rewrite spawns exactly one process for the whole file instead of two million — roughly a **158x** speedup (1m38s down to about 0.6 seconds), entirely attributable to eliminating per-line process-spawning overhead, exactly the pattern from this module's core benchmark.

✅ Exercise complete — you've measured a real bottleneck, correctly diagnosed *why* it was slow (process-spawning overhead multiplied by a million loop iterations), and fixed it with a single-pass `awk` rewrite that produces identical output in a small fraction of the time.

---

## ✅ Module Completion Checklist

- [ ] I can explain why script performance matters on the job — blocked pipelines, missed deadlines, wasted compute cost
- [ ] I can measure a script's runtime with `time`, and correctly explain what `real`, `user`, and `sys` each mean and why they can differ
- [ ] I can customize `time`'s output with `TIMEFORMAT`, and time individual sections of a script using `date +%s%N` or `$SECONDS`
- [ ] I can identify the "Useless Use of Cat" (UUOC) anti-pattern and other unnecessary-subshell patterns, and fix them
- [ ] I can explain why spawning an external process once per loop iteration is slow, and prefer Bash builtins instead
- [ ] I can rewrite a bash loop calling external commands line-by-line as a single `awk`/`sed` pass, and benchmark the difference
- [ ] I can parallelize independent work with `xargs -P`, know when GNU `parallel` is a better fit, and recognize when parallelizing does not help
- [ ] I can profile a script's resource usage with `/usr/bin/time -v`, and I know `strace -c` exists for syscall-level profiling

## Next Step

Continue to [Module 18: Security Auditing Scripts](../module18-security-auditing/)
