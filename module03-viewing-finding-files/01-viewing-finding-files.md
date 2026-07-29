# Module 3: Viewing & Finding Files 🟢

**Difficulty:** 🟢 Beginner
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-2

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] View file contents with `cat`, `cat -n`, `less`, and (briefly) `more`, and navigate confidently inside `less`
- [ ] Peek at the start or end of a file with `head` and `tail`, including `head -n` / `tail -n`
- [ ] Watch a file grow live with `tail -f` — the command you'll reach for constantly when debugging a running service
- [ ] Count lines, words, and characters in a file with `wc`, `wc -l`, `wc -w`, and `wc -c`
- [ ] Search for text inside files with `grep`, including `-i`, `-v`, `-r`/`-R`, `-n`, `-c`, `-l`, and `-w`
- [ ] Combine commands with pipes to answer real troubleshooting questions in one line
- [ ] Locate files by name, type, size, or modification time with `find`, including a first look at `-exec`
- [ ] Find files faster with `locate` (and know when it's out of date), and locate executables with `which` and `whereis`

---

## Module Goal

By the end of this module, you'll be able to open, skim, search, and locate any file on a Linux system without ever needing a graphical file manager or text editor.

🎯 **On the job:** Picture this — a teammate pings you: "the checkout service is throwing errors in production, can you take a look?" You don't have a GUI. You have SSH access to a server and a terminal. Within thirty seconds you need to find the right log file, watch it live as errors happen, search it for the word "error" or an order ID, and count how many times it occurred today. Every command in this module exists to make that thirty-second response possible. This is the toolkit that separates "I can use Linux" from "I can actually do my job on Linux."

---

## Core Concepts

### 1. Viewing vs. finding — two different jobs

This module covers two related but distinct skills:

- **Viewing** a file means looking at content you've already located — reading what's inside it, in full or in part.
- **Finding** a file means locating *which* file(s) you need in the first place, based on their name, location, size, age, or type — or locating text *inside* files you haven't opened yet.

💡 **Analogy — the librarian:** Think of a librarian who can help you two completely different ways. Ask "where's the book with the word 'dragon' on page 40?" and she searches by **content** — that's what `grep` does, hunting through the actual text inside files. Ask "where's the book titled *Dragons of Fire*, published after 2020, larger than 300 pages?" and she searches the **catalog** — metadata like name, date, and size, without reading a single page inside. That's exactly what `find` does. Keep this distinction in mind: `grep` looks *inside* files for matching text; `find` looks *at* files' names and metadata to decide which files qualify.

### 2. A file's contents can be small or huge — pick the right viewer

A tiny config file with 10 lines is easy — just dump the whole thing to your screen. But a log file with two million lines is a different problem entirely — dumping all of it would flood your terminal and be useless. Different tools exist for these different situations, and knowing which one to reach for is itself a skill.

### 3. Streams and redirection — a quick preview

When you run a command like `cat file.txt`, the file's content is sent to your terminal as **standard output** (often shortened to "stdout") — the normal text-output channel a program writes to. You'll go much deeper into redirecting and combining streams in a later module; for now, just know that piping (`|`), which you'll use heavily in this module, works by taking the standard output of one command and feeding it in as the standard input of the next.

### 4. A pattern is just "text you're looking for"

A **pattern** is the text (or eventually, a more powerful expression) you hand to a search tool so it knows what counts as a match. This module keeps patterns simple — literal words or short pieces of text. `grep` can do far more powerful pattern matching using **regular expressions** (a mini-language for describing text patterns) — we'll go much deeper into `grep` regex in Module 9.

### 5. Metadata — data about a file, not its content

**Metadata** means information *about* a file rather than what's written inside it — its name, size, type (regular file vs. directory), and timestamps (when it was last modified or accessed). `find` searches almost entirely by metadata; `grep` searches by content. Keep this straight and the rest of the module falls into place.

---

## Detailed Explanations

### Viewing a whole file: `cat`

`cat` (short for "concatenate," meaning "join things together") prints a file's entire contents straight to your terminal, from top to bottom, all at once.

| Command | What it does |
|---|---|
| `cat file.txt` | Prints the whole file to the terminal |
| `cat -n file.txt` | Same, but prefixes every line with a line number |
| `cat file1 file2` | Prints both files back to back, one after another (this is the "concatenate" part) |

⚠️ **Warning:** `cat` is great for small files, but running `cat` on a 2-million-line log file will scroll your terminal into oblivion. Use `less` or `head`/`tail` instead once a file gets big.

### Viewing a file page by page: `less` (and briefly, `more`)

`less` opens a file in an interactive, scrollable viewer — it only loads what's on screen, so it stays fast even on enormous files. It's the modern, default tool for this job on virtually every Linux distribution, including Ubuntu.

| Key | Action inside `less` |
|---|---|
| `Space` | Scroll forward one full page |
| `b` | Scroll backward one full page |
| `/pattern` | Search **forward** for `pattern` |
| `?pattern` | Search **backward** for `pattern` |
| `n` | Jump to the **next** match of your last search |
| `N` | Jump to the **previous** match of your last search |
| `g` | Jump to the top of the file |
| `G` | Jump to the bottom of the file |
| `q` | Quit and return to the shell |

💡 **Tip:** There's an older, more limited pager called `more` that only scrolls forward. You'll still see it mentioned in old tutorials and scripts, but `less` can do everything `more` does and much more (the name `less` is a pun — "less is more"). On modern Ubuntu, always reach for `less`.

🎯 **On the job:** `less` is what actually opens when you run `man <command>` — so you've already been using it since Module 1 without necessarily realizing it.

### Peeking at the start: `head`

`head` prints just the **first** lines of a file — the default is 10 lines if you don't specify.

| Command | What it does |
|---|---|
| `head file.txt` | Prints the first 10 lines |
| `head -n 5 file.txt` | Prints the first 5 lines |
| `head -n 3 file1 file2` | Prints the first 3 lines of each file, with a header naming each one |

🎯 **On the job:** `head` is perfect for a quick sanity check — "does this CSV file even have the header row I expect?" — without printing the whole thing.

### Peeking at the end: `tail`

`tail` is `head`'s mirror image — it prints the **last** lines of a file, again defaulting to 10.

| Command | What it does |
|---|---|
| `tail file.txt` | Prints the last 10 lines |
| `tail -n 20 file.txt` | Prints the last 20 lines |
| `tail -f file.txt` | **Follows** the file — keeps printing new lines live as they're appended |

**Why does `tail -f` matter so much?** Log files are almost always written **append-only** — new events get added to the *end* of the file as they happen. `tail -f` opens the last few lines and then keeps the connection open, printing every new line the instant it's written. This is how you "watch" a running application in real time from the terminal.

🎯 **On the job:** This is the single most common command for live debugging. You deploy a change, then immediately run `tail -f /var/log/myapp/app.log`, hit the feature in your browser, and watch the log scroll in real time to see exactly what happened — no need to keep re-running `cat` over and over.

💡 **Tip:** Press `Ctrl+C` to stop `tail -f` and get your prompt back — it runs forever until you interrupt it, because it has no way of knowing the file will ever stop growing.

### Counting things: `wc`

`wc` stands for "word count," and it counts lines, words, and characters (or bytes) in a file.

| Command | Counts |
|---|---|
| `wc file.txt` | Lines, words, and characters (bytes), all three, in that order |
| `wc -l file.txt` | Just the number of **lines** |
| `wc -w file.txt` | Just the number of **words** |
| `wc -c file.txt` | Just the number of **characters** (bytes) |

🎯 **On the job:** `wc -l` is everywhere — "how many rows are in this CSV?", "how many entries matched my search?", "how many files are in this listing?" It's almost always paired with a pipe from another command, which you'll see in the examples below.

### Searching file contents: `grep`

`grep` ("**g**lobally search a **r**egular **e**xpression and **p**rint") searches the contents of a file (or files) for lines that contain a given pattern, and prints those matching lines.

⚠️ This module keeps `grep` patterns **simple and literal** — plain words or short phrases. `grep`'s true power comes from **regular expressions**, a much richer pattern language for describing text — that's a full topic of its own, covered deeply in Module 9 alongside `sed` and `awk`. For now, think of `grep` as "find the lines containing this text."

| Command | What it does |
|---|---|
| `grep "pattern" file.txt` | Prints every line in `file.txt` containing `pattern` |
| `grep -i "pattern" file.txt` | Case-**insensitive** search (matches `Error`, `ERROR`, `error`, all the same) |
| `grep -v "pattern" file.txt` | **Invert** the match — prints every line that does *NOT* contain `pattern` |
| `grep -n "pattern" file.txt` | Prefixes each matching line with its **line number** |
| `grep -c "pattern" file.txt` | Prints just a **count** of matching lines, not the lines themselves |
| `grep -w "pattern" file.txt` | Matches only whole **words** (so `grep -w "cat"` won't match `concatenate`) |
| `grep -l "pattern" *.log` | Prints only the **filenames** that contain at least one match, not the matching lines |
| `grep -r "pattern" dir/` | **Recursively** searches every file inside `dir/` and its subdirectories |
| `grep -R "pattern" dir/` | Same as `-r`, but also follows symbolic links into other directories |

💡 **Tip:** Always quote your pattern — `grep "out of memory" file.txt`, not `grep out of memory file.txt`. Without quotes, the shell treats each word as a separate argument and `grep` gets confused about what you actually meant to search for.

🎯 **On the job:** `grep -ri "error" /var/log/` is close to a universal first move when troubleshooting anything on a Linux server — search every log recursively, ignoring case, for the word "error."

### Combining commands with pipes

A **pipe**, written as `|`, takes the output of the command on its left and feeds it directly in as the input of the command on its right, without any temporary file in between. You can chain several pipes together to build up a small "recipe" of filtering steps.

🎯 **On the job:** This is where these small tools stop being toys and become genuinely powerful — `grep` narrows things down, `wc -l` counts, `sort`/`uniq` clean things up, all chained together in a single line typed directly into a live troubleshooting session.

### Sorting and de-duplicating: a first look at `sort` and `uniq`

Two more small, composable tools worth knowing at a basic level right now:

| Command | What it does |
|---|---|
| `sort file.txt` | Prints the file's lines sorted alphabetically |
| `sort -n file.txt` | Sorts **numerically** instead of alphabetically (so `10` comes after `9`, not before `2`) |
| `uniq file.txt` | Removes **consecutive** duplicate lines |
| `sort file.txt \| uniq` | The classic combo — sort first so duplicates become consecutive, *then* dedupe them |

⚠️ **Warning:** `uniq` only removes duplicates that are directly next to each other. If your duplicate lines aren't already adjacent, `uniq` alone won't catch them — that's why `sort` almost always comes first in the pipe.

💡 We're only scratching the surface here — the full power of `sort` (multiple keys, custom fields) and `uniq` (`-c` to count occurrences, and more) is part of the text-processing power tools covered in Module 9.

### Finding files by name and type: `find`

`find` searches a directory tree for files and directories that match criteria you specify — by name, type, size, age, and more. Unlike `grep`, it doesn't look at file *contents* by default; it looks at file *metadata*.

**Basic shape of a `find` command:**

```
find <where-to-look> <what-to-match> <what-to-do>
```

| Command | What it does |
|---|---|
| `find .` | Lists every file and directory under the current directory, recursively |
| `find . -name "*.txt"` | Finds files/directories whose name matches `*.txt`, **case-sensitively** |
| `find . -iname "*.txt"` | Same, but **case-insensitively** (matches `Notes.TXT` too) |
| `find . -type f` | Only regular **f**iles (not directories, not links) |
| `find . -type d` | Only **d**irectories |
| `find . -size +10M` | Files larger than 10 megabytes |
| `find . -size -1k` | Files smaller than 1 kilobyte |
| `find . -mtime -7` | Files **m**odified within the last 7 days |
| `find . -mmin -30` | Files modified within the last 30 **min**utes |

⚠️ **Warning:** Quote your `-name`/`-iname` patterns, e.g. `-name "*.txt"`. Without quotes, the shell expands the wildcard itself against files in your *current* directory before `find` even runs — which usually isn't what you want, and can even make `find` error out or silently misbehave.

💡 **Tip:** `-mtime -7` means "modified less than 7 days ago." A plus sign instead of a minus (`-mtime +7`) flips it to mean "more than 7 days ago." This plus/minus convention shows up across several `find` options, including `-size`.

### Doing something with what you found: `-exec`

`-exec` lets `find` run a command on every file it matches, instead of just printing the list.

```bash
find . -name "*.tmp" -exec rm {} \;
```

- `{}` is a placeholder that `find` substitutes with each matching filename, one at a time.
- `\;` marks the end of the `-exec` command (the backslash stops the shell from treating the semicolon specially).

⚠️ **Warning:** This is an introductory look at `-exec` — it's genuinely easy to accidentally delete or modify the wrong files with it. Always run the `find` command *without* `-exec` first (just to see the list of matches), confirm the list looks right, and only then add the `-exec ... \;` part.

### A faster alternative: `locate`

`find` walks the actual filesystem live every time you run it, which can be slow on a huge disk. `locate` instead searches a pre-built **index** — a database of file paths — which makes it far faster.

| Command | What it does |
|---|---|
| `locate filename` | Instantly searches the index for paths containing `filename` |
| `sudo updatedb` | Manually rebuilds the index right now |

⚠️ **Warning:** That index is usually only refreshed once a day (via a scheduled background job). A file created five minutes ago may not show up in `locate` yet, even though it genuinely exists — this trips up beginners constantly. If a file feels like it "should" be found but isn't, either run `sudo updatedb` to refresh the index, or fall back to `find`, which always sees the live filesystem.

💡 **Tip:** `locate` isn't installed by default on every minimal Ubuntu image — you may need `sudo apt install mlocate` (or `plocate`, its modern faster replacement on recent Ubuntu/Debian releases) first. If `locate` isn't found, that's why.

### Finding executables: `which` and `whereis`

Sometimes you don't want to find a data file — you want to know exactly which program runs when you type a command name.

| Command | What it does |
|---|---|
| `which grep` | Prints the full path of the `grep` executable that would actually run |
| `whereis grep` | Prints the executable path, **plus** any related man page and source locations it knows about |

🎯 **On the job:** `which python3` is a classic first move when a script mysteriously uses the "wrong" version of a tool — it tells you exactly which installed copy is winning on your `PATH`.

---

## Practical Examples

### Example 1 — Viewing a small config file with `cat`

```bash
cat -n /etc/hostname
```

Expected output:
```
     1	prod-web-01
```

Line-by-line:
- `cat -n` prints the file's content with a line number prepended — useful even on a one-line file, and essential once you start referencing "line 42 of the config" out loud to a teammate.

### Example 2 — Browsing a bigger file with `less`

```bash
less /var/log/dpkg.log
```

Once inside `less`:
- Press `Space` to page forward through the install/upgrade history.
- Type `/python` and press Enter to jump to the next line mentioning "python."
- Press `n` to jump to the *next* match, `N` to jump back to the *previous* one.
- Press `q` to quit back to your shell prompt.

💡 **Tip:** Nothing you do inside `less` changes the file — it's a read-only viewer, which makes it completely safe to explore with.

### Example 3 — `head` and `tail` on a log file

```bash
head -n 5 /var/log/syslog
```

Expected output (abbreviated, your timestamps/content will differ):
```
Jul 28 08:00:01 prod-web-01 CRON[1021]: (root) CMD (run-parts /etc/cron.hourly)
Jul 28 08:01:12 prod-web-01 systemd[1]: Starting Daily apt...
Jul 28 08:02:03 prod-web-01 kernel: [12345.678] eth0: link up
Jul 28 08:05:44 prod-web-01 sshd[2044]: Accepted publickey for weki
Jul 28 08:10:00 prod-web-01 CRON[1099]: (root) CMD (run-parts /etc/cron.hourly)
```

```bash
tail -n 5 /var/log/syslog
```

Expected output (abbreviated):
```
Jul 28 11:58:02 prod-web-01 myapp[3391]: INFO request completed in 42ms
Jul 28 11:58:10 prod-web-01 myapp[3391]: ERROR database connection timeout
Jul 28 11:58:11 prod-web-01 myapp[3391]: WARN retrying database connection
Jul 28 11:58:12 prod-web-01 myapp[3391]: INFO database connection restored
Jul 28 11:58:15 prod-web-01 myapp[3391]: INFO request completed in 88ms
```

Line-by-line:
- `head -n 5` shows the oldest 5 lines currently in the file — good for confirming the file's format or where logging started.
- `tail -n 5` shows the newest 5 lines — the freshest events, which is almost always what you care about when something just broke.

### Example 4 — 🎯 `tail -f`: watching a log live during an incident

This is the real-world scenario from the Module Goal. Imagine you just deployed a fix and want to watch it take effect in real time:

```bash
tail -f /var/log/myapp/app.log
```

Expected behavior (this command does **not** exit on its own):
```
Jul 28 12:01:00 prod-web-01 myapp[4410]: INFO server started, listening on :8080
Jul 28 12:01:32 prod-web-01 myapp[4410]: INFO GET /checkout 200 OK
Jul 28 12:01:45 prod-web-01 myapp[4410]: ERROR payment gateway timeout, order #88231
Jul 28 12:01:46 prod-web-01 myapp[4410]: INFO retry succeeded, order #88231
```

- New lines appear the instant the application writes them — you're watching the log grow live, in real time, while it happens.
- Press `Ctrl+C` to stop following and return to your prompt.

✅ **Best Practice:** Open `tail -f` in one terminal window, then reproduce the issue (click the button, run the request, trigger the job) in another. Watching cause and effect side by side is far faster than repeatedly re-running `cat`.

### Example 5 — Counting with `wc`

```bash
wc -l /var/log/syslog
```

Expected output:
```
1284 /var/log/syslog
```

```bash
wc -l -w -c /var/log/syslog
```

Expected output:
```
1284 9912 98304 /var/log/syslog
```

Line-by-line:
- `wc -l` alone reports just the line count: 1,284 lines.
- Adding `-w` and `-c` reports lines, words, and characters (bytes) together, in that fixed order.

### Example 6 — Basic `grep` searches

```bash
grep "ERROR" /var/log/myapp/app.log
```

Expected output:
```
Jul 28 11:58:10 prod-web-01 myapp[3391]: ERROR database connection timeout
```

```bash
grep -i "error" /var/log/myapp/app.log
```

Expected output (now also catches lowercase and mixed-case):
```
Jul 28 11:58:10 prod-web-01 myapp[3391]: ERROR database connection timeout
Jul 28 09:14:02 prod-web-01 myapp[3391]: Error: retry limit exceeded
```

```bash
grep -v "INFO" /var/log/myapp/app.log
```

Expected output (every line that is NOT an INFO line):
```
Jul 28 11:58:10 prod-web-01 myapp[3391]: ERROR database connection timeout
Jul 28 11:58:11 prod-web-01 myapp[3391]: WARN retrying database connection
```

Line-by-line:
- Plain `grep "ERROR"` is case-sensitive, so it misses a line that starts with `Error:` instead of `ERROR`.
- `grep -i` catches both, ignoring case entirely.
- `grep -v "INFO"` flips the logic — show me everything *except* the noisy informational lines, which is often the fastest way to cut clutter out of a busy log.

### Example 7 — 🎯 A realistic multi-command pipe: "how many errors happened today?"

```bash
grep -i "error" /var/log/myapp/app.log | wc -l
```

Expected output:
```
2
```

Now let's find out *which* order numbers were involved, and make sure we don't count the same order twice:

```bash
grep -i "error" /var/log/myapp/app.log | grep -o "order #[0-9]*" | sort | uniq
```

Expected output:
```
order #88231
```

Line-by-line:
- The first `grep -i "error"` filters the log down to just error lines.
- The output of that is piped into a second `grep -o "order #[0-9]*"` (`-o` prints only the matched text itself, not the whole line — a small preview of `grep`'s regex power that Module 9 covers properly).
- `sort` puts the results in a predictable order so identical entries end up next to each other.
- `uniq` then removes consecutive duplicates, leaving one entry per distinct order number.

🎯 **On the job:** This exact pattern — filter with `grep`, extract with a second `grep`, then `sort | uniq` — is one of the most common "quick and dirty data analysis" recipes on a Linux terminal, long before anyone opens a proper log-analysis tool.

### Example 8 — Recursive search across a whole project

```bash
grep -rn "TODO" ~/projects/demo/
```

Expected output:
```
/home/weki/projects/demo/src/app.py:14:# TODO: handle empty input
/home/weki/projects/demo/README.md:8:<!-- TODO: add screenshots -->
```

Line-by-line:
- `-r` searches every file under `~/projects/demo/`, recursing into subdirectories.
- `-n` prefixes each match with its line number, so you can jump straight to it in an editor.

### Example 9 — Finding files with `find`

```bash
find ~/projects -name "*.log"
```

Expected output:
```
/home/weki/projects/demo/logs/app.log
/home/weki/projects/demo/logs/error.log
```

```bash
find ~/projects -type d
```

Expected output:
```
/home/weki/projects
/home/weki/projects/demo
/home/weki/projects/demo/logs
/home/weki/projects/demo/src
```

```bash
find ~/projects -type f -mtime -1
```

Expected output (files modified in roughly the last 24 hours):
```
/home/weki/projects/demo/logs/app.log
```

Line-by-line:
- `-name "*.log"` matches by filename pattern only, regardless of type.
- `-type d` restricts results to directories only.
- `-type f -mtime -1` combines two filters at once — only regular files, and only ones modified within the last day — this is exactly how you'd hunt for "what changed recently" on a server.

### Example 10 — A safe, introductory `-exec`

```bash
find ~/projects -name "*.tmp"
```

Expected output:
```
/home/weki/projects/demo/build/cache.tmp
```

Only after confirming that list looks right do we act on it:

```bash
find ~/projects -name "*.tmp" -exec rm {} \;
```

This deletes every `.tmp` file found, one at a time, substituting each match into `{}`.

✅ **Best Practice:** Always run the plain `find` command first (no `-exec`) to see exactly what will be affected, before adding the destructive action.

### Example 11 — `locate` vs `find`

```bash
locate app.log
```

Expected output (assuming the index is up to date):
```
/home/weki/projects/demo/logs/app.log
```

```bash
sudo updatedb
locate newly-created-file.txt
```

Line-by-line:
- `locate` answers instantly from its pre-built index, rather than scanning the disk live.
- If a file was just created and doesn't show up yet, `sudo updatedb` forces an immediate index refresh before searching again.

### Example 12 — Finding an executable

```bash
which python3
```

Expected output:
```
/usr/bin/python3
```

```bash
whereis python3
```

Expected output:
```
python3: /usr/bin/python3 /usr/lib/python3 /usr/share/man/man1/python3.1.gz
```

Line-by-line:
- `which` tells you exactly which binary will run when you type `python3` — critical when multiple versions might be installed.
- `whereis` gives a broader picture: the binary, related library locations, and the man page, all at once.

---

## Common Pitfalls & Best Practices

- **Running `cat` on a huge file.** It floods your terminal with thousands of lines instantly. Reach for `less` (to browse) or `head`/`tail` (to peek) instead once a file is more than a screen or two long.
- **Forgetting to quote `grep` patterns with spaces.** `grep out of memory file.txt` will not do what you want — always write `grep "out of memory" file.txt`.
- **Forgetting to quote `find -name` patterns.** `find . -name *.txt` (unquoted) lets the shell expand `*.txt` against your current directory *before* `find` runs, which can silently produce wrong or empty results. Always quote it: `find . -name "*.txt"`.
- **Confusing `find -name` and `-iname`.** `-name` is case-sensitive; `-iname` is not. A search for `-name "*.TXT"` will silently miss `notes.txt` — reach for `-iname` whenever case shouldn't matter.
- **Assuming `locate` sees files created seconds ago.** Its index is typically refreshed once a day. If a brand-new file isn't showing up, that's expected — run `sudo updatedb` or just use `find` instead.
- **Running `-exec rm {} \;` before checking the plain `find` output first.** Always preview the match list without `-exec`, confirm it's exactly right, and only then bolt on the destructive action.
- **Leaving `tail -f` running forever by accident.** It's designed to never exit on its own — remember `Ctrl+C` to stop it and get your prompt back.
- **Expecting `uniq` to remove all duplicates on its own.** It only removes *consecutive* duplicate lines — always `sort` first if the duplicates aren't already adjacent.
- **Forgetting `grep`'s exit code matters in scripts.** `grep` isn't just for printing matches — it also signals success/failure through its exit status, which matters once you start writing conditional logic in scripts (covered properly in a later module).

---

## Hands-on Exercise

**Task:**

You're investigating a flaky service. Build a small log file and use this module's tools to answer real questions about it.

1. Create a file called `service.log` in a `~/logs` directory with the following 10 lines (use any method you like — `cat` with a heredoc, or your own trick from earlier modules):
   ```
   2026-07-28 09:00:01 INFO service started
   2026-07-28 09:01:15 INFO health check ok
   2026-07-28 09:02:40 WARN slow response 900ms
   2026-07-28 09:03:12 ERROR failed to connect to cache
   2026-07-28 09:03:13 INFO retry attempt 1
   2026-07-28 09:03:14 INFO retry succeeded
   2026-07-28 09:05:00 INFO health check ok
   2026-07-28 09:07:22 ERROR failed to connect to cache
   2026-07-28 09:07:23 INFO retry succeeded
   2026-07-28 09:10:00 INFO health check ok
   ```
2. Show only the first 3 lines, then only the last 3 lines.
3. Count how many total lines are in the file.
4. Count how many lines contain the word `ERROR`.
5. Print only the lines that are **not** `INFO` lines.
6. Search recursively under `~/logs` for the word `cache`, showing line numbers.
7. Use `find` to locate every `.log` file under `~` modified in roughly the last day.

Try this yourself before reading the solution.

### Solution

```bash
# 1. Build the file
mkdir -p ~/logs
cat > ~/logs/service.log << 'EOF'
2026-07-28 09:00:01 INFO service started
2026-07-28 09:01:15 INFO health check ok
2026-07-28 09:02:40 WARN slow response 900ms
2026-07-28 09:03:12 ERROR failed to connect to cache
2026-07-28 09:03:13 INFO retry attempt 1
2026-07-28 09:03:14 INFO retry succeeded
2026-07-28 09:05:00 INFO health check ok
2026-07-28 09:07:22 ERROR failed to connect to cache
2026-07-28 09:07:23 INFO retry succeeded
2026-07-28 09:10:00 INFO health check ok
EOF
```

💡 That `cat > file << 'EOF' ... EOF` block is called a **heredoc** — a way of feeding multiple lines of text into a command as if you'd typed them at a prompt. You'll formalize this technique in a later scripting module; for now, just use it as shown.

```bash
# 2. First 3, then last 3 lines
head -n 3 ~/logs/service.log
```
Expected output:
```
2026-07-28 09:00:01 INFO service started
2026-07-28 09:01:15 INFO health check ok
2026-07-28 09:02:40 WARN slow response 900ms
```

```bash
tail -n 3 ~/logs/service.log
```
Expected output:
```
2026-07-28 09:07:22 ERROR failed to connect to cache
2026-07-28 09:07:23 INFO retry succeeded
2026-07-28 09:10:00 INFO health check ok
```

```bash
# 3. Total line count
wc -l ~/logs/service.log
```
Expected output:
```
10 /home/weki/logs/service.log
```

```bash
# 4. Count ERROR lines
grep -c "ERROR" ~/logs/service.log
```
Expected output:
```
2
```

```bash
# 5. Everything that's NOT an INFO line
grep -v "INFO" ~/logs/service.log
```
Expected output:
```
2026-07-28 09:02:40 WARN slow response 900ms
2026-07-28 09:03:12 ERROR failed to connect to cache
2026-07-28 09:07:22 ERROR failed to connect to cache
```

```bash
# 6. Recursive search with line numbers
grep -rn "cache" ~/logs/
```
Expected output:
```
/home/weki/logs/service.log:4:2026-07-28 09:03:12 ERROR failed to connect to cache
/home/weki/logs/service.log:8:2026-07-28 09:07:22 ERROR failed to connect to cache
```

```bash
# 7. Find recently modified .log files
find ~ -name "*.log" -mtime -1
```
Expected output:
```
/home/weki/logs/service.log
```

Explanation: I used `head`/`tail` to sample the edges of the file, `wc -l` to get an exact total, `grep -c` for a fast count without reading every line myself, `grep -v` to strip out the noisy `INFO` lines and focus on what matters, `grep -rn` to search the whole `~/logs` directory (not just this one file) while keeping line numbers for reference, and finally `find -mtime -1` to confirm the file counts as "recently modified" by metadata alone, without opening it at all — the same distinction between searching *content* versus searching *metadata* from this module's opening analogy.

✅ Exercise complete — you've now built, sampled, counted, filtered, and located a real log file using nothing but this module's toolkit.

---

## ✅ Module Completion Checklist

- [ ] I can view file contents with `cat`, `cat -n`, `less`, and (briefly) `more`, and navigate confidently inside `less`
- [ ] I can peek at the start or end of a file with `head` and `tail`, including `head -n` / `tail -n`
- [ ] I can watch a file grow live with `tail -f`
- [ ] I can count lines, words, and characters in a file with `wc`, `wc -l`, `wc -w`, and `wc -c`
- [ ] I can search for text inside files with `grep`, including `-i`, `-v`, `-r`/`-R`, `-n`, `-c`, `-l`, and `-w`
- [ ] I can combine commands with pipes to answer real troubleshooting questions in one line
- [ ] I can locate files by name, type, size, or modification time with `find`, including a first look at `-exec`
- [ ] I can find files faster with `locate` (and know when it's out of date), and locate executables with `which` and `whereis`

## Next Step

Continue to [Module 4: Permissions, Users & Ownership](../module04-permissions-users/)
