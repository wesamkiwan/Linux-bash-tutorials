# Module 16: Production Scripting & Security Hardening 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 2.5 hours
**Prerequisites:** Modules 1-14 (Shell Fundamentals through Error Handling, Traps & Debugging). This module assumes you're already comfortable with `set -euo pipefail`, `trap`, and `die()` from Module 14, and with permissions/`chmod`/`chown`/`sudo` from Module 4 — both are used directly and extensively here.

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Write a script from a standard production template — shebang, header comment, strict mode, cleanup trap, logging setup, argument parsing, functions, and a `main "$@"` call at the bottom
- [ ] Validate and sanitize every piece of user input *before* using it — checking that a path exists, that a value is genuinely numeric before doing arithmetic on it, and never trusting an argument just because it was supplied
- [ ] Explain exactly why unquoted variables and `eval` create command-injection vulnerabilities, and rewrite vulnerable code to use arrays and proper quoting instead
- [ ] Explain why passing secrets as command-line arguments is dangerous — including a working demonstration of reading another user's "secret" argument via `ps aux` — and use environment variables, stdin, or restricted-permission files instead
- [ ] Keep secrets out of `.bash_history` and log files
- [ ] Apply the principle of least privilege to scripts: avoiding unnecessary `sudo`, dropping privileges where possible, and locking down file permissions on scripts/configs that carry sensitive logic (`chmod 700`/`600`, tying back to Module 4)
- [ ] Build a logging setup that writes timestamped entries to a logfile and, where appropriate, to syslog via `logger` — and explain how `logrotate` keeps that logfile from growing forever
- [ ] Design a script to be idempotent — safe to run twice, or twenty times, without causing duplicate work or corrupted state
- [ ] Take a genuinely insecure script and harden it against injection, secret leakage, and repeat-run damage, using nothing but the techniques in this module

---

## Module Goal

This module is about the gap between a script that *works* and a script that's *safe to run unattended, on someone else's infrastructure, against real customer data.*

🎯 **On the job:** Picture a script you wrote on your own laptop. It takes a customer ID and a date, pulls some records, and emails a report. It works great — you tested it a dozen times, it does exactly what you want, and your teammate is impressed. Then it gets "promoted": it starts running as a scheduled job (Module 15) on a shared server, triggered automatically by a ticket system that passes it whatever the customer typed into a web form. Now the customer ID isn't something you typed carefully — it's arbitrary text from a stranger on the internet, and if that text happens to contain `; rm -rf /var/data`, and your script builds a shell command by gluing strings together, you don't have a report generator anymore — you have a remote-command-execution tool with someone else's credentials. Meanwhile, the API key your script needs was hardcoded at the top of the file so you wouldn't forget it, and last month a junior engineer with read access to that file quietly noticed it, and now it's floating around in a chat log. And because nobody thought about what happens if the job runs twice — say, a scheduler hiccup fires it back-to-back — it emailed the customer the same report three times before anyone noticed. None of these things were bugs in the "the logic is wrong" sense. The script did exactly what it was written to do. It just wasn't written to survive contact with production. This module is about making sure yours is.

---

## Core Concepts

### 1. Production-readiness as a mindset, not a checklist

A script is "production-ready" when it can survive three things a laptop script never has to face: **input it didn't expect**, **an environment it doesn't fully control**, and **being run more than once, by more than one person, possibly at 3 a.m. with nobody watching.** Everything in this module is really an answer to one of those three pressures.

💡 **Analogy — hardening a script is like hardening a house.** A house you live in alone, with the doors unlocked, is perfectly comfortable — right up until someone you didn't invite walks in. Hardening a house means: putting **locks on the doors** (file permissions — Concept 6, and Module 4's `chmod`), **not leaving a spare key under the doormat** (secrets — Concept 4), and **not letting a stranger who rings the doorbell rearrange your furniture just because they asked nicely** (input validation and injection — Concepts 2 and 3). None of these individually makes the house "hacker-proof" — a determined enough attacker with enough time can still get through a good lock — but a house with all three is a fundamentally different proposition than one with none, and shipping a production script without them today is the equivalent of leaving the front door open on a house full of other people's belongings.

### 2. Never trust input — validate before you use it

**Input validation** means checking that a piece of data is what you expect *before* you act on it, rather than assuming it's fine because it arrived. Every argument, environment variable, file, or piece of user-supplied text a script touches should be treated as **guilty until proven innocent**.

Two checks come up constantly:

