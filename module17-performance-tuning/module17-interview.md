# 🎤 Module 17 Interview Prep — Performance Tuning & Profiling Scripts

## Conceptual Questions

### 🟢 Beginner

**Q1: Why does script performance actually matter on the job? Isn't it fine if a script "eventually" finishes?**

> "It matters because 'eventually' has real consequences. If a nightly job is supposed to finish before a downstream job starts, a slow script pushes everything after it later — sometimes into business hours, where people are actually waiting on a result that used to just be ready by morning. It also costs real money: cloud compute is billed by the second, so a script that takes 60x longer than it needs to burns through 60x the compute budget for the exact same output, every single time it runs. And often the slowdown is hidden until the input grows — a script that's fine on a small test file can secretly be doing something like spawning a process per line, and that only becomes visible once the file has millions of lines in production."

**Q2: What does the `time` command measure, and what are the three numbers it reports?**

> "`time` runs a command and reports how long it took, broken into three numbers: `real`, which is wall-clock time — the actual time that passed, what a person watching the clock would see; `user`, which is CPU time spent running the program's own code; and `sys`, which is CPU time spent inside the kernel doing system calls on the program's behalf, like reading a file or creating a process. They answer different questions, and reading them together tells you a lot more than just the runtime."

**Q3: What is 'Useless Use of Cat' (UUOC)?**

> "It's piping `cat file` into a command that could just read the file itself directly — `cat file | grep pattern` instead of `grep pattern file`. Both produce identical output, but the first spawns an extra process just to stream bytes into a pipe, when nearly every standard text tool — `grep`, `awk`, `sed`, `wc`, `sort` — already accepts a filename argument and can read the file on its own. On one file it costs a few milliseconds; the reason it's worth knowing is that this exact pattern shows up constantly, often inside loops, where that small cost gets multiplied."

**Q4: What's a Bash 'builtin,' and why is it faster than an external command for small, repeated operations?**

> "A builtin is a command implemented inside the shell itself — things like `[[`, `(( ))`, parameter expansion, and `read` — so it runs as part of the shell process that's already running, with zero extra process-creation cost. An external command like `test`, `expr`, or `grep` has to be forked and exec'd as a brand-new process every single time it runs. For one call, that overhead is invisible. Inside a loop that runs thousands of times, it adds up into real, measurable seconds or minutes, purely from repeatedly paying that process-creation cost."

**Q5: What does `xargs -P` do?**

> "It runs multiple copies of a command in parallel instead of one at a time, reading its arguments from stdin. `xargs -P 4` means up to four instances of the command run simultaneously — useful for independent tasks, like compressing a directory of unrelated log files, where running four at once can use up to four CPU cores instead of one core doing everything sequentially."

### 🟡 Intermediate

**Q6: Explain `real` vs. `user` vs. `sys` time in more depth — specifically, why can `real` be bigger than `user + sys`, and why can it sometimes be smaller?**

> "`real` can be bigger than `user + sys` when a program spends time waiting on something outside the CPU entirely — a slow disk, a network call, another process it's blocked on. That waiting time counts toward wall-clock time but the CPU isn't actually doing anything during it, so it doesn't show up in `user` or `sys` at all. That's the signature of an I/O-bound workload. On the flip side, `user + sys` can add up to *more* than `real` on a multi-core machine, if multiple cores are each burning CPU time during the same wall-clock window — for example, a script that forks off several CPU-heavy children that run in parallel. Each core's CPU-seconds get added into the total `user`/`sys`, but they all happened during overlapping real time, so the sum exceeds the wall-clock duration."

**Q7: Why is calling external commands inside a loop slow? Walk through what actually happens at the OS level.**

> "Every time Bash runs an external command, the operating system has to `fork` — create a new process that's a copy of the calling process — and then `exec` — replace that copy's memory with the new program's code — before the command can do anything at all. That setup is usually well under a millisecond, so for one call it's genuinely negligible. But it's paid every single time, and a loop that calls `grep`, `sed`, `expr`, or `echo | something` once per line of a million-line file pays that fork/exec cost a million times over, for what might be one brick's worth of actual work each time. The fix is almost always the same: replace 'spawn a process per line' with 'spawn one process for the entire file' — which is exactly what a single `awk` or `sed` pass does."

