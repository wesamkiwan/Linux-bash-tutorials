# 📋 Module 3 Cheat Sheet — Viewing & Finding Files

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Pager** | A program (like `less`) that displays a file one screen at a time |
| **Pattern** | The text you're searching for |
| **Regular expression (regex)** | A mini-language for describing text patterns — full depth in Module 9 |
| **Metadata** | Data *about* a file (name, size, type, timestamps) rather than its content |
| **Pipe (`\|`)** | Feeds one command's output directly into the next command's input |
| **Standard output (stdout)** | The normal text-output channel a program writes to |
| **Append-only** | New content is added to the *end* of a file, never modifying earlier lines — how logs typically grow |
| **Index (for `locate`)** | A pre-built database of file paths, refreshed periodically by `updatedb` |

## Viewing Files

| Command | What it shows |
|---|---|
| `cat file` | Whole file, dumped at once |
| `cat -n file` | Whole file with line numbers |
| `less file` | Interactive scrollable pager (modern default) |
| `more file` | Older forward-only pager — prefer `less` |
| `head file` | First 10 lines |
| `head -n N file` | First `N` lines |
| `tail file` | Last 10 lines |
| `tail -n N file` | Last `N` lines |
| `tail -f file` | Live-follow — keep printing new lines as they're appended (`Ctrl+C` to stop) |

### `less` Navigation Keys

| Key | Action |
|---|---|
| `Space` | Page forward |
| `b` | Page backward |
| `/pattern` | Search forward |
| `?pattern` | Search backward |
| `n` | Next match |
| `N` | Previous match |
| `g` / `G` | Jump to top / bottom |
| `q` | Quit |

## Counting: `wc`

| Command | Counts |
|---|---|
| `wc file` | Lines, words, characters (all three) |
| `wc -l file` | Lines only |
| `wc -w file` | Words only |
| `wc -c file` | Characters (bytes) only |

## `grep` Flags Reference

| Flag | Meaning | Example |
|---|---|---|
| (none) | Print matching lines | `grep "error" file` |
| `-i` | Case-insensitive | `grep -i "error" file` |
| `-v` | Invert match — show non-matching lines | `grep -v "INFO" file` |
| `-n` | Show line numbers | `grep -n "error" file` |
| `-c` | Show a count of matches, not the lines | `grep -c "error" file` |
| `-l` | Show only filenames with a match | `grep -l "error" *.log` |
| `-w` | Match whole words only | `grep -w "cat" file` |
| `-r` / `-R` | Recurse into a directory (`-R` also follows symlinks) | `grep -r "TODO" src/` |
| `-o` | Print only the matched text, not the whole line | `grep -o "order #[0-9]*" file` |

⚠️ Keep patterns **quoted**: `grep "out of memory" file.txt`, not `grep out of memory file.txt`.
💡 Full regex power (`.`, `*`, `[...]`, anchors, groups) is Module 9 — this module = literal/simple patterns only.

## `sort` and `uniq` (preview — full power in Module 9)

| Command | What it does |
|---|---|
| `sort file` | Sort lines alphabetically |
| `sort -n file` | Sort lines numerically |
| `uniq file` | Remove **consecutive** duplicate lines |
| `sort file \| uniq` | Sort first, then dedupe — the standard combo |

## `find` Predicate Reference

| Predicate | Matches | Example |
|---|---|---|
| `-name "pattern"` | Filename, case-**sensitive** | `find . -name "*.txt"` |
| `-iname "pattern"` | Filename, case-**insensitive** | `find . -iname "*.TXT"` |
| `-type f` | Regular files only | `find . -type f` |
| `-type d` | Directories only | `find . -type d` |
| `-size +10M` | Larger than 10 MB | `find . -size +10M` |
| `-size -1k` | Smaller than 1 KB | `find . -size -1k` |
| `-mtime -7` | Modified less than 7 days ago | `find . -mtime -7` |
| `-mtime +7` | Modified more than 7 days ago | `find . -mtime +7` |
| `-mmin -30` | Modified less than 30 minutes ago | `find . -mmin -30` |
| `-exec cmd {} \;` | Run `cmd` on each match | `find . -name "*.tmp" -exec rm {} \;` |

⚠️ Quote `-name`/`-iname` patterns or the shell may expand the wildcard itself before `find` runs.
⚠️ Always dry-run `find` *without* `-exec` first, confirm the match list, then add `-exec`.

## Finding Files Fast: `locate` vs `find`

| Tool | Speed | Freshness | Notes |
|---|---|---|---|
| `find` | Slower (scans live) | Always accurate, right now | No setup needed |
| `locate` | Very fast (searches an index) | Only as fresh as the last `updatedb` run (often once/day) | Install via `apt install mlocate` / `plocate` if missing |
| `sudo updatedb` | — | Forces an immediate index refresh | Run this if a new file "should" be findable but isn't |

## Finding Executables

| Command | What it shows |
|---|---|
| `which cmd` | Full path of the executable that would actually run |
| `whereis cmd` | Executable path + man page + related file locations |

## 🔁 The "Where Did That Error Come From" Workflow

Do this every time you're handed a "something's broken, go look" ticket:

1. **Find the right log file first.** `find /var/log -iname "*app*"` or `locate app.log` if you already know its rough name.
2. **Watch it live while reproducing the issue.** `tail -f /var/log/myapp/app.log` in one terminal, trigger the action in another.
3. **Once you've seen an error, search history for it.** `grep -in "error" /var/log/myapp/app.log`
4. **Count how often it's happened.** `grep -ci "error" /var/log/myapp/app.log`
5. **Pull out a specific detail (like an order or request ID) and dedupe it.** `grep -i "error" file.log | grep -o "order #[0-9]*" | sort | uniq`
6. **Narrow to a time window if the log is huge.** `find /var/log -name "*.log" -mmin -60` to see what's changed in the last hour.
7. **Report back with exact counts and line numbers**, not vague impressions — `grep -n` gives you the receipts.

## 🔁 The Quick File Triage Workflow

Do this the moment you open an unfamiliar file:

1. `wc -l file` — how big is it, roughly?
2. `head -n 5 file` — what does the format/header look like?
3. `tail -n 5 file` — what's the most recent/last content?
4. If it's still unclear and the file is large: `less file` to browse properly.
