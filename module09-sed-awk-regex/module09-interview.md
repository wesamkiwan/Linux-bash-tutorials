# 🎤 Module 9 Interview Prep — sed, awk & Regex

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a regular expression, and how is it different from a plain literal search?**

> "A regular expression, or regex, is a pattern language for describing the *shape* of text, not one specific literal string. A literal search for `error123` only matches that exact text. A regex like `error[0-9]+` matches `error1`, `error42`, or `error999` — anything starting with `error` followed by one or more digits — with a single pattern. It's not specific to Linux either; the same underlying ideas show up in grep, sed, awk, and most programming languages, so it's a skill you reuse constantly."

**Q2: What does the `^` anchor mean, and how is its meaning different inside a character class?**

> "On its own, `^` anchors a match to the **start of a line** — it doesn't consume a character, it just marks a position. But inside square brackets, like `[^abc]`, it means something completely different: negate the set, match any character that is **not** `a`, `b`, or `c`. Same symbol, two unrelated meanings depending on context — that's a classic beginner trip-up worth memorizing explicitly."

**Q3: What does the `.` (dot) match in regex, and what's a common mistake people make with it?**

> "A bare `.` matches any single character — letter, digit, symbol, basically anything except a newline in most contexts. The common mistake is treating it as if it means 'anything at all, any length,' when really it's exactly one character. People also forget that if they actually want to match a literal period — like in an IP address or file extension — they need to escape it as `\.`, or an unescaped `.` will also match things like `192X168` instead of only `192.168`."

**Q4: What does `sed` stand for, and what's its basic job?**

> "Stream editor. It reads input one line at a time and applies editing commands — most commonly substitution — to each line, streaming the result to standard output. It never loads a whole file into memory the way a text editor would, which is exactly why it can handle multi-gigabyte files that wouldn't fit comfortably in an interactive editor."

**Q5: In awk, what do `$0`, `$1`, and `NF` each refer to?**

> "`$0` is the entire current line, unmodified. `$1` is the first field — the first whitespace-separated (or custom-separator-separated) chunk of that line. `NF` is the **number** of fields on the current line, so `$NF` — using NF as the field number — always refers to the *last* field, whatever position that happens to be, which is handy since it works even if the field count varies from line to line."

### 🟡 Intermediate

**Q6: What's the difference between BRE and ERE, and why does it matter in practice?**

> "BRE, Basic Regular Expressions, is the older, default flavor that plain `grep` and plain `sed` use. In BRE, characters like `+`, `?`, `{`, `}`, `(`, `)`, and `|` are treated as **literal** characters unless you escape them with a backslash — so grouping is `\(...\)`, one-or-more is `\+`. ERE, Extended Regular Expressions, flips that: those same characters work directly, with no escaping, and you get it with `grep -E` or `sed -E`/`-r`. It matters in practice because it's probably the single most common source of 'why isn't my regex matching' bugs on Linux — someone writes `grep "a|b"` expecting alternation and it silently searches for a literal pipe character instead, because plain grep is BRE by default. My habit is to just default to `-E` whenever I need grouping, alternation, or those quantifiers, rather than keep BRE's escaping rules memorized."

**Q7: When would you reach for `awk` instead of `sed` for a task?**

> "It comes down to whether I'm thinking in terms of whole-line text transformation or in terms of columns and structured data. If I need to find-and-replace text, delete certain lines, or do targeted rewrites within a line, that's sed's job — it's built for that. The moment I need to reason about specific fields — 'sum the third column,' 'print only rows where the fourth field is over 100,' 'filter based on one column's value' — that's awk's job, because it automatically splits every line into fields I can reference by number and gives me real variables, arithmetic, and conditionals to work with. A rough rule: sed edits text; awk processes structured, column-shaped data."

**Q8: How do sed capture groups work, and what's a realistic use case?**

> "I wrap part of my pattern in parentheses — `\(...\)` in BRE or plain `(...)` with `-E` — and sed remembers whatever that group matched, letting me reference it in the replacement as `\1`, `\2`, and so on, in whatever order I want. A realistic case is reformatting a date: matching `([0-9]{4})-([0-9]{2})-([0-9]{2})` as year, month, day, then writing the replacement as `\2/\3/\1` to output it as month/day/year — I'm not just replacing the match, I'm rearranging pieces of it."

**Q9: What's the difference between `grep -E` and `grep -P`?**

> "`-E` switches on Extended Regular Expressions — the POSIX ERE flavor, giving you grouping, alternation, and quantifiers like `+`, `?`, `{n,m}` without escaping. `-P` goes further and switches on PCRE, Perl-Compatible Regular Expressions, which adds things POSIX regex doesn't have at all — shorthand classes like `\d` and `\w`, lazy (non-greedy) quantifiers like `.*?`, and lookahead/lookbehind assertions. `-P` is genuinely more powerful, but it's worth knowing it isn't guaranteed to be compiled into every grep binary on every minimal system, so I'd sanity-check it's available before relying on it in a portable script, whereas `-E` is essentially always there."

**Q10: What's "greedy matching," and how does it commonly surprise people?**

