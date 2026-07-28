# 🎤 Module 14 Interview Prep — Error Handling, Traps & Debugging

## Conceptual Questions

### 🟢 Beginner

**Q1: Why does error handling matter in a Bash script, if the commands themselves already print errors when they fail?**

> "Printing an error message and stopping are two completely different things. By default, Bash prints whatever error a failing command produces and then just moves on to the next line — nothing about that behavior halts the script or flags the overall run as failed. In an unattended script, that's dangerous, because nobody's watching the output scroll by in real time. A script can fail three lines in, keep running the other forty lines against a broken state, and still print a cheerful 'success' message at the end, because that message is just the last line running — it has no idea anything upstream went wrong. Error handling is what turns 'an error was printed somewhere' into 'the script actually stopped and told someone.'"

**Q2: What does `set -e` do?**

> "It tells Bash to exit the script immediately if any command exits with a non-zero status, instead of the default behavior of just continuing to the next line. It's a good default for straight-line scripts, but it has well-known exceptions — it doesn't fire for commands used as the condition of an `if`/`while`, or on either side of `&&`/`||`, because in those cases Bash assumes you're deliberately checking the result yourself."

**Q3: What does `set -u` catch, and what's a concrete example of a bug it prevents?**

> "It makes referencing an unset variable a hard error instead of silently treating it as an empty string. The classic example is a typo: if I write `TARGT_DIR=/tmp/build` instead of `TARGET_DIR=/tmp/build`, and later run `rm -rf "$TARGET_DIR"/*`, without `set -u` that silently becomes `rm -rf ""/*` because the variable was never actually set — which can behave in ways you really don't want next to `rm -rf`. With `set -u`, Bash stops immediately with an 'unbound variable' error the moment `$TARGET_DIR` is referenced, before `rm` ever runs."

**Q4: What is `trap` used for, in one sentence?**

> "It registers a command or function to run automatically when a specific signal or shell event happens — most commonly cleanup code on `EXIT`, so temp files and locks always get removed no matter how the script ends."

**Q5: What's the difference between `bash -n script.sh` and actually running the script?**

> "`bash -n` only parses the script for syntax errors — unmatched quotes, a missing `fi` or `done` — without executing a single line of it. It's the fastest possible sanity check, but it says nothing about whether the script's logic is correct or whether a command inside it will actually succeed. Running it is the only way to see real behavior, which is why `bash -n` is a quick pre-flight check, not a substitute for testing."

### 🟡 Intermediate

**Q6: What does `set -euo pipefail` do, and what are its limits?**

> "It's three independent safety nets combined. `-e` exits the script on most command failures in straight-line code. `-u` turns unset-variable references into hard errors, catching typos before they do damage. `-o pipefail` makes a pipeline's exit status reflect whether *any* stage failed, not just the last one — without it, `set -e` alone only ever looks at the last command in a pipeline, so a completely broken first command can still report success if the last command in the chain happens to exit 0. Together, they're a strong default for production scripts and I put them on the line right after the shebang in basically everything I write. But they're not a silver bullet — `set -e`'s conditional gotchas still apply even with all three enabled: a command used as the condition of an `if`, or a function called that way, can still fail silently as far as `set -e` is concerned. I still write explicit checks and a `die()` call for anything genuinely critical, rather than assuming the strict-mode idiom alone covers it."

**Q7: How does `trap` actually work under the hood?**

> "`trap` tells the shell: when this signal arrives, or this event happens, run this command instead of (or in addition to) the shell's default behavior for it. It takes a command string and one or more signal names or pseudo-signals — `EXIT` fires on any script exit for any reason, `ERR` fires when a command fails in a way `set -e` would react to, `INT` fires on `SIGINT` (Ctrl+C). Multiple traps can be registered for different signals in the same script, and later `trap` calls for the same signal replace earlier ones rather than stacking. It's implemented at the shell level, so it only applies to that shell process and its trap table — a `SIGKILL` sent to the process bypasses the whole trap mechanism entirely, since `SIGKILL` can't be caught by any process, trap handlers included."

**Q8: Why doesn't `set -e` catch every failure? Give a concrete example.**

