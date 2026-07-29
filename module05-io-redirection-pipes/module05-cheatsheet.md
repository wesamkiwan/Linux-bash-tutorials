# 📋 Module 5 Cheat Sheet — I/O Redirection, Pipes & Filters

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Stream** | A flowing channel of data connected to a running program |
| **stdin** (0) | Standard input — data flowing into a program, default: keyboard |
| **stdout** (1) | Standard output — normal results, default: terminal screen |
| **stderr** (2) | Standard error — error/diagnostic messages, default: terminal screen |
| **Redirection** | Pointing a stream somewhere other than its default (file, `/dev/null`, another stream) |
| **Pipe** (`\|`) | Connects one command's stdout directly to the next command's stdin, in memory |
| **Filter** | A command that reads stdin, transforms data, writes stdout (`sort`, `uniq`, `wc`, `grep`) |
| **`/dev/null`** | A special file that discards anything written to it — a bottomless drain |
| **Command substitution** | `$(command)` — runs a command and substitutes its stdout as text |

## Redirection Operators

| Operator | Meaning | Overwrites or Appends? |
|---|---|---|
| `command > file` | Redirect stdout to `file` | **Overwrites** (truncates first) |
| `command >> file` | Redirect stdout to `file` | **Appends** |
| `command < file` | Feed `file` in as stdin | n/a |
| `command 2> file` | Redirect stderr only to `file` | Overwrites |
| `command 2>> file` | Redirect stderr only to `file` | Appends |
| `command &> file` | Redirect **both** stdout and stderr to `file` | Overwrites (Bash-specific shortcut) |
| `command > file 2>&1` | Redirect **both** stdout and stderr to `file` | Overwrites — **portable, correct order** |
| `command > /dev/null` | Discard stdout, keep stderr on screen | — |
| `command > /dev/null 2>&1` | Discard everything | — |

⚠️ **The ordering rule:** the shell processes redirections left to right. `2>&1` means "point stream 2 at wherever stream 1 currently points" — a snapshot, not a live link. Always write the file redirect **first**:

```
✅ command > file 2>&1     (stdout → file, then stderr follows stdout → file)
❌ command 2>&1 > file     (stderr → screen, THEN stdout → file — streams split!)
```

Memorize it as: **"file first, then `2>&1`."**

## File Descriptor Numbers

| Number | Stream | Shorthand you'll see |
|---|---|---|
| `0` | stdin | `0<` (rarely written explicitly) |
| `1` | stdout | `1>` (same as plain `>`) |
| `2` | stderr | `2>` |

💡 `>` alone is shorthand for `1>` — the `1` is implied because stdout is the default stream `>` acts on.

## Pipes

| Syntax | Behavior |
|---|---|
| `cmd1 \| cmd2` | `cmd1`'s stdout becomes `cmd2`'s stdin — no temp file, streamed in memory |
| `cmd1 \| cmd2 \| cmd3` | Chain as many stages as you need — one conveyor belt, many stations |

## Common Pipeline Filter Commands

| Command | Purpose |
|---|---|
| `sort` | Sort lines alphabetically |
| `sort -n` | Sort **numerically** (plain `sort` treats `"10"` as before `"9"`) |
| `sort -r` | Reverse the sort order |
| `sort -rn` | Reverse + numeric together — "biggest number first" |
| `uniq` | Remove **adjacent** duplicate lines only — sort first! |
| `uniq -c` | Same, prefixed with a count of consecutive occurrences |
| `wc -l` | Count lines (from Module 3) — common pipeline endpoint |
| `tee file` | Write to `file` **and** stdout simultaneously (overwrites `file`) |
| `tee -a file` | Same, but appends to `file` |

## `xargs` Patterns

| Pattern | What it does |
|---|---|
| `cmd \| xargs other-cmd` | Turns piped stdin items into arguments for `other-cmd` |
| `find . -name "*.tmp" \| xargs rm` | Deletes every match — ⚠️ unsafe if filenames contain spaces |
| `... \| xargs -I {} cmd {}` | `{}` placeholder — run `cmd` once per item, inserted wherever `{}` appears |
| `find . -name "*.log" -print0 \| xargs -0 rm` | ✅ **Safe pattern** — NUL-separated, immune to spaces/newlines in filenames |
| `xargs -n 1 cmd` | Run `cmd` once per single item instead of batching all items into one call |

✅ **Always pair `find -print0` with `xargs -0`** when the result touches real filenames on a real filesystem.

## Command Substitution

| Syntax | Notes |
|---|---|
| `$(command)` | ✅ Modern, preferred — nests cleanly: `$(cmd1 $(cmd2))` |
| `` `command` `` | ⚠️ Legacy backticks — recognize them, don't write new ones |

## 🔁 The Log Investigation Workflow

Do this every time you need to answer "what's happening the most in this log file?":

1. **Isolate the lines you care about** — `grep "pattern" file.log` (or several chained `grep`s).
2. **Extract just the field you want to rank** — e.g. `awk '{print $1}'` for the first column.
3. **Group identical values together** — `sort` (required before `uniq` can see adjacent duplicates).
4. **Count each group** — `uniq -c`.
5. **Rank by count, highest first** — `sort -rn`.
6. **Trim to the top offenders** — `| head -10` (or `head -n 20`, etc.).
7. **Optionally save the result while still watching it live** — pipe the whole thing into `tee report.txt` instead of running it twice.

```bash
grep "ERROR" app.log | awk '{print $2}' | sort | uniq -c | sort -rn | head -10 | tee top-errors.txt
```

## 🔁 The Safe Bulk-File Workflow (xargs)

Do this every time you're about to run a command against many files found by `find`:

1. **Find the files** with `find . -name "pattern" -print0`.
2. **Rehearse first** — pipe into `xargs -0 -I {} echo "Would act on: {}"` and read the output.
3. **Only then** swap `echo "Would act on: {}"` for the real command (`rm {}`, `mv {} dest/`, etc.).
4. Never drop the `-print0` / `-0` pair for anything touching real-world filenames.