**Q8: When does parallelizing NOT help — or actively hurt?**

> "A few cases. First, if the tasks aren't actually independent — each one depends on the previous one's result — parallelizing them produces wrong results, not just no speedup, so that's disqualifying before performance even enters the discussion. Second, if the workload is I/O-bound on a single shared resource — one disk, one rate-limited API, a database with a small connection pool — throwing more parallel jobs at it can overwhelm that one resource and make everything slower for every job hitting it, not faster. Third, for genuinely CPU-bound work, setting the parallelism level far above the number of physical cores (`nproc`) just adds context-switching overhead without adding real throughput, since there's no extra hardware to actually run those extra jobs simultaneously. Parallelizing helps specifically when tasks are independent and there's real slack to exploit — either waiting time that can overlap, or spare cores that can each do real computation."

**Q9: What's the difference between `TIMEFORMAT` and `/usr/bin/time -v`?**

> "`TIMEFORMAT` only customizes the output of the `time` shell keyword — it reshapes the same three numbers (`real`/`user`/`sys`) using `%`-escapes like `%R`, `%U`, `%S`, and `%P` for percent CPU. `/usr/bin/time -v` is a completely different program — note the full path, since it's not the shell keyword at all — and it reports a much richer resource-usage breakdown: maximum memory used, page faults, voluntary and involuntary context switches, and more, in addition to timing. You'd reach for `TIMEFORMAT` just to make `time`'s existing three numbers easier to log or parse, and reach for `/usr/bin/time -v` when duration alone doesn't answer the question — for example, when a script is getting killed on a smaller server and you need to know its actual peak memory use."

**Q10: How would you time just one section of a long script, rather than the whole thing?**

> "Capture a timestamp before and after the section I'm interested in, using `date +%s%N` for nanosecond precision — subtracting the two and dividing by a million gives milliseconds — or, if second-level precision is good enough, just reset Bash's built-in `$SECONDS` variable to 0 right before the section and read it again right after; it auto-increments once per second on its own. This turns 'the whole script is slow' into 'this specific stage is 90% of the runtime,' which is the difference between guessing where to optimize and actually knowing."

### 🔴 Advanced

**Q11: You're handed a script where `time` shows `real: 45s`, `user: 3s`, `sys: 0.8s`. A teammate wants to parallelize the CPU-heavy parsing step to speed it up. Do you agree?**

> "No, and the numbers explain exactly why. `real` (45s) is vastly larger than `user + sys` (about 4s combined) — that roughly 41-second gap is time the CPU was essentially idle, waiting on something external, almost certainly a slow network call or disk I/O based on the shape of the script. The parsing step, which is the CPU-bound part, only accounts for about 3 seconds total — parallelizing it might shave a second or two off best case, while completely missing the actual bottleneck. The right move is to address the wait itself: figure out what's being waited on, and if it's genuinely independent (multiple downloads, for instance), parallelize *that* instead — not the small CPU-bound step that was never the problem."

**Q12: Design an approach for finding the bottleneck in a data-processing script you've never seen before, that a colleague says 'got slow.'**

> "First, reproduce it and get a baseline with `time`, reading the `real`/`user`/`sys` ratio to decide whether I'm looking at an I/O-bound or CPU-bound problem before I touch anything. If `sys` is unexpectedly high relative to `user`, that's a strong early hint toward excessive system calls — often too many spawned processes, since fork/exec itself is a syscall — rather than genuinely heavy computation. Next, I'd add section timers (`$SECONDS` or `date +%s%N`) around the script's major stages to find which single stage dominates the runtime, since it's almost always concentrated in one place rather than spread evenly. Once I've found the hot stage, I'd read its code looking specifically for a loop that calls external commands per iteration — `grep`, `sed`, `expr`, `echo | grep` patterns — since that's the single most common cause of this exact symptom. If I need more than the loop-reading to confirm it, `/usr/bin/time -v` for memory/page-fault data, or `strace -c` to get a hard syscall count — a table dominated by thousands of `fork` calls is a direct confirmation, not just a guess. Only after all of that would I start rewriting anything."

