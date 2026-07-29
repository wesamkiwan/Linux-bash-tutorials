# Module 7: Control Flow 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2.5 hours
**Prerequisites:** Modules 1-6 (Shell Fundamentals, Filesystem Navigation, Viewing/Finding Files, Permissions & Users, I/O Redirection & Pipes, Bash Scripting Fundamentals)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Write `if`/`elif`/`else`/`fi` blocks to branch a script's behavior based on a condition
- [ ] Explain what `test`, `[ ]`, and `[[ ]]` actually are, and confidently choose `[[ ]]` for new Bash scripts while still recognizing `[ ]`
- [ ] Use the full range of comparison operators — string (`=`, `!=`, `-z`, `-n`), numeric (`-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`), and file test (`-f`, `-d`, `-e`, `-r`, `-w`, `-x`, `-s`)
- [ ] Combine conditions with logical operators `&&`, `||`, and `!`
- [ ] Write a `case`/`esac` statement with pattern matching, including `|` for multiple patterns and `*` as a catch-all
- [ ] Write `for` loops over lists, globs, C-style ranges, and command output
- [ ] Write `while` and `until` loops, including a retry loop
- [ ] Use `break` and `continue` (with a level number) to control loop execution, and recognize the `if command; then` pattern as testing a command's real exit status, not just `[ ]`/`[[ ]]`

---

## Module Goal

In Module 6 you learned to build a script that runs a fixed sequence of steps every time. But real automation almost never wants to do the *exact same thing* regardless of circumstances — it needs to **make decisions**.

🎯 **On the job:** Picture a deployment script that has to roll out an update across ten servers. For each server it needs to: check if a required config file actually exists before touching anything, retry the connection a few times if the network hiccups, react differently depending on which environment it's targeting (`dev`, `staging`, or `production`), and skip any server that's already up to date instead of wasting time on it. None of that is possible with a script that just runs top to bottom — it requires **control flow**: the ability to branch, loop, and repeat based on real conditions. This module gives you exactly that toolkit — the one every production script, CI/CD pipeline, and sysadmin one-liner relies on.

---

## Core Concepts

### 1. What is control flow?

**Control flow** is the mechanism that lets a script decide *which* commands to run, and *how many times* to run them, instead of always running every line once, in order, top to bottom.

💡 **Analogy — a flowchart at a fork in the road:** Imagine driving and reaching a fork with a sign: "If the bridge is open, go straight. Otherwise, take the detour." That's a decision — exactly what `if` gives you in Bash. Now imagine a second sign a mile later: "Repeat this loop of the park until you've collected all five checkpoints." That's a loop — exactly what `for`, `while`, and `until` give you. Every script you write from here on is really just a flowchart: some diamonds (decisions) and some loops (repeated segments), connected by plain sequential commands.

### 2. Conditions and exit status — the foundation everything else builds on

Before you can branch or loop, you need a way to ask Bash a yes/no question. Here's the single most important fact in this entire module:

**In Bash, "true" and "false" are not special keywords — they're exit statuses.** You already met exit statuses in Module 6: every command, when it finishes, returns a number from 0-255. By convention, `0` means "succeeded," and any non-zero number means "failed." Bash's `if`, `while`, and `until` don't check for some special boolean value — they simply run a command and look at **its exit status**. Exit status `0` is treated as "true" (proceed with the `then`/loop body); any non-zero exit status is treated as "false."

This is why `[ ]`, `[[ ]]`, and `test` even exist: they're just **commands** — like `ls` or `grep` — whose entire job is to evaluate a condition and exit `0` (true) or `1` (false) accordingly. There's no special "boolean type" in Bash; it's exit statuses all the way down.

### 3. `test`, `[ ]`, and `[[ ]]` — three ways to ask the same question

- **`test`** is an actual external-or-builtin **command** named `test`. You call it like any other command: `test -f myfile.txt`. It evaluates its arguments as a condition and exits `0` or `1`.
- **`[ ]`** is simply **another name for the exact same `test` command** — `[` is a program/builtin that behaves identically to `test`, with one quirk: it requires a literal closing `]` as its very last argument. `[ -f myfile.txt ]` and `test -f myfile.txt` do exactly the same thing.
- **`[[ ]]`** is different — it's a **Bash keyword** (parsed specially by the shell itself, not a regular command), introduced later to fix several rough edges in `[ ]`.

