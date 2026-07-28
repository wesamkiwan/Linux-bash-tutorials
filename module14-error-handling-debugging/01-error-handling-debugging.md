# Module 14: Error Handling, Traps & Debugging 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 3 hours
**Prerequisites:** Modules 1-10 (Shell Fundamentals through Process Management & Signals). Module 6's exit codes (`$?`, `exit N`) and Module 10's signals (`SIGTERM`, `SIGKILL`, `SIGHUP`) are used directly and extensively in this module.

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain why a script that "ran without printing an error" isn't the same thing as a script that succeeded, and why that gap matters in production
- [ ] Use `set -e` to exit on error, and correctly identify the specific situations where it silently does **not** trigger (conditionals, pipelines, function calls used as conditions)
- [ ] Use `set -u` to turn unset-variable references into hard errors, catching typos before they corrupt data or delete the wrong thing
- [ ] Use `set -o pipefail` to make a pipeline fail if **any** stage fails, not just the last one, and explain the specific failure it prevents
- [ ] Combine all three into the `set -euo pipefail` idiom, and explain both why it's a strong default and why it is not a silver bullet
- [ ] Use `set -x` / `set +x` for command tracing, and customize the trace prefix with `PS4` (including showing line numbers)
- [ ] Use `trap` to run cleanup code on `EXIT`, custom handling on `ERR`, and graceful shutdown on `INT`
- [ ] Write a reusable `die()` function that logs a message and exits with a specific code
- [ ] Check a script for syntax errors without running it (`bash -n`), and run static analysis with `shellcheck` to catch entire classes of bugs before they ship

---

## Module Goal

By the end of this module, you'll be able to take a script that merely *works when everything goes right* and turn it into one that is **safe when things go wrong** — which, in production, they eventually always do.

🎯 **On the job:** Picture this: a deployment script runs every night. It pulls the latest code, runs database migrations, restarts the application, and sends a Slack message saying "Deploy succeeded ✅". One night, the database migration step fails halfway through — maybe the disk was full, maybe a network blip dropped the connection. But the script has no error handling. It doesn't check whether the migration actually succeeded; it just marches on to the next line, restarts the application against a half-migrated database, and — because the *last* command in the script (the Slack notification) succeeded — reports total success. Nobody notices anything is wrong until customers start seeing corrupted data the next morning. The script wasn't buggy in the sense of having wrong logic; it was buggy in the sense of having **no opinion about failure at all**. This module is about building scripts that have an opinion: scripts that stop, loudly and immediately, the moment something goes wrong, and that clean up after themselves regardless of how they exit.

---

## Core Concepts

### 1. Why "no error printed" doesn't mean "it worked"

By default, Bash is extremely forgiving. If a command in your script fails — a file doesn't exist, a network call times out, a program isn't installed — Bash's default behavior is to print whatever error message *that command* produces (if any) and then simply **continue to the next line**, exactly as if nothing had happened. Nothing about Bash itself stops the script or flags the failure unless you tell it to.

This is fine for a quick one-off command you're watching interactively — you'll see the error scroll by and react. It's dangerous in an unattended script, because unattended means **nobody is watching**. A silent failure three lines into a fifty-line deployment script can leave the remaining forty-seven lines running against a broken, half-finished state, and the only signal anyone gets is whatever the script prints at the very end — which, as in the Module Goal scenario, might be a cheerful "success" message that has no idea anything upstream went wrong.

💡 **Analogy — a car's safety systems:** Think of a car built with no seatbelt, no airbag, and no collision-detection system. It drives just fine on a normal day. The failure modes only reveal themselves in a crash — and by then it's too late to add them. `set -e`, `set -u`, and `set -o pipefail` are three independent safety systems, each catching a *different* category of failure, the same way a seatbelt (keeps you in your seat), an airbag (cushions the impact), and collision-detection (brakes before impact even happens) each address a different moment and different kind of danger. None of them prevents every possible accident — you can still crash badly enough to get hurt even with all three — but a car with all three is unambiguously safer than one with none, and it's professional negligence to ship one without them today.

### 2. `set -e` — exit immediately on error

