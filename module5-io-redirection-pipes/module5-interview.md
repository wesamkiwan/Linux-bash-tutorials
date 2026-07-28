# 🎤 Module 5 Interview Prep — I/O Redirection, Pipes & Filters

## Conceptual Questions

### 🟢 Beginner

**Q1: What are the three standard streams, and what does each one default to?**

> "Every command-line program automatically gets three streams connected to it. Stream 0 is standard input, stdin — it defaults to the keyboard. Stream 1 is standard output, stdout — normal, successful results, defaulting to the terminal screen. Stream 2 is standard error, stderr — error and diagnostic messages, which also defaults to the terminal screen. Because both stdout and stderr print to the same screen by default, you don't normally notice they're separate streams at all until you start redirecting one of them."

**Q2: Explain stdout vs stderr — why does Linux bother keeping them separate if they both print to the same screen by default?**

> "They're kept separate because you often want to treat 'what worked' differently from 'what went wrong.' If they were a single combined stream, there'd be no way to, say, save a program's normal results to a log file while still letting error messages show up immediately on-screen, or vice versa. Keeping them as two independent streams means each one can be redirected on its own — that's the whole reason the split exists. A cron job is the classic example: you want routine output quietly logged, but you want failures to stand out or get emailed, and that's only possible because stderr isn't mixed into stdout."

**Q3: What's the difference between `>` and `>>`?**

> "`>` redirects stdout to a file and overwrites — truncates — that file first, so any previous contents are gone before the new output is written. `>>` redirects stdout to the same file but appends to the end instead, creating the file if it doesn't exist yet, but never wiping what's already there. Mixing these two up is one of the most common and most damaging redirection mistakes people make."

**Q4: What does `2>` do differently from plain `>`?**

> "Plain `>` (which is really shorthand for `1>`) redirects stdout — the normal output. `2>` redirects only stream 2, stderr — the error output — leaving stdout going to its normal destination. You'd use `2>` when you want to keep seeing normal results on-screen but capture just the error messages into a file for later review."

**Q5: What is a pipe, and how is it different from just redirecting to a file and then reading that file back?**

> "A pipe, written `|`, connects one command's stdout directly to the next command's stdin, entirely in memory — no file is created at any point. Redirecting to a file and then reading it back would work too, functionally, but it's slower, needs cleanup afterward, and needs disk space for potentially huge intermediate data. A pipe streams the data straight from one process into the next as it's produced."

### 🟡 Intermediate

**Q6: Why does `command > file 2>&1` work but `command 2>&1 > file` doesn't combine the streams the way people expect?**

> "The shell processes redirections strictly left to right, and `2>&1` doesn't mean 'send stderr to this file' — it means 'point stderr at wherever stdout currently points, right now.' It's a snapshot, not a live link. In `command > file 2>&1`, the `> file` runs first, so stdout is already pointing at `file` by the time `2>&1` executes — so stderr gets pointed at `file` too, and both streams end up combined there. In `command 2>&1 > file`, the `2>&1` runs first, while stdout is still pointing at the terminal screen — so stderr locks onto the screen. Then `> file` moves stdout to the file afterward, but stderr already committed to the screen and doesn't follow. The result is stdout going to the file and stderr staying on the screen — split, not combined. I always remember it as 'file first, then `2>&1`.'"

**Q7: What does `uniq` actually do, and what's the most common mistake people make with it?**

> "`uniq` removes duplicate lines, but only if they're directly adjacent to each other in the input — it doesn't scan the whole file for duplicates scattered throughout. The most common mistake is running `uniq` on unsorted data and expecting it to catch every duplicate anywhere in the file. It won't. That's why `sort` almost always comes right before `uniq` in a pipeline — sorting groups identical lines together so they become adjacent, and only then can `uniq` collapse them."

**Q8: Why does a command like `find . -name "*.tmp" | rm` fail to do what a beginner expects?**

