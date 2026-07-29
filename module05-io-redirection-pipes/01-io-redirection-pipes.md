# Module 5: I/O Redirection, Pipes & Filters 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-4

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain the three standard streams — stdin, stdout, stderr — and why errors are kept separate from normal output
- [ ] Redirect output with `>` (overwrite) and `>>` (append), and explain the classic "why did my file get wiped" mistake
- [ ] Redirect input with `<`, and redirect errors with `2>`
- [ ] Combine stdout and stderr correctly with `&>` or `> file 2>&1`, and explain why the *order* of `2>&1` matters
- [ ] Discard unwanted output by sending it to `/dev/null`
- [ ] Build pipelines with `|` and explain the pipeline as a conveyor-belt/assembly-line model
- [ ] Use the core filter commands — `sort`, `sort -n`, `sort -r`, `uniq`, `uniq -c`, `wc -l` — inside a pipeline
- [ ] Use `tee` to write to a file and the screen at the same time, and use `xargs` to feed piped output as arguments to commands that can't read stdin directly
- [ ] Use basic command substitution with `$(...)` and know why it's preferred over legacy backticks

---

## Module Goal

By the end of this module, you'll be able to chain small, simple commands together into a single line that answers a real question — instead of reaching for a specialized tool or writing a full script.

🎯 **On the job:** Imagine your web server is getting hammered and you need to know, right now, which IP addresses are hitting it the hardest. The raw access log has 500,000 lines. You don't have a monitoring dashboard open, and writing a Python script would take ten minutes you don't have. What you actually type is something like:

```bash
grep "GET" access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```

One line. Five seconds. Top 10 offending IPs, ranked. This module teaches you every piece of that line — the streams, the redirection operators, the pipe, and the filter commands — so that by the end, building lines like this becomes second nature instead of a mystery you copy-paste from Stack Overflow.

---

## Core Concepts

### 1. What is a "stream"?

A **stream** is just a flowing channel of data — think of it like a hose: data goes in one end and comes out the other, one chunk after another, in order. In Linux, every running program is automatically connected to streams the moment it starts, whether it asks for them or not.

### 2. The three standard streams

Every command-line program gets exactly three of these streams connected to it automatically, and each one has both a name and a number:

| # | Name | Abbreviation | Default destination | What flows through it |
|---|---|---|---|---|
| 0 | **Standard Input** | stdin | Your keyboard | Data flowing *into* the program |
| 1 | **Standard Output** | stdout | Your terminal screen | Normal, successful output |
| 2 | **Standard Error** | stderr | Your terminal screen | Error messages and diagnostics |

💡 **Analogy — the water hose:** Picture your terminal program as a machine with three hoses attached. One hose (stdin) feeds water *in* — normally from your keyboard, one keystroke at a time. Two hoses drain water *out* — stdout carries the "good" water (normal results), and stderr carries the "dirty" water (errors and warnings). By default, both output hoses drain into the same bucket — your screen — which is why you don't normally notice they're separate at all.

**Why does it matter that stdout and stderr are separate, even though they both print to the screen by default?** Because they're separate *pipes*, you can redirect each one independently. You can send the good results to a file for later review while still letting error messages show up immediately on your screen — or the reverse. If they were the same single stream, you'd have no way to separate "what worked" from "what broke."

🎯 **On the job:** A cron job (a scheduled task) that runs a backup script every night is a perfect example. You want the normal progress output saved quietly to a log file, but you want any *errors* emailed to you immediately or written to a separate `errors.log` so you notice a failure fast — that split is only possible because stdout and stderr are two different streams.

### 3. Redirection — pointing a stream somewhere else

**Redirection** means telling the shell "instead of the default destination for this stream, send it here instead." The shell handles this *before* the command even runs — the command itself has no idea its output is going to a file instead of the screen.

💡 **Analogy continued:** Redirection is like taking one of those hoses and pointing it at a bucket (a file) instead of letting it spray onto the floor (the screen). The water (data) doesn't change — only where it lands.

