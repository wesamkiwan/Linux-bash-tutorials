# Module 9: Text Processing Power Tools — sed, awk & Regex 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 3.5 hours
**Prerequisites:** Modules 1-8 (Shell Fundamentals through Functions, Arrays & String Manipulation)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain the difference between Basic Regular Expressions (BRE) and Extended Regular Expressions (ERE), and know which tools/flags use which by default
- [ ] Build regex patterns using anchors (`^`, `$`), character classes (`[abc]`, `[^abc]`, `[a-z]`, POSIX classes like `[[:digit:]]`), the `.` wildcard, and quantifiers (`*`, `+`, `?`, `{n,m}`)
- [ ] Use `grep -E` (extended regex) and know when `grep -P` (Perl-compatible regex) exists as a more powerful option
- [ ] Use `sed` for stream editing: substitution (`s/pattern/replacement/`), the `g` and `i` flags, in-place editing with `-i` (safely, with a backup), deleting lines, and printing specific line ranges
- [ ] Use sed address ranges (`2,4d`, `/start/,/end/d`) and back-references (`&`, `\1`, `\2`) in replacements
- [ ] Explain awk's field-based mental model (`$0`, `$1`...`$NF`, `NR`, `NF`, `FS`, `OFS`) and set a custom field separator with `-F`
- [ ] Write awk `BEGIN`/`END` blocks and pattern-action pairs to filter, transform, and summarize column data
- [ ] Combine `grep`, `sed`, and `awk` into a single realistic pipeline to parse logs or CSV data
- [ ] Recognize the common pitfalls that trip up even experienced engineers when using these tools under time pressure

---

## Module Goal

By the end of this module, you'll be able to sit down at a production server that has no Python, no pandas, no fancy log-analysis dashboard — just a plain Ubuntu shell — and still confidently answer questions like "how many 500 errors did we serve in the last hour?" or "extract every email address out of this 4GB file" or "fix this typo across 200 config files without opening a single one by hand."

🎯 **On the job:** Picture this real scenario: it's 2 a.m., production is throwing errors, and you SSH into a server that has nothing installed beyond the base OS. The access log is 6GB. You can't `scp` that down to your laptop and open it in a spreadsheet — it wouldn't even load. What you *do* have, guaranteed, on every Linux box on Earth, is `grep`, `sed`, and `awk`. These three tools, chained together with pipes, can slice, filter, transform, and summarize gigabytes of text data in seconds, using nowhere near the memory a "real" program would need, because they process data one line at a time as a **stream** instead of loading the whole file into memory. This is the single most valuable skill set in this entire course for anyone who will touch a production Linux server for a living — SRE, DevOps, backend engineer, support engineer, all of it.

This module is also the course's first 🔴 **Advanced** module. That's not decoration — regex, sed, and awk each have their own small "language" to learn, and combining all three takes real practice. Go slowly through the Core Concepts section below; everything after it builds directly on getting the fundamentals of regex right first.

---

## Core Concepts

### 1. Why this module exists, and the order we'll tackle it in

You met `grep` in Module 3, but only with **literal** patterns — plain words, no special meaning. That was a deliberate simplification. `grep`'s real power (and sed's, and part of awk's) comes from **regular expressions**, a pattern language far richer than "match this exact text." So we'll do this in three stages, each building on the last:

1. **Regex fundamentals** — the pattern language itself, tool-agnostic.
2. **sed** — the "stream editor," which uses regex to find-and-transform text, line by line.
3. **awk** — a full mini programming language built around thinking of each line as a row of **columns** (fields), which uses regex for pattern-matching and adds real logic (variables, arithmetic, conditionals) on top.

💡 **Analogy — the stencil, the correction tape, and the spreadsheet:** Think of **regex** as a **stencil** — a shape you hold up against text to decide "does this count as a match, and if so, exactly which part?" Think of **sed** as a roll of **correction tape with your stencil built in** — it runs down every line of a document, and wherever the stencil matches, it can delete that spot or write something new over it, then moves on — it never "opens" the whole document in an editor, it streams through once. Think of **awk** as a **spreadsheet program that only understands columns** — it treats every line as a row split into cells (`$1`, `$2`, `$3`...), and you write tiny formulas ("if column 3 is greater than 100, print column 1 and column 5") instead of clicking cells with a mouse.

### 2. What is a "regular expression"?

A **regular expression** (shortened to **regex**, plural regexes or regexen) is a sequence of characters that describes a *pattern* of text, rather than one specific literal string. Instead of asking "does this line contain the exact text `error123`?", a regex can ask "does this line contain the word `error` followed by any number of digits?" — matching `error1`, `error42`, and `error999` all with one pattern.

Regex is not specific to Linux or to any one tool — it's used across `grep`, `sed`, `awk`, most programming languages, and even the "Find" box in many text editors. Learn it once here, and you'll reuse it for the rest of your career.