**Q13: A colleague parallelized a script with `xargs -P 32` on an 8-core machine and it actually got slower. Why might that happen, and how would you fix it?**

> "If the workload is genuinely CPU-bound, running 32 jobs simultaneously on only 8 physical cores means the kernel has to constantly context-switch between far more runnable processes than there are cores to run them on — that switching itself has a real cost, and past a certain point it outweighs whatever benefit parallelism was providing, so throughput can actually drop below what a smaller, more reasonable parallelism level would achieve. There's also a possibility all 32 jobs are contending for one shared bottleneck — a single disk, a rate-limited API, a database connection pool — in which case more concurrency just means more contention, not more real throughput. The fix is to check `nproc` first, and set `-P` at or modestly above the physical core count for CPU-bound work, or — if the real bottleneck is a shared external resource — reduce parallelism to whatever that resource can actually sustain, which might genuinely be lower than the core count."

---

## Practical/Coding Questions

**Q1: Fix the UUOC in this line, and explain the concrete difference between the two versions:**

```bash
cat access.log | grep "500" | wc -l
```

Solution:
```bash
grep -c "500" access.log
```
Explanation: The original spawns three processes (`cat`, `grep`, `wc`) connected by two pipes, purely to count matching lines. `grep -c` counts matching lines itself, natively, so the whole pipeline collapses into a single process with no `cat` and no separate `wc` needed at all — `grep`'s own `-c` flag already does exactly what the pipeline was built to do.

**Q2: Rewrite this bash loop as a single `awk` pass, and explain why the rewrite is faster:**

```bash
total=0
while IFS=, read -r name amount; do
    total=$(echo "$total + $amount" | bc)
done < sales.csv
echo "Total: $total"
```

Solution:
```bash
awk -F, '{ total += $2 } END { printf "Total: %s\n", total }' sales.csv
```
Explanation: The original spawns two processes (`echo` and `bc`) per line to do floating-point addition that `awk` can do natively with `+=`. The rewrite reads and sums the entire file inside one running `awk` process, with zero process spawns per line — on a large file this is the difference between minutes and a fraction of a second, purely from eliminating the fork/exec cost repeated once per line.

**Q3: Given this loop that runs 50,000 times, replace every external-process call with a Bash builtin equivalent:**

```bash
for i in $(seq 1 50000); do
    if [ $(expr "$i" % 3) -eq 0 ]; then
        echo "$i" | sed 's/$/: multiple of 3/'
    fi
done
```

Solution:
```bash
for (( i=1; i<=50000; i++ )); do
    if (( i % 3 == 0 )); then
        echo "$i: multiple of 3"
    fi
done
```
Explanation: `seq` is replaced with a C-style `for` loop (a builtin construct); `expr` and `[ ]` are replaced with `(( ))` arithmetic evaluation, a builtin; and `echo "$i" | sed 's/$/: multiple of 3/'` — which spawned two processes just to append fixed text — is replaced with a single builtin `echo` that constructs the same string directly via normal string interpolation. Zero external processes are spawned anywhere in the rewritten loop.

**Q4: You need to resize 500 independent image files with `convert` (ImageMagick). Write a command that does this in parallel, and explain how you'd choose the parallelism level.**

Solution:
```bash
nproc   # check available cores, e.g. 8
find . -name "*.jpg" | xargs -P 8 -I {} convert {} -resize 50% resized/{}
```
Explanation: Resizing each image is independent of every other image, and it's CPU-bound work (image processing math), so parallelizing genuinely helps — running multiple `convert` processes lets multiple cores each do real computation simultaneously instead of one core doing all 500 sequentially. The parallelism level (`-P 8`) is chosen to match `nproc`'s output rather than picking an arbitrary large number, since going meaningfully above the physical core count for CPU-bound work adds context-switching overhead without adding real throughput.