Because `[` and `test` are ordinary commands, Bash expands their arguments *before* they ever run — exactly like it would for any other command. That means all of `[ ]`'s classic gotchas (word splitting on unquoted variables, `*` being expanded by the shell as a glob) apply to it, the same as they'd apply to `ls` or `grep`. `[[ ]]`, being a shell keyword rather than a command, gets special parsing that avoids those problems.

### 4. Why `[[ ]]` is generally preferred in modern Bash

`[[ ]]` fixes three real, common problems with `[ ]`:

1. **No word-splitting or globbing surprises.** Inside `[[ ]]`, you can safely write `[[ $name == "Mary Jane" ]]` without quoting `$name` — Bash treats it as a single word regardless of spaces. Inside `[ ]`, the same unquoted variable can split into multiple arguments and break the whole test, exactly like the unquoted-variable bugs from Module 6.
2. **`&&`, `||`, and `<`/`>` work naturally inside it.** `[[ -f file.txt && -r file.txt ]]` just works. Inside `[ ]`, `&&` doesn't mean what you'd expect (it would end the `[ ]` command and try to run a second command with `&&` joining them), so you're forced to use the single-character `-a`/`-o` operators — which are also officially considered deprecated by Bash's own documentation.
3. **Pattern and regex matching.** `[[ ]]` supports `==` with glob patterns (`[[ $file == *.txt ]]`) and `=~` with full regular expressions (`[[ $line =~ ^[0-9]+$ ]]`) — neither is available in `[ ]` at all.

✅ **Best Practice:** For any script you're writing today, targeting Bash specifically (not a POSIX `sh` script that must run under `dash`), use `[[ ]]`. Recognize `[ ]`/`test` fluently, because you'll read them constantly in existing scripts, tutorials, and any script that must stay POSIX-portable — but write new Bash code with `[[ ]]`.

⚠️ **Warning:** If a script's shebang is `#!/bin/sh` (or it's explicitly written to be POSIX-portable), you **cannot** use `[[ ]]` — it's a Bash-only extension and will error under `dash`. This course targets `#!/bin/bash`, so `[[ ]]` is fair game throughout.

### 5. `if` / `elif` / `else` / `fi` — the basic decision structure

```bash
if condition; then
    # runs if condition's exit status is 0
elif other_condition; then
    # runs if condition failed but other_condition succeeded
else
    # runs if nothing above matched
fi
```

