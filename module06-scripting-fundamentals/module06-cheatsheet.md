# 📋 Module 6 Cheat Sheet — Bash Scripting Fundamentals

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Script** | A plain text file of shell commands, run as a single unit |
| **Shebang** | The `#!/bin/bash` first line telling the kernel which interpreter to use |
| **Shell variable** | A named value visible only in the current shell/script |
| **Environment variable** | A shell variable marked `export`-ed, inherited by child processes |
| **Command substitution** | `$(command)` — captures a command's output as a string |
| **Arithmetic expansion** | `$((expression))` — integer math, no external process |
| **Positional parameter** | `$1`, `$2`, ... — the arguments a script was called with |
| **Exit status / exit code** | Number 0-255 a command/script returns on finishing; `0` = success |
| **Word splitting** | Bash splitting an unquoted value into multiple arguments on whitespace |
| **Globbing** | Bash expanding an unquoted `*`/`?`/`[...]` against real filenames |

## Shebang & Execution

| Line/Command | Meaning |
|---|---|
| `#!/bin/bash` | Must be the literal first line — tells the kernel to run this file with `/bin/bash` |
| `#!/usr/bin/env bash` | Same idea, but finds `bash` via `PATH` — more portable |
| `chmod +x script.sh` | Grants execute permission, required for `./script.sh` |
| `./script.sh` | Kernel honors the shebang; requires execute permission |
| `bash script.sh` | Runs via Bash explicitly, ignoring the shebang; no execute permission needed |
| `sh script.sh` | Runs via whatever `sh` is (on Ubuntu: `dash`, **not** Bash) — Bash-only syntax may break |

## Variables

| Syntax | Meaning |
|---|---|
| `name="value"` | Assign — **no spaces** around `=` |
| `$name` / `${name}` | Read the value — `${}` avoids ambiguity next to other text |
| `"$name"` | **Always quote** — prevents word splitting & globbing |
| `readonly name="value"` | Locks the variable against reassignment |
| `unset name` | Removes the variable entirely |
| `export name="value"` | Marks it as an environment variable — inherited by child processes |
| `env` / `printenv` | List / print environment variables |

### Quoting Rules — Quick Decision Table

| Situation | Do this |
|---|---|
| Value might contain spaces | Always `"$var"` |
| Value might be empty | Always `"$var"` |
| Value might contain `*`, `?`, `[` | Always `"$var"` |
| Passing along all script arguments | Always `"$@"`, never `$@`, `$*`, `"$*"` |
| You deliberately want one joined string | `"$*"` (rare) |

## Command Substitution & Arithmetic

| Syntax | Purpose |
|---|---|
| `$(command)` | Modern command substitution — capture output, nestable |
| `` `command` `` | Legacy backtick form — avoid in new scripts |
| `$((expr))` | Modern integer arithmetic — preferred |
| `let var=expr` | Older arithmetic built-in — still works, less clear |
| `expr $var + 1` | **Legacy** external program — avoid in new scripts |
| `$((a + b))`, `$((a - b))`, `$((a * b))`, `$((a / b))`, `$((a % b))`, `$((a ** b))` | add, subtract, multiply, integer-divide, remainder, power |

## Positional Parameters

| Variable | Meaning |
|---|---|
| `$0` | Script's own invocation name |
| `$1`, `$2`, ... | 1st, 2nd, ... argument |
| `$#` | Count of arguments |
| `$@` | All arguments |
| `$*` | All arguments |
| `shift` | Drops `$1`, renumbers everything down by one |

### `"$@"` vs `"$*"` — the difference that matters

| Form | Quoted behavior |
|---|---|
| `"$@"` | Expands to **each argument as its own separate word**, exactly as passed — use this |
| `"$*"` | Expands to **one single string**, all arguments joined by a space |

```bash
for a in "$@"; do echo "[$a]"; done   # one iteration per original argument — correct
for a in "$*"; do echo "[$a]"; done   # one iteration total, over the whole joined blob — usually wrong
```

## Other Special Variables

| Variable | Holds |
|---|---|
| `$?` | Exit status of the **last** command (check immediately — it's overwritten fast) |
| `$$` | PID of the current shell/script |
| `$!` | PID of the most recent background job |

## Reading Input

| Command | Behavior |
|---|---|
| `read var` | Waits for input, stores it in `var` |
| `read -p "Prompt: " var` | Shows a prompt, then waits, same line |
| `read -s -p "Prompt: " var` | Silent — input not echoed (passwords/secrets) |

## Exit Codes

| Code | Convention |
|---|---|
| `0` | Success |
| `1`-`255` | Some kind of failure (meaning is yours to define) |
| `exit 0` / `exit 1` | Explicitly set the script's own exit code |
| `$?` | Check the exit code of the last finished command |

```bash
some-command
if [ "$?" -eq 0 ]; then echo "ok"; else echo "failed"; fi
# or, more idiomatic:
if some-command; then echo "ok"; else echo "failed"; fi
```

## 🔁 The New Script Bootstrap Workflow

Do this every time you start a new script:

1. **Create the file** with a `.sh` name, e.g. `deploy.sh`.
2. **Add the shebang as line one** — `#!/bin/bash` (nothing above it, not even a blank line).
3. **Add a short description comment** — what the script does and how to use it (`# Usage: ./deploy.sh <env>`).
4. **`chmod +x deploy.sh`** so it can be run with `./deploy.sh`.
5. **Validate arguments early** — check `$#` before touching `$1`, `$2`, etc.
6. **Quote every variable expansion** — `"$var"`, `"$@"`, `"${name}"` — by default, no exceptions without a reason.
7. **Give it a real exit code** at every ending point — `exit 0` for success, non-zero for each distinct failure case.
8. **Test it three ways**: with valid input, with no arguments, and with input containing spaces — that last one catches the quoting bugs.

## 🔁 The Safe Argument-Handling Workflow

Do this any time a script accepts arguments:

1. Check `$#` first — bail out with a usage message (to `>&2`) and a non-zero exit code if the count is wrong.
2. Copy positional parameters into clearly-named variables (`environment="$1"`) instead of scattering `$1`/`$2` throughout the script.
3. Quote every reference to that variable from then on.
4. If you need to forward all arguments to another command, use `"$@"` — never anything else.