> "By default, regex quantifiers try to match the longest possible piece of text that still lets the whole pattern succeed, not the shortest. The classic example is a pattern like `<.*>` against a line containing multiple tags, like `<b>bold</b> and <i>italic</i>` — you might expect it to stop at the very first `>`, matching just `<b>`, but greedy `.*` actually consumes everything up through the very last `>` on the line. The fix in plain POSIX regex, since there's no lazy quantifier available, is to be more specific about what you don't want — using a negated character class like `[^>]*` instead of `.*`, which correctly stops at the next `>` rather than the last one."

### 🔴 Advanced

**Q11: What does `NR==FNR` mean, and when would you use it?**

> "`NR` is the total record (line) count across everything awk has read so far, while `FNR` is the record count **within the current file only** — it resets back to 1 every time awk starts reading a new file. When awk is given two files, `NR==FNR` is true only while it's still processing the **first** file, because up through the end of that file, the running total and the per-file count are identical; the moment awk starts the second file, `FNR` resets to 1 but `NR` keeps climbing, so they diverge. This is the classic idiom for 'load data from file A into memory, then use it while processing file B' — for example, `awk 'NR==FNR{lookup[$1]=$2; next} FNR in lookup{print $0, lookup[FNR]}' fileA fileB` builds a lookup table from the first file, and applies it while reading the second. It's a bit of a gotcha because it looks cryptic if you haven't seen the pattern before, but it's genuinely one of the most useful idioms once two-file joins come up."

**Q12: Why doesn't sed modify a file by default, and what does that design choice buy you?**

> "sed was designed as a filter — read from input, write the transformed result to standard output, exactly like most classic Unix tools. `-i` is a deliberate, explicit opt-in for the specific case where you actually want the source file overwritten, rather than the default behavior. The practical benefit is that I can always test a substitution safely: run it without `-i`, read the output, and only commit to modifying the real file once I'm sure the pattern does exactly what I think it does. If sed modified files by default, there'd be no safe way to 'preview' an edit before committing to it."

**Q13: How would you design a text-processing pipeline to handle a multi-gigabyte log file that won't fit comfortably in memory, and why would sed/awk/grep be a reasonable choice over, say, loading it into a scripting language's data structures?**

> "The key property of grep, sed, and awk is that they're all **streaming** tools — they process input one line at a time and never need to hold the entire file in memory at once, so their memory footprint stays roughly constant regardless of whether the file is 10 megabytes or 10 gigabytes. I'd chain them so each does the part it's best at: `grep` to cheaply filter down to only the lines that matter first (cutting the volume early, before more expensive processing), then `awk` to parse structure, extract fields, and aggregate — sums, counts, group-bys — and finally `sort`/`uniq -c` to rank results. Loading the same file into, say, a list of dictionaries in a general-purpose scripting language risks blowing available memory or taking far longer, especially on a production box where you might not even have that language's runtime installed, let alone spare RAM to spare during an incident."

---

## Practical/Coding Questions

**Q1: Write a command that extracts every IP-address-like string from a log file.**

Solution:
```bash
grep -Eo "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" access.log
```
Explanation: `-E` enables the `{1,3}` repetition counts without escaping; `-o` prints only the matched text, not the whole line. Each `[0-9]{1,3}` matches one to three digits, separated by literal (escaped) dots. This is deliberately "IP-like" rather than a strict validator — it would also match something like `999.999.999.999` — which is fine for quickly scraping candidate IPs out of a log, but not for validating real IP addresses.

**Q2: Write a `sed` command that deletes every blank line from a file, safely, with a backup.**

Solution:
```bash
sed 's/^$/BLANK-CHECK/' file.txt   # test first: confirm which lines would match
sed -i.bak '/^$/d' file.txt        # then apply in place with a backup
diff file.txt.bak file.txt
```
Explanation: `^$` anchors match a line that goes directly from start to end with nothing in between — an empty line. I test first by substituting a visible marker so I can see exactly which lines qualify, then run the real deletion (`/^$/d`) with `-i.bak` so the original is preserved, and `diff` proves exactly which lines were removed.

**Q3: Write an awk one-liner that prints the second and fourth columns of a colon-separated file, like `/etc/passwd`.**

Solution:
```bash
awk -F':' '{ print $2, $4 }' /etc/passwd
```
Explanation: `-F':'` sets the field separator to a colon, matching `/etc/passwd`'s actual format, instead of relying on the default whitespace splitting (which would be wrong here since the fields are colon-delimited and can contain no whitespace at all between them).

**Q4: Write a command that reformats `"LastName, FirstName"` into `"FirstName LastName"` using sed capture groups.**

Solution:
```bash
echo "Nguyen, Priya" | sed -E 's/([A-Za-z]+), ([A-Za-z]+)/\2 \1/'
```
Explanation: The pattern captures the last name as group 1 and the first name as group 2, then the replacement `\2 \1` prints them back in reversed order with a plain space between them — `-E` lets the grouping parentheses work without backslash-escaping.

**Q5: Write a pipeline that counts how many times each unique value appears in the third column of a CSV file, sorted from most to least common.**