---

## Gotcha Questions

**Q1: "I ran `time ./script.sh` twice and got different `real` times but nearly identical `user`/`sys` times. Is `time` unreliable?"**

> Trap: `time` isn't unreliable — this is expected, and it's actually informative. `user` and `sys` measure CPU time the process itself consumed, which tends to be fairly consistent run-to-run for the same input and code. `real` (wall-clock time), on the other hand, is also affected by everything *else* happening on the machine at that moment — other processes competing for the CPU, a disk that's momentarily busier, network latency that varies. A `real` time that jumps around while `user`/`sys` stay stable is actually a good diagnostic: it tells you the variation is coming from external contention or waiting, not from the script's own logic behaving inconsistently.

**Q2: "I parallelized my script with GNU `parallel -j 16` and my `user` time in the `time` output went UP compared to running it sequentially. Doesn't parallel mean faster, so shouldn't `user` go down?"**

> Trap: `user` time is the *total* CPU-seconds consumed across all cores, added together — it is not a per-core figure, and parallelizing doesn't reduce the total amount of work being done, it just spreads that same total work across more cores at the same time. So `user` staying the same, or even going up slightly (extra CPU spent on coordination/process management with more parallel jobs), while `real` drops sharply is exactly the expected, healthy signature of successful parallelization — check `real`, not `user`, to judge whether parallelizing actually helped.

**Q3: "My script only calls `grep` a handful of times, not in a loop, but a teammate says it has a UUOC problem. Is a few extra milliseconds really worth fixing?"**

> Trap: The honest answer for an isolated instance is: probably not, on its own — a single `cat file | grep pattern` costs on the order of tens of milliseconds, genuinely not worth a special trip to fix for its own sake. The reason it's still worth *knowing about and fixing when you see it* is that the same pattern, written the same lazy way, tends to get copy-pasted into loops, into other scripts, and into places where the "a few extra milliseconds" assumption quietly stops being true. Treat it as a habit to build (write `grep pattern file` by default), not because any one instance is a crisis, but because the habit is what protects you from the version of this pattern that runs a million times.

---

## Quick-Fire Rapid Review

- **Q: What does `real` time measure?** A: Wall-clock time — the actual elapsed time.
- **Q: What does `user` time measure?** A: CPU time in the program's own (non-kernel) code.
- **Q: What does `sys` time measure?** A: CPU time spent in the kernel on system calls.
- **Q: `real` ≫ `user + sys` suggests what kind of bottleneck?** A: I/O-bound — waiting, not computing.
- **Q: `user` ≈ `real` suggests what kind of bottleneck?** A: CPU-bound — single-core computation dominates.
- **Q: What is UUOC?** A: "Useless Use of Cat" — `cat file | cmd` instead of `cmd file`.
- **Q: Why is calling an external command per loop iteration slow?** A: Each call pays a fork/exec (process-creation) cost, repeated every iteration.
- **Q: What variable customizes `time`'s output format?** A: `TIMEFORMAT`.
- **Q: What two ways can you time just one section of a script?** A: `date +%s%N` (nanosecond precision) or `$SECONDS` (whole seconds).
- **Q: What tool runs commands in parallel by reading arguments from stdin?** A: `xargs -P`.
- **Q: What's a more feature-rich alternative to `xargs -P`?** A: GNU `parallel`.
- **Q: How do you check how many CPU cores are available before picking a parallelism level?** A: `nproc`.
- **Q: What does `/usr/bin/time -v` show that plain `time` doesn't?** A: Max memory (resident set size), page faults, context switches, and more.
- **Q: What does `strace -c` show?** A: A summary table of system calls made and time spent in each.
- **Q: Name three Bash builtins that replace external processes in a loop.** A: `[[ ]]`, `(( ))`/`$(( ))`, and parameter expansion (e.g. `${var%pattern}`).
- **Q: When does parallelizing NOT help?** A: When tasks aren't independent, or the workload is bottlenecked on one shared resource, or parallelism level far exceeds available cores for CPU-bound work.