> "Because Bash deliberately treats a command's exit status differently depending on the context it runs in. If a command is the condition of an `if`, `while`, or `until`, or sits on either side of `&&`/`||`, Bash assumes you're intentionally testing its result yourself, not asking Bash to fail the whole script on your behalf — so `set -e` doesn't fire there, on purpose. For example: `if grep "ERROR" logfile.txt; then echo "found"; fi` — if `grep` finds nothing, it exits 1, but the script keeps running past the `if` block regardless, because that's exactly what testing a condition is supposed to do. The subtler version: if a *function* is called as the condition of an `if`, `set -e` is suspended for everything inside that function call too, even a failing command buried a few lines deep inside it — which surprises almost everyone the first time they hit it."

**Q9: What's the difference between `ERR` and `EXIT` traps?**

> "`EXIT` fires exactly once, whenever the script's process is about to end, for any reason at all — normal completion, an explicit `exit`, or an uncaught failure under `set -e`. It's the reliable place for cleanup that must always happen. `ERR` fires specifically when a command fails in a way that would trigger `set -e` — so it inherits the exact same gotchas as `set -e` itself; a failure inside a conditional won't trigger an `ERR` trap either. `ERR` is useful for centralized logging of exactly what failed and where, using `$LINENO` and `$?` at the moment it fires; `EXIT` is what you use for cleanup you want guaranteed regardless of the failure path."

**Q10: Why is `shellcheck` valuable, and what kinds of bugs does it catch that a human reviewer might miss?**

> "It's static analysis built specifically for shell scripts — it parses the script without running it and flags well-known Bash pitfalls, each tagged with a stable code like SC2086. The single most common one is unquoted variable expansions, which will break on spaces or get expanded as filename globs — something that looks completely fine to a human skimming the code, because it works perfectly on any test path without spaces in it, and only breaks in production when a real file or directory name happens to contain one. It also catches legacy syntax like backtick command substitution, quoting issues inside loops, and dozens of other patterns that are individually easy to miss but collectively responsible for a large share of real-world shell bugs. That's exactly why teams run it automatically in CI rather than relying on code review alone to catch it."

### 🔴 Advanced

**Q11: You inherited a deployment script that uses `set -euo pipefail`, but it still silently continued after a failed step last week. What would you investigate?**

> "First I'd check whether the failing command was inside any kind of conditional — an `if`, a `&&`/`||` chain, or called through a function that was itself used as a condition — since all of those suspend `set -e` by design, and that's the most common reason 'strict mode' doesn't behave as expected. Second, I'd check whether the failure happened inside a pipeline where `pipefail` genuinely applied, or inside a subshell/command substitution where exit-status propagation can behave differently than people expect. Third, I'd check whether the failing command was itself something like a background job (`&`) — `set -e` doesn't wait around to fail on a backgrounded process's eventual exit status the same way it does for a foreground command. I'd add a `trap 'echo "failed at $LINENO"' ERR` temporarily to pinpoint exactly where execution actually continued past, rather than guessing."

**Q12: Explain a scenario where relying only on `set -euo pipefail` gives a false sense of safety, even though the flags are all correctly set.**

> "Imagine a script that checks disk space before starting a big data import: `if check_disk_space; then import_data; else die 'not enough space'; fi`, where `check_disk_space` internally runs `df / | awk '...' | some_comparison` and has a bug — say, a `grep` inside it that fails outright for an unrelated reason, like a missing binary. Because the whole function is called as the condition of an `if`, `set -e` is suspended for everything inside it, so that internal failure is swallowed silently, and the function's overall return value might come back as 0 or 1 for entirely the wrong reason. The script confidently proceeds to import data believing it correctly checked disk space, when in fact the check itself silently malfunctioned. `set -euo pipefail` was on the whole time; it never had a chance to catch this, because the failure happened entirely inside a context those flags are documented not to cover."

**Q13: How would you design error handling for a script that absolutely must not leave partial state behind — for example, one that modifies a production database schema?**