### 4. Pipes — connecting one hose directly to the next machine

A **pipe**, written as `|`, connects the stdout of one command directly to the stdin of the next command — no file involved at all. The data flows straight from one program into the next, in memory, in order.

💡 **Analogy — the assembly line:** Think of a pipeline of commands like a factory conveyor belt. Raw material goes onto the belt at station 1. Station 1 does one small, specific job to it (say, cuts it to size) and puts the result back on the belt. Station 2 takes whatever comes down the belt, does its own small job (say, sands the edges), and passes it on. Nobody at station 3 cares how the part looked before station 1 touched it — they only see what's on the belt right now. Each command in a pipeline is one station: it does one focused job, and passes its output down the belt to the next command.

This is also the core Unix philosophy you first touched in Modules 1-3: small, single-purpose tools (`grep`, `sort`, `uniq`...) that do one thing well, combined together to do something powerful that no single one of them could do alone.

### 5. Filters — commands built to live in the middle of a pipeline

A **filter** is a command that reads from stdin, transforms the data in some way, and writes the result to stdout — making it perfectly suited to sit in the middle of a pipeline (an assembly-line "station"). `sort`, `uniq`, `wc`, and `grep` (which you met in Module 3) are all filters.

---

## Detailed Explanations

### Output redirection: `>` and `>>`

| Operator | Behavior |
|---|---|
| `command > file` | Sends stdout to `file`. If `file` already exists, its **entire contents are wiped first**, then replaced with the new output. |
| `command >> file` | Sends stdout to `file`, **appending** to the end. If `file` doesn't exist yet, it's created. |

⚠️⚠️ **CRITICAL WARNING — the "why did my file get wiped" mistake:** `>` doesn't ask permission and doesn't merge anything. The instant the shell sees `command > important-notes.txt`, it truncates `important-notes.txt` to zero bytes — *before command even runs* — and only then starts writing the new output into it. If `command` fails immediately, or produces no output at all, you're left with a completely empty file where your old data used to be. This has destroyed real production log files and config files more times than anyone wants to admit.

✅ **Best Practice:** When you mean "add to this file," always reach for `>>`. Reserve `>` for cases where you genuinely intend to replace the file's entire contents, and double-check the filename before running it — exactly the same discipline you learned for `rm -rf` in Module 2.

### Input redirection: `<`

`command < file` feeds the contents of `file` into a command's stdin, as if you had typed it at the keyboard. Many commands can already take a filename as an argument (`sort file.txt`), so explicit `<` is used less often in daily work — but it matters for commands that only know how to read from stdin, and it's the conceptual mirror image of `>` that makes the "three streams" model click into place.

```bash
sort < names.txt
```

This reads `names.txt` as if it had been typed into `sort`'s keyboard input, and prints the sorted result to the screen (stdout, still going to its default destination).

### Error redirection: `2>`

Since stderr is stream number **2**, `2>` redirects *only* the error stream, leaving stdout alone.

```bash
grep "config" /etc/*.conf 2> errors.log
```

Here, normal matches still print to the screen, but any "Permission denied" or "No such file" errors get written into `errors.log` instead of cluttering your terminal.

### Combining stdout and stderr — and the ordering gotcha

Sometimes you want *both* streams to go to the same place — for example, saving a full transcript of everything a command did, good or bad. There are two ways to do this:

**The modern shortcut:**
```bash
command &> logfile.txt
```
`&>` redirects both stdout and stderr to `logfile.txt` in one shot. It's a Bash-specific shortcut — clean and simple.

**The explicit, portable form:**
```bash
command > logfile.txt 2>&1
```

⚠️ **This is the single most misunderstood redirection pattern in Bash — read this carefully.** The shell processes redirections **left to right**, and `2>&1` doesn't mean "send stream 2 to the file" — it means "make stream 2 point to wherever stream 1 *currently* points."

