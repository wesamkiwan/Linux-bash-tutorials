# 🎤 Module 3 Interview Prep — Viewing & Finding Files

## Conceptual Questions

### 🟢 Beginner

**Q1: What's the difference between `cat`, `less`, `head`, and `tail`, and when would you use each?**

> "`cat` dumps a file's entire content to the terminal at once — fine for small files, but it floods your screen on a huge one. `less` opens a file in an interactive, scrollable pager that only loads what's on screen, so it stays fast on files of any size — I'd reach for that to actually browse something large. `head` shows just the first lines of a file, which is great for a quick sanity check like confirming a CSV's header row. `tail` shows the last lines, which is what you want for the freshest entries in a log file."

**Q2: Why is `less` considered the modern default over `more`?**

> "`more` is an older pager that can only scroll forward through a file — once you pass a section, you can't easily go back. `less` can do everything `more` does plus scroll backward, search in both directions, and jump to specific points, and the name is even a pun on 'less is more.' On any modern Ubuntu system, `less` is what you should reach for; `more` mostly survives in old scripts and tutorials."

**Q3: What does `tail -f` do, and why is it so commonly used?**

> "It prints the last lines of a file and then keeps the connection open, printing every new line the instant it's appended. Log files are almost always written append-only, so `tail -f` is how you watch an application's behavior live, in real time, instead of repeatedly re-running `cat` to see if anything new showed up. It's probably the single most common command I'd use while actively debugging a running service."

**Q4: What's the difference between `wc -l`, `wc -w`, and `wc -c`?**

> "`wc -l` counts lines, `wc -w` counts words, and `wc -c` counts characters, or more precisely bytes. Plain `wc` with no flags prints all three together, in that order. I use `wc -l` constantly — piped after a `grep`, it turns 'here are the matching lines' into 'here's exactly how many matched.'"

**Q5: In plain terms, what does `grep` do?**

> "It searches through a file's contents, line by line, for lines that contain a given pattern, and prints those matching lines. The name itself comes from an old text-editor command — 'globally search a regular expression and print.' At a beginner level I keep the pattern simple, just literal text I'm looking for; `grep`'s true power comes from regular expressions, which is its own deep topic."

### 🟡 Intermediate

**Q6: What's the core difference between what `grep` searches and what `find` searches?**

> "`grep` searches file *content* — the actual text written inside a file. `find` searches file *metadata* — things like the filename, its type, its size, and its timestamps — without caring what's written inside. I think of it like a librarian: `grep` is 'find me the book with this word on some page,' and `find` is 'find me the book with this title, published after this date, over this size' — pure catalog information, no reading required."

**Q7: Explain the difference between `grep -r` and `grep -R`.**

> "Both recurse into a directory and search every file inside it, including subdirectories. The difference is symbolic links: `-R` will follow a symlink into whatever directory it points at and search there too, while lowercase `-r`'s exact symlink behavior is more conservative depending on the implementation. In day-to-day use they behave the same most of the time, but if you're searching a tree with symlinks in it and results look incomplete, that's worth checking."

**Q8: Why does `find . -name *.txt` (unquoted) sometimes behave unexpectedly?**

> "Because the shell expands wildcards *before* the command even runs. If you don't quote `*.txt`, Bash tries to expand it against filenames in your current directory first — so if there happen to be one or more `.txt` files sitting right there, `find` receives a list of literal filenames instead of the pattern `*.txt`, and it searches for exactly those names throughout the tree, not a wildcard. If there are zero matching files in the current directory, some shells pass the literal, unexpanded string through, which happens to work but only by accident. Always quote the pattern — `find . -name "*.txt"` — to guarantee `find` itself does the wildcard matching."

**Q9: What's the practical difference between `-name` and `-iname` in `find`?**

> "`-name` matches the filename case-sensitively, so `-name '*.TXT'` would miss a file called `notes.txt`. `-iname` does the same match but ignores case entirely. On a case-sensitive filesystem like ext4 on Linux, that distinction matters a lot more than it would on Windows, where filenames aren't case-sensitive by default."

**Q10: How would you explain what a pipe does to someone who's never seen one?**