> "I'd treat `set -euo pipefail` as the baseline, not the whole plan. For anything genuinely irreversible, I'd wrap the risky operations in an explicit transaction where the underlying tool supports it, so a mid-operation failure rolls back automatically rather than relying on the script's own control flow. I'd register a `trap` on both `EXIT` and `ERR` that checks whether the operation completed fully and, if not, either rolls back or clearly flags the database as being in a known-bad intermediate state rather than leaving it ambiguous. I'd validate every precondition — connectivity, disk space, an existing backup — with explicit `die()` calls *before* touching anything destructive, since those are exactly the checks that `set -e`'s conditional gotchas require me to handle manually rather than trust to the flags alone. And I'd make sure the trap handlers themselves are simple and unlikely to fail, because a cleanup handler that itself throws an error partway through is one of the nastier edge cases to reason about afterward."

---

## Practical/Coding Questions

**Q1: Fix this script so a missing environment variable can't silently cause a destructive command:**

```bash
#!/bin/bash
rm -rf "$BUILD_DIR"/*
```

Solution:
```bash
#!/bin/bash
set -euo pipefail
: "${BUILD_DIR:?BUILD_DIR must be set}"
rm -rf "${BUILD_DIR:?}"/*
```
Explanation: `set -u` alone would already turn a missing `$BUILD_DIR` into a hard error, but `: "${BUILD_DIR:?BUILD_DIR must be set}"` is the more explicit, self-documenting idiom — it fails immediately with a clear custom message if `BUILD_DIR` is unset *or empty*, which `set -u` by itself wouldn't catch (an empty-but-set variable still passes `set -u`). The `:?` on the actual `rm -rf` line is a second, defense-in-depth guard directly at the point of danger.

**Q2: Given this pipeline, explain what its exit status will be with and without `pipefail`, and fix it so a broken `curl` is never masked:**

```bash
curl -s https://api.example.com/health | grep -q '"status":"ok"'
```

Solution:
```bash
set -o pipefail
curl -s https://api.example.com/health | grep -q '"status":"ok"'
```
Explanation: Without `pipefail`, the pipeline's exit status is whatever `grep -q` returns — 0 if the pattern was found, 1 if not — regardless of whether `curl` itself succeeded at all. If the API is completely unreachable, `curl` fails and produces no output, `grep -q` finds no match (exits 1) purely because there was nothing to search, and depending on how the caller checks this, an unreachable API and a reachable-but-unhealthy API can look identical, or worse, `curl`'s specific network failure is invisible entirely. With `pipefail`, the pipeline's exit status reflects `curl`'s failure directly if it's the one that failed, giving the caller an honest signal about *what* actually went wrong.

**Q3: Write a `trap`-based cleanup for a script that creates a lock file, ensuring the lock is removed even if the script is interrupted with Ctrl+C.**

Solution:
```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/tmp/myscript.lock"
[ -e "$LOCKFILE" ] && { echo "Already running" >&2; exit 1; }
touch "$LOCKFILE"

cleanup() {
    rm -f "$LOCKFILE"
}
trap cleanup EXIT INT

echo "Doing work..."
sleep 30
echo "Done."
```
Explanation: `trap cleanup EXIT INT` registers the same handler for both events — `EXIT` covers normal completion and any `set -e`-triggered failure, `INT` explicitly covers Ctrl+C so the lock file is removed even if the user interrupts the script mid-run. Registering the trap immediately after `touch "$LOCKFILE"` (rather than at the bottom of the script) means the lock is protected from the earliest possible moment it could be left behind.

**Q4: A colleague's script has `if [ $STATUS = "ok" ]` and no `set` options at all. Identify at least two problems `shellcheck` would flag, and fix them.**

Solution:
```bash
if [ "$STATUS" = "ok" ]; then
    echo "All good"
fi
```
Explanation: The unquoted `$STATUS` (SC2086) breaks if `$STATUS` is unset or empty — `[ = "ok" ]` is a syntax error at runtime — or contains spaces/special characters. Quoting it (`"$STATUS"`) guarantees it's always treated as a single argument to `[`, even if empty. Separately, `shellcheck` would also flag the missing `set -u`/`set -e` context as worth considering for a script with no error-handling discipline at all, though that's more of a project-wide recommendation than a single-line fix.

---

## Gotcha Questions

**Q1: "I have `set -e` at the top of my script, and it's still not stopping when `mycommand | grep pattern` fails. Isn't `set -e` supposed to catch every failure?"**