Let's trace `command > logfile.txt 2>&1` step by step:
1. `> logfile.txt` — stdout (1) is redirected to point at `logfile.txt`.
2. `2>&1` — stderr (2) is redirected to point at *wherever stdout currently points* — which, thanks to step 1, is now `logfile.txt`. So stderr ends up going to the file too.

Now compare `command 2>&1 > logfile.txt` — the same two pieces, in the opposite order:
1. `2>&1` — stderr (2) is redirected to point at wherever stdout *currently* points — which at this moment is still the terminal screen (nothing has redirected stdout yet). So stderr now points at the screen.
2. `> logfile.txt` — stdout (1) is redirected to point at `logfile.txt`.

The result: stdout goes to the file, but stderr is still pointing at the screen, because it was locked onto the screen *before* stdout ever moved. The two streams end up in different places, even though it looks like you asked for them to be combined.

💡 **Analogy:** Think of `2>&1` as "aim stream 2 at whatever stream 1 is *currently* aimed at" — a snapshot, taken at that exact moment, not an ongoing link. If you take the snapshot before moving stream 1 to its new target, stream 2 just gets a photo of the old, now-abandoned target.

✅ **Best Practice:** Memorize the working order as a fixed phrase: **"file first, then `2>&1`."** `command > file 2>&1` always works. If you ever see it written the other way around and it's not producing the combined file you expect, this ordering rule is almost always why.

### Discarding output: `/dev/null`

`/dev/null` is a special file that Linux provides which **discards anything written to it** and always reports "end of file" immediately when read from. Think of it as a bottomless drain — data goes in, and nothing ever comes back out.

```bash
noisy-command > /dev/null
```

This throws away the normal output but still lets errors show on-screen. To silence *everything*:

```bash
noisy-command > /dev/null 2>&1
```

🎯 **On the job:** You'll use this constantly in scripts and cron jobs to suppress expected, harmless output ("Command completed successfully" printed 500 times) while still letting genuine failures surface.

### Pipes: `|`

```bash
command1 | command2
```

This connects `command1`'s stdout directly to `command2`'s stdin. No temporary file is created — the data streams from one process to the next.

You can chain as many as you like:

```bash
command1 | command2 | command3 | command4
```

Each command only ever needs to worry about two things: what it reads from stdin, and what it writes to stdout. That's the entire mental model — one conveyor belt, as many stations as you need.

### Filter commands for pipelines

| Command | What it does |
|---|---|
| `sort` | Sorts lines alphabetically (default) |
| `sort -n` | Sorts **numerically** — critical for numbers, since plain `sort` treats `"10"` as coming before `"9"` alphabetically |
| `sort -r` | Sorts in **reverse** order (combine as `sort -rn` for "biggest number first") |
| `uniq` | Removes **adjacent** duplicate lines — it does *not* find duplicates scattered throughout a file, only ones sitting next to each other |
| `uniq -c` | Same as `uniq`, but prefixes each line with a **count** of how many times it appeared consecutively |
| `wc -l` | Counts lines — you met `wc` in Module 3 alongside `grep`/`find`; it's one of the most common pipeline endpoints when you just want a total |

⚠️ **Warning:** `uniq` only collapses duplicates that are directly next to each other. If your duplicate lines are scattered throughout the file, `uniq` alone will do nothing useful — you almost always need `sort` *before* `uniq` so that identical lines end up adjacent first. This is why `sort | uniq -c` is such a common pair — sort groups the duplicates together, then uniq -c counts each group.

### `tee` — split the stream, don't just redirect it

`tee` reads from stdin and writes the data to **both** a file **and** stdout at the same time — unlike `>`, which sends output *only* to the file. Named after a plumbing "T-shaped" pipe fitting that splits one flow into two directions.

```bash
some-command | tee output.txt
```

You still see the output live in your terminal, *and* it's saved to `output.txt` for later — you don't have to choose one or the other.

