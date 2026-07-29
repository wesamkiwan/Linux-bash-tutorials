# 📋 Module 9 Cheat Sheet — sed, awk & Regex

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Regex** | A pattern language describing text shapes, not just literal strings |
| **BRE** | Basic Regular Expressions — grep/sed default; `+ ? { } ( ) \|` need `\` to be special |
| **ERE** | Extended Regular Expressions — `grep -E`/`sed -E`; those symbols are special with no escaping |
| **Anchor** | `^` (start of line) / `$` (end of line) — matches a position, not a character |
| **Character class** | `[...]` — matches exactly one character from a set |
| **Quantifier** | `* + ? {n,m}` — controls how many times the previous item repeats |
| **Greedy matching** | Quantifiers match the *longest* possible text by default |
| **Capture group** | `(...)` — a piece of a pattern you can reuse via `\1`, `\2`... in a replacement |
| **Stream editor (sed)** | Processes input line-by-line, applying edit commands, without loading the whole file |
| **Field (awk)** | One whitespace- (or `-F`-) separated chunk of a line; `$1`, `$2`, ... |
| **Record (awk)** | One line of input, by default |

## Regex Metacharacter Reference — BRE vs. ERE

| Meaning | BRE (plain grep/sed) | ERE (`grep -E`, `sed -E`/`-r`) |
|---|---|---|
| Any one character | `.` | `.` |
| Start / end of line | `^` / `$` | `^` / `$` |
| Character class | `[abc]`, `[^abc]`, `[a-z]` | same |
| POSIX class | `[[:digit:]]`, `[[:alpha:]]`, `[[:space:]]` | same |
| Zero or more | `*` | `*` |
| One or more | `\+` | `+` |
| Zero or one | `\?` | `?` |
| Exact/range count | `\{n,m\}` | `{n,m}` |
| Grouping | `\(...\)` | `(...)` |
| Alternation (OR) | `\|` (GNU ext.) | `\|` |
| Literal special char | escape with `\` (e.g. `\.` for a literal dot) | same |

💡 **Rule of thumb:** Need `+`, `?`, `{}`, `()`, or `\|`? Reach for `-E` instead of memorizing BRE escapes.

## grep Regex Flags

| Flag | Meaning |
|---|---|
| `grep -E "pattern"` | Extended regex (ERE) — no escaping needed for `+ ? {} () \|` |
| `egrep "pattern"` | Older alias for `grep -E` |
| `grep -P "pattern"` | Perl-Compatible Regex (PCRE) — `\d \w \s`, lazy quantifiers, lookaround; not always installed |
| `grep -o "pattern"` | Print only the matched text, not the whole line |
| `grep -c` / `-n` / `-i` / `-v` / `-w` / `-r` | Count / line numbers / case-insensitive / invert / whole-word / recursive (from Module 3) |

## sed Command Reference

| Command | Meaning |
|---|---|
| `sed 's/pat/repl/' file` | Substitute **first** match per line; prints to stdout, file untouched |
| `sed 's/pat/repl/g' file` | Substitute **every** match per line (`g` = global) |
| `sed 's/pat/repl/i' file` | Case-**i**nsensitive match (GNU extension) |
| `sed 's/pat/repl/gi' file` | Combine flags: global + case-insensitive |
| `sed -i 's/pat/repl/' file` | Edit **in place** — overwrites the file, no backup, no undo |
| `sed -i.bak 's/pat/repl/' file` | Edit in place, saving original as `file.bak` first — **use this by default** |
| `sed -E 's/pat/repl/' file` | Use ERE inside the pattern |
| `sed '3d' file` | Delete line 3 |
| `sed '2,4d' file` | Delete lines 2 through 4 (address range) |
| `sed '/START/,/END/d' file` | Delete from a line matching `START` through a line matching `END` |
| `sed -n '5,10p' file` | Print only lines 5-10 (`-n` suppresses default auto-print; `p` prints explicitly) |
| `sed -n '/pattern/p' file` | Print only lines matching `pattern` |
| `sed -e 'expr1' -e 'expr2' file` | Apply multiple expressions in one pass |
| `&` in replacement | The whole matched text |
| `\1`, `\2`, ... in replacement | Text captured by the 1st, 2nd, ... `(...)` group |

## awk Built-in Variables

| Variable | Meaning |
|---|---|
| `$0` | The whole current line |
| `$1`, `$2`, ... | 1st, 2nd, ... field of the current line |
| `$NF` | The **last** field (whatever its position) |
| `NF` | Number of fields on the current line |
| `NR` | Number of records (lines) read so far — a running counter |
| `FS` | Input Field Separator (default: any whitespace) |
| `OFS` | Output Field Separator used by `print` when rebuilding fields (default: single space) |
| `FNR` | Record number **within the current file** (matters with multiple input files; resets per file, unlike `NR`) |

## awk Structure

| Syntax | Meaning |
|---|---|
| `awk -F',' '...' file` | Set field separator to comma |
| `pattern { action }` | Run `action` only on lines matching `pattern` (regex, comparison, or blank = every line) |
| `BEGIN { ... }` | Runs once, before any input is read |
| `END { ... }` | Runs once, after all input is processed |
| `{ print $1, $3 }` | Print fields 1 and 3, separated by `OFS` |
| `$3 > 100 { print $1 }` | Comparison used directly as the pattern |
| `total += $3` | Running sum accumulator (auto-initializes to 0) |
| `count++` | Running counter (auto-initializes to 0) |

## Ready-to-Copy One-Liners

```bash
# 1. Extract every email-like string from a file
grep -Eo "[[:alnum:]._%+-]+@[[:alnum:].-]+\.[[:alpha:]]{2,}" file.txt