> "Because `rm` doesn't read the files it should delete from stdin at all — it expects them as command-line arguments. Piping `find`'s output into `rm` just hands `rm` a stream of text it never looks at; `rm` runs with no arguments and does nothing useful, possibly complaining that it's missing an operand. This is exactly the gap `xargs` is built to close — it reads items from stdin and converts them into arguments for whatever command you give it, so `find . -name "*.tmp" | xargs rm` actually works."

**Q9: Why is `find ... -print0 | xargs -0 ...` considered the safe pattern, and what breaks without it?**

> "By default, both `find`'s plain output and `xargs` split items on whitespace and newlines. If a filename contains a space — which is extremely common in the real world, like `Q3 Report.pdf` — plain `find | xargs` will split that single filename into two separate arguments, corrupting the operation. `-print0` makes `find` separate entries with a NUL byte instead of a newline, and `xargs -0` reads that NUL-separated format. Since a NUL byte can never legally appear inside a filename, this combination is unambiguous and safe no matter what characters — spaces, tabs, even newlines — appear in the name."

### 🔴 Advanced

**Q10: In a pipeline like `grep "ERROR" app.log | sort | uniq -c | sort -rn | head -10`, at what point does each command actually start running, and what does that imply about performance on a huge file?**

> "All five commands in the pipeline start essentially at the same time — the shell forks a process for each stage and wires their streams together before any of them produce a byte of output. Data streams through the whole pipeline continuously rather than each stage running to completion before the next one starts. In practice, though, `sort` is a blocking point: it has to see every line of its input before it can emit anything, because it doesn't know if a 'smaller' line might still be coming. So even though all five processes are technically alive concurrently, the effective throughput of this particular pipeline is gated by however long it takes `grep` to finish feeding `sort` the entire matching set — the stages after `sort` can't meaningfully start producing final results until then."

**Q11: What's the practical difference between `$(...)` command substitution and legacy backticks beyond just style preference?**

> "Functionally they do the same thing — run a command and substitute its stdout as text — but `$(...)` nests far more cleanly. To nest backticks you have to escape the inner ones with backslashes, like `` `echo \`date\`` ``, which gets unreadable fast and is easy to get wrong. `$(...)` nests naturally: `$(echo $(date))` just works, with no escaping needed. `$(...)` is also visually distinct from a stray single quote, whereas a backtick can be easy to misread or mistype as one. There's no real functional advantage to backticks anymore — they exist purely for backward compatibility with old scripts."

---

## Practical/Coding Questions

**Q1: You need to run a script and save both its normal output and any errors into one combined log file, in the correct working order. Show the command.**

Solution:
```bash
./deploy.sh > deploy-combined.log 2>&1
```
Explanation: `> deploy-combined.log` redirects stdout to the file first; `2>&1` then points stderr at wherever stdout currently is — which is now the file — so both streams land in `deploy-combined.log` together, in the order the shell interleaves them internally.

**Q2: Given a file `requests.log` where each line starts with an IP address, show a pipeline that prints the single most frequent IP address.**

Solution:
```bash
awk '{print $1}' requests.log | sort | uniq -c | sort -rn | head -1
```
Explanation: extract the first field (IP) from each line, sort so identical IPs become adjacent, collapse each run with a count via `uniq -c`, sort those counts numerically in reverse so the biggest is first, then `head -1` keeps only that top line.

**Q3: You have a directory full of `.bak` files, some with spaces in their names, and need to delete all of them safely. Show the command and explain the safety mechanism.**

Solution:
```bash
find . -name "*.bak" -print0 | xargs -0 rm
```
Explanation: `-print0` makes `find` separate each match with a NUL byte instead of a newline or space, and `xargs -0` parses that NUL-separated stream back into individual arguments correctly — since a NUL byte can never appear inside a real filename, this is unambiguous even for names containing spaces, and is safer than plain `find | xargs rm`, which would break on those same filenames.

**Q4: Show how you'd watch a long-running build's output live in your terminal while also saving it to `build.log`, without running the build twice.**

