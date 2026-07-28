# 📋 Module 7 Cheat Sheet — Control Flow

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Control flow** | Deciding which commands run, and how many times, instead of always running top to bottom |
| **Condition** | A command whose exit status (`0`=true, non-zero=false) an `if`/`while`/`until` checks |
| **`test`** | A command that evaluates a condition and exits `0` or `1` |
| **`[ ]`** | Another name for the `test` command |
| **`[[ ]]`** | A Bash keyword (special parsing) that fixes `[ ]`'s quoting/word-splitting issues |
| **Glob pattern** | Wildcard syntax (`*`, `?`, `[...]`) used in `case` patterns and filename matching |

## `[ ]` vs `[[ ]]` vs `test` — Decision Table

| Situation | Use |
|---|---|
| Writing a new Bash script (`#!/bin/bash`) | ✅ `[[ ]]` |
| Script must be POSIX-portable (`#!/bin/sh`/`dash`) | `[ ]` / `test` (no `[[ ]]` allowed) |
| Need `&&`, `\|\|` inside the condition | `[[ ]]` |
| Need glob (`==  *.txt`) or regex (`=~`) matching | `[[ ]]` (only option) |
| Reading someone else's older script | Recognize `[ ]`/`test`, don't need to rewrite it |
| Variable might be empty or contain spaces | `[[ ]]` is safe unquoted; `[ ]` needs careful quoting |

## Comparison Operators

### String

| Operator | Meaning |
|---|---|
| `=` / `==` | Equal (use `==` inside `[[ ]]`) |
| `!=` | Not equal |
| `-z "$s"` | String is empty |
| `-n "$s"` | String is not empty |

### Numeric

| Operator | Meaning |
|---|---|
| `-eq` | Equal |
| `-ne` | Not equal |
| `-lt` | Less than |
| `-le` | Less than or equal |
| `-gt` | Greater than |
| `-ge` | Greater than or equal |

### File Test

| Operator | True when path... |
|---|---|
| `-f` | Exists and is a regular file |
| `-d` | Exists and is a directory |
| `-e` | Exists (any type) |
| `-r` | Exists and is readable |
| `-w` | Exists and is writable |
| `-x` | Exists and is executable |
| `-s` | Exists and is non-empty (size > 0) |

## Logical Operators

| Operator | Meaning | Example |
|---|---|---|
| `&&` | AND | `[[ -f "$f" && -r "$f" ]]` |
| `\|\|` | OR | `[[ -f "$f" \|\| -f "$f.bak" ]]` |
| `!` | NOT | `[[ ! -f "$f" ]]` |

## `if` / `elif` / `else`

```bash
if condition; then
    ...
elif other_condition; then
    ...
else
    ...
fi

# Testing a real command directly — no brackets needed:
if grep -q "ERROR" file.log; then
    echo "found it"
fi
```

## `case` / `esac`

```bash
case "$value" in
    pattern1)
        ...
        ;;
    pattern2|pattern3)   # multiple patterns, either matches
        ...
        ;;
    *.txt)                # glob pattern
        ...
        ;;
    *)                    # catch-all — always include this
        ...
        ;;
esac
```

`;;&` — after a match, keep checking later patterns instead of stopping (rare, advanced fallthrough).

## Loop Syntax Quick Reference

| Loop | Syntax | Use when... |
|---|---|---|
| `for` over a list | `for x in a b c; do ...; done` | You have a fixed set of items |
| `for` over a glob | `for f in *.txt; do ...; done` | You want matching files |
| `for` over command output | `for line in $(command); do ...; done` | You want each word of a command's output |
| C-style `for` | `for ((i=0; i<10; i++)); do ...; done` | You need a numeric counter with start/stop/step |
| `while` | `while condition; do ...; done` | Repeat while condition is true (`0`) |
| `until` | `until condition; do ...; done` | Repeat while condition is false (non-zero) — mirror of `while` |

```bash
# Safe glob loop — guard against zero matches
for f in *.log; do
    [[ -f "$f" ]] || continue
    echo "$f"
done
```

## `break` / `continue`

| Statement | Effect |
|---|---|
| `break` | Exit the innermost loop immediately |
| `continue` | Skip to the next iteration of the innermost loop |
| `break N` | Exit N levels of nested loops |
| `continue N` | Skip to the next iteration, N levels out |

## 🔁 The Safe Conditional Workflow

Do this every time you write an `if` statement:

1. **Decide what you're really testing** — a string, a number, a file, or a real command's success — and pick the matching operator/pattern (`=`/`==` for strings, `-eq` for numbers, `-f`/`-d`/etc. for files, no brackets at all for a plain command).
2. **Default to `[[ ]]`** unless the script must stay POSIX-portable.
3. **Quote your variables anyway** (`"$var"`) even inside `[[ ]]` — it's still good habit and required practice if you ever fall back to `[ ]`.
4. **Always add an `else` (or `case`'s `*)`)** for the "nothing matched" case — don't let unexpected input fall through silently.
5. **Test the boundary cases** — empty input, a numeric string like `"05"`, a missing file — before trusting the condition in production.

## 🔁 The Safe Loop Workflow

Do this every time you write a loop:

1. **Pick the right loop shape** — list/glob/command-output `for`, numeric C-style `for`, or `while`/`until` — using the table above.
2. **Guard globs** with a `[[ -f "$item" ]]` check in case nothing matched.
3. **Make sure the loop's condition variable actually changes inside the body** (increment counters, update retry state) — this is the #1 cause of infinite loops.
4. **Use `break`/`continue` deliberately**, with a level number the moment loops are nested and you need to affect an outer loop.
5. **Cap retries** with a `max_attempts`-style variable — never loop "until success" with no upper bound in production code.