> "A pipe, written as a vertical bar, takes whatever the command on its left would normally print to the screen and instead feeds it directly in as the input to the command on the right — no temporary file involved. That's how you chain small, single-purpose tools together into a bigger recipe, like piping `grep`'s matching lines into `wc -l` to get a count, or into `sort` and `uniq` to clean up duplicates."

### 🔴 Advanced

**Q11: `locate` returned nothing for a file you just created five seconds ago, but `find` found it instantly. Why, and what would you do about it?**

> "`locate` doesn't search the live filesystem — it searches a pre-built index, a database of file paths that's normally only refreshed once a day by a scheduled job called `updatedb`. A file created seconds ago simply isn't in that index yet. `find`, by contrast, always walks the actual filesystem in real time, so it's slower on a huge disk but never stale. If I needed `locate` to see the new file immediately, I'd run `sudo updatedb` to force a refresh; otherwise I'd just use `find` when I need guaranteed up-to-the-second accuracy."

**Q12: What are the risks of using `find ... -exec rm {} \;` in a production environment, and how would you mitigate them?**

> "The biggest risk is that `-exec` will happily run the destructive command against every single match, with no confirmation step, so a slightly wrong `-name` pattern or an unintended `-type`/`-size`/`-mtime` filter can delete far more than intended. My mitigation is always to run the exact same `find` command *without* `-exec` first, review the full list of matches carefully, and only add `-exec rm {} \;` once I'm confident the list is exactly right. For anything really sensitive, I'd also consider `-exec mv {} /tmp/quarantine/ \;` instead of an outright delete, so there's a recovery path if the match set turns out to be wrong."

---

## Practical/Coding Questions

**Q1: You need to see just the last 20 lines of `/var/log/nginx/error.log`, and then keep watching it live for new entries. Show the commands.**

Solution:
```bash
tail -n 20 /var/log/nginx/error.log
tail -f /var/log/nginx/error.log
```
Explanation: The first command gives an immediate snapshot of the most recent 20 lines. The second then switches into live-follow mode, printing new lines as they're appended, until interrupted with `Ctrl+C`. In practice I'd often just run `tail -n 20 -f file` in one command to get both at once.

**Q2: Count how many lines in `access.log` mention the string "500" (an HTTP server error), case-insensitively, without printing the matching lines themselves.**

Solution:
```bash
grep -ic "500" access.log
```
Explanation: `-i` makes the match case-insensitive (relevant if "500" ever appears inside mixed-case text around it), and `-c` tells `grep` to print only the count of matching lines rather than the lines themselves.

**Q3: Find every file under `/var/log` larger than 50 megabytes that hasn't been touched in more than 30 days — good candidates for archiving or deletion.**

Solution:
```bash
find /var/log -type f -size +50M -mtime +30
```
Explanation: `-type f` restricts to regular files (skip directories), `-size +50M` keeps only files over 50 MB, and `-mtime +30` keeps only files whose last modification was more than 30 days ago. I'd run this exact command first to review the list before ever attaching `-exec rm {} \;` to it.

**Q4: You have a directory of log files and want to know, across all of them, which distinct IP addresses appear on lines containing "login failed." Show a full pipe.**