Solution:
```bash
./build.sh | tee build.log
```
Explanation: `tee` reads the build's stdout and writes it to both the terminal (so you can watch it happen) and `build.log` (so you have a permanent record) at the same time, from a single execution of `build.sh`.

---

## Gotcha Questions

**Q1: "I ran `command > output.txt` and the command crashed immediately with an error — but now `output.txt` is completely empty even though it had important data in it before. Was this a bug?"**

> Trap: This is not a bug — it's exactly how `>` works, and it's the classic "why did my file get wiped" mistake. The shell truncates `output.txt` to zero bytes as part of setting up the redirection, *before* the command itself ever runs. If the command then fails or produces nothing, the file is simply left empty — the old data was destroyed the instant the redirection was set up, regardless of whether the command afterward succeeded. The lesson: `>` is destructive immediately, independent of whether the command that follows actually works.

**Q2: "I wrote `grep 'error' log.txt | uniq -c` and it printed way more lines than the number of distinct error messages I expected. Why didn't `uniq` collapse them?"**

> Trap: `uniq` only removes/counts lines that are *adjacent* to each other in the input — it does not scan the entire stream for matching lines wherever they occur. If the matching `grep` lines aren't already sorted, identical error messages scattered at different points in the file are never next to each other, so `uniq -c` treats each as its own separate run of length 1. The fix is always to insert `sort` before `uniq`: `grep 'error' log.txt | sort | uniq -c`.

**Q3: "My script does `find . -name '*.txt' | xargs cat` and it works fine in testing, but breaks in production where filenames sometimes have spaces. What's the underlying issue, and is `xargs` broken?"**

> Trap: `xargs` isn't broken — it's behaving exactly per its (unsafe-by-default) design. Both `find`'s default output and `xargs`'s default input parsing split on whitespace, so a filename like `march sales.txt` gets treated as two separate arguments, `march` and `sales.txt`, neither of which exists as a file. This isn't a rare edge case in production — filenames with spaces are extremely common (reports, downloads, user uploads). The fix is `find . -name '*.txt' -print0 | xargs -0 cat`, which uses NUL bytes as the separator instead of whitespace, sidestepping the problem entirely.

---

## Quick-Fire Rapid Review

- **Q: What number is stdin?** A: 0.
- **Q: What number is stdout?** A: 1.
- **Q: What number is stderr?** A: 2.
- **Q: Does `>` overwrite or append?** A: Overwrite (truncates the file first).
- **Q: Does `>>` overwrite or append?** A: Append.
- **Q: What must come first for `> file 2>&1` to correctly combine streams?** A: The `> file` part — file redirection before `2>&1`.
- **Q: What's the Bash shortcut for combining both streams to one file?** A: `&>`.
- **Q: What does `/dev/null` do to anything written to it?** A: Discards it permanently.
- **Q: What does a pipe (`|`) connect?** A: One command's stdout to the next command's stdin.
- **Q: Does `uniq` find duplicates anywhere in a file, or only adjacent ones?** A: Only adjacent ones — sort first.
- **Q: What does `uniq -c` add to its output?** A: A count of consecutive occurrences per line.
- **Q: What does `sort -n` do differently from plain `sort`?** A: Sorts numerically instead of alphabetically (so `9` comes before `10`).
- **Q: What does `tee` do that plain `>` doesn't?** A: Writes to a file AND stdout at the same time, instead of only the file.
- **Q: Why can't you just pipe `find` output straight into `rm`?** A: `rm` reads filenames as arguments, not from stdin.
- **Q: What does `xargs` do?** A: Converts piped stdin items into command-line arguments for another command.
- **Q: What's the safe pairing for filenames with spaces?** A: `find -print0` piped into `xargs -0`.
- **Q: What does `{}` mean in `xargs -I {}`?** A: A placeholder substituted with each input item.
- **Q: What's the modern syntax for command substitution?** A: `$(command)`.
- **Q: What's the legacy syntax for command substitution?** A: Backticks: `` `command` ``.
- **Q: Why prefer `$(...)` over backticks?** A: Easier to read and nests cleanly without needing escape characters.