Solution:
```bash
awk -F',' '{ print $3 }' data.csv | sort | uniq -c | sort -rn
```
Explanation: `awk` extracts just the third field from every row; `sort` groups identical values together (a prerequisite for `uniq` to work correctly, since `uniq` only collapses *consecutive* duplicates); `uniq -c` collapses those duplicates and prefixes each unique value with its count; the final `sort -rn` sorts numerically, in reverse, so the highest counts appear first.

---

## Gotcha Questions

**Q1: "I wrote `grep "cost: $5"` to find lines where the cost is exactly 5 dollars, but it matched nothing, even though I can see the line right there. What went wrong?"**

> Trap: The candidate needs to recognize that `$` is a regex anchor meaning "end of line," not a literal dollar sign, in this position. `"cost: $5"` is interpreted as "the literal text `cost: ` followed by end-of-line, followed by a literal `5`" — which can never match, since nothing can come after the end of a line. The fix is to escape it: `grep "cost: \$5"` — treating the dollar sign as a literal character instead of the end-of-line anchor. The broader lesson: any regex metacharacter (`. * [ ] ^ $ \`, and more with `-E`) that appears in your data as a literal character needs to be escaped, or it silently changes what your pattern means instead of erroring out.

**Q2: "My `sed -i` command ran without any errors, but now half my config file is gone and I don't have a backup. What happened, and how do you prevent it?"**

> Trap: This tests whether the candidate has actually internalized the safety habit or just knows the syntax. `sed -i` with no suffix overwrites the file **immediately**, with **zero confirmation and zero built-in undo** — if the pattern matched far more than intended (a common risk with a greedy or overly broad regex), the entire file can be silently gutted with no error message at all, because sed did exactly what it was told, just not what was meant. Prevention is entirely about habit: always test the exact same expression **without** `-i` first and actually read the output, then apply it with `-i.bak` (never bare `-i`) so a recoverable original exists, and `diff` the backup against the result to confirm the change was exactly what was intended.

**Q3: "I used `grep "a|b" file.txt` expecting it to match lines with 'a' OR 'b', but it returned nothing. Why?"**

> Trap: Classic BRE-vs-ERE trap. Plain `grep` defaults to Basic Regular Expressions, where `|` is a **literal** pipe character, not the alternation operator — so the pattern is actually searching for the literal three-character string `a|b`, which doesn't appear anywhere in the file. The fix is `grep -E "a|b"` (or the older `egrep "a|b"`) to enable Extended Regular Expressions, where `|` means OR. The deeper lesson interviewers are checking for: do you know *why* it failed silently instead of erroring, which is that BRE treats `|` as ordinary text rather than rejecting the pattern outright.

**Q4: "My pattern `<.*>` was supposed to grab just one HTML tag, but it grabbed everything from the first `<` all the way to the very last `>` on the line, spanning multiple tags. Why, and how do you fix it without lazy quantifiers?"**

> Trap: This is greedy matching. POSIX regex quantifiers (used by grep, sed, and awk) always try to consume the *longest* possible match that still lets the overall pattern succeed — there's no built-in "lazy"/non-greedy option in plain BRE/ERE the way some other languages provide (`grep -P`'s PCRE dialect does support `.*?`, but plain POSIX tools don't). The standard fix isn't "make it lazy" — it's to be more specific about what should **not** appear inside the match: replacing `.*` with a negated character class like `[^>]*` (any characters that are not a `>`) forces the match to stop at the very next `>` instead of the last one on the line.

---

## Quick-Fire Rapid Review

- **Q: What does `^` mean outside a character class? Inside one, as `[^abc]`?** A: Start of line; "NOT one of these characters."
- **Q: Does plain `grep`/`sed` use BRE or ERE by default?** A: BRE.
- **Q: How do you enable ERE in grep? In sed?** A: `grep -E`; `sed -E` (or `-r`).
- **Q: In BRE, how do you write "one or more"?** A: `\+` (escaped); in ERE it's plain `+`.
- **Q: What does a bare `.` match?** A: Any single character (except usually a newline).
- **Q: How do you match a literal dot?** A: Escape it — `\.`.
- **Q: What flag makes a sed substitution replace every match on a line, not just the first?** A: `g`.
- **Q: What's the safest way to run `sed -i`?** A: `sed -i.bak '...' file` — always with a backup suffix, tested without `-i` first.
- **Q: In sed, what does `&` mean in a replacement?** A: The entire text that was matched.
- **Q: In sed, what do `\1`, `\2` refer to?** A: Text captured by the 1st, 2nd, ... `(...)` group in the pattern.
- **Q: In awk, what does `$0` mean? `$NF`?** A: The whole current line; the last field, whatever its position.
- **Q: What does `NR` track in awk? `FNR`?** A: Total records read so far across all input; records read within the current file only.
- **Q: How do you set awk's field separator to a comma?** A: `awk -F','`.
- **Q: What runs exactly once, before any input is read, in awk?** A: A `BEGIN { }` block.
- **Q: Is regex quantifier matching greedy or lazy by default in POSIX tools?** A: Greedy — it matches the longest possible text.
- **Q: What pipeline pattern ranks how often values appear in a column?** A: `... | sort | uniq -c | sort -rn`.