| Flag | Behavior |
|---|---|
| `tee file` | Writes to `file`, overwriting it (like `>`) |
| `tee -a file` | **Appends** to `file` instead of overwriting (like `>>`) |
| `tee file1 file2` | Writes to multiple files at once, plus stdout |

🎯 **On the job:** Running a long deployment script and want to watch it live *and* keep a log for the post-mortem if something breaks? `./deploy.sh | tee deploy.log` gives you both without running the command twice.

### `xargs` — feeding piped output as command-line arguments

Here's a subtlety that trips up almost everyone the first time: **piping output into a command is not the same as passing it as a command-line argument.** Many commands — like `rm`, `cp`, or `echo` — don't read their main input from stdin at all; they expect filenames or values to be listed as *arguments* after the command name.

```bash
find . -name "*.tmp" | rm
```

This does **not** work the way a beginner expects. `rm` doesn't read filenames from stdin — it needs them as arguments — so this pipeline just hands `rm` nothing useful to delete, and `rm` (with no arguments) does nothing (or complains it's missing an operand).

`xargs` exists to bridge exactly this gap: it reads items from stdin and converts them into arguments for the command you give it.

```bash
find . -name "*.tmp" | xargs rm
```

Now `xargs` reads each filename from `find`'s output and builds a command line like `rm file1.tmp file2.tmp file3.tmp ...`, then actually runs it. This is the fix for the broken example above.

**The `-I {}` placeholder** lets you control exactly *where* each item gets inserted in the command, and lets you run the command once *per item* instead of all-at-once:

```bash
find . -name "*.log" | xargs -I {} mv {} archive/{}
```

Here, `{}` is a placeholder that `xargs` substitutes with each individual input item, one at a time, letting you build a more complex command around each one.