# 2. Safely rename text in-place with a backup
sed -i.bak 's/old_value/new_value/g' config.conf

# 3. Delete blank lines
sed '/^$/d' file.txt

# 4. Swap two capture groups (rearrange "Last, First" -> "First Last")
echo "Smith, Alice" | sed -E 's/([A-Za-z]+), ([A-Za-z]+)/\2 \1/'

# 5. Print a specific column from a CSV
awk -F',' '{ print $2 }' data.csv

# 6. Sum a numeric column
awk -F',' '{ total += $3 } END { print total }' data.csv

# 7. Filter rows by a numeric condition
awk -F',' '$4 > 100 { print $1, $4 }' data.csv

# 8. Count occurrences of each value in a column, ranked
awk -F',' '{ print $2 }' data.csv | sort | uniq -c | sort -rn

# 9. Print only the last field of every line, whatever its position
awk '{ print $NF }' file.txt

# 10. Strip trailing whitespace from every line
sed -E 's/[[:space:]]+$//' file.txt
```

## 🔁 The Safe `sed -i` Edit Workflow

Do this every time you edit a file in place:

1. **Write the `s///` (or `d`/range) expression and run it WITHOUT `-i` first** — read the printed output carefully; the real file is untouched at this point.
2. **Confirm the output is exactly what you expect** — check the specific lines that changed, not just "it looks fine at a glance."
3. **Re-run with `-i.bak`**, never bare `-i`, so a recoverable original exists (`sed -i.bak 's/.../.../ ' file`).
4. **`diff file.bak file`** to see precisely what changed — this is your proof the edit did exactly what you intended, nothing more.
5. **Delete the `.bak` only once you're fully confident** — on anything that matters, keep it around at least until the change is verified in production.

## 🔁 The grep → sed → awk Decision Workflow

Do this when you're not sure which tool fits the task:

1. **Just need to find/filter lines matching a pattern?** → `grep` (add `-E` for grouping/alternation/quantifiers).
2. **Need to find AND transform/rewrite text on matching lines (including in-place)?** → `sed`.
3. **Need to think in columns — extract, filter by a field's value, sum, count, or do arithmetic?** → `awk`.
4. **Need more than one of the above?** → Pipe them together: `grep` to narrow down → `awk` to extract/aggregate → `sort | uniq -c` to rank.