**`set -e`** (also written `set -o errexit`) tells Bash: if any command exits with a non-zero status (Module 6's exit codes), stop the whole script immediately instead of continuing to the next line.

```bash
set -e
cp important-file.txt /backup/
echo "Backup complete"
```

Without `set -e`, if `cp` fails (say, `/backup/` doesn't exist), Bash prints `cp`'s own error message and then happily runs `echo "Backup complete"` anyway — a flatly false claim. With `set -e`, the script halts the instant `cp` fails, and `echo "Backup complete"` never runs.

⚠️ **This is not a complete solution by itself** — see Concept 3 for the specific situations where `set -e` does *not* fire, which is the single most misunderstood part of production Bash scripting.

### 3. The `set -e` gotchas — where it silently doesn't fire

`set -e` has a documented, deliberate list of exceptions where a failing command does **not** trigger an exit. Missing these is how people end up with a false sense of security. Three matter most:

**Gotcha A — commands inside a conditional.** If a command is the condition of an `if`, `while`, or `until`, or is combined with `&&`/`||`, `set -e` treats its exit code as *information the script is deliberately checking*, not a fatal error — so it never triggers an exit, no matter what `set -e` is doing elsewhere in the script:

```bash
set -e
if grep "ERROR" logfile.txt; then
    echo "Found errors"
fi
echo "This line always runs, even if grep found nothing (exit 1)"
```

`grep` returning 1 (no match found) does **not** stop the script here — and that's actually correct, expected behavior, because the whole point of putting a command in an `if` is to react to its result yourself, not have Bash react for you.

**Gotcha B — every command except the last in a pipeline (without `pipefail`).** Covered in depth in Concept 4 below, since it deserves its own full explanation.

**Gotcha C — a function's exit status, when the function call itself sits in a conditional.** This is a more subtle version of Gotcha A:

```bash
set -e
check_disk_space() {
    df / | grep -q "100%"   # returns non-zero if disk is NOT full
    echo "Disk is full!"    # only reached if grep found "100%"
}

if check_disk_space; then
    echo "Handling full disk..."
fi
echo "Script continues normally"
```

Because `check_disk_space` is called as the condition of an `if`, `set -e` is suspended for *everything that happens inside that function call* — including the `grep -q` line inside it — even though that `grep` isn't directly written next to the `if`. This surprises almost everyone the first time: people assume `set -e` protects the inside of a function unconditionally, but the *context the function was called in* matters, not just what's written inside the function itself.

✅ **Best Practice:** Never rely on `set -e` alone to catch failures inside conditionals or functions used as conditions. Check the specific command's exit status explicitly (`if ! cmd; then ...`) when you genuinely need to react to failure, and reserve bare `set -e` for the straight-line, unconditional parts of your script.

### 4. `set -u` — error on unset variables

**`set -u`** (also `set -o nounset`) tells Bash to treat any reference to an **unset variable** as a hard error instead of silently substituting an empty string.

```bash
#!/bin/bash
set -u
rm -rf "$TARGET_DIR"/*
```

Without `set -u`, if `TARGET_DIR` was never set (a typo like `TARGT_DIR=/tmp/build` earlier in the script, or a missing argument), `$TARGET_DIR` silently expands to an empty string, and `rm -rf ""/*` becomes... well, on some shells and setups that expands in ways you really don't want anywhere near `rm -rf`. With `set -u`, Bash immediately errors out with `TARGET_DIR: unbound variable` the moment it's referenced, before `rm` ever runs.

💡 **Tip:** `set -u` is one of the cheapest safety nets available — it costs you nothing to add, and its single most common real-world catch is exactly this: a variable-name typo that would otherwise silently become an empty string in a destructive command.

⚠️ **Gotcha:** `set -u` also flags legitimately-optional variables (like positional parameters that weren't passed, e.g. `$1` in a script called with no arguments). Use a default value with `${VAR:-default}` for anything that's allowed to be unset — this is the standard idiom for working safely alongside `set -u`.

### 5. `set -o pipefail` — catching failures hidden inside a pipeline

Bash pipelines (`cmd1 | cmd2 | cmd3`, from Module 5) have a quirk: by default, the **exit status of the whole pipeline is just the exit status of the last command**, regardless of whether anything earlier in the pipeline failed.

**`set -o pipefail`** changes that: the pipeline's exit status becomes non-zero if **any** command in it fails, not just the last one.

This matters enormously combined with `set -e`. Without `pipefail`, `set -e` only ever looks at the *last* command's exit code in a pipeline — so a pipeline can have a completely broken first command and still report success, because the last command in the chain happened to succeed on empty or partial input.

### 6. The combined idiom: `set -euo pipefail`

Putting the three together at the top of a script is a widely-adopted convention:

```bash
#!/bin/bash
set -euo pipefail
```

This is often described as "Bash strict mode." Each flag catches something the others don't:
- `-e` stops the script on most straight-line command failures.
- `-u` stops it on typo'd/missing variable references.
- `-o pipefail` stops it on failures hidden inside a pipeline.

✅ **Best Practice:** Start every new production script with `set -euo pipefail` on the line right after the shebang. It costs nothing and catches an entire category of "it silently kept going" bugs before they ever reach production.

⚠️ **It is not a silver bullet.** All the gotchas from Concept 3 still apply — `set -e` still won't fire inside conditionals or on functions called as conditions, even with `-u` and `pipefail` also enabled. `set -euo pipefail` is a strong *default safety net*, not a guarantee that every possible failure will be caught. You still need to think about error handling deliberately at the specific points where failure is likely (Concept 8's `die()` pattern), especially anywhere a command is deliberately used inside an `if`.

### 7. `set -x` and `PS4` — command tracing for debugging

**`set -x`** (also `set -o xtrace`) makes Bash print every command it's about to execute, with variables already expanded, right before running it — prefixed with `+` by default. Turn it back off with **`set +x`** (a rare case where `+` means "disable" rather than "enable," for historical reasons).

```bash
set -x
name="World"
echo "Hello, $name"
set +x
```

Output:
```
+ name=World
+ echo Hello, World
Hello, World
```

Note the two lines from the `+` trace versus the actual program output below it — tracing shows you the *command as Bash is about to run it*, with substitutions already resolved, which is invaluable when you suspect a variable doesn't contain what you think it does.

**`PS4`** is the variable controlling that `+` trace prefix. A very common customization adds the script name and current line number, which turns tracing from "here's *a* command that ran" into "here's *exactly which line* ran it":

```bash
export PS4='+ ${BASH_SOURCE}:${LINENO}: '
```

### 8. `trap` — running code on a signal or event

**`trap`** registers a command (or function) to run automatically when a specified signal or shell event occurs. General form:

```bash
trap 'command_to_run' EVENT_OR_SIGNAL
```

Three events matter most for production scripts:

- **`EXIT`** — a pseudo-signal that fires whenever the script exits, for *any* reason: normal completion, an explicit `exit`, or an uncaught error under `set -e`. This is the single most reliable place to put cleanup code, because it fires no matter how the script ends.
- **`ERR`** — fires whenever a command fails in a way that would trigger `set -e` (same gotchas from Concept 3 apply — it doesn't fire in the same situations `set -e` wouldn't). Useful for centralized logging of exactly what failed and where.
- **`INT`** — fires on `SIGINT` (Ctrl+C, from Module 10), letting you intercept a user's interrupt and do something graceful (like "are you sure?" or an orderly shutdown) instead of stopping wherever the script happened to be.

💡 **Analogy:** `trap EXIT` is like a "leaving the building" checklist a responsible employee always runs through — lock the door, turn off the lights — regardless of whether they're leaving because their shift ended normally, they got called home in an emergency, or they got sick and had to leave early. The trigger for leaving varies, but the checklist runs every single time.

### 9. The `die()` pattern — a reusable error-handling function

A very common professional convention is a small `die()` function: log a message (often to stderr) and exit with a specific, meaningful code, so error handling looks consistent everywhere in the script instead of being reinvented at every failure point:

```bash
die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}
```

Called as `die "config file not found" 2` — prints the message to stderr and exits with code 2. `${2:-1}` (a `set -u`-safe default, per Concept 4) means the exit code argument is optional and falls back to `1` if omitted.

### 10. Checking syntax without running: `bash -n`

**`bash -n script.sh`** parses the script for syntax errors — unmatched quotes, missing `fi`/`done`, malformed substitutions — **without executing a single line of it**. This is the fastest possible sanity check before running something that might otherwise start doing real (possibly destructive) work with a typo three lines from the end.

```bash
bash -n deploy.sh
```

If the syntax is clean, this produces **no output at all** and exits `0` — silence is success here, which surprises people the first time.

⚠️ **What `bash -n` does *not* catch:** it only checks that the script is grammatically valid Bash. It has no idea whether `$TYPO_VARIABLE` should have been `$TARGET_DIR`, whether a command exists, or whether your logic is correct. For that, you need `shellcheck` (Concept 11) and actually running/testing the script.

### 11. Static analysis with `shellcheck`

**`shellcheck`** is a static analysis tool built specifically for shell scripts — it reads your script without running it and flags an enormous range of common bugs: unquoted variables that will break on spaces, use of `[ ]` where `[[ ]]` was probably intended, comparing strings with `=` in ways that quietly misbehave, unreachable code, and dozens of other well-known Bash pitfalls, each with a stable warning code (like `SC2086`) you can look up.

Install it on Ubuntu/Debian:

```bash
sudo apt update
sudo apt install shellcheck
```

Run it against a script:

```bash
shellcheck deploy.sh
```

✅ **Best Practice:** Professional teams run `shellcheck` automatically in CI (continuous integration — an automated pipeline that checks every proposed change) on every shell script before it's allowed to merge. This catches an entire category of bugs — the kind a human reviewer skims right past because the script *looks* fine — before they ever reach production, for the cost of a few seconds of automated checking.

### 12. Debugging techniques, recap and tying together

You now have a full toolbox for figuring out *why* a script misbehaves:

- **Echo/print debugging** — the simplest technique: temporarily add `echo "DEBUG: variable is $variable"` lines at suspicious points. Crude, but always available, and often the fastest first move.
- **`set -x` / `set +x`** — bracket a suspicious section of an existing script with these to trace exactly what ran and with what values, without modifying the script's actual logic (unlike scattering `echo` everywhere, tracing shows *every* command automatically).
- **`bash -x script.sh`** — the external equivalent of putting `set -x` at the very top of the script, without editing the script's own file at all. Perfect for a quick one-off trace of a script you don't want to modify, or one you don't own.

🎯 **On the job:** In practice, these aren't competing choices — they're often used together. You might run `bash -x script.sh 2>trace.log` to capture a full trace of a failing production script, grep that log for the last few lines before the failure, and use what you find to add a targeted `echo` or two once you've narrowed down which function or loop is actually misbehaving.

---

## Detailed Explanations

### The `set -e` conditional gotcha, in full

The rule, precisely: `set -e` does **not** trigger an exit when the failing command is part of a construct where its exit status is being *tested*, rather than treated as an unconditional error. That includes: the condition of `if`/`elif`, the condition of `while`/`until`, either side of `&&` or `||` (except the very last command in a list, in some Bash versions/configurations — this is exactly the kind of edge case that makes relying on memory dangerous and testing essential), and any command whose result is negated with `!`.

```bash
#!/bin/bash
set -e

echo "Step 1"
false && echo "This never prints"    # false fails, but && short-circuits — no exit triggered
echo "Step 2 — still running!"

if false; then
    echo "unreachable"
fi
echo "Step 3 — still running, even though the if's condition failed!"
```

Realistic output:
```
Step 1
Step 2 — still running!
Step 3 — still running, even though the if's condition failed!
```

Nothing here stops the script, even though `false` (a command that always exits 1) runs twice. This is **correct, documented Bash behavior**, not a bug — but it's precisely why people say "`set -e` doesn't catch everything I expected it to." The fix isn't a different flag; it's recognizing that anywhere you deliberately test a command's result, you've told Bash "I'm handling this myself," and you need to actually handle it — usually with an explicit `else` branch or a `die()` call.

### The `pipefail` masked-failure gotcha, in full

This is the single most dangerous silent-failure pattern in everyday Bash scripting, because it looks completely innocent:

```bash
#!/bin/bash
set -e   # note: no pipefail yet

cat /var/log/does-not-exist.log | grep "ERROR" > /tmp/errors.txt
echo "Exit status of the pipeline: $?"
echo "Log check complete."
```

Realistic output:
```
cat: /var/log/does-not-exist.log: No such file or directory
Exit status of the pipeline: 1
Log check complete.
```

Wait — `$?` printed `1`, so `grep` found no matches (correct, since `cat` produced no output to search). But look closely: `cat` itself failed outright — the file doesn't exist — and the script **did not stop**, because `set -e` (without `pipefail`) only looks at the pipeline's overall exit status, which is `grep`'s exit status, not `cat`'s. `cat`'s real failure is completely invisible to `set -e` here. The script marches on to `"Log check complete."` as if everything were fine, even though the actual log file was never read at all.

Now add `pipefail`:

```bash
#!/bin/bash
set -eo pipefail

cat /var/log/does-not-exist.log | grep "ERROR" > /tmp/errors.txt
echo "Log check complete."
```

Realistic output:
```
cat: /var/log/does-not-exist.log: No such file or directory
```

The script stops immediately after `cat`'s failure — `"Log check complete."` never prints, because with `pipefail` enabled, the pipeline's exit status reflects `cat`'s failure, not just `grep`'s, and `set -e` now correctly sees the whole pipeline as failed.

🎯 **On the job:** This exact pattern — `some_command_that_might_fail | grep pattern` — shows up constantly in log-checking, health-check, and monitoring scripts. Without `pipefail`, a monitoring script checking `curl https://api.internal/health | grep '"status":"ok"'` can report "all clear" on a night the API endpoint was completely unreachable, because `curl`'s failure is invisible and `grep` on empty input just reports "no match" — which some scripts, wrongly, treat as "healthy, just no alerts" rather than "the check itself never actually ran."

---

## Practical Examples

### Example 1 — The `set -e` gotcha, demonstrated end-to-end

```bash
#!/bin/bash
set -e

check_config() {
    grep -q "enabled=true" /etc/myapp/config.ini
}

echo "Checking configuration..."
if check_config; then
    echo "Config says enabled"
else
    echo "Config says disabled, or grep failed for another reason"
fi
echo "Script finished normally"
```

Realistic output (assuming `/etc/myapp/config.ini` doesn't actually exist):
```
Checking configuration...
grep: /etc/myapp/config.ini: No such file or directory
Config says disabled, or grep failed for another reason
Script finished normally
```

Line-by-line:
- `grep -q "enabled=true" /etc/myapp/config.ini` fails — not because the pattern didn't match, but because the **file itself doesn't exist**. `grep` still prints its own error to stderr.
- Because `check_config` was called as the condition of `if`, `set -e` is suspended for that entire call (Gotcha C from Concept 3) — the script does **not** stop, even though the underlying reason for failure (a missing config file) is very different from "config says disabled."
- The `else` branch runs, printing a message that's technically true but misleading about *why* it went there — a missing file and an intentionally-disabled config look identical to this script.

⚠️ **Warning:** This is exactly the kind of bug `set -e` alone will never catch, because the script author explicitly asked Bash to test the result rather than fail on it. The fix is to distinguish the failure reasons explicitly, e.g. checking `[ -f /etc/myapp/config.ini ]` first and calling `die` if it's missing, before ever calling `grep` on it.

### Example 2 — The `pipefail` masked-failure demo

```bash
#!/bin/bash
set -eu   # deliberately WITHOUT pipefail, to show the problem first

echo "=== Without pipefail ==="
false | grep "ERROR" || true
echo "Reached the end (without pipefail) — but the first command actually failed!"
```

Realistic output:
```
=== Without pipefail ===
Reached the end (without pipefail) — but the first command actually failed!
```

Now the fixed version:

```bash
#!/bin/bash
set -euo pipefail

echo "=== With pipefail ==="
false | grep "ERROR" || true
echo "This line only runs because of the trailing || true"
```

Realistic output:
```
=== With pipefail ===
This line only runs because of the trailing || true
```

Line-by-line:
- `false` always exits `1` — it simulates any command that fails (a broken `curl`, a missing file, whatever).
- Without `pipefail`, the pipeline's exit status is `grep`'s (`1`, since there's no output to match "ERROR" in), and the trailing `|| true` (which forces the whole line to exit `0` no matter what) masks even that — the script reaches the end either way in this specific example.
- The point of the second block isn't that it behaves differently here (the explicit `|| true` masks both versions identically) — it's that **without** `pipefail`, `false`'s failure was never even visible to `set -e` in the first place. Remove the `|| true` from both versions and re-run them to see the real difference: the `pipefail` version stops immediately at `false`'s failure; the non-`pipefail` version does not, because it only ever looks at `grep`'s result.

💡 **Tip:** When testing whether `pipefail` matters for a given pipeline, temporarily remove any trailing `|| true` or similar exit-code overrides — they can mask the very difference you're trying to observe.

### Example 3 — A full `trap EXIT` cleanup example

```bash
#!/bin/bash
set -euo pipefail

TMPDIR=$(mktemp -d)

cleanup() {
    echo "Cleaning up temporary directory: $TMPDIR"
    rm -rf "$TMPDIR"
}
trap cleanup EXIT

echo "Working in $TMPDIR"
echo "some data" > "$TMPDIR/scratch.txt"
echo "Processing..."
cat "$TMPDIR/scratch.txt"

false   # simulate an unexpected failure partway through
echo "This line never runs"
```

Realistic output:
```
Working in /tmp/tmp.Xk3pQr9F2A
Processing...
some data
Cleaning up temporary directory: /tmp/tmp.Xk3pQr9F2A
```

Line-by-line:
- `mktemp -d` creates a fresh, uniquely-named temporary directory and stores its path in `TMPDIR`.
- `trap cleanup EXIT` registers the `cleanup` function to run automatically whenever the script exits — for **any** reason.
- The script writes and reads a scratch file normally, then hits `false`, an unconditional failure. Under `set -euo pipefail`, this immediately stops the script — `"This line never runs"` genuinely never runs.
- Despite that abrupt, unplanned exit, `cleanup` still runs and removes the temp directory — because `EXIT` fires on *any* exit path, not just a clean, planned one.

✅ **Best Practice:** Register your `trap cleanup EXIT` **immediately after** creating anything that needs cleaning up (a temp file, a lock file, a background process) — not at the bottom of the script. If the script fails somewhere in between, a trap registered too late never gets the chance to run.

### Example 4 — `trap ERR` for centralized error logging

```bash
#!/bin/bash
set -euo pipefail

log_error() {
    echo "ERROR: command failed at line $1 (exit code $2)" >&2
}
trap 'log_error $LINENO $?' ERR

echo "Starting deployment steps..."
echo "Step 1: pulling latest code... OK"
cp /nonexistent/source/app.tar.gz /opt/releases/
echo "Step 2: this never runs"
```

Realistic output:
```
Starting deployment steps...
Step 1: pulling latest code... OK
cp: cannot stat '/nonexistent/source/app.tar.gz': No such file or directory
ERROR: command failed at line 10 (exit code 1)
```

Line-by-line:
- `trap 'log_error $LINENO $?' ERR` registers a handler that runs whenever a command fails in a way `set -e` would react to. `$LINENO` captures the line number where the failure happened; `$?` captures its exit code — both evaluated at the moment the trap fires, not when it was registered.
- `cp` fails because the source file doesn't exist. Its own error prints first (from `cp` itself), then the `ERR` trap fires, printing a structured, consistent error line naming the exact line and exit code.
- Because `set -e` is active and this failure isn't inside a conditional (Concept 3's gotchas don't apply here), the script actually stops right after — `"Step 2: this never runs"` is accurate.

🎯 **On the job:** A single `ERR` trap like this gives you one consistent, greppable error format across an entire script, rather than each individual command failure looking different depending on which program produced it — genuinely useful when scripts ship their output to a centralized logging system.

### Example 5 — The `die()` function pattern in a realistic script

```bash
#!/bin/bash
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

CONFIG_FILE="/etc/myapp/config.ini"

[ -f "$CONFIG_FILE" ] || die "config file not found: $CONFIG_FILE" 2

command -v curl >/dev/null 2>&1 || die "curl is required but not installed" 3

echo "All prerequisites satisfied, continuing..."
```

Realistic output (assuming the config file is genuinely missing):
```
ERROR: config file not found: /etc/myapp/config.ini
```

Then, checking the exit code:
```bash
echo "Exit code: $?"
```
```
Exit code: 2
```

Line-by-line:
- `die()` is defined once, at the top, and reused everywhere a fatal precondition check is needed — every failure in this script speaks the same "ERROR: ... " format on the same stream (stderr, via `>&2`), and exits with a specific, documented code rather than a generic `1` everywhere.
- `[ -f "$CONFIG_FILE" ] || die ...` is the idiomatic way to react to a test's result under `set -e` — because it's on the *left* side of `||`, `set -e` correctly treats this whole line as "handled" and doesn't need to fire, exactly as intended (this is Concept 3's gotcha used *deliberately and correctly*, not accidentally).
- `command -v curl >/dev/null 2>&1` is the standard portable way to check whether a program is installed without printing its path; redirecting both stdout and stderr to `/dev/null` keeps this check silent either way.

✅ **Best Practice:** Give every distinct failure category in a script its own exit code (documented in a header comment), the same convention introduced for plain `exit N` back in Module 6 — a `die()` function just makes that convention easy to apply consistently everywhere.

### Example 6 — Reading `shellcheck` output on a flawed script

Given this flawed script, `backup.sh`:

```bash
#!/bin/bash
DIR=$1
FILES=`ls $DIR`
for f in $FILES
do
  if [ $f = "important.txt" ]
  then
    cp $DIR/$f /backup
  fi
done
```

Running `shellcheck`:

```bash
shellcheck backup.sh
```

Realistic output:
```

In backup.sh line 2:
DIR=$1
    ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In backup.sh line 3:
FILES=`ls $DIR`
      ^-- SC2006 (style): Use $(...) notation instead of legacy backquoted `...`.
         ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In backup.sh line 4:
for f in $FILES
            ^-- SC2231 (warning): Quote expansions in this for loop glob to prevent wordsplitting, e.g. "$f".

In backup.sh line 6:
  if [ $f = "important.txt" ]
       ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In backup.sh line 8:
    cp $DIR/$f /backup
       ^-- SC2086 (info): Double quote to prevent globbing and word splitting.
```

Line-by-line:
- **SC2086** (appears four times) — the single most common ShellCheck warning: an unquoted variable will be split on whitespace and expanded as a filename glob pattern by the shell. If `$DIR` ever contains a space (`/home/my backups`), `$DIR/$f` silently becomes two separate words instead of one path.
- **SC2006** — backticks (`` `ls $DIR` ``) are the legacy command-substitution syntax; `$(...)` is the modern, nestable, more readable equivalent (this is purely a style/modernization warning, not a correctness bug by itself).
- **SC2231** — looping over an unquoted glob/variable expansion in a `for` risks the exact same word-splitting problem as SC2086, specific to the loop context.
- Notice there's no warning at all about the *logic* being wrong — `shellcheck` doesn't know your intent, only recognized bad patterns. This script could still back up the wrong file entirely if `$DIR` had a typo; `shellcheck` catches how it's written, not what it's supposed to do.

✅ **Best Practice:** Treat `shellcheck` output as a checklist to work through before merging, not noise to silence. Fixing every `SC2086` in this file alone (`"$DIR"`, `"$f"`, `"$FILES"` quoted throughout) would have prevented an entire category of "worked on my machine, broke on a path with a space" bugs.

---

## Common Pitfalls & Best Practices

- **Assuming `set -e` catches everything.** As Concepts 3 and Example 1 covered in depth, `set -e` does not fire inside conditionals, `&&`/`||` chains, or on a function called as a condition. If a failure needs to be reacted to inside one of those constructs, you must check it explicitly — `set -e` will not save you there.
- **Forgetting traps don't fire on `kill -9`.** `SIGKILL` (Module 10) cannot be caught by *any* process, and that includes trap handlers — a script killed with `kill -9` skips `trap ... EXIT` entirely, no cleanup code runs, and any temp files or locks it was holding are simply abandoned. This is exactly the same rule from Module 10 applied to scripts: `SIGTERM` (plain `kill`) still lets your `EXIT`/`INT` traps run; `SIGKILL` never does.
- **Over-trusting a script that "ran without errors."** Especially without `set -euo pipefail`, "no error was printed" and "everything genuinely worked" are not the same claim — as the Module Goal's deployment scenario shows, a script can complete a `false`-success message while having failed silently three steps earlier.
- **Adding `set -euo pipefail` and calling it done.** It's a strong default, not a replacement for actually thinking through failure points. Critical operations (deleting data, deploying to production, financial transactions) deserve explicit checks and a `die()` call with a clear message, even in a script that already has the strict-mode idiom at the top.
- **Forgetting `${VAR:-default}` for genuinely-optional variables under `set -u`.** Turning on `set -u` and then having the script immediately fail on an unset-but-intentionally-optional argument is a common early frustration — use the default-value syntax rather than disabling `set -u` altogether.
- **Registering a cleanup trap too late.** A `trap cleanup EXIT` placed at the bottom of the script does nothing for a failure that happens before that line ever executes. Register it immediately after creating whatever needs cleaning up.
- **Silencing `shellcheck` warnings with a blanket disable instead of understanding them.** `# shellcheck disable=SC2086` on every line defeats the entire point. Fix the underlying issue (usually: add quotes) unless you have a specific, understood reason the warning doesn't apply.

✅ **Best Practice — the production-readiness mindset:** Before shipping any script that isn't purely read-only and interactive, ask: "If any single line in this script fails unexpectedly, what happens to everything after it — and what happens to anything it created along the way?" `set -euo pipefail` plus a cleanup trap plus a `die()` function is how you make the answer to both halves of that question "nothing bad."

---

## Hands-on Exercise

**Task:** Below is a "naive" backup script with no error handling at all. Harden it into a production-grade version.

Naive starting point, `naive-backup.sh`:

```bash
#!/bin/bash
SRC=$1
DEST=$2
TMPDIR=/tmp/backup-work
mkdir $TMPDIR
cp -r $SRC/* $TMPDIR
tar czf $DEST $TMPDIR
rm -rf $TMPDIR
echo "Backup complete"
```

Your job:
1. Check its syntax with `bash -n`.
2. Run it through `shellcheck` and note every warning.
3. Rewrite it with `set -euo pipefail`, a `die()` function for precondition failures (missing arguments, missing source directory), a `trap cleanup EXIT` that removes the temp directory no matter how the script exits, and properly quoted variables throughout.
4. Re-run `shellcheck` on your hardened version to confirm it's clean.

Try hardening it yourself before reading the solution below.

### Solution

**Step 1 — syntax check:**
```bash
bash -n naive-backup.sh
```
```
(no output — syntax is valid Bash; this doesn't mean it's safe, just parseable)
```

**Step 2 — shellcheck:**
```bash
shellcheck naive-backup.sh
```
```

In naive-backup.sh line 2:
SRC=$1
    ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In naive-backup.sh line 3:
DEST=$2
     ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In naive-backup.sh line 5:
mkdir $TMPDIR
      ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In naive-backup.sh line 6:
cp -r $SRC/* $TMPDIR
      ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In naive-backup.sh line 7:
tar czf $DEST $TMPDIR
        ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

In naive-backup.sh line 8:
rm -rf $TMPDIR
       ^-- SC2086 (info): Double quote to prevent globbing and word splitting.
```

**Step 3 — hardened version, `backup.sh`:**

```bash
#!/bin/bash
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

SRC="${1:-}"
DEST="${2:-}"

[ -n "$SRC" ] || die "usage: $0 <source-dir> <dest.tar.gz>" 2
[ -n "$DEST" ] || die "usage: $0 <source-dir> <dest.tar.gz>" 2
[ -d "$SRC" ] || die "source directory does not exist: $SRC" 3

TMPDIR=$(mktemp -d /tmp/backup-work.XXXXXX)

cleanup() {
    echo "Cleaning up: $TMPDIR"
    rm -rf "$TMPDIR"
}
trap cleanup EXIT

echo "Copying $SRC into working directory..."
cp -r "$SRC"/* "$TMPDIR"/

echo "Creating archive at $DEST..."
tar czf "$DEST" -C "$TMPDIR" .

echo "Backup complete: $DEST"
```

**Step 4 — re-run shellcheck on the hardened version:**
```bash
shellcheck backup.sh
```
```
(no output — clean)
```

Explanation of the changes: `set -euo pipefail` at the top means any unexpected failure (a bad `tar` invocation, a permissions error on `cp`) stops the script instead of limping forward to the misleading `"Backup complete"` line. `die()` gives every precondition failure a consistent message-and-exit-code shape, and each check (`$SRC`/`$DEST` provided, `$SRC` actually exists) runs **before** anything destructive or time-consuming happens — checking preconditions early is exactly the discipline `set -e`'s conditional gotcha (Concept 3) requires you to handle yourself, since `[ -n "$SRC" ] || die ...` deliberately sits on the left of `||`. `mktemp -d` replaces the hardcoded `/tmp/backup-work` (which would collide if two backups ever ran at once, or fail confusingly if it already existed from a previous crashed run) with a guaranteed-unique directory. `trap cleanup EXIT`, registered immediately after `TMPDIR` is created, guarantees the temp directory is removed whether the script finishes normally, hits a `die()` call, or fails on an unexpected error — closing the exact "vanishing deploy script" gap Module 10 introduced with `nohup`, applied here to cleanup instead of signal survival. Every variable is quoted (`"$SRC"`, `"$DEST"`, `"$TMPDIR"`), eliminating every one of the six `SC2086` warnings from the naive version.

✅ Exercise complete — you've taken a script with zero error handling and turned it into one with a documented exit-code convention, guaranteed cleanup regardless of how it exits, and a clean `shellcheck` report.

---

## ✅ Module Completion Checklist

- [ ] I can explain why a script that "ran without printing an error" isn't the same thing as a script that succeeded, and why that gap matters in production
- [ ] I can use `set -e` to exit on error, and correctly identify the specific situations where it silently does not trigger (conditionals, pipelines, function calls used as conditions)
- [ ] I can use `set -u` to turn unset-variable references into hard errors, catching typos before they corrupt data or delete the wrong thing
- [ ] I can use `set -o pipefail` to make a pipeline fail if any stage fails, not just the last one, and explain the specific failure it prevents
- [ ] I can combine all three into `set -euo pipefail`, and explain both why it's a strong default and why it is not a silver bullet
- [ ] I can use `set -x` / `set +x` for command tracing, and customize the trace prefix with `PS4`
- [ ] I can use `trap` to run cleanup code on `EXIT`, custom handling on `ERR`, and graceful shutdown on `INT`
- [ ] I can write a reusable `die()` function that logs a message and exits with a specific code
- [ ] I can check a script for syntax errors without running it (`bash -n`), and run static analysis with `shellcheck` to catch bugs before they ship

## Next Step

Continue to [Module 15: Automation & Scheduling](../module15-automation-scheduling/)