Every `if` block must be closed with `fi` ("if" spelled backwards — a Bash convention you'll see again with `case`/`esac` and `do`/`done`). `elif` and `else` are both optional — you can have a bare `if ... fi` with nothing else.

### 6. `case` / `esac` — matching one value against many patterns

A `case` statement compares one value against a list of **patterns**, running the block for the first pattern that matches:

```bash
case "$value" in
    pattern1)
        # commands
        ;;
    pattern2|pattern3)
        # commands for either pattern2 OR pattern3
        ;;
    *)
        # catch-all, matches anything not matched above
        ;;
esac
```

`case` is Bash's equivalent to a `switch` statement in other languages, but its real superpower is **pattern matching** (using the same glob syntax as filename matching), not just exact equality.

### 7. Loops — `for`, `while`, and `until`

A **loop** repeats a block of commands, either once per item in a list, or for as long as (or until) a condition holds:

| Loop | Repeats... |
|---|---|
| `for item in list; do ... done` | Once per item in a fixed list, glob, or command output |
| `for ((i=0; i<10; i++)); do ... done` | C-style — while a numeric condition holds, with an explicit counter |
| `while condition; do ... done` | For as long as `condition`'s exit status is `0` (true) |
| `until condition; do ... done` | For as long as `condition`'s exit status is **non-zero** (false) — the mirror image of `while` |

💡 **Tip:** `until` is just `while` with the condition's meaning flipped. Most people reach for `while` almost all the time and save `until` for cases where phrasing it as "keep going *until* this becomes true" reads more naturally — like "retry until the server responds."

### 8. `break` and `continue`

Inside any loop:
- **`break`** exits the loop immediately — no more iterations run.
- **`continue`** skips the *rest of the current iteration* and jumps straight to the next one.

Both accept an optional **level number** — `break 2` or `continue 2` — meaning "apply this to the loop two levels out," which matters once loops are nested inside each other.

### 9. The `if command; then` pattern — testing real commands, not just `[ ]`

Here's something that trips up a lot of learners: `if` doesn't require `[ ]` or `[[ ]]` at all. Since `if` just runs whatever command follows it and checks the exit status, you can put **any command** there:

```bash
if grep -q "ERROR" logfile.txt; then
    echo "Found an error in the log"
fi

if ping -c 1 example.com > /dev/null; then
    echo "Host is reachable"
fi
```

`[ ]` and `[[ ]]` are just two commands among many that happen to be *designed* for use in conditions — but `if` works with literally any command, including the ones you already know like `grep`, `ping`, `mkdir`, or a script you wrote yourself.

🎯 **On the job:** This is exactly the pattern from Module 6's `if my-deploy-script.sh; then echo "deployed"; fi` — no brackets in sight, because the condition being tested is simply "did this real command succeed?"

---

## Detailed Explanations

### `[ ]` vs `[[ ]]` vs `test` — side by side

| | `test` | `[ ]` | `[[ ]]` |
|---|---|---|---|
| **What it is** | A command | The exact same command, invoked as `[` | A Bash keyword (special shell syntax) |
| **Closing token required?** | No | Yes — literal `]` as the last argument | Yes — `]]` |
| **Word-splitting/globbing on unquoted variables?** | Yes — must quote carefully | Yes — must quote carefully | No — safe even unquoted (still good habit to quote) |
| **`&&` / `\|\|` inside the brackets** | Not supported directly (use `-a`/`-o`, deprecated) | Not supported directly (use `-a`/`-o`, deprecated) | Fully supported: `[[ a && b ]]` |
| **Pattern matching (`==` with `*`)** | No | No | Yes: `[[ $f == *.txt ]]` |
| **Regex matching (`=~`)** | No | No | Yes: `[[ $s =~ ^[0-9]+$ ]]` |
| **Portable to POSIX `sh`/`dash`?** | Yes | Yes | **No** — Bash (and a few other modern shells) only |
| **Recommended for new Bash scripts?** | Recognize, don't write | Recognize, don't write | ✅ **Yes — use this** |

✅ **Best Practice:** The simplest rule that will serve you correctly almost every time: **write `[[ ]]`, read and recognize `[ ]`/`test`.**

### Why the spaces inside `[ ]` and `[[ ]]` are not optional

Both `[` and `[[` are their own distinct "words" to the shell — `[` is genuinely a command name, and `[[` is a keyword token. Bash only recognizes them as such if they're **surrounded by spaces**, exactly like any other command needs a space before its arguments. `[-f file.txt]` (no spaces) isn't parsed as "the `[` command with arguments `-f`, `file.txt`, and `]`" — it looks like one long, meaningless word to Bash, and you'll get a confusing "command not found" or syntax error. Always write `[ -f file.txt ]` and `[[ -f file.txt ]]` with a space right after the opening bracket(s) and right before the closing one.

### String comparison operators

| Operator | Meaning | Example |
|---|---|---|
| `=` (or `==` inside `[[ ]]`) | Strings are equal | `[[ "$a" == "$b" ]]` |
| `!=` | Strings are not equal | `[[ "$a" != "$b" ]]` |
| `-z` | String is empty (zero length) | `[[ -z "$a" ]]` |
| `-n` | String is not empty | `[[ -n "$a" ]]` |

⚠️ **Warning:** Inside `[ ]`, only a single `=` is officially correct for string comparison (`==` happens to work too in Bash's `[ ]` as an extension, but isn't portable to other `test`/`[` implementations). Inside `[[ ]]`, `==` is the idiomatic form and also enables glob pattern matching. Either way — `=`/`==` is for **strings**. Using it to compare numbers is one of the most common real-world bugs, covered below.

### Numeric comparison operators

| Operator | Meaning |
|---|---|
| `-eq` | Equal |
| `-ne` | Not equal |
| `-lt` | Less than |
| `-le` | Less than or equal |
| `-gt` | Greater than |
| `-ge` | Greater than or equal |

These only make sense for **integers**. `[[ "$count" -eq 5 ]]` asks "does the number in `$count` equal 5?" — a completely different question from `[[ "$count" == "5" ]]`, which asks "is the *text* in `$count` character-for-character the string `5`?" They often give the same answer, but not always — `"05"` equals `5` numerically (`-eq`) but not as a string (`==`).

### File test operators

| Operator | True when... |
|---|---|
| `-f` | Path exists **and is a regular file** |
| `-d` | Path exists **and is a directory** |
| `-e` | Path exists (any type at all) |
| `-r` | Path exists and is **readable** by you |
| `-w` | Path exists and is **writable** by you |
| `-x` | Path exists and is **executable** by you |
| `-s` | Path exists and has a **size greater than zero** (not empty) |

🎯 **On the job:** `-f` and `-e` are easy to mix up. `-e` just means "something is there" — could be a directory, a device file, anything. `-f` specifically means "a normal file is there." A deployment script checking for a config file should almost always use `-f` — you don't want it to "succeed" just because a directory happens to share that name.

### Logical operators — combining conditions

| Operator | Meaning |
|---|---|
| `&&` | AND — both sides must succeed (exit `0`) |
| `\|\|` | OR — at least one side must succeed |
| `!` | NOT — inverts a condition's result |

These work at **two levels** in Bash, and it's worth being precise about which one you're using:

- **Inside `[[ ]]`**, `&&`/`\|\|` combine conditions *within the same test*: `[[ -f "$file" && -r "$file" ]]`.
- **Between whole commands**, `&&`/`\|\|` chain entire commands together (this is the Module 6-adjacent usage): `mkdir -p /tmp/out && cd /tmp/out` — run `cd` only if `mkdir` succeeded.

```bash
if [[ -f "$file" ]] && [[ -r "$file" ]]; then
    echo "readable file"
fi
# equivalent, and more common:
if [[ -f "$file" && -r "$file" ]]; then
    echo "readable file"
fi
```

`!` negates: `if [[ ! -f "$file" ]]; then echo "missing"; fi` reads naturally as "if NOT a file, then...".

### `case` pattern matching in depth

`case` patterns use the same wildcard syntax as filename globbing (Module 2/3), not regular expressions:

| Pattern | Matches |
|---|---|
| `*` | Anything (including nothing) — the standard catch-all/default |
| `y\|yes\|Y\|Yes` | Any one of several exact alternatives, separated by `\|` |
| `*.txt` | Anything ending in `.txt` |
| `[0-9]*` | Anything starting with a digit |

By default each pattern's block ends with `;;`, which stops checking further patterns once one matches. Bash also supports `;;&` — after running a matched block, it **keeps checking subsequent patterns** instead of stopping, allowing intentional fallthrough. This is a niche, advanced feature; most `case` statements you write will use plain `;;`, but recognize `;;&` if you see it in someone else's script.

---

## Practical Examples

### Example 1 — `if` / `elif` / `else`: checking a file before acting on it

```bash
#!/bin/bash
#
# check-config.sh — checks whether a config file exists, is readable, or is missing
# Usage: ./check-config.sh <path>

config_path="$1"

if [[ -z "$config_path" ]]; then
    echo "Usage: $0 <path>" >&2
    exit 2
fi

if [[ -f "$config_path" && -r "$config_path" ]]; then
    echo "OK: $config_path exists and is readable."
elif [[ -f "$config_path" ]]; then
    echo "WARNING: $config_path exists but is not readable — check permissions."
    exit 1
else
    echo "ERROR: $config_path does not exist."
    exit 1
fi
```

```bash
chmod +x check-config.sh
./check-config.sh /etc/hostname
./check-config.sh /etc/does-not-exist.conf
```

Expected output:
```
OK: /etc/hostname exists and is readable.
ERROR: /etc/does-not-exist.conf does not exist.
```

Line-by-line:
- `[[ -z "$config_path" ]]` guards against a missing argument first — same defensive habit from Module 6.
- The first real check combines two file tests with `&&` inside a single `[[ ]]`: exists as a regular file **and** is readable.
- `elif` only runs if the first condition was false — here, it narrows down to "it's a file, but not readable," giving a more specific message than a generic failure.
- `else` catches everything else — the file simply isn't there.

💡 **Tip:** Ordering your `elif` chain from most specific to least specific (or vice versa, depending on what reads clearest) gives users much more useful error messages than one generic "something went wrong."

### Example 2 — `case`/`esac`: branching on an environment name

```bash
#!/bin/bash
#
# deploy-target.sh — shows different behavior per environment using case
# Usage: ./deploy-target.sh <environment>

environment="$1"

case "$environment" in
    dev|development)
        echo "Deploying to DEV — no confirmation needed, auto-restart enabled."
        ;;
    staging|stage)
        echo "Deploying to STAGING — running smoke tests first."
        ;;
    prod|production)
        echo "Deploying to PRODUCTION — manual approval required."
        ;;
    "")
        echo "No environment given." >&2
        exit 2
        ;;
    *)
        echo "Unknown environment: '$environment'" >&2
        exit 1
        ;;
esac
```

```bash
chmod +x deploy-target.sh
./deploy-target.sh staging
./deploy-target.sh qa
```

Expected output:
```
Deploying to STAGING — running smoke tests first.
Unknown environment: 'qa'
```

Line-by-line:
- `dev|development` matches either exact word — the `|` lets one block cover multiple accepted spellings.
- `""` matches an empty string, catching the case where no argument was passed at all (`$environment` is empty).
- `*)` is the catch-all — it must come **last**, since `case` stops at the first pattern that matches, and `*` would swallow everything if placed earlier.

✅ **Best Practice:** Always include a `*)` catch-all in production scripts — without one, an unexpected input just silently falls through the whole `case` with no error at all, which is a much harder bug to notice.

### Example 3 — `for` loop over a list, and over a `*.txt` glob

```bash
#!/bin/bash
#
# for-list-glob.sh — for loop over a fixed list, then over matching files

echo "--- Looping over a fixed list ---"
for server in web01 web02 db01; do
    echo "Checking server: $server"
done

echo "--- Looping over a glob ---"
mkdir -p /tmp/reports
touch /tmp/reports/jan.txt /tmp/reports/feb.txt /tmp/reports/notes.md

for report in /tmp/reports/*.txt; do
    echo "Found report file: $report"
done
```

```bash
chmod +x for-list-glob.sh
./for-list-glob.sh
```

Expected output:
```
--- Looping over a fixed list ---
Checking server: web01
Checking server: web02
Checking server: db01
--- Looping over a glob ---
Found report file: /tmp/reports/jan.txt
Found report file: /tmp/reports/feb.txt
```

Line-by-line:
- The first loop iterates once per word in the plain list `web01 web02 db01` — the classic "for each item in this list" form.
- `for report in /tmp/reports/*.txt` lets Bash expand the glob **before** the loop runs, producing a list of exactly the matching files — `notes.md` is correctly excluded since it doesn't match `*.txt`.

⚠️ **Warning:** If a glob matches **nothing**, by default Bash leaves the literal pattern text (`*.txt`) as a single "match" instead of an empty list, so the loop body would run once with a nonsense value like `/tmp/reports/*.txt`. Guarding with `[[ -f "$report" ]]` inside the loop (as in Example 9 below) protects against this.

### Example 4 — C-style `for` loop

```bash
#!/bin/bash
#
# c-style-for.sh — classic counting loop

for ((i = 1; i <= 5; i++)); do
    echo "Attempt number $i"
done
```

```bash
chmod +x c-style-for.sh
./c-style-for.sh
```

Expected output:
```
Attempt number 1
Attempt number 2
Attempt number 3
Attempt number 4
Attempt number 5
```

Line-by-line:
- `((i = 1; i <= 5; i++))` mirrors the C-language for-loop: an initializer (`i = 1`), a condition checked before every iteration (`i <= 5`), and an update run after every iteration (`i++`).
- Notice there's no `$` needed on `i` inside the double parentheses — arithmetic context (from Module 6's `$((...))`) doesn't require it, though `$i` also works there.