⚠️ **Important terminology check:** A regex is not the same thing as a shell **glob** (`*.txt`, `file?.log`) that you used with `find` and wildcard expansion in earlier modules. Globs match *filenames*; regex matches *text content*, and the two languages use similar-looking symbols (`*`, `?`) to mean **completely different things**. This mix-up is one of the most common sources of confusion for people learning this module — keep them mentally separate.

### 3. BRE vs. ERE — the split that causes 90% of regex confusion on Linux

This is the single most important thing to get straight before writing a single pattern. GNU tools that use regex support **two different flavors** of the language:

- **BRE — Basic Regular Expressions.** The older, more limited flavor. Plain `grep` and plain `sed` use BRE **by default**. In BRE, the characters `+`, `?`, `{`, `}`, `(`, `)`, and `|` are treated as **literal characters** unless you escape them with a backslash (`\+`, `\?`, `\{`, `\}`, `\(`, `\)`, `\|`) — backslash *adds* special meaning in BRE, the opposite of what you might expect.
- **ERE — Extended Regular Expressions.** The newer, more powerful flavor. `+`, `?`, `{n,m}`, `(...)` (grouping), and `|` (alternation) all work **directly**, with no backslash needed. You get ERE with `grep -E` (or the older alias `egrep`), and with `sed -E` (GNU sed also accepts `sed -r` as an equivalent, non-standard flag).

| Feature | BRE (default `grep`/`sed`) | ERE (`grep -E`, `sed -E`/`sed -r`) |
|---|---|---|
| Literal characters, anchors `^ $`, `.`, `*`, `[ ]` | Same in both | Same in both |
| One-or-more (`+`) | `\+` | `+` |
| Zero-or-one (`?`) | `\?` | `?` |
| Repetition count `{n,m}` | `\{n,m\}` | `{n,m}` |
| Grouping `(...)` | `\(...\)` | `(...)` |
| Alternation (OR) `\|` | `\|` (GNU extension) | `\|` |