⚠️ **Warning — the filename-with-spaces trap:** By default, `xargs` (and `find`'s plain text output) splits items on whitespace and newlines. If a filename contains a space — like `my report.txt` — plain `find | xargs` will mangle it, treating `my` and `report.txt` as two separate items.

✅ **Best Practice — the safe combo:** Use `find -print0` together with `xargs -0`. Both flags switch from newline-separated output to **NUL-byte-separated** output — and since a NUL byte (`\0`) can never legally appear inside a filename, this is 100% safe even for filenames containing spaces, tabs, or newlines.

```bash
find . -name "*.log" -print0 | xargs -0 rm
```

🎯 **On the job:** Any time you're bulk-deleting, bulk-moving, or bulk-processing files found by `find`, on a real server where filenames might contain spaces (very common — think `"Q3 Report.pdf"`), `-print0 | xargs -0` is the professional default, not an edge case you can skip.

### Command substitution: `$(...)` (and legacy backticks)

**Command substitution** runs a command and replaces it, in place, with whatever that command printed to stdout — letting you use a command's output as part of another command or as a value.

```bash
echo "Today is $(date +%A)"
```

Expected output:
```
Today is Tuesday
```

The shell runs `date +%A` first, captures its stdout, and substitutes that text directly into the `echo` command before `echo` ever runs.

You may also see the older syntax using backticks:

```bash
echo "Today is `date +%A`"
```

This does the exact same thing. ⚠️ **Backticks are legacy syntax.** They're harder to read (especially nested — nesting backticks requires escaping with backslashes), and easy to visually confuse with a regular single quote. `$(...)` nests cleanly (`$(command1 $(command2))`) and is the modern standard.

✅ **Best Practice:** Always prefer `$(...)` over backticks in anything you write today. You'll see backticks in older scripts and should recognize them, but don't write new ones.

💡 We're only scratching the surface of command substitution here — full scripting, variables, and control flow are coming in Module 6.

---

## Practical Examples

### Example 1 — Watching stdout and stderr split apart

```bash
ls existing-file.txt missing-file.txt
```

Expected output (mixed together on screen, since both default to the screen):
```
missing-file.txt: cannot access 'missing-file.txt': No such file or directory
existing-file.txt
```

```bash
ls existing-file.txt missing-file.txt 2> errors-only.txt
cat errors-only.txt
```

Expected output:
```
existing-file.txt
```
```
missing-file.txt: cannot access 'missing-file.txt': No such file or directory
```

Line-by-line:
- The first `ls` mixes stdout (the successful listing) and stderr (the error about the missing file) together on-screen — you can't visually tell which is which just by looking.
- Adding `2> errors-only.txt` redirects *only* the error stream into a file, leaving the successful result (`existing-file.txt`) still printing to the screen as normal stdout.
- `cat errors-only.txt` proves the error message really was captured separately.

### Example 2 — `>` overwrite vs `>>` append

```bash
echo "First line" > log.txt
cat log.txt
```
Expected output:
```
First line
```

```bash
echo "Second line" > log.txt
cat log.txt
```
Expected output:
```
Second line
```

⚠️ Notice "First line" is **gone** — `>` truncated the file before writing again.

```bash
echo "First line" > log.txt
echo "Second line" >> log.txt
cat log.txt
```
Expected output:
```
First line
Second line
```

Line-by-line: the second block starts fresh with `>` (wiping the file), then uses `>>` for the second write — this time both lines survive, because `>>` adds to the end instead of replacing.

### Example 3 — The `2>&1` ordering gotcha, proven

```bash
{ echo "to stdout"; echo "to stderr" >&2; } > both.txt 2>&1
cat both.txt
```
Expected output:
```
to stdout
to stderr
```

Now the broken order:

```bash
{ echo "to stdout"; echo "to stderr" >&2; } 2>&1 > broken.txt
cat broken.txt
```
Expected output (only stdout landed in the file):
```
to stdout
```
(and `to stderr` printed straight to your terminal screen instead, because stderr was locked onto the screen *before* stdout got redirected to `broken.txt`.)

Line-by-line:
- In the first command, `> both.txt` runs first (stdout now points at `both.txt`), then `2>&1` points stderr at wherever stdout currently is — `both.txt`. Both lines land in the file.
- In the second command, `2>&1` runs first, while stdout is still the screen — so stderr gets pointed at the screen. *Then* `> broken.txt` moves stdout to the file, but stderr already committed to the screen and doesn't follow. Result: split streams, not combined ones.
- 🎯 **On the job:** this is the exact bug that causes "but my log file is missing all the error messages!" — always write `> file 2>&1`, file first.

### Example 4 — `tee`: see it and save it

```bash
echo "Deploying version 2.3.1..." | tee deploy.log
cat deploy.log
```
Expected output (printed twice — once live, once from `cat`):
```
Deploying version 2.3.1...
```
```
Deploying version 2.3.1...
```

Line-by-line: `tee` printed the line to the screen immediately (that's the first block of output) *and* saved it into `deploy.log` at the same time — `cat deploy.log` afterward proves it was actually written, not just displayed.

### Example 5 — `xargs` in action, safely

```bash
mkdir -p sandbox && cd sandbox
touch "report one.tmp" "report-two.tmp" "notes.tmp"
find . -name "*.tmp" -print0 | xargs -0 -I {} echo "Would delete: {}"
```
Expected output:
```
Would delete: ./report one.tmp
Would delete: ./report-two.tmp
Would delete: ./notes.tmp
```

Line-by-line:
- `find . -name "*.tmp" -print0` finds every `.tmp` file and prints them NUL-separated instead of newline-separated, so `report one.tmp` (which has a space) stays intact as one item.
- `xargs -0` reads that NUL-separated stream correctly, and `-I {}` runs the given command once per item, substituting `{}` with the full filename each time.
- ✅ **Best Practice:** I echoed the command first (a "dry run") before ever swapping in a real `rm` — always rehearse a destructive `xargs` command this way first.

### Example 6 — Full multi-stage pipeline: analyzing a log file

This ties everything in the module together — the exact kind of one-liner described in the Module Goal.

```bash
cat sample-access.log
```
Expected output (a small sample log):
```
203.0.113.5 - - "GET /index.html" 200
198.51.100.9 - - "GET /login" 200
203.0.113.5 - - "GET /admin" 403
203.0.113.5 - - "GET /login" 200
198.51.100.9 - - "GET /index.html" 200
192.0.2.14 - - "GET /admin" 403
203.0.113.5 - - "GET /admin" 403
```

```bash
grep "403" sample-access.log | awk '{print $1}' | sort | uniq -c | sort -rn
```
Expected output:
```
      2 203.0.113.5
      1 192.0.2.14
```

Line-by-line, station by station on the conveyor belt:
1. `grep "403"` — keeps only lines representing failed/forbidden requests (you met `grep` in Module 3).
2. `awk '{print $1}'` — prints just the first field of each line (the IP address). `awk` itself is a deeper tool for another day, but this common one-column-extraction pattern is worth recognizing.
3. `sort` — puts identical IP addresses next to each other, so `uniq` has adjacent duplicates to work with.
4. `uniq -c` — collapses each run of identical adjacent IPs into one line, prefixed with a count.
5. `sort -rn` — sorts those counts **numerically**, in **reverse** (biggest offender first).

🎯 **On the job:** Swap `"403"` for whatever you're hunting — failed logins, 500 errors, a specific username — and this five-stage pipeline pattern (`grep | awk | sort | uniq -c | sort -rn`) answers "who/what is doing this the most" almost instantly, on any server, with tools that are installed everywhere by default.

---

## Common Pitfalls & Best Practices

- **Using `>` when you meant `>>`.** This is the single most common redirection mistake — always pause and ask "do I want to replace this file, or add to it?" before you press Enter.
- **Forgetting that `2>&1` cares about order.** Always write `command > file 2>&1` — file redirection first, then point stderr at wherever stdout now lives. `2>&1 > file` will not combine the streams the way you expect.
- **Assuming `uniq` finds all duplicates.** It only removes *adjacent* duplicate lines. Sort first, or your duplicates will sail right through unnoticed.
- **Piping into commands that don't read stdin.** `find ... | rm` silently fails to do what you want, because `rm` expects arguments, not stdin data. Reach for `xargs`.
- **Running `xargs` on filenames without `-print0`/`-0` in production.** Any filename with a space will get split into multiple bogus arguments. Make `-print0 | xargs -0` your default habit for `find`-driven pipelines that touch real files, not just a special case for "weird filenames."
- **Nesting backticks instead of `$(...)`.** Legacy syntax, awkward to nest, easy to misread. Use `$(...)` in everything you write going forward.
- **Forgetting `/dev/null` silences data permanently.** There's no way to recover what you sent there — treat it the same way you treat `rm`: understand it's genuinely gone.

✅ **Best Practice — rehearse before you destroy:** For any pipeline ending in something destructive (`rm`, `mv` over an existing file), swap the final command for `echo` first and read the output carefully. Only replace `echo` with the real command once you've confirmed exactly what would happen — the same discipline as the Safe Delete Workflow from Module 2.

---

## Hands-on Exercise

**Task:**

1. Create a sample log file called `web.log` in a fresh `logs-exercise` directory, containing at least 10 lines in the format `IP_ADDRESS STATUS_CODE`, with some IPs and status codes repeating (you can type them by hand or generate them — see the solution).
2. Find every line containing a `404` status code and save just those lines to `not-found.log` — using `>>` correctly so a re-run doesn't wipe previous results.
3. Build a one-line pipeline that prints each distinct IP address that appears in `web.log`, along with how many times it appears, sorted from most frequent to least frequent.
4. Use `tee` to run that same pipeline so the ranked results are both shown on-screen *and* saved to `ip-report.txt`.
5. Use `find` and `xargs` to safely list (not delete!) every `.log` file in `logs-exercise`, using the `-print0`/`-0` safe pattern.
6. Combine both stdout and stderr of a command that deliberately references one real file and one missing file into a single file called `combined.log`, using the correct `2>&1` order.

Try this yourself before reading the solution.

### Solution

```bash
# 1. Build the sample log file
mkdir -p logs-exercise && cd logs-exercise
cat > web.log << 'EOF'
203.0.113.5 200
198.51.100.9 200
203.0.113.5 404
203.0.113.5 200
198.51.100.9 404
192.0.2.14 500
203.0.113.5 404
198.51.100.9 200
192.0.2.14 404
203.0.113.5 200
EOF
cat web.log
```
Expected output: the 10 lines exactly as written above.

```bash
# 2. Extract 404s, append-safe
grep "404" web.log >> not-found.log
cat not-found.log
```
Expected output:
```
203.0.113.5 404
198.51.100.9 404
203.0.113.5 404
192.0.2.14 404
```

```bash
# 3. Rank IPs by frequency
awk '{print $1}' web.log | sort | uniq -c | sort -rn
```
Expected output:
```
      4 203.0.113.5
      3 198.51.100.9
      2 192.0.2.14
```

```bash
# 4. Same pipeline, shown AND saved with tee
awk '{print $1}' web.log | sort | uniq -c | sort -rn | tee ip-report.txt
cat ip-report.txt
```
Expected output: the same ranked list, printed twice (once live from the pipeline, once from `cat` proving it was saved).

```bash
# 5. Safely list every .log file, spaces-proof
find . -name "*.log" -print0 | xargs -0 -I {} echo "Found: {}"
```
Expected output:
```
Found: ./web.log
Found: ./not-found.log
Found: ./ip-report.txt
```
(Note: `ip-report.txt` won't actually match `*.log`, so if you named it with a `.log` extension instead it would appear here too — the point of this step is the safe `-print0`/`-0` pattern, not the exact file list.)

```bash
# 6. Combine stdout and stderr correctly
ls web.log this-file-does-not-exist.txt > combined.log 2>&1
cat combined.log
```
Expected output:
```
this-file-does-not-exist.txt: cannot access 'this-file-does-not-exist.txt': No such file or directory
web.log
```

Explanation: I wrote `> combined.log` **before** `2>&1`, so stdout was redirected to the file first, and stderr then followed it there — both the successful listing and the error message ended up in the same file, in the order the shell happened to process them internally. Had I written `2>&1 > combined.log` instead, the error message would have printed to my terminal screen instead of landing in the file, because stderr would have locked onto the screen before stdout ever moved.

✅ Exercise complete — you've now redirected output safely, built and saved a ranked pipeline, and handled files safely with `xargs`, all using real log-analysis patterns you'll use on the job.

---

## ✅ Module Completion Checklist

- [ ] I can explain the three standard streams — stdin, stdout, stderr — and why errors are kept separate from normal output
- [ ] I can redirect output with `>` (overwrite) and `>>` (append), and explain the classic "why did my file get wiped" mistake
- [ ] I can redirect input with `<`, and redirect errors with `2>`
- [ ] I can combine stdout and stderr correctly with `&>` or `> file 2>&1`, and explain why the order of `2>&1` matters
- [ ] I can discard unwanted output by sending it to `/dev/null`
- [ ] I can build pipelines with `|` and explain the pipeline as a conveyor-belt/assembly-line model
- [ ] I can use the core filter commands — `sort`, `sort -n`, `sort -r`, `uniq`, `uniq -c`, `wc -l` — inside a pipeline
- [ ] I can use `tee` to write to a file and the screen at the same time, and use `xargs` to feed piped output as arguments to commands that can't read stdin directly
- [ ] I can use basic command substitution with `$(...)` and know why it's preferred over legacy backticks

## Next Step

Continue to [Module 6: Bash Scripting Fundamentals](../module06-scripting-fundamentals/)