💡 **Tip:** Reach for the C-style `for` when you need a numeric counter with fine control (start, stop, step); reach for `for item in list` when you're iterating over real things (files, servers, words) rather than counting.

### Example 5 — `while` loop reading input, and a retry loop

```bash
#!/bin/bash
#
# while-retry.sh — a while loop that retries a flaky command up to N times

max_attempts=3
attempt=1
success=false

while [[ "$attempt" -le "$max_attempts" ]]; do
    echo "Attempt $attempt of $max_attempts..."

    # Simulate a flaky command: fails unless attempt is 3 (for this demo)
    if [[ "$attempt" -eq 3 ]]; then
        echo "  Connected successfully."
        success=true
        break
    else
        echo "  Connection failed. Retrying..."
    fi

    attempt=$((attempt + 1))
done

if [[ "$success" == "true" ]]; then
    echo "Result: SUCCESS after $attempt attempt(s)."
    exit 0
else
    echo "Result: FAILED after $max_attempts attempts." >&2
    exit 1
fi
```

```bash
chmod +x while-retry.sh
./while-retry.sh
```

Expected output:
```
Attempt 1 of 3...
  Connection failed. Retrying...
Attempt 2 of 3...
  Connection failed. Retrying...
Attempt 3 of 3...
  Connected successfully.
Result: SUCCESS after 3 attempt(s).
```