✅ **Best Practice:** Unless you have a specific reason not to (like matching an old script's exact BRE syntax), reach for the extended flavor — `grep -E` and `sed -E` — every time you need grouping, alternation, `+`, `?`, or `{n,m}`. It reads far closer to the regex you'll see in every other programming language, and you stop needing to remember which characters need escaping.

🎯 **On the job:** This is a real, recurring source of "why isn't my regex working?!" bugs. Someone writes `grep "error|warning" file.log` expecting an OR match, and it silently does something else entirely — because plain `grep` treats `|` as a literal pipe character in BRE, not as alternation. The fix is `grep -E "error|warning" file.log`.

### 4. Anchors — pinning a match to the start or end of a line

An **anchor** doesn't match a character at all — it matches a *position* in the line.

- `^` matches the **start** of a line.
- `$` matches the **end** of a line.

```
^Error    matches lines that START with "Error"
done$     matches lines that END with "done"
^$        matches a completely EMPTY line (start immediately followed by end)
```

⚠️ **Context matters:** Inside a character class (`[^abc]`, covered next), `^` means something totally different — "NOT one of these characters" — not "start of line." Same symbol, different meaning, depending on where it appears. This trips people up constantly.

### 5. Character classes — matching "one of these characters"

A **character class** is written in square brackets, `[...]`, and matches **exactly one character** — any single character that's a member of the set you list inside the brackets.

| Pattern | Matches |
|---|---|
| `[abc]` | One character: `a`, `b`, or `c` |
| `[a-z]` | One lowercase letter, `a` through `z` (a **range**) |
| `[A-Za-z]` | One letter, upper or lower case |
| `[0-9]` | One digit |
| `[^abc]` | One character that is **NOT** `a`, `b`, or `c` (the `^` here means "negate," not "start of line") |
| `[^0-9]` | One character that is **NOT** a digit |

GNU tools also support **POSIX character classes**, written with double brackets, which are more portable and more readable than spelling out ranges yourself:

| POSIX class | Equivalent to | Meaning |
|---|---|---|
| `[[:digit:]]` | `[0-9]` | A digit |
| `[[:alpha:]]` | `[A-Za-z]` | A letter |
| `[[:alnum:]]` | `[A-Za-z0-9]` | A letter or digit |
| `[[:space:]]` | (space, tab, newline, etc.) | Any whitespace character |
| `[[:upper:]]` | `[A-Z]` | An uppercase letter |
| `[[:lower:]]` | `[a-z]` | A lowercase letter |
| `[[:punct:]]` | (punctuation) | Any punctuation character |

💡 **Tip:** The outer `[ ]` in `[[:digit:]]` is the character-class bracket you already know; the `[:digit:]` inside it is the POSIX class name. You almost always use POSIX classes *inside* one more layer of brackets, like `[[:digit:]]` or even combined, like `[[:digit:].]` (a digit OR a literal dot).

### 6. The `.` wildcard — matching "any one character"

A bare `.` (dot) matches **any single character** — letter, digit, symbol, even a space — except (in most contexts) a newline. It does **not** mean "any number of characters" by itself; that's a common beginner misread.

```
c.t    matches "cat", "cot", "cut", "c t", "c9t" — anything with exactly one character between c and t
```

⚠️ **Warning:** If you actually want to match a **literal dot** (like in an IP address or a filename extension), you must escape it: `\.`. An unescaped `.` in `192\.168\.1\.1` vs `192.168.1.1` — the second version would also match `192X168Y1Z1`, since every unescaped `.` matches any character, not just a literal period.

### 7. Quantifiers — how many times should the previous thing repeat?

A **quantifier** attaches to whatever comes immediately before it (a single character, a character class, or a group) and controls *how many times* that thing must repeat for a match.

| Quantifier | Meaning | BRE | ERE |
|---|---|---|---|
| `*` | Zero or more | `*` | `*` |
| `+` | One or more | `\+` | `+` |
| `?` | Zero or one (optional) | `\?` | `?` |
| `{n}` | Exactly `n` times | `\{n\}` | `{n}` |
| `{n,}` | `n` or more times | `\{n,\}` | `{n,}` |
| `{n,m}` | Between `n` and `m` times | `\{n,m\}` | `{n,m}` |

```
[0-9]*      zero or more digits (matches even an empty string!)
[0-9]+      one or more digits (ERE) — needs at least one digit present
[0-9]\+     the same thing, in BRE
colou?r     matches "color" AND "colour" (the "u" is optional)
[0-9]{3}    exactly three digits
[0-9]{2,4}  between two and four digits
```

✅ **Best Practice:** If you catch yourself reaching for `+`, `?`, or `{n,m}`, that's your signal to add `-E` (grep/sed) rather than hunt down backslash-escaping rules for BRE.

### 8. Grouping and alternation — `(...)` and `|`

**Grouping**, `(...)`, treats everything inside the parentheses as a single unit — so a quantifier can apply to a whole sequence, not just one character, and so you can capture that piece of text for reuse (more on that in the sed section).

**Alternation**, `|`, means "OR" — match whatever is on the left, or whatever is on the right.

```
(ha)+          matches "ha", "haha", "hahaha", ... (one or more repeats of "ha")
cat|dog        matches "cat" OR "dog"
(error|warn)ing   matches "erroring" or "warning" — grouping combined with alternation
```

Both `(...)` for grouping and `|` for alternation need `-E` in grep, or escaping (`\(`, `\)`, `\|`) in plain BRE grep/sed.

### 9. Greedy matching — regex takes as much as it can

By default, quantifiers in POSIX regex (BRE and ERE alike) are **greedy** — they match the *longest* possible piece of text that still lets the overall pattern succeed, not the shortest.

```
Input:   <b>bold</b> and <i>italic</i>
Pattern: <.*>
```

You might expect this to match just `<b>`, stopping at the first `>`. It doesn't. `.*` greedily consumes as much as possible, so the match actually spans all the way from the very first `<` to the very **last** `>` in the line: `<b>bold</b> and <i>italic</i>` — the entire thing. This surprises almost everyone the first time they hit it.

⚠️ **Warning:** POSIX grep/sed/awk regex has no built-in "lazy" (non-greedy) quantifier the way some other languages do (Perl-compatible regex, `grep -P`, does support that, using `.*?`). In plain POSIX regex, the fix is to be more specific about what you *don't* want, rather than relying on laziness — e.g., `<[^>]*>` (any characters that are **not** a `>`) instead of `<.*>`, which correctly stops at the very next `>`.

### 10. `grep -E` and `grep -P` — leveling up grep's regex power

You already know plain `grep` from Module 3. Two extra flags unlock more powerful pattern matching:

- **`grep -E "pattern" file`** switches on ERE — grouping, alternation, `+`, `?`, `{n,m}` all work without escaping. `egrep` is an older, equivalent alias for the same thing, though `grep -E` is the more modern, portable form to write in new scripts.
- **`grep -P "pattern" file`** switches on **PCRE** — Perl-Compatible Regular Expressions — an even richer dialect that adds features like lazy quantifiers (`.*?`), lookahead/lookbehind assertions, and `\d`/`\w`/`\s` shorthand classes. It's genuinely more powerful, but ⚠️ it isn't always installed by default on every minimal system (it depends on grep being built with PCRE support) — check with `grep -P "test" <<< "test"` before relying on it in a script meant to run on unknown machines. On a standard Ubuntu/Debian install, GNU grep ships with `-P` support out of the box.

💡 **Tip:** For this module's scope, ERE (`-E`) covers the vast majority of real, on-the-job regex needs. Reach for `-P` when you specifically need something ERE can't do, like `\d` shorthand or a lookahead.

### 11. Practical regex building blocks you'll actually reuse

A few patterns come up constantly enough to memorize:

| Goal | Pattern (ERE) |
|---|---|
| A digit | `[0-9]` or `[[:digit:]]` |
| One or more digits (an integer) | `[0-9]+` |
| A decimal number | `[0-9]+\.[0-9]+` |
| An IP-address-*like* pattern (not fully strict) | `[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}` |
| An email-*like* pattern (simplified, not RFC-perfect) | `[[:alnum:]._%+-]+@[[:alnum:].-]+\.[[:alpha:]]{2,}` |
| A word at the start of a line | `^[[:alpha:]]+` |
| Trailing whitespace on a line | `[[:space:]]+$` |

⚠️ **Warning:** Both the IP-like and email-like patterns above are deliberately simplified for everyday log-scraping and cleanup tasks — they're not fully spec-compliant validators. `999.999.999.999` would match the "IP-like" pattern even though it's not a valid IP. For real validation (as opposed to "find things that look roughly like an IP in a log file"), you'd want a stricter pattern or a proper library — that's outside this module's scope, which is about fast, practical text processing, not formal validation.

### 12. sed — the stream editor

**sed** stands for **s**tream **ed**itor. It reads input **one line at a time**, applies one or more editing commands to each line based on patterns you give it, and prints the (possibly transformed) result — all without ever loading the whole file into memory or opening an interactive editor. That's why it can process a multi-gigabyte log file that would choke a text editor.

The most common sed command by far is **substitution**:

```bash
sed 's/pattern/replacement/' file.txt
```

Read this as: "**s**ubstitute `pattern` with `replacement`." By default, sed only replaces the **first** match on each line, and it prints its result to standard output — it does **not** modify the file unless you explicitly ask it to (with `-i`, covered below).

Key sed flags and behaviors you'll use constantly:

| Feature | Syntax | Meaning |
|---|---|---|
| Global flag | `s/pattern/replacement/g` | Replace **every** match on the line, not just the first |
| Case-insensitive flag | `s/pattern/replacement/i` | Match regardless of case (a GNU extension, not in strict POSIX sed) |
| In-place edit | `sed -i 's/.../.../ ' file` | Writes changes **directly back into the file** |
| In-place edit with backup | `sed -i.bak 's/.../.../ ' file` | Same, but first saves the original as `file.bak` |
| Delete a line | `sed '3d' file` | Deletes line 3 |
| Delete a range | `sed '2,4d' file` | Deletes lines 2 through 4 |
| Delete by pattern range | `sed '/START/,/END/d' file` | Deletes everything from a line matching `START` through a line matching `END` |
| Print only matching lines | `sed -n '/pattern/p' file` | `-n` suppresses default output; `p` explicitly prints only lines that matched |
| Print a line range | `sed -n '5,10p' file` | Prints only lines 5 through 10 |
| Multiple edits at once | `sed -e 's/a/b/' -e 's/c/d/' file` | Applies more than one expression in one pass |

Two special replacement symbols matter a lot:

- **`&`** in the replacement means "whatever the whole pattern matched" — you can wrap the original match in something else without retyping it.
- **`\1`, `\2`, ...** refer to **capture groups** — pieces of the pattern you wrapped in parentheses (escaped `\(...\)` in BRE, or plain `(...)` in ERE with `sed -E`). This lets you **rearrange** parts of the matched text in the replacement, not just wrap it.

```bash
echo "2026-07-28" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# output: 28/07/2026 — day/month/year rearranged from year-month-day
```

⚠️ **Critical safety note:** `sed -i` overwrites the file **immediately** and **without confirmation** — there's no "are you sure?" prompt, and no built-in undo. Always test your `s///` expression **without** `-i` first (just let it print to your terminal), confirm the output looks exactly right, and only *then* add `-i` — ideally `-i.bak` so a backup exists. This is covered in detail, with a full worked example, in the Practical Examples section below.

### 13. awk — thinking in columns

**awk** (named after its three original authors — Aho, Weinberger, and Kernighan) is a full pattern-scanning and text-processing *language*, not just a single command. Where sed thinks in terms of "lines and substitutions," awk thinks in terms of **records and fields** — by default, each **line** is a record, and each **whitespace-separated chunk** of that line is a **field**.

💡 **Analogy:** If sed is correction tape, awk is a **spreadsheet program that reads plain text**. Every line becomes a row; every field becomes a cell in that row, automatically split for you. You then write small rules like "if the value in the third cell is bigger than 100, print the first cell" — that's an entire awk program.

Awk's core built-in variables:

| Variable | Meaning |
|---|---|
| `$0` | The **whole current line**, unchanged |
| `$1`, `$2`, `$3`, ... | The 1st, 2nd, 3rd, ... **field** (column) of the current line |
| `NF` | **N**umber of **F**ields in the current line (so `$NF` is always the *last* field, whatever its position) |
| `NR` | **N**umber of **R**ecords (lines) processed so far — effectively a running line counter |
| `FS` | **F**ield **S**eparator — what awk splits fields on (default: any run of whitespace) |
| `OFS` | **O**utput **F**ield **S**eparator — what awk uses to glue fields back together when you rebuild `$0` (default: a single space) |

By default, awk splits fields on **any amount of whitespace** (one or more spaces/tabs), and automatically ignores leading/trailing whitespace when doing so. You can force a different separator with `-F`:

```bash
awk -F',' '{ print $1 }' data.csv    # split on commas instead — reading a CSV file
awk -F':' '{ print $1 }' /etc/passwd # split on colons — reading /etc/passwd's format
```

An awk program is built from **pattern-action pairs**:

```awk
pattern { action }
```

- **pattern** decides *which* lines this rule applies to — it can be empty (meaning "every line"), a regex like `/error/`, or a comparison like `$3 > 100`.
- **action** is what to do to matching lines, wrapped in `{ }` — most often `print`, but it can include variables, arithmetic, and conditionals.

Two special patterns bookend the whole run:

- **`BEGIN { ... }`** runs **once**, before any input is read — used to set things up (like `FS`, or initializing a counter to zero).
- **`END { ... }`** runs **once**, after all input has been processed — used to print a final summary (like a total or an average).

```bash
awk 'BEGIN { print "Starting..." } { print $1 } END { print "Done. Total lines:", NR }' file.txt
```

Awk supports real arithmetic and comparisons directly:

```bash
awk '{ total += $3 } END { print "Sum:", total }' data.txt
awk '$3 > 100 { print $1, $3 }' data.txt      # a comparison used AS the pattern
awk '{ if ($2 == "ERROR") count++ } END { print count }' data.txt   # a conditional inside an action
```

🎯 **On the job:** The single most common real-world awk one-liner is "sum a column" or "print specific columns from a CSV/log," which is exactly what's shown in the Practical Examples below with real sample data.

### 14. Combining grep, sed, and awk into pipelines

None of these tools live in isolation — the real power move is **piping** them together, each doing the one thing it's best at:

- `grep` **filters** — keep only the lines you care about.
- `sed` **transforms** — rewrite/clean/extract text within lines.
- `awk` **structures and aggregates** — split into columns, count, sum, summarize.

```bash
grep "ERROR" app.log | awk -F'|' '{ print $2 }' | sort | uniq -c | sort -rn
```

Read left to right: filter down to error lines, split each on `|` and grab the second field, sort so identical values are adjacent, collapse and count duplicates, then sort those counts from highest to lowest. That single line is a complete, real "top error codes" report — you'll build exactly this in the Practical Examples section.

---

## Detailed Explanations

### Why does BRE even still exist if ERE is strictly more convenient?

BRE is the original POSIX regex flavor, dating back decades, and countless existing scripts, cron jobs, and tools still rely on plain `grep`/`sed` defaulting to it — changing that default would break an enormous amount of legacy software across every Linux distribution. GNU tools solve this by keeping BRE as the default for backward compatibility, while offering `-E`/`-r` as an explicit opt-in to ERE. You inherit both: recognize BRE when you read someone else's older script, but default to ERE (`-E`) for anything new you write, since it's clearer and matches what most other languages call "regex."

### Why sed doesn't modify files by default

sed's original design goal was to be a filter in a pipeline — read from standard input (or a file), write to standard output, exactly like `grep` or `cat`. That's why plain `sed 's/a/b/' file.txt` prints the transformed result to your terminal and leaves `file.txt` completely untouched. `-i` is a deliberate, explicit opt-in specifically for the (common, but riskier) case where you want to edit the file itself rather than just view the transformed output. This design means you can always safely experiment with a sed expression against a real file, see the result, and only commit to changing the file once you're sure — as long as you remember not to add `-i` prematurely.

### Why awk's default field splitting is "any whitespace," and when that's the wrong choice

Awk's default `FS` behaves specially: any run of one or more spaces or tabs counts as a single separator, and leading whitespace is ignored entirely. This is usually exactly what you want for the ragged, inconsistently-spaced output of commands like `ps`, `ls -l`, or `df -h`, where columns don't line up with a single, fixed-width space. But it becomes the *wrong* choice the moment your data uses a **specific delimiter** that isn't plain whitespace — a comma-separated CSV, or a colon-separated file like `/etc/passwd`. In those cases, `-F','` or `-F':'` tells awk to split on that exact character instead, which is essential — using the default whitespace-splitting on a CSV where fields might legitimately be empty (`a,,c`) or contain internal spaces (`"New York",b,c`) will misalign your columns.

### Anchors and multi-line files — a subtlety worth knowing

`^` and `$` in `grep`/`sed`/`awk` regex refer to the start and end of **each individual line** being processed, not the start/end of the whole file. Since these tools process input **line by line** by default, this is usually exactly what you'd expect — but it's worth being explicit about, since some other regex engines (in some programming languages) let `^`/`$` optionally mean "start/end of the whole string," which can span multiple lines. In classic grep/sed/awk usage, one line in, one line's worth of `^`/`$` per pass — keep that mental model and you won't be surprised.

---

## Practical Examples

### Example 1 — Character classes and quantifiers in `grep -E`

Sample file `contacts.txt`:
```
Order #4471 shipped to 192.168.1.42
Order #88 delayed, contact: sales_team@example.com
Invalid entry: ORDER-ABC
Order #100234 delivered
```

```bash
grep -E "Order #[0-9]{3,}" contacts.txt
```

Expected output:
```
Order #4471 shipped to 192.168.1.42
Order #100234 delivered
```

Line-by-line:
- `-E` turns on extended regex so `{3,}` works without escaping.
- `Order #` is matched literally.
- `[0-9]{3,}` matches **three or more digits in a row** — `#4471` (4 digits) and `#100234` (6 digits) both qualify; `#88` (only 2 digits) is correctly excluded.

💡 **Tip:** Compare this to `grep -E "Order #[0-9]+"`, which would also match `#88` (one-or-more, no minimum count). The `{3,}` form gives you precise control that `+` alone can't.

### Example 2 — `sed -i` in-place editing with a backup, tested safely first

Sample file `app.conf`:
```
debug_mode=true
timeout=30
host=old-server.internal
log_level=verbose
```

**Step 1 — ALWAYS test without `-i` first**, so you only ever see the result printed to your screen, with the real file completely untouched:

```bash
sed 's/old-server\.internal/new-server.internal/' app.conf
```

Expected output (printed to terminal only — `app.conf` is unchanged on disk):
```
debug_mode=true
timeout=30
host=new-server.internal
log_level=verbose
```

⚠️ **Warning — read this before you ever touch `-i`:** `sed -i` edits the file **immediately and irreversibly**, with **no confirmation prompt** and **no undo**. If your pattern was subtly wrong — matched more than you intended, or matched nothing at all and you don't notice — you can silently corrupt or fail to fix a production config file with zero warning. Get in the unbreakable habit of running the exact same expression **without** `-i` first, reading the output carefully, and only adding `-i` once you're confident.

**Step 2 — once confirmed, run with `-i.bak`** so a backup is created automatically before the file is overwritten:

```bash
sed -i.bak 's/old-server\.internal/new-server.internal/' app.conf
```

**Step 3 — diff to confirm exactly what changed:**

```bash
diff app.conf.bak app.conf
```

Expected output:
```
3c3
< host=old-server.internal
---
> host=new-server.internal
```

Line-by-line:
- `sed -i.bak '...'` first copies the original file to `app.conf.bak`, then overwrites `app.conf` with the transformed result — the `.bak` suffix is required immediately after `-i` with no space (GNU sed also accepts `-i` with no suffix at all, which skips the backup entirely — never do that on anything you can't afford to lose).
- `diff app.conf.bak app.conf` proves exactly one line changed, exactly as intended, giving you a paper trail of the edit.

✅ **Best Practice:** `sed -i.bak` should be your **default** habit for any in-place edit on a file that matters, not an occasional precaution. Deleting a `.bak` file once you've confirmed success costs you nothing; needing one you didn't make can cost you a production outage.

### Example 3 — sed capture groups: rearranging matched text

Sample file `access-dates.txt`:
```
2026-07-28 request from 10.0.0.5
2026-01-15 request from 10.0.0.9
```

```bash
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\2\/\3\/\1/' access-dates.txt
```

Expected output:
```
07/28/2026 request from 10.0.0.5
01/15/2026 request from 10.0.0.9
```

Line-by-line:
- `-E` enables ERE so the plain `(...)` grouping works without escaping.
- Three capture groups are defined: `([0-9]{4})` = the year → `\1`, `([0-9]{2})` = the month → `\2`, `([0-9]{2})` = the day → `\3`.
- The replacement `\2\/\3\/\1` rebuilds the date as `month/day/year`, referencing the captured groups in a new order. The `\/` escapes the forward slash so sed doesn't mistake it for the delimiter ending the replacement.

💡 **Tip:** You can also use `&` when you want to keep the whole match intact but wrap something around it: `echo "total: 500" | sed -E 's/[0-9]+/[&]/'` produces `total: [500]` — `&` stands for "whatever was actually matched," here the whole number.

### Example 4 — awk column extraction

Sample file `employees.csv`:
```
name,department,salary
Alice,Engineering,95000
Bob,Sales,72000
Carol,Engineering,101000
```

```bash
awk -F',' 'NR > 1 { print $1, $3 }' employees.csv
```

Expected output:
```
Alice 95000
Bob 72000
Carol 101000
```

Line-by-line:
- `-F','` tells awk to split each line on commas instead of the default whitespace, since this is a CSV.
- `NR > 1` is the pattern — it's true for every line **except the first** (the header row, where `NR` equals 1), so the header is skipped.
- `{ print $1, $3 }` is the action — print the 1st field (name) and 3rd field (salary), separated by a space (awk's default `OFS`).

### Example 5 — awk: sum a column

Using the same `employees.csv`:

```bash
awk -F',' 'NR > 1 { total += $3; count++ } END { print "Total:", total; print "Average:", total/count }'  employees.csv
```

Expected output:
```
Total: 268000
Average: 89333.3
```

Line-by-line:
- For every non-header row, `total += $3` adds that row's salary to a running total (awk variables start at `0`/empty automatically — no need to initialize `total=0` yourself), and `count++` increments a row counter.
- `END { ... }` runs exactly once, after every line has been processed, printing the final sum and the average.

🎯 **On the job:** "Sum a column" and "average a column" are two of the single most common real-world awk uses — anytime someone hands you a CSV or a log and asks "what's the total/average of this number across every row," this is the one-liner.

### Example 6 — 🎯 Combined pipeline: parsing an access log for status-code counts

Sample file `access.log` (a simplified, realistic web server access log):
```
203.0.113.5 - - [28/Jul/2026:10:15:01] "GET /home HTTP/1.1" 200 512
203.0.113.7 - - [28/Jul/2026:10:15:03] "GET /login HTTP/1.1" 200 340
198.51.100.2 - - [28/Jul/2026:10:15:05] "POST /login HTTP/1.1" 401 128
203.0.113.5 - - [28/Jul/2026:10:15:09] "GET /cart HTTP/1.1" 500 89
198.51.100.2 - - [28/Jul/2026:10:15:12] "GET /home HTTP/1.1" 200 512
203.0.113.9 - - [28/Jul/2026:10:15:20] "GET /checkout HTTP/1.1" 500 76
```

**Goal:** count how many requests returned each HTTP status code, most frequent first.

```bash
awk '{ print $9 }' access.log | sort | uniq -c | sort -rn
```

Expected output:
```
      3 200
      2 500
      1 401
```

Line-by-line:
- `awk '{ print $9 }'` — in this log format, the HTTP status code is the 9th whitespace-separated field (count them: IP, dash, dash, `[timestamp]`, `"METHOD`, `path`, `HTTP/1.1"`, `status`, `size` — that's fields 1 through 9). Awk's default whitespace splitting works here because the log format is space-separated (with the bracketed timestamp and quoted request treated by awk as containing internal spaces — in a real production parser you'd often use a more precise field-separator or a dedicated log-parsing regex, but plain whitespace-splitting works well enough for this common Apache/Nginx "combined-ish" layout since the fields we want land in fixed positions).
- `sort` groups identical status codes next to each other, a prerequisite for `uniq` to work.
- `uniq -c` collapses consecutive duplicates and prefixes each with a **count**.
- `sort -rn` sorts those counts **r**everse **n**umerically — highest count first — so the most common status code appears at the top.

✅ **Best Practice:** `sort | uniq -c | sort -rn` is a workflow worth memorizing on its own — "count and rank anything" — you'll reuse it constantly, on status codes, error messages, IP addresses, user agents, anything.

🎯 **On the job:** This exact pipeline (or a close variant) is one of the most common real commands run directly on a production server during an incident — no dashboard needed, just `grep`/`awk`/`sort`/`uniq` chained together to answer "what's actually happening right now?" in seconds.

---

## Common Pitfalls & Best Practices

- **BRE vs. ERE escaping confusion.** Forgetting that plain `grep`/`sed` need `\+`, `\?`, `\(`, `\)`, `\|` (BRE) while `-E`/`-r` don't is the single most common regex bug on Linux. If your pattern uses grouping, alternation, or `+`/`?`/`{}`, default to adding `-E` rather than hand-escaping BRE.
- **Forgetting the `-i` backup.** `sed -i` with no suffix overwrites with zero backup and zero undo. Make `sed -i.bak` (or testing without `-i` first) a non-negotiable habit on anything that matters.
- **Greedy match surprises.** `.*` between two delimiters (like `<.*>`) grabs the *longest* possible match, often spanning much further than intended. Prefer a negated character class (`[^>]*`) when you want to stop at the *next* occurrence of a delimiter rather than the *last* one.
- **awk field-splitting on multiple spaces.** Awk's default `FS` treats any run of whitespace as one separator and ignores leading whitespace — usually helpful, but it means a genuinely empty field in whitespace-delimited data can silently disappear or shift every later column by one. When the data has a real, fixed delimiter (CSV commas, `/etc/passwd` colons), always set `-F` explicitly rather than relying on the default.
- **Unescaped `.` in patterns meant to match a literal dot.** `192.168.1.1` as a regex also matches `192X168Y1Z1` — always escape a literal dot as `\.` in an IP, version number, domain, or filename extension pattern.
- **Confusing shell globs with regex.** `*` alone means "zero or more of the previous thing" in regex, but "any characters" in a shell glob — the same character, two unrelated languages, depending on which tool you're feeding it to.
- **Not testing a sed/awk one-liner against a small sample first.** Always try a new pattern against a handful of representative lines (or `head -20 file | your-pipeline`) before running it against a multi-gigabyte production file or, worse, with `-i`.

---

## Hands-on Exercise

**Sample file — save this as `orders.csv`:**
```
order_id,customer,status,amount
1001,Alice Smith,SHIPPED,49.99
1002,Bob Jones,PENDING,120.50
1003,Carol Diaz,SHIPPED,15.00
1004,Dan Lee,CANCELLED,89.25
1005,Erin Cole,SHIPPED,200.00
```

**Task:**

1. Use `sed` to change every occurrence of `PENDING` to `IN_PROGRESS` in the file, **safely** — test first, then apply the change in-place with a backup, then confirm with `diff`.
2. Use `awk` to print only the `customer` and `amount` columns for orders whose status is `SHIPPED`.
3. Use `awk` to calculate the **total dollar amount** across all `SHIPPED` orders only.

Try writing the commands yourself before reading the solution below.

### Solution

**1. Safely rename `PENDING` to `IN_PROGRESS`:**

```bash
# Test first — no -i, just look at the output
sed 's/PENDING/IN_PROGRESS/' orders.csv
```
```
order_id,customer,status,amount
1001,Alice Smith,SHIPPED,49.99
1002,Bob Jones,IN_PROGRESS,120.50
1003,Carol Diaz,SHIPPED,15.00
1004,Dan Lee,CANCELLED,89.25
1005,Erin Cole,SHIPPED,200.00
```

```bash
# Looks correct — now apply it for real, with a backup
sed -i.bak 's/PENDING/IN_PROGRESS/' orders.csv
diff orders.csv.bak orders.csv
```
```
3c3
< 1002,Bob Jones,PENDING,120.50
---
> 1002,Bob Jones,IN_PROGRESS,120.50
```

**2. Print customer and amount for `SHIPPED` orders:**

```bash
awk -F',' '$3 == "SHIPPED" { print $2, $4 }' orders.csv
```
```
Alice Smith 49.99
Carol Diaz 15.00
Erin Cole 200.00
```

**3. Total dollar amount across `SHIPPED` orders:**

```bash
awk -F',' '$3 == "SHIPPED" { total += $4 } END { print "Total shipped revenue:", total }' orders.csv
```
```
Total shipped revenue: 264.99
```

Explanation: I always tested the `sed` substitution without `-i` first, confirmed the exact line that would change, and only then re-ran it with `-i.bak` so a recoverable original still exists on disk — `diff` proved exactly one line changed and nothing else. For both `awk` steps, `-F','` was required since this is comma-separated data, not whitespace-separated; `$3 == "SHIPPED"` as the **pattern** filters which rows the action applies to, without needing an explicit `if` inside `{ }`. The final `total += $4` / `END` combination is the same "sum a column" pattern from Example 5, just filtered down to a subset of rows first.

✅ Exercise complete — you've safely edited a file in-place with sed, and filtered, extracted, and aggregated CSV data with awk, using nothing but tools guaranteed to exist on any Linux box.

---

## ✅ Module Completion Checklist

- [ ] I can explain the difference between BRE and ERE, and know which tools/flags use which by default
- [ ] I can build regex patterns using anchors, character classes (including POSIX classes), `.`, and quantifiers
- [ ] I can use `grep -E` for extended regex and know when `grep -P` exists as a more powerful (not always installed) option
- [ ] I can use `sed` for substitution, the `g`/`i` flags, safe in-place editing with `-i.bak`, deleting lines, and printing line ranges
- [ ] I can use sed address ranges and back-references (`&`, `\1`, `\2`) in replacements
- [ ] I can explain awk's field-based mental model (`$0`, `$1`...`$NF`, `NR`, `NF`, `FS`, `OFS`) and set a field separator with `-F`
- [ ] I can write awk `BEGIN`/`END` blocks and pattern-action pairs to filter, transform, and sum column data
- [ ] I can combine `grep`, `sed`, and `awk` into one realistic pipeline to parse logs or CSV data
- [ ] I can recognize the common pitfalls in this module and apply the corresponding best practices

## Next Step

Continue to [Module 10: Process Management & Job Control](../module10-process-management/)