- **Does this path actually exist, and is it the kind of thing I expect (file vs. directory)?** Test with `[ -f "$path" ]` or `[ -d "$path" ]` (Module 7's test operators) before reading, writing, or deleting anything at that path.
- **Is this string actually a number, before I do arithmetic on it?** Bash's `$(( ))` arithmetic and comparisons like `-gt`/`-lt` (Module 7) will produce confusing errors — or in some cases just silently misbehave — if fed something that isn't a valid integer. Check first with a pattern match: `[[ "$value" =~ ^[0-9]+$ ]]`.

⚠️ **Warning:** "The script ran without an error" is not the same as "the input was valid." A script that receives a garbage argument and just quietly does the wrong thing with it — instead of stopping and saying so — is arguably worse than one that crashes loudly, because nobody notices until the damage is already done.

### 3. Command injection — why unquoted input can run arbitrary commands

**Command injection** is what happens when a script builds a command out of untrusted text and the shell re-interprets part of that text as *shell syntax* rather than as plain data. This is the single most dangerous class of bug covered in this module, because the impact isn't "wrong output" — it's "an attacker (or an unlucky user) ran whatever command they wanted, as whatever user your script runs as."

The root cause is almost always one of two things:

- **Unquoted variable expansion**, which lets the shell perform *word splitting* (breaking one variable into multiple words on whitespace) and *globbing* (expanding `*`/`?` into filenames) on data that was never supposed to be split or expanded — and, in the worst cases, lets shell metacharacters like `;`, `&&`, `|`, or backticks inside that data get interpreted as actual command separators instead of literal text.
- **`eval`**, a builtin that takes a string and runs it *as if it had been typed directly into the shell* — meaning any shell syntax embedded in that string, intentional or not, executes with full shell privileges.

Both are covered with full working examples in Detailed Explanations below, because seeing the actual exploit is the only way this really sinks in.

✅ **Best Practice:** The fix for almost every injection bug is the same two habits: **quote every variable expansion** (`"$var"`, never bare `$var`), and **build dynamic commands with arrays instead of concatenated strings** (Concept 3 continued in Detailed Explanations). Get in the habit of typing the quotes as you type the variable name — don't write it unquoted and plan to "add quotes later."

### 4. Secrets don't belong in your script, your arguments, or your history

A **secret** is any piece of data whose exposure causes harm on its own — passwords, API keys, tokens, private keys, database connection strings with credentials embedded. Three specific mistakes come up constantly:

- **Hardcoding a secret directly in the script.** Anyone who can read the file — including, eventually, anyone who can read a backup of it, a Git history containing it, or a chat log where someone pasted "here's the script" — now has the secret, permanently, with no way to revoke just their access without also invalidating everyone else's.
- **Passing a secret as a command-line argument.** This is more dangerous than most people realize, and Concept 5 below is dedicated entirely to demonstrating why: on a shared Linux system, **other logged-in users can see the full command line of any process**, secret included, using nothing more than `ps aux`.
- **Letting a secret land in `.bash_history` or a log file.** If you type a command containing a raw password directly at the interactive prompt, Bash saves that entire line — password and all — into `~/.bash_history` by default, readable by anyone with access to that file. The same risk applies to any script that `echo`s or logs a variable containing a secret "just for debugging."

✅ **Best Practice:** Prefer, in roughly this order: an **environment variable** set by a secrets manager or a restricted-permission file the script sources at startup; a value **read from stdin** so it never appears in `ps` output or shell history at all; or a **file with `chmod 600` permissions** that only the script's own user can read. Never a CLI argument, never a hardcoded literal.

### 5. Command-line arguments are not private — the `ps aux` problem

This deserves its own concept because it surprises almost everyone the first time they see it. On a standard Linux system, `ps aux` (Module 10) shows the full command line — including every argument — of every running process on the machine, to every user who can run `ps`. That includes arguments you might have assumed were private just because you didn't `echo` them anywhere.

```bash
./deploy.sh --db-password "SuperSecret123"
```

The instant that command runs, anyone else logged into the same machine can run:

```bash
ps aux | grep deploy.sh
```

...and see `SuperSecret123` sitting right there in the output, in plain text, for as long as the process is alive (and sometimes it lingers briefly in process-accounting logs even after). This isn't a bug or a misconfiguration — it's how process listing has always worked on Unix-like systems, and it's exactly why "just pass it as an argument, it's quick" is one of the most common — and most avoidable — security mistakes in production scripting. Concept 4's ordering (env var / stdin / restricted file, never a CLI arg) exists specifically because of this.

### 6. Least privilege — running with only the access you actually need

**The principle of least privilege** means a process should have the *minimum* permissions required to do its job, and nothing more. In scripting terms, that shows up in two places:

- **Don't run the whole script as root "just in case."** If only one line in a fifty-line script genuinely needs elevated privileges (say, writing to `/etc/`), wrap **only that line** with `sudo`, rather than requiring the entire script to run as root. A script that only ever needs `sudo` for one specific command shouldn't have blanket root access to every other line in it, where a bug or an injection vulnerability (Concept 3) would then have full-root blast radius instead of a narrow one.
- **Lock down permissions on the script itself and any config/secrets file it reads**, using `chmod 700` (owner: read/write/execute; nobody else: anything) on scripts containing sensitive logic, and `chmod 600` (owner: read/write; nobody else: anything, no execute) on files holding secrets — directly applying Module 4's permission model to the specific case of "this file must not be even *readable* by other users on the box."

🎯 **On the job:** A shared build server with a dozen engineers on it is a common place this bites people. If your deployment script is world-readable and hardcodes a production database password, every one of those engineers — not just the ones who should have production access — can read that password with a plain `cat`. `chmod 600` on the file holding it (and better yet, not hardcoding it there at all — see Concept 4) closes that door entirely.

### 7. Logging — leaving a trail someone can actually use later

A production script should **write down what it did**, with **timestamps**, so that six weeks from now, when something's gone wrong at 2 a.m., there's a record to look at instead of a shrug. Two complementary approaches:

- **A dedicated logfile**, where every meaningful step writes a timestamped line — typically via a small `log()` helper function that prefixes every message with the current date/time.
- **`logger`**, a command that sends a message to the system's central logging service (**syslog**), rather than (or in addition to) a private logfile. This is the standard way for a script to show up alongside every other system service's logs, viewable with `journalctl` on modern systemd-based systems (Module 15 covered systemd timers; `journalctl` is how you'd read what they logged).

Neither of these is useful forever unless something eventually stops the logfile from growing without bound — that's **`logrotate`**, a standard Ubuntu/Debian utility that automatically compresses, archives, and eventually deletes old log files on a schedule you configure, so a script that logs diligently for years doesn't quietly fill up the disk. You don't need to become a `logrotate` expert for this module — just know it exists, that it's usually already installed and managing `/var/log/*` on Ubuntu, and that any custom logfile your script writes to should eventually be handed off to it (a short config snippet in `/etc/logrotate.d/`) rather than left to grow forever.

### 8. Idempotency — safe to run twice

An operation is **idempotent** if running it once and running it five times produce the *same end result* — the fifth run doesn't pile on duplicate effects on top of the first. This matters enormously for production scripts because, sooner or later, something *will* cause a script to run more than once when you only meant it to run once: a scheduler hiccup (Module 15), a nervous engineer re-triggering a deploy that looked like it hung, a retry after a flaky network blip.

Three habits make a script idempotent almost for free:

- **Check before you create.** Instead of assuming a directory or file doesn't exist yet, check first (`[ -d "$dir" ] || mkdir "$dir"`), or use a tool flag that already makes creation idempotent by design.
- **`mkdir -p` instead of `mkdir`.** `mkdir -p` creates a directory (and any missing parent directories) only if it doesn't already exist, and simply succeeds silently if it does — versus plain `mkdir`, which errors out the second time with "File exists," which under `set -e` (Module 14) would kill the whole script on a harmless re-run.
- **Check whether a service is already running before starting it**, rather than blindly issuing a start command every time — starting an already-running service is sometimes harmless, but re-running "download and install," "append a config line," or "add a user" without checking first can leave duplicated data, duplicated config lines, or an error that halts an otherwise-fine re-run.

💡 **Tip:** A good mental test for idempotency: "If a cron job fired this script twice in a row by accident, would anything be different, worse, or duplicated?" If the honest answer is yes, it isn't idempotent yet.

---

## Detailed Explanations

### Command injection, shown end to end

Here's a small script that looks completely reasonable at first glance — it searches a directory for files matching a pattern the caller supplies:

```bash
#!/bin/bash
# VULNERABLE — do not use
search_term="$1"
target_dir="$2"

eval "grep -r $search_term $target_dir"
```

Called normally, this behaves exactly as expected:

```bash
./search.sh "TODO" /home/alice/project
```

But watch what happens with a specially-crafted `search_term`:

```bash
./search.sh "TODO; rm -rf /home/alice/project" /home/alice/project
```

Because the script uses `eval` on a string built by directly gluing `$search_term` into it, Bash doesn't see "the literal text `TODO; rm -rf /home/alice/project`" — it sees **two separate commands separated by `;`**: `grep -r TODO ...` followed immediately by `rm -rf /home/alice/project`. The semicolon, which the caller controlled completely, was interpreted as a command separator, not as part of a search term. Anyone who can influence `$1` — a web form, a ticket field, another script calling this one — can run *any command they want*, with whatever privileges `search.sh` itself has.

Even without `eval`, unquoted variables alone are exploitable through word-splitting and globbing:

```bash
#!/bin/bash
# STILL VULNERABLE — no eval, but still broken
filename=$1
cat $filename          # unquoted — word-splitting and globbing both apply here
```

```bash
./read.sh "/etc/passwd /etc/shadow"
```

The unquoted `$filename` gets split on the space into *two* separate arguments to `cat`, so this single call reads two files instead of the one the script author probably imagined — and if `$filename` contained a glob character like `*`, it would expand against real filenames in the current directory instead of being treated as literal text.

The fix, both for the `eval` version and the unquoted version, is the same two changes:

```bash
#!/bin/bash
# FIXED
search_term="$1"
target_dir="$2"

grep -r -- "$search_term" "$target_dir"
```

```bash
#!/bin/bash
# FIXED
filename="$1"
cat -- "$filename"
```

Line-by-line: dropping `eval` entirely means Bash never re-parses the search term as shell syntax — it's passed to `grep` as a single, literal, inert string, semicolons and all, because it's fully quoted. The `--` before the argument tells `grep`/`cat` "everything after this is a positional argument, not a flag" — a small but important habit, since a search term that happens to start with a dash (like `-rf`) could otherwise be misread as an option instead of data. Quoting `"$search_term"` and `"$target_dir"` means no word-splitting and no globbing happens on them at all — the exact same string, semicolon included, is now just inert data as far as the shell is concerned, because `eval`'s re-interpretation step is gone and quoting suppressed the shell's other reinterpretation mechanisms too.

⚠️ **`eval` is almost never actually necessary.** If you find yourself reaching for it, there is very likely a safer, more direct way to express what you're trying to do — building an array of arguments (below) covers the overwhelming majority of cases people reach for `eval` to solve.

### Building dynamic commands safely with arrays

The problem `eval` (badly) tries to solve is usually: "I don't know in advance how many arguments I need to pass, or what they are." Arrays (Module 8) solve this cleanly, with no re-parsing and no injection risk, because each array element stays a single, distinct argument no matter what characters it contains:

```bash
#!/bin/bash
set -euo pipefail

# Build a dynamic rsync command based on optional flags
target_dir="$1"
dry_run="${2:-false}"

rsync_args=(-avh --delete)

if [ "$dry_run" = "true" ]; then
    rsync_args+=(--dry-run)
fi

rsync_args+=("$target_dir/" "/backup/destination/")

# Each array element is passed as exactly one argument, whatever it contains
rsync "${rsync_args[@]}"
```

Line-by-line: `rsync_args=(-avh --delete)` starts an array with two fixed flags. `rsync_args+=(--dry-run)` conditionally appends a third element — note this is appending *one array element*, not concatenating text onto a string. `rsync_args+=("$target_dir/" "/backup/destination/")` appends two more elements, each individually quoted, so even a `$target_dir` containing spaces or special characters is passed to `rsync` as a single, correct argument rather than being split apart. Finally, `"${rsync_args[@]}"` — with the quotes and the `[@]` — expands the array back out as separate, individually-quoted words, exactly preserving argument boundaries no matter what's inside them. This is the array equivalent of `"$@"` from Module 8, and it's the reason arrays, not string concatenation, are the correct tool whenever a command's arguments are built up dynamically.

✅ **Best Practice:** Any time you catch yourself building a command by writing `cmd="$cmd extra stuff"` and later running `$cmd` (or worse, `eval "$cmd"`), stop and rewrite it as an array instead. String concatenation for commands is the on-ramp to almost every injection bug in this section.

### The `ps aux` secrets demo, in full

This is worth actually running yourself once, because reading about it and seeing it are different experiences. Open two terminals on the same machine (or two SSH sessions as the same or different users).

**Terminal 1** — start a long-running process that takes a "secret" as a plain argument:

```bash
sleep 300 --my-fake-password="SuperSecret123"
```

(`sleep` ignores unknown arguments, so this just sleeps for 300 seconds — it's a safe stand-in for "any real script that takes a secret as an argument.")

**Terminal 2** — while that's still running, as *any user on the same machine*:

```bash
ps aux | grep sleep
```

Realistic output:
```
alice     48213  0.0  0.0   8192  1536 pts/1    S+   14:02   0:00 sleep 300 --my-fake-password=SuperSecret123
```

There it is — `SuperSecret123`, in plain text, visible to anyone on the box who can run `ps`, for the entire 300 seconds that process is alive. Nothing exotic was required; `ps aux` is a completely ordinary, unprivileged command every user can run. This is precisely why real production tooling (database CLIs, cloud CLIs, deployment scripts) increasingly refuses to accept passwords as plain arguments at all, and instead prompts, reads an environment variable, or reads from a file — exactly the alternatives in Concept 4 and demonstrated in Practical Examples below.

---

## Practical Examples

### Example 1 — The production script template

This is the skeleton to start every production script from. Save it, then fill in the middle.

```bash
#!/bin/bash
#
# deploy-report.sh — generates a nightly customer usage report and emails it.
# Usage: deploy-report.sh --customer-id <id> --output-dir <dir>
#
# Exit codes:
#   1 - invalid/missing arguments
#   2 - required dependency missing
#   3 - report generation failed
#
set -euo pipefail

# ---- Logging setup -------------------------------------------------------
LOG_FILE="/var/log/deploy-report.log"
log() {
    printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1" "$2" | tee -a "$LOG_FILE"
}

# ---- die() — consistent fatal errors (Module 14) -------------------------
die() {
    log "ERROR" "$1"
    exit "${2:-1}"
}

# ---- Cleanup trap ---------------------------------------------------------
WORKDIR=""
cleanup() {
    [ -n "$WORKDIR" ] && rm -rf "$WORKDIR"
}
trap cleanup EXIT

# ---- Argument parsing ------------------------------------------------------
CUSTOMER_ID=""
OUTPUT_DIR=""

usage() {
    echo "Usage: $0 --customer-id <id> --output-dir <dir>" >&2
    exit 1
}

parse_args() {
    while [ $# -gt 0 ]; do
        case "$1" in
            --customer-id)
                CUSTOMER_ID="${2:-}"
                shift 2
                ;;
            --output-dir)
                OUTPUT_DIR="${2:-}"
                shift 2
                ;;
            -h|--help)
                usage
                ;;
            *)
                die "unknown argument: $1"
                ;;
        esac
    done
}

# ---- Validation -------------------------------------------------------------
validate_inputs() {
    [[ "$CUSTOMER_ID" =~ ^[0-9]+$ ]] || die "customer-id must be numeric, got: $CUSTOMER_ID"
    [ -d "$OUTPUT_DIR" ] || die "output directory does not exist: $OUTPUT_DIR"
}

# ---- Main logic, in functions ------------------------------------------------
generate_report() {
    WORKDIR=$(mktemp -d)
    log "INFO" "Generating report for customer $CUSTOMER_ID in $WORKDIR"
    echo "report data for customer $CUSTOMER_ID" > "$WORKDIR/report.txt"
    cp "$WORKDIR/report.txt" "$OUTPUT_DIR/report-$CUSTOMER_ID.txt"
    log "INFO" "Report written to $OUTPUT_DIR/report-$CUSTOMER_ID.txt"
}

main() {
    parse_args "$@"
    validate_inputs
    generate_report
    log "INFO" "Done."
}

main "$@"
```

Line-by-line: the header comment names the script, states its usage, and documents every exit code — a habit carried over directly from Module 14's `die()` convention. `set -euo pipefail` goes right after the shebang, per Module 14. The `log()` function timestamps every message and writes it both to the terminal and to `$LOG_FILE` via `tee -a` (Module 5's `tee`, appending). `trap cleanup EXIT` is registered early, and `cleanup()` safely no-ops if `WORKDIR` was never set (Concept 8's idempotency habit applied to cleanup itself). `parse_args` is a `while`/`case` loop (Module 7) that walks `"$@"` and fills in named variables — no positional-argument guessing required, and it's easy to add new flags later. `validate_inputs` runs *before* `generate_report`, applying Concept 2 — nothing destructive or expensive happens until both arguments are confirmed sane. All real logic lives in named functions, and the very last line, `main "$@"`, is what actually kicks everything off — this ordering means every function is fully defined before it's ever called, and the script reads top-to-bottom as "here's what I can do" followed by one line that says "now actually do it."

✅ **Best Practice:** Keep `main "$@"` as the literal last line of the file, every time. It's a small convention, but it means anyone reading an unfamiliar script can jump straight to the bottom to see the actual entry point before reading how any of the pieces work.

### Example 2 — Input validation in isolation

```bash
#!/bin/bash
set -euo pipefail

validate_path() {
    local path="$1"
    [ -e "$path" ] || { echo "ERROR: path does not exist: $path" >&2; exit 1; }
    [ -r "$path" ] || { echo "ERROR: path is not readable: $path" >&2; exit 1; }
}

validate_number() {
    local value="$1"
    local label="$2"
    [[ "$value" =~ ^[0-9]+$ ]] || { echo "ERROR: $label must be a positive integer, got: '$value'" >&2; exit 1; }
}

# --- demo calls ---
validate_path "/etc/hostname"
echo "Path OK"

validate_number "42" "retry-count"
echo "Number OK"

validate_number "abc" "retry-count"
echo "This line never runs"
```

Realistic output:
```
Path OK
Number OK
ERROR: retry-count must be a positive integer, got: 'abc'
```

Line-by-line: `validate_path` checks existence (`-e`) and readability (`-r`) separately, since a path can exist but not be readable by the current user (a permissions problem, tying back to Module 4) — reporting the right one of those two saves whoever's debugging it a wasted step. `validate_number` uses `[[ ... =~ ^[0-9]+$ ]]`, an extended regex match (Module 9) anchored at both ends (`^`...`$`), so `"42abc"` or `"  42"` correctly fail too — a naive check without the anchors could be fooled by a number with trailing garbage. The final call, with `"abc"`, prints its error and exits before "This line never runs" ever executes, because `exit 1` inside the function's `||` block actually exits the whole script, not just the function.

⚠️ **Warning:** Never do arithmetic (`$(( $value * 2 ))`) on a value before validating it's numeric. Depending on what garbage ends up in `$value`, arithmetic expansion can throw a confusing "syntax error" at best — or, in some Bash configurations and versions, be tricked into evaluating something you didn't intend. Validate first, always.

### Example 3 — Command injection: vulnerable vs. fixed, side by side

```bash
#!/bin/bash
# VULNERABLE — accepts a filename to archive, builds the command via eval
archive_name="$1"
eval "tar czf backup-$archive_name.tar.gz /data"
```

```bash
./archive.sh "$(whoami).txt; touch /tmp/pwned"
```

Realistic output (the vulnerability triggering):
```
tar: /data: Cannot stat: No such file or directory
```
...and a new empty file silently appears:
```bash
ls /tmp/pwned
```
```
/tmp/pwned
```

The `tar` command failed (there's no `/data` directory on this demo machine) — but notice that's irrelevant to the actual damage: the `touch /tmp/pwned` after the semicolon ran as a completely separate command, injected entirely through `$1`, regardless of whether the `tar` part succeeded or failed.

Fixed version:

```bash
#!/bin/bash
set -euo pipefail
archive_name="$1"
# Reject anything that isn't a plausible simple filename component
[[ "$archive_name" =~ ^[A-Za-z0-9_.-]+$ ]] || { echo "ERROR: invalid archive name: $archive_name" >&2; exit 1; }

tar czf "backup-${archive_name}.tar.gz" /data
```

```bash
./archive.sh "$(whoami).txt; touch /tmp/pwned"
```

Realistic output:
```
ERROR: invalid archive name: alice.txt; touch /tmp/pwned
```

Line-by-line: dropping `eval` means the string is never re-parsed as shell syntax in the first place — that alone closes most of the hole. The regex whitelist (`^[A-Za-z0-9_.-]+$`) is a second, independent layer: even without `eval`, this rejects the semicolon (and any other character outside a deliberately narrow, known-safe set) before it ever reaches `tar`, so a legitimate simple filename passes and anything containing shell metacharacters is refused outright.

✅ **Best Practice:** Whitelisting — defining the narrow set of characters something is *allowed* to contain, and rejecting everything else — is generally safer than blacklisting specific "bad" characters, because it's very easy to forget one dangerous character (backticks, `$()`, `|`, newlines) when trying to list them all, and much harder to accidentally leave a hole in "only letters, digits, underscore, dot, and hyphen are allowed."

### Example 4 — Secrets via environment variable, with the `ps aux` danger demonstrated

**The dangerous version:**

```bash
#!/bin/bash
# VULNERABLE — secret visible in ps aux for the process's whole lifetime
db_password="$1"
mysql -u appuser -p"$db_password" -h db.internal.example.com -e "SELECT 1;"
```

```bash
./query.sh "SuperSecretDbPass1"
```

While that's running (even briefly), from another session:
```bash
ps aux | grep mysql
```
```
alice    51902  0.1  0.2  23456  9876 pts/2    S+   14:10   0:00 mysql -u appuser -pSuperSecretDbPass1 -h db.internal.example.com -e SELECT 1;
```

The password is right there in plain text, exactly as Concept 5 described.

**The fixed version, using an environment variable:**

```bash
#!/bin/bash
set -euo pipefail
# Requires DB_PASSWORD to be set in the environment before running,
# e.g. by a secrets manager, or a restricted-permission file the caller sources.
: "${DB_PASSWORD:?DB_PASSWORD environment variable must be set}"

mysql -u appuser -p"$DB_PASSWORD" -h db.internal.example.com -e "SELECT 1;"
```

```bash
export DB_PASSWORD="SuperSecretDbPass1"
./query.sh
```

Checking `ps aux` while this runs still shows the `mysql` command line — but critically, **the password itself is not in it**, because it was never passed as a literal argument; `mysql` read it from the shell's own expansion of `$DB_PASSWORD` inside the script's process, not from a value visible in the argument list `ps` displays... 

⚠️ Careful here — some tools (including certain `mysql` client versions) still put the *expanded* password into their own argument list, which `ps` would still show, because the vulnerability isn't really about "environment variable vs. CLI argument" in the abstract — it's about **whether the secret ends up as a literal command-line argument to any process at all.** The genuinely safe pattern for a tool like `mysql` is to avoid `-p<password>` entirely and use its own environment-variable or config-file support instead:

```bash
#!/bin/bash
set -euo pipefail
: "${MYSQL_PWD:?MYSQL_PWD environment variable must be set}"
export MYSQL_PWD   # mysql reads this directly; nothing is ever passed as an argument
mysql -u appuser -h db.internal.example.com -e "SELECT 1;"
```

```bash
ps aux | grep mysql
```
```
alice    52011  0.1  0.2  23456  9876 pts/2    S+   14:15   0:00 mysql -u appuser -h db.internal.example.com -e SELECT 1;
```

No password anywhere in that line.

🎯 **On the job:** This is the real lesson, and it's a level deeper than "use environment variables instead of arguments": the goal isn't the mechanism (env var, stdin, file) for its own sake — it's making sure the secret **never becomes a literal argument to any process**, anywhere in the chain, including the ones your script calls. Passing a secret to *your own script* via an env var is a wasted effort if your script then turns around and hands it to `mysql` as a `-p` flag.

### Example 5 — Idempotent script: safe to run twice

```bash
#!/bin/bash
set -euo pipefail

APP_DIR="/opt/myapp"
SERVICE_NAME="myapp"

log() { printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1"; }

ensure_directory() {
    if [ -d "$APP_DIR" ]; then
        log "Directory already exists, skipping: $APP_DIR"
    else
        mkdir -p "$APP_DIR"
        log "Created directory: $APP_DIR"
    fi
}

ensure_config_line() {
    local line="LOG_LEVEL=info"
    local config_file="$APP_DIR/config.env"
    touch "$config_file"
    if grep -qxF "$line" "$config_file"; then
        log "Config line already present, skipping"
    else
        echo "$line" >> "$config_file"
        log "Added config line"
    fi
}

ensure_service_running() {
    if systemctl is-active --quiet "$SERVICE_NAME"; then
        log "$SERVICE_NAME is already running, nothing to do"
    else
        sudo systemctl start "$SERVICE_NAME"
        log "Started $SERVICE_NAME"
    fi
}

main() {
    ensure_directory
    ensure_config_line
    ensure_service_running
}

main "$@"
```

First run, realistic output:
```
2026-07-28 09:00:01 Created directory: /opt/myapp
2026-07-28 09:00:01 Added config line
2026-07-28 09:00:02 Started myapp
```

Second run (moments later), realistic output:
```
2026-07-28 09:00:15 Directory already exists, skipping: /opt/myapp
2026-07-28 09:00:15 Config line already present, skipping
2026-07-28 09:00:15 myapp is already running, nothing to do
```

Line-by-line: `ensure_directory` checks `[ -d "$APP_DIR" ]` before creating anything — using `mkdir -p` here too would have worked on its own, but the explicit check lets us log a clearly different, more informative message for the "already there" case. `ensure_config_line` uses `grep -qxF` — `-x` matches the *whole line* exactly, `-F` treats the search text as a literal string rather than a regex, `-q` suppresses output and just reports found/not-found via exit status — so the config line is appended once, ever, no matter how many times this runs; without the check, every re-run would append a duplicate line. `ensure_service_running` checks `systemctl is-active --quiet` (Module 11/15 territory) before calling `sudo systemctl start` — starting an already-running service is often harmless by itself, but skipping the check here also means the `sudo` call — and its associated privilege escalation, per Concept 6 — only happens when it's actually needed.

✅ **Best Practice:** Notice that `sudo` appears on exactly one line, for exactly the one operation that needs it — not at the top of the script wrapping the whole thing. That's least privilege (Concept 6) in practice, not just in theory.

---

## Common Pitfalls & Best Practices

- **Reaching for `eval` because it's the fastest way to make dynamic commands "just work."** As Detailed Explanations showed, `eval` re-parses a string as shell syntax, which means any shell metacharacter inside untrusted data becomes a command the attacker chose, not the caller's data. Arrays (Example 3's fixed version, and Concept 3) solve nearly every legitimate use case people reach for `eval` to cover.
- **Hardcoding secrets "temporarily" and forgetting to remove them.** Temporary hardcoded secrets have a way of becoming permanent the moment the script is committed to version control, copied to a second server, or pasted into a chat for debugging help. Treat "temporarily hardcoded" as already leaked.
- **Running the whole script as root "just in case something later needs it."** This maximizes the blast radius of every other bug in the script — an injection vulnerability, a typo'd `rm -rf`, an unvalidated path — turning what would have been a limited, recoverable mistake into one with full-system consequences. Elevate only the single command that actually needs it (Concept 6, Example 5).
- **Assuming a script only ever runs once.** Schedulers retry, engineers re-trigger stuck-looking jobs, networks blip mid-run. A script that isn't idempotent (Concept 8) turns a harmless accidental re-run into duplicated emails, duplicated config lines, or a crash on the second attempt.
- **Passing secrets as CLI arguments because "it's just for testing."** Test environments get logged, shared, and screen-recorded too, and the habit of typing secrets as arguments tends to follow people straight into production. Practice the safe pattern (env var / stdin / restricted file) even in throwaway test scripts.
- **Logging a secret "just to debug it."** A `log()` call or a stray `echo "password is: $DB_PASSWORD"` left in during debugging routinely ends up shipped, and log files are frequently readable by more people, and kept for longer, than anyone realizes.
- **Trusting a path or number just because an argument was provided.** "The user gave me *something*" and "the user gave me something *valid*" are different claims — validate explicitly (Concept 2), every time, before using the value for anything consequential.

✅ **Best Practice — the production-promotion mindset:** Before any script moves from "runs on my laptop" to "runs unattended, on shared infrastructure, on real data," ask three questions: *What happens if the input is malicious or malformed? What happens if this runs twice? Who else can read the secrets and permissions this script depends on?* This module exists to make sure your honest answer to all three is "nothing bad happens."

---

## Hands-on Exercise

**Task:** Below is an insecure script that backs up a user-specified directory to a remote host using credentials baked into the file. Harden it completely.

Insecure starting point, `remote-backup.sh`:

```bash
#!/bin/bash
API_KEY="sk_live_abc123fakekeydonotuse"
SRC=$1
mkdir /tmp/backup-staging
cp -r $SRC/* /tmp/backup-staging
eval "curl -X POST -H \"Authorization: Bearer $API_KEY\" -F file=@/tmp/backup-staging https://backup.example.com/upload"
echo "Backup uploaded"
```

Your job:
1. Remove the hardcoded `API_KEY` and require it via an environment variable instead.
2. Fix the command-injection risk (the `eval` call and the unquoted `$SRC`).
3. Make it idempotent — safe to run back-to-back without erroring on the second run.
4. Tighten file permissions on the script itself, since it handles a sensitive API key.

Try hardening it yourself before reading the solution below.

### Solution

**Step 1-3 — hardened script, `remote-backup.sh`:**

```bash
#!/bin/bash
#
# remote-backup.sh — archives a directory and uploads it to the backup service.
# Usage: BACKUP_API_KEY=<key> ./remote-backup.sh <source-dir>
#
# Exit codes:
#   1 - missing/invalid arguments or missing API key
#   2 - source directory does not exist
#
set -euo pipefail

STAGING_DIR="/tmp/backup-staging"

log() { printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1"; }
die() { log "ERROR: $1"; exit "${2:-1}"; }

cleanup() {
    rm -rf "$STAGING_DIR"
}
trap cleanup EXIT

SRC="${1:-}"
[ -n "$SRC" ] || die "usage: $0 <source-dir>" 1
[ -d "$SRC" ] || die "source directory does not exist: $SRC" 2
: "${BACKUP_API_KEY:?BACKUP_API_KEY environment variable must be set}"

# Idempotent: fresh staging dir every run, no leftover state from a prior run
rm -rf "$STAGING_DIR"
mkdir -p "$STAGING_DIR"

log "Copying $SRC into staging..."
cp -r "$SRC"/. "$STAGING_DIR"/

log "Uploading backup..."
curl -sS -X POST \
     -H "Authorization: Bearer ${BACKUP_API_KEY}" \
     -F "file=@${STAGING_DIR}" \
     "https://backup.example.com/upload"

log "Backup uploaded"
```

**Step 4 — tighten permissions:**

```bash
chmod 700 remote-backup.sh
ls -l remote-backup.sh
```

Realistic output:
```
-rwx------ 1 alice alice 1024 Jul 28 09:30 remote-backup.sh
```

**Running it, the safe way:**

```bash
export BACKUP_API_KEY="sk_live_abc123fakekeydonotuse"
./remote-backup.sh /home/alice/project
./remote-backup.sh /home/alice/project   # run again — no error, no stale state
```

Explanation of the changes: the hardcoded `API_KEY` is gone entirely, replaced by `: "${BACKUP_API_KEY:?...}"` (the same fail-fast idiom from Module 14's exercises), so the key exists only in the caller's environment for as long as needed, never inside the file itself, and never as a `curl` command-line argument, since it's interpolated into a quoted `-H` header value rather than typed as a bare CLI flag containing the raw secret. Dropping `eval` and quoting every expansion (`"$SRC"`, `"${STAGING_DIR}"`, `"${BACKUP_API_KEY}"`) closes the injection hole — a `$SRC` like `/tmp; rm -rf /` is now just an invalid, nonexistent directory path that `[ -d "$SRC" ]` rejects before anything runs, rather than shell syntax that gets executed. Idempotency comes from `rm -rf "$STAGING_DIR"` immediately before `mkdir -p "$STAGING_DIR"` — every run starts from a guaranteed-clean staging directory, so a second run never trips over "directory already exists" or uploads stale leftovers from a previous, possibly-failed run. `chmod 700` ensures that even while `BACKUP_API_KEY` is out of the file, the script's own logic — including the URL it uploads to and exactly how it authenticates — isn't readable by anyone but its owner, applying Module 4's permission model directly to Concept 6's least-privilege principle.

✅ Exercise complete — you've removed a hardcoded secret, closed a command-injection hole, made repeat runs safe, and locked the file down to its owner only.

---

## ✅ Module Completion Checklist

- [ ] I can write a script from the standard production template — shebang, header comment, strict mode, cleanup trap, logging setup, argument parsing, functions, and `main "$@"`
- [ ] I can validate and sanitize arguments before using them — checking a path exists and confirming a value is genuinely numeric before doing arithmetic on it
- [ ] I can explain why unquoted variables and `eval` create command-injection vulnerabilities, and rewrite vulnerable code using arrays and proper quoting
- [ ] I can explain why CLI arguments are visible to other users via `ps aux`, and demonstrate it, and I know to prefer environment variables, stdin, or restricted-permission files for secrets instead
- [ ] I know to keep secrets out of `.bash_history` and log files
- [ ] I can apply least privilege — avoiding unnecessary `sudo`, and locking down sensitive scripts/configs with `chmod 700`/`600`
- [ ] I can build a timestamped logging setup, and I understand where `logger`/syslog and `logrotate` fit in
- [ ] I can design a script to be idempotent — safe to run more than once without duplicating work or corrupting state
- [ ] I hardened a genuinely insecure script against injection, secret leakage, and repeat-run damage

## Next Step

Continue to [Module 17: Performance Tuning & Profiling](../module17-performance-tuning/)
