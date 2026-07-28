# 📋 Module 14 Cheat Sheet — Error Handling, Traps & Debugging

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **`set -e` (`errexit`)** | Exit the script immediately if any command exits non-zero (with documented exceptions) |
| **`set -u` (`nounset`)** | Treat referencing an unset variable as a hard error instead of silently using an empty string |
| **`pipefail`** | Makes a pipeline's exit status reflect the first failing command, not just the last one |
| **`set -x` (`xtrace`)** | Print every command (with substitutions expanded) right before running it |
| **`PS4`** | The prefix string shown before each traced command under `set -x` (default `+ `) |
| **`trap`** | Registers code to run automatically when a signal or shell event occurs |
| **`die()`** | A conventional, hand-written function: log a message, exit with a specific code |
| **`shellcheck`** | Static analysis tool that finds common Bash bugs without running the script |

## `set` Options Reference

| Option | Long form | What it catches | Well-known gotcha |
|---|---|---|---|
| `set -e` | `set -o errexit` | Any command exiting non-zero, in straight-line code | Does **not** fire inside `if`/`while`/`until` conditions, `&&`/`||` chains, or on a function called as a condition |
| `set -u` | `set -o nounset` | References to unset/undeclared variables (typos, missing args) | Flags intentionally-optional variables too — use `${VAR:-default}` |
| `set -o pipefail` | (no short flag) | A pipeline where any stage fails, not just the last | Without it, `cmd_that_fails \| grep foo` can report success if `grep`'s own exit code happens to be 0/handled |
| `set -x` | `set -o xtrace` | N/A — a tracing tool, not an error-catcher | Very verbose on large scripts; turn off with `set +x` when done |

## The Combined Idiom

```bash
#!/bin/bash
set -euo pipefail
```

| Claim | True? |
|---|---|
| Catches most straight-line command failures | ✅ Yes (`-e`) |
| Catches typo'd variable names | ✅ Yes (`-u`) |
| Catches failures hidden inside pipelines | ✅ Yes (`pipefail`) |
| Catches every possible failure, no exceptions | ❌ No — `-e`'s conditional gotchas still apply |
| Replaces the need to think about error handling | ❌ No — still write explicit checks + `die()` for critical operations |

## `set -x` / `PS4` Quick Reference

| Command | Effect |
|---|---|
| `set -x` | Turn tracing on (prints `+ command` before each command runs) |
| `set +x` | Turn tracing off |
| `bash -x script.sh` | Trace an entire script externally, without editing it |
| `export PS4='+ ${BASH_SOURCE}:${LINENO}: '` | Customize the trace prefix to show file + line number |
| `bash -x script.sh 2> trace.log` | Capture a full trace to a file for later review |

## `trap` Reference

| Signal/Event | Fires when | Typical use |
|---|---|---|
| `EXIT` | The script exits, for **any** reason (normal, `exit N`, uncaught error) | Cleanup: remove temp files/dirs, release locks |
| `ERR` | A command fails in a way `set -e` would react to (same gotchas apply) | Centralized error logging (`$LINENO`, `$?`) |
| `INT` | `SIGINT` received (Ctrl+C) | Graceful shutdown / confirmation prompt instead of abrupt stop |
| `TERM` | `SIGTERM` received | Same graceful-shutdown idea, for `kill <pid>` instead of Ctrl+C |

```bash
trap 'cleanup_function' EXIT
trap 'echo "failed at line $LINENO"; exit 1' ERR
trap 'echo "Interrupted, cleaning up..."; exit 130' INT
```

⚠️ **Traps never fire on `SIGKILL` (`kill -9`).** It cannot be caught by any process — cleanup code, `EXIT` traps included, is simply skipped. This is the same rule from Module 10 applied to scripts.

## `die()` Function Pattern

```bash
die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

# Usage:
[ -f "$CONFIG" ] || die "config not found: $CONFIG" 2
command -v curl >/dev/null 2>&1 || die "curl is required" 3
```

## Debugging Technique Reference

| Technique | When to use |
|---|---|
| `echo "DEBUG: x=$x"` | Fastest first move; crude but always available |
| `set -x` / `set +x` (inside script) | Trace a suspicious *section* without touching its logic |
| `bash -x script.sh` | Trace a whole script externally, without editing the file |
| `bash -n script.sh` | Syntax check only — catches unmatched quotes/`fi`/`done`; catches **no** logic bugs |
| `shellcheck script.sh` | Static analysis — catches quoting bugs, legacy syntax, dozens of known pitfalls |

## Syntax Check vs. Static Analysis vs. Running

| Check | Command | Catches | Doesn't catch |
|---|---|---|---|
| Syntax check | `bash -n script.sh` | Malformed grammar (unmatched quotes, missing `fi`) | Logic errors, typos in variable names, bad quoting |
| Static analysis | `shellcheck script.sh` | Unquoted variables, legacy backticks, common anti-patterns | Whether your business logic is actually correct |
| Actually running it | `bash -x script.sh` / normal run | Real runtime behavior, actual output | Nothing — but it's the only way to see truth, so test on non-production data first |

## Common `shellcheck` Warning Codes

| Code | Meaning |
|---|---|
| `SC2086` | Unquoted variable — will break on spaces/globs; the single most common warning |
| `SC2006` | Legacy backtick command substitution; use `$(...)` instead |
| `SC2231` | Unquoted glob/variable expansion inside a `for` loop |
| `SC2164` | `cd` without checking/handling failure (pair with `|| exit`) |
| `SC2181` | Checking `$?` instead of testing the command directly (`if cmd; then` is preferred) |
| `SC2046` | Unquoted command substitution — same word-splitting risk as SC2086 |

## Installing shellcheck (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install shellcheck
shellcheck myscript.sh
```

## 🔁 The Production-Ready Script Workflow

Do this every time before shipping a script:

1. **Add `set -euo pipefail`** right after the shebang.
2. **Add a `trap cleanup EXIT`** immediately after creating any temp file/dir/lock — not at the bottom of the script.
3. **Add a `die()` function** and use it for every precondition check (missing args, missing files, missing dependencies).
4. **Quote every variable expansion** (`"$VAR"`, not `$VAR`).
5. **Run `bash -n script.sh`** — confirm it's at least syntactically valid.
6. **Run `shellcheck script.sh`** — fix every warning, or document why a specific one doesn't apply.
7. **Test the failure paths deliberately** — rename a required file, unset a variable, kill it mid-run — don't only test the happy path.

## 🔁 The "Debug a Misbehaving Script" Workflow

Do this any time a script isn't doing what you expect:

1. **Check syntax first** — `bash -n script.sh` — rule out a basic parse error.
2. **Trace it externally** — `bash -x script.sh 2> trace.log` — capture exactly what ran.
3. **Find the last thing that ran before it went wrong** in the trace log.
4. **Narrow with targeted `echo`s** around that specific spot if the trace alone isn't enough.
5. **Run `shellcheck`** — often the bug is a quoting/word-splitting issue it would have already flagged.