Line-by-line:
- `while [[ "$attempt" -le "$max_attempts" ]]` keeps looping as long as we haven't exhausted our retry budget.
- `break` exits the loop the moment we succeed — no point in burning remaining attempts.
- `attempt=$((attempt + 1))` (Module 6's arithmetic expansion) advances the counter — forgetting this line entirely is the classic way to write an infinite loop, covered in Pitfalls below.

🎯 **On the job:** This exact shape — loop, try, check, break on success, increment, fail after N attempts — is the backbone of retry logic for flaky network calls, waiting for a service to become healthy after a restart, or polling a long-running job until it finishes.

### Example 6 — `until` loop: the mirror image of `while`

```bash
#!/bin/bash
#
# until-demo.sh — counts up using until instead of while

count=1

until [[ "$count" -gt 5 ]]; do
    echo "Count is: $count"
    count=$((count + 1))
done

echo "Done — count exceeded 5."
```

```bash
chmod +x until-demo.sh
./until-demo.sh
```

Expected output:
```
Count is: 1
Count is: 2
Count is: 3
Count is: 4
Count is: 5
Done — count exceeded 5.
```

Line-by-line:
- `until [[ "$count" -gt 5 ]]` runs the loop body for as long as the condition is **false** — i.e., "keep going *until* count is greater than 5."
- This is functionally identical to `while [[ "$count" -le 5 ]]` — pick whichever phrasing reads more naturally for the situation. "Retry until the server responds" reads better as `until`; "while there's still work to do" reads better as `while`.

### Example 7 — `if command; then`: testing a real command, not brackets

```bash
#!/bin/bash
#
# if-command-demo.sh — uses a real command's exit status directly in if

echo "--- Checking if a package is installed ---"
if dpkg -s curl > /dev/null 2>&1; then
    echo "curl is installed."
else
    echo "curl is NOT installed."
fi

echo "--- Checking if a log line contains an error ---"
echo "2026-07-28 INFO: startup complete" > /tmp/app.log
if grep -q "ERROR" /tmp/app.log; then
    echo "Errors found in log!"
else
    echo "No errors found."
fi
```

```bash
chmod +x if-command-demo.sh
./if-command-demo.sh
```

Expected output (assuming curl is installed and the log has no errors):
```
--- Checking if a package is installed ---
curl is installed.
--- Checking if a log line contains an error ---
No errors found.
```

Line-by-line:
- `if dpkg -s curl > /dev/null 2>&1; then` runs `dpkg -s curl` for real, discards its normal output (`> /dev/null`) and its error output (`2>&1` redirects stderr to wherever stdout is now pointing, from Module 5), and branches purely on **whether `dpkg` itself exited `0` or not**.
- `grep -q "ERROR" /tmp/app.log` — the `-q` flag makes `grep` "quiet" (no printed output), returning only its exit status: `0` if a match was found, `1` if not. `if grep -q ...; then` is one of the single most common real-world uses of this pattern.

✅ **Best Practice:** Whenever you're checking "did this real command succeed/find something," put the command directly after `if` — don't wrap it in `[ $(command) ... ]` gymnastics. Save `[[ ]]`/`[ ]` for testing strings, numbers, and files, not for re-checking a command's result indirectly.

### Example 8 — nested loops with `break 2` and `continue 2`

```bash
#!/bin/bash
#
# nested-break-continue.sh — demonstrates break/continue with a level number

for server in web01 web02 web03; do
    echo "Checking server: $server"

    for service in nginx cron ssh; do
        # Simulate: web02's cron check "fails badly" and should abandon this server entirely
        if [[ "$server" == "web02" && "$service" == "cron" ]]; then
            echo "  $service: FATAL — abandoning $server entirely"
            continue 2
        fi

        # Simulate: web03's ssh check "isn't relevant", skip just this one service
        if [[ "$server" == "web03" && "$service" == "ssh" ]]; then
            echo "  $service: skipped (not applicable)"
            continue
        fi

        echo "  $service: OK"
    done

    echo "  Finished checking $server"
done
```

```bash
chmod +x nested-break-continue.sh
./nested-break-continue.sh
```

Expected output:
```
Checking server: web01
  nginx: OK
  cron: OK
  ssh: OK
  Finished checking web01
Checking server: web02
  nginx: OK
  cron: FATAL — abandoning web02 entirely
Checking server: web03
  nginx: OK
  cron: OK
  ssh: skipped (not applicable)
  Finished checking web03
```

Line-by-line:
- The plain `continue` (no number) for `web03`/`ssh` only skips the rest of the **inner** loop's current iteration — the inner loop moves on, and the outer loop's "Finished checking web03" still prints.
- `continue 2` for `web02`/`cron` means "skip the rest of the current iteration, two loops out" — it abandons the rest of the *inner* loop **and** the rest of the *outer* loop's current iteration, jumping straight to the outer loop's next server. Notice "Finished checking web02" never prints.
- `break 2` (not shown running here, but written the same way) would instead **exit both loops entirely** rather than continuing the outer loop's next iteration — use it when one failure means "stop everything," not just "move on to the next item."

⚠️ **Warning:** Plain `break`/`continue` with no number only ever affects the **innermost** loop they're written inside. It's an easy mistake to assume `break` inside a nested loop stops everything — it doesn't, unless you give it a level number.

### Example 9 — putting it together: safe glob loop with a guard

```bash
#!/bin/bash
#
# safe-glob-loop.sh — guards against a glob matching nothing

mkdir -p /tmp/empty-dir

for file in /tmp/empty-dir/*.log; do
    if [[ ! -f "$file" ]]; then
        echo "No .log files found in /tmp/empty-dir."
        break
    fi
    echo "Processing: $file"
done
```

```bash
chmod +x safe-glob-loop.sh
./safe-glob-loop.sh
```

Expected output:
```
No .log files found in /tmp/empty-dir.
```

Line-by-line:
- Since `/tmp/empty-dir` has no `.log` files, the glob `*.log` doesn't expand to anything, and by default Bash leaves the literal text `*.log` as the one "item" the loop receives.
- `[[ ! -f "$file" ]]` catches exactly that situation — the literal string `*.log` is not a real file — and reports it cleanly instead of trying (and failing) to process a fake filename.

💡 **Tip:** This guard pattern is worth memorizing any time you loop over a glob whose match count you can't guarantee in advance — which, on real systems, is most of the time.

---

## Common Pitfalls & Best Practices

- **Missing spaces inside `[ ]`/`[[ ]]`.** `[-f file.txt]` or `[[$a == $b]]` are not valid — both brackets need a space right after the opening one and right before the closing one, because `[`/`[[` must be recognized as separate "words."
- **`=` vs `==` vs `-eq`.** `=`/`==` compare **strings**; `-eq` compares **numbers**. `[[ "$count" == "5" ]]` and `[[ "$count" -eq 5 ]]` usually agree, but `[[ "05" == "5" ]]` is false (different strings) while `[[ "05" -eq 5 ]]` is true (same number) — pick the operator that matches what you're actually comparing.
- **Off-by-one loop errors.** `for ((i = 0; i < 10; i++))` runs 10 times (`0` through `9`); `for ((i = 0; i <= 10; i++))` runs 11 times (`0` through `10`). Always double-check whether you want `<` or `<=` against your intended count.
- **Infinite `while` loops from a forgotten increment.** If you write `while [[ "$attempt" -le "$max_attempts" ]]; do ... done` and forget `attempt=$((attempt + 1))` inside the body, `$attempt` never changes and the loop runs forever. Always double-check that whatever the condition depends on is actually updated somewhere inside the loop body.
- **Forgetting a `*)` catch-all in `case`.** Without one, an unexpected value silently falls through the entire `case` with no output and no error — much harder to debug than a clear "unknown value" message.
- **Using `[ ]` and forgetting to quote variables.** `[ ]`'s word-splitting/globbing quirks (same as any unquoted variable from Module 6) still apply — `[ $name == "" ]` can break if `$name` is empty or contains spaces. `[[ ]]` avoids this, which is itself one of the strongest reasons to prefer it.
- **Assuming a glob with no matches produces an empty loop.** As shown in Example 9, an unmatched glob leaves the literal pattern text behind by default — always guard with a file test inside the loop if the match count isn't guaranteed.
- **Confusing `break`/`continue` scope in nested loops.** Plain `break`/`continue` only ever affects the *innermost* loop. Use `break 2`/`continue 2` (or higher) deliberately when you mean to affect an outer loop.

---

## Hands-on Exercise

**Task:** Write a script called `fleet-check.sh` that:

1. Defines a fixed list of server names: `web01`, `web02`, `db01`.
2. For each server, uses a `case` statement to print a different message depending on whether the name starts with `web` (pattern `web*`) or `db` (pattern `db*`) — anything else should print "Unknown server type" (catch-all).
3. Separately, loops over the numbers 1 through 3 using a C-style `for` loop, simulating a "health check attempt," and uses a `while` loop nested inside to simulate retrying a connection up to 2 times per health check attempt, printing whether it succeeded (treat attempt number 2 of any retry loop as "success" for this simulation) using `break` once it succeeds.
4. Uses an `if `command`; then` pattern to check whether the file `/etc/hostname` exists using the real `test`-free approach — i.e., use `[[ -f /etc/hostname ]]` directly in the `if`, and report "Host identity file present" or "Host identity file missing."
5. Exits `0` if everything ran, exiting `1` only if `/etc/hostname` is missing.

Try writing this yourself before reading the solution below.

### Solution

```bash
#!/bin/bash
#
# fleet-check.sh — loops over servers, retries a simulated health check,
# and verifies the host identity file, reporting status throughout.
# Usage: ./fleet-check.sh

servers=(web01 web02 db01)

# 2. Case statement per server, based on name pattern
echo "--- Server type check ---"
for server in "${servers[@]}"; do
    case "$server" in
        web*)
            echo "$server: web tier server"
            ;;
        db*)
            echo "$server: database tier server"
            ;;
        *)
            echo "$server: Unknown server type"
            ;;
    esac
done

# 3. C-style for loop over health-check attempts, with a nested retry while loop
echo "--- Health check simulation ---"
for ((attempt = 1; attempt <= 3; attempt++)); do
    echo "Health check attempt $attempt:"
    retry=1
    max_retries=2
    connected=false

    while [[ "$retry" -le "$max_retries" ]]; do
        echo "  Retry $retry of $max_retries..."
        if [[ "$retry" -eq 2 ]]; then
            echo "  Connected."
            connected=true
            break
        fi
        retry=$((retry + 1))
    done

    if [[ "$connected" != "true" ]]; then
        echo "  Could not connect after $max_retries retries."
    fi
done

# 4. if command; then pattern — checking a real file test directly
echo "--- Host identity check ---"
if [[ -f /etc/hostname ]]; then
    echo "Host identity file present."
else
    echo "Host identity file missing." >&2
    exit 1
fi

exit 0
```

```bash
chmod +x fleet-check.sh
./fleet-check.sh
```

Expected output (on a normal Ubuntu system where `/etc/hostname` exists):
```
--- Server type check ---
web01: web tier server
web02: web tier server
db01: database tier server
--- Health check simulation ---
Health check attempt 1:
  Retry 1 of 2...
  Retry 2 of 2...
  Connected.
Health check attempt 2:
  Retry 1 of 2...
  Retry 2 of 2...
  Connected.
Health check attempt 3:
  Retry 1 of 2...
  Retry 2 of 2...
  Connected.
--- Host identity check ---
Host identity file present.
```

```bash
echo "Exit code: $?"
```
```
Exit code: 0
```

Explanation: I used an **array** (`servers=(...)`, a preview of Module 8) with `"${servers[@]}"` so the `for` loop iterates over each server name cleanly — you'll cover arrays in depth next module, but this shows control flow already working naturally alongside them. The `case` uses `web*`/`db*` glob patterns rather than exact matches, since that's the whole point of `case` over a simple `if`/`elif` chain of equality checks. The nested loop shows a C-style outer counter driving repeated attempts, with a `while`-based retry loop inside each attempt — `break` exits just the inner `while` the moment `connected` becomes true, letting the outer `for` continue to its next attempt normally. Finally, the host identity check uses `[[ -f /etc/hostname ]]` directly after `if`, with no separate `test`/`[` needed — this is the same "test a real condition" pattern as `if command; then`, just applied to a file test instead of a full external command.

✅ Exercise complete — you've written a script combining `case` pattern matching, nested loops with a retry pattern, and a direct `if [[ ]]` file check with a meaningful exit code.

---

## ✅ Module Completion Checklist

- [ ] I can write `if`/`elif`/`else`/`fi` blocks to branch a script's behavior based on a condition
- [ ] I can explain what `test`, `[ ]`, and `[[ ]]` actually are, and confidently choose `[[ ]]` for new Bash scripts while still recognizing `[ ]`
- [ ] I can use string (`=`, `!=`, `-z`, `-n`), numeric (`-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`), and file test (`-f`, `-d`, `-e`, `-r`, `-w`, `-x`, `-s`) operators correctly
- [ ] I can combine conditions with `&&`, `||`, and `!`
- [ ] I can write a `case`/`esac` statement with pattern matching, including `|` for multiple patterns and `*` as a catch-all
- [ ] I can write `for` loops over lists, globs, C-style ranges, and command output
- [ ] I can write `while` and `until` loops, including a retry loop
- [ ] I can use `break` and `continue` (with a level number) to control loop execution, and recognize the `if command; then` pattern as testing a command's real exit status

## Next Step

Continue to [Module 8: Functions, Arrays & String Manipulation](../module08-functions-arrays-strings/)