> Trap: The candidate needs to recognize this is the classic pipefail gap — `set -e` alone only ever looks at a pipeline's *overall* exit status, which by default is just the exit status of the *last* command in the pipeline (`grep`, here), regardless of whether `mycommand` itself failed. If `mycommand` fails but produces no output, `grep pattern` legitimately finds no match and exits non-zero too, coincidentally making this specific example look fine — but if `mycommand` fails while still producing *some* output that happens to satisfy `grep`, the pipeline's exit status can even come back 0, fully masking a real upstream failure. `set -e` was never the wrong tool here; it's simply not designed to see inside a pipeline at all — that's specifically what `set -o pipefail` is for.

**Q2: "I added `trap cleanup EXIT` to my script, ran it, and then killed it with `kill -9` mid-run — the temp files it created are still sitting there. Isn't `trap` supposed to guarantee cleanup?"**

> Trap: `trap` guarantees cleanup for every signal and event *that can be caught* — but `SIGKILL` (`kill -9`) is specifically designed by the kernel to be uncatchable by any process, no exceptions, so that there's always a guaranteed way to stop a process even if it's deliberately ignoring everything else. That includes trap handlers — the process is torn down before any trap logic gets a chance to run at all. `trap ... EXIT` genuinely does fire for a normal exit, an explicit `exit N`, or a `SIGTERM` (plain `kill`, which the process *can* catch and act on before actually terminating). `kill -9` bypasses this whole mechanism entirely, by design — the fix isn't a different trap, it's using `kill` (SIGTERM) instead of `kill -9` whenever you need cleanup to actually happen, exactly the same graceful-first discipline from Module 10's signal handling.

**Q3: "My script has `if do_the_thing; then echo ok; fi` under `set -e`, and `do_the_thing` is a function with a command in it that fails — but the script just keeps going past the whole `if` block like nothing happened. Is `set -e` broken inside functions?"**

> Trap: `set -e` isn't broken, and it isn't specifically about functions either — the rule is about *context*, not about whether the failing code lives inside a function. Any command (or function call) used as the condition of an `if`/`while`/`until`, or on either side of `&&`/`||`, has its exit status treated as information being deliberately tested, not as an unconditional error — and that suspension applies to *everything that runs as part of evaluating that condition*, including every command inside a function if the function itself is what's being tested. This is documented, intentional Bash behavior, not a bug, but it's genuinely the single most common surprise people report the first time they lean on `set -e` inside anything beyond simple straight-line scripts.

---

## Quick-Fire Rapid Review

- **Q: What does `set -e` do?** A: Exits the script immediately if any command exits non-zero (with documented exceptions).
- **Q: What does `set -u` do?** A: Turns referencing an unset variable into a hard error.
- **Q: What does `set -o pipefail` do?** A: Makes a pipeline's exit status reflect any failing stage, not just the last one.
- **Q: What's the combined "strict mode" idiom?** A: `set -euo pipefail`.
- **Q: Does `set -e` fire for a command used as an `if` condition?** A: No — that's one of its documented exceptions.
- **Q: What turns tracing on, and what turns it off?** A: `set -x` on, `set +x` off.
- **Q: What variable customizes the `set -x` trace prefix?** A: `PS4`.
- **Q: What event should you use `trap` on to guarantee cleanup code always runs?** A: `EXIT`.
- **Q: Does a `trap ... EXIT` fire if the process receives `SIGKILL`?** A: No — `SIGKILL` can't be caught by any process, trap handlers included.
- **Q: What's the purpose of a `die()` function?** A: A reusable pattern to log an error message and exit with a specific code.
- **Q: What does `bash -n script.sh` check?** A: Syntax only — no execution, no logic checking.
- **Q: What tool performs static analysis on shell scripts to catch common bugs?** A: `shellcheck`.
- **Q: How do you install `shellcheck` on Ubuntu/Debian?** A: `sudo apt install shellcheck`.
- **Q: What does `shellcheck` warning code SC2086 mean?** A: An unquoted variable that risks word-splitting/globbing.
- **Q: What's the standard idiom for a variable that's allowed to be unset under `set -u`?** A: `${VAR:-default}`.