Solution:
```bash
grep -h "login failed" *.log | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" | sort | uniq
```
Explanation: `grep -h` (suppressing filenames since we don't need them here) filters for the relevant lines across all `.log` files, a second `grep -o` with a simple regex pulls out just the IP-shaped text (a small preview of the regex depth coming in Module 9), and `sort | uniq` collapses that down to the distinct set of IPs involved. Note: `-E` enables extended regex syntax for that pattern — full regex mechanics belong to Module 9, but it's worth knowing this option exists.

**Q5: A teammate says "I don't remember the exact filename, but I know it's some kind of `.conf` file somewhere under `/etc`, and it was edited earlier today." How would you find it?**

Solution:
```bash
find /etc -iname "*.conf" -mmin -720
```
Explanation: `-iname "*.conf"` matches the extension case-insensitively, and `-mmin -720` narrows it to files modified within the last 720 minutes (12 hours) — a reasonable stand-in for "edited earlier today." I'd adjust the exact `-mmin`/`-mtime` value based on how precisely "today" needs to be interpreted.

---

## Gotcha Questions

**Q1: "My script checks `if grep 'pattern' file; then ...` — what is that actually checking, and what's the trap?"**

> Trap: Beginners often think `grep` only prints text and forget it also has an **exit code** — `0` if it found at least one match, and a non-zero code (`1`) if it found nothing. That exit code is exactly what `if grep ...` is testing, not the printed output itself. The deeper trap: a *third* exit code, `2`, means something actually went wrong (like the file not existing), which a naive `if grep ...` treats identically to "no match found" — so scripts that need to tell "no match" apart from "an error occurred" have to check more carefully than a simple `if`.

**Q2: "I typed `find . -name report.txt` and got nothing, but the file is right there when I `ls`. What went wrong?"**

> Trap: The most common cause is a case mismatch — `-name` is case-sensitive, so if the real file is `Report.txt` or `REPORT.TXT`, a search for `report.txt` finds nothing. The fix is `-iname` for a case-insensitive match. A second common cause: forgetting quotes around a wildcard pattern lets the shell expand it early, which behaves differently than intended, though that specific example has no wildcard so it points straight at the case issue.

**Q3: "I need to search for the exact phrase `error code` (two words with a space) using `grep`, but typing it without quotes gave a weird error or wrong result." Why?**

> Trap: Without quotes, the shell splits `grep error code file.txt` into three separate arguments — `grep`, `error`, `code`, and `file.txt` — so `grep` thinks `error` is the pattern and treats `code` as another filename to search (which likely doesn't exist, producing a "no such file" error). The fix is always to quote multi-word patterns: `grep "error code" file.txt`. This same quoting rule applies to `find -name` patterns containing wildcards, for a related reason — the shell, not the command, is what needs to be told "don't touch this."

**Q4: "`locate somefile` returns nothing, but I can see the file with `ls`. Is `locate` broken?"**

> Trap: `locate` isn't broken — it's just reading a possibly-stale index rather than the live filesystem. If the file was created recently, or `updatedb` hasn't run yet (it's often scheduled just once a day), the index simply doesn't know the file exists yet. The candidate should know to try `sudo updatedb` or just fall back to `find` rather than assuming a functional bug.

**Q5: "Doesn't `head file1 file2` just print the first 10 lines total, combined across both files?"**

> Trap: No — `head` (and `tail`) print up to 10 lines from *each* file given, not 10 lines total across all of them, and by default it prints a header line naming each file (`==> file1 <==`) so you can tell where one file's output ends and the next begins. This surprises people who expect it to behave like `cat`, which really does just concatenate everything together with no separators.

---

## Quick-Fire Rapid Review

- **Q: Which pager is the modern default on Ubuntu, `less` or `more`?** A: `less`.
- **Q: Key to quit `less`?** A: `q`.
- **Q: Key inside `less` to search forward?** A: `/pattern`, then `n` for next match.
- **Q: Command to watch a log file live as it grows?** A: `tail -f file`.
- **Q: How do you stop `tail -f`?** A: `Ctrl+C`.
- **Q: Flag to `wc` for just a line count?** A: `-l`.
- **Q: Flag to `grep` for case-insensitive search?** A: `-i`.
- **Q: Flag to `grep` to invert the match (show non-matching lines)?** A: `-v`.
- **Q: Flag to `grep` to search recursively?** A: `-r` (or `-R` to also follow symlinks).
- **Q: Flag to `grep` for just a count of matches?** A: `-c`.
- **Q: Flag to `grep` for whole-word-only matches?** A: `-w`.
- **Q: Does `find` search file content or file metadata?** A: Metadata (name, type, size, timestamps) — `grep` searches content.
- **Q: Difference between `-name` and `-iname` in `find`?** A: `-name` is case-sensitive, `-iname` is not.
- **Q: What does `-type f` restrict `find` to?** A: Regular files only (not directories or links).
- **Q: What does `-mtime -7` mean?** A: Modified less than 7 days ago.
- **Q: What's the safety habit before using `find ... -exec rm {} \;`?** A: Run the same `find` without `-exec` first to review the match list.
- **Q: Why might `locate` miss a brand-new file?** A: Its index (`updatedb`) is only refreshed periodically, often once a day.
- **Q: Command to find which executable actually runs for a given command name?** A: `which`.
- **Q: What does `sort file | uniq` do?** A: Sorts lines, then removes consecutive duplicates.
- **Q: Why quote `grep`/`find` patterns with spaces or wildcards?** A: To stop the shell from splitting or expanding them before the command runs.
