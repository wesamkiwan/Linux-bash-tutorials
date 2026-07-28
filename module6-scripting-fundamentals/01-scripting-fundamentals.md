# Module 6: Bash Scripting Fundamentals 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 3 hours
**Prerequisites:** Modules 1-5 (Shell Fundamentals, Filesystem Navigation, Viewing/Finding Files, Permissions & Users, I/O Redirection & Pipes)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain what a bash script is, how it differs from typing commands interactively, and why scripts matter on the job (repeatability, automation, version control)
- [ ] Write a correct shebang line (`#!/bin/bash` or `#!/usr/bin/env bash`) and explain what it does and why it matters
- [ ] Make a script executable with `chmod +x` and explain the difference between running it as `./script.sh`, `bash script.sh`, and `sh script.sh`
- [ ] Declare and use variables correctly (`name="value"`, `$name`, `${name}`), and explain why quoting variables matters
- [ ] Explain the difference between shell variables and environment variables, and use `export`, `env`, and `printenv`
- [ ] Use command substitution `$(...)` and arithmetic expansion `$((...))` confidently, and explain why `expr` is considered legacy
- [ ] Work with positional parameters (`$0`, `$1`, `$@`, `$*`, `$#`, `shift`) and explain precisely how `"$@"` differs from `"$*"`
- [ ] Read user input with `read`, `read -p`, and `read -s`, and use exit codes (`exit 0`, `exit 1`, `$?`) so a script communicates success or failure to whatever runs it

---

## Module Goal

By the end of this module, you'll be able to take a series of commands you'd normally type by hand — one at a time, from memory, hoping you don't forget a step — and turn them into a single, reliable, repeatable **script**.

🎯 **On the job:** Imagine your team has a deployment checklist: SSH into the server, pull the latest code, check a config file exists, ask the on-call engineer to confirm before restarting the service, log the result, and report success or failure back to a monitoring dashboard. Doing that by hand every single time is slow and error-prone — someone will eventually skip a step at 2 a.m. A script turns that checklist into one command: `./deploy.sh production`. It runs the same way every time, it can be reviewed and improved like any other code, and it can be checked into Git so the whole team shares the exact same process. This module is where you stop being "someone who runs commands" and start being "someone who builds tools other people rely on."

---

## Core Concepts

### 1. What is a script?

A **script** is just a plain text file containing a sequence of shell commands, saved so it can be run as a single unit instead of being typed one line at a time. Anything you can type at the prompt, you can put in a script — the shell reads the file top to bottom and runs each line exactly as if you'd typed it yourself.

💡 **Analogy — recipe card vs. cooking live:** Typing commands interactively is like cooking a dish from memory, one step at a time, in front of a guest — if you forget an ingredient or do a step out of order, the dish suffers and there's no record of what you actually did. A script is a written recipe card: every step is written down in order, in advance, so anyone (including future-you) can follow it exactly, hand it to a colleague, or improve it over time. You cook the same excellent meal every time because the recipe doesn't rely on memory.

### 2. Why scripts matter on the job

- **Repeatability** — a script runs the exact same steps every time, with no steps forgotten or done out of order.
- **Automation** — a script can be triggered by a schedule (`cron`, which you'll meet in a later module), a CI/CD pipeline, or another program, with no human sitting at the keyboard.
- **Version control** — because a script is just a text file, it can live in Git alongside your application code. Changes are reviewed, tracked, and can be rolled back like any other code change.
- **Documentation as code** — a well-written script *is* the documentation for a process. Instead of a wiki page that goes stale, the script itself is always up to date, because it's what actually runs.

### 3. The shebang line

A **shebang** (also written `she-bang`, from "hash-bang" — the `#` and `!` characters) is the very first line of a script, and it must look like this:

```bash
#!/bin/bash
```

This tells the operating system **which interpreter should run this file**. When you execute a script directly (`./script.sh`), the kernel reads that first line and hands the rest of the file to whatever program is named there — in this case, `/bin/bash`, the full path to the Bash interpreter.

An alternative, increasingly common form is:

```bash
#!/usr/bin/env bash
```

This uses the `env` command to *search your `PATH`* for `bash`, rather than hardcoding `/bin/bash`. It's slightly more portable across systems where Bash might live somewhere other than `/bin/bash` (some macOS setups, for example). Both are correct; this course will use `#!/bin/bash` since we're targeting Ubuntu/Debian where it's guaranteed to exist, but you should recognize both forms.

⚠️ **Warning:** The shebang **must** be the very first line of the file — not even a blank line or comment before it. If it's missing, the script will still often run (because your *current* shell will fall back to interpreting it as best it can), but you lose control over exactly which interpreter runs your code — a common source of "works on my machine" bugs.

### 4. Making a script executable and running it

Creating a text file with a shebang doesn't automatically make it runnable — the file also needs the **execute permission** (you covered permissions in depth in Module 4). You grant it with:

```bash
chmod +x script.sh
```

Once it's executable, you can run it three different ways, and the differences matter:

| Way to run it | What actually happens |
|---|---|
| `./script.sh` | The kernel reads the shebang line and uses **exactly the interpreter named there**. Requires execute permission (`chmod +x`). |
| `bash script.sh` | Explicitly tells Bash to run the file, **ignoring the shebang entirely**. Does **not** require execute permission — `bash` just reads the file as input. |
| `sh script.sh` | Runs the file using whatever `sh` points to on this system. On Ubuntu, `/bin/sh` is a symlink to **`dash`**, a smaller, stricter, POSIX-only shell — **not** Bash. Bash-only features (like arrays or `[[ ]]`) will break or behave differently under `sh`. |

🎯 **On the job:** This trips people up constantly. If a script starts with `#!/bin/bash` but a teammate runs it with `sh script.sh`, they may hit confusing errors because `dash` doesn't understand Bash-specific syntax. The safe habit: always run your own scripts with `./script.sh` (respects the shebang) once they're executable, and never assume `sh` and `bash` are interchangeable.

### 5. Comments

A **comment** is a line (or part of a line) the shell ignores completely — it's there purely for humans reading the code. In Bash, anything after a `#` on a line is a comment (except, of course, the shebang's `#!`, which the kernel treats specially only on line one).

```bash
# This whole line is a comment and does nothing
echo "Hello"  # This part after the # is also a comment
```

🎯 **On the job:** Comments explain *why*, not *what* — the code already shows *what* it does. A good comment says "we sleep 5 seconds here because the API needs time to register the new record," not "this sleeps for 5 seconds."

### 6. Variables — declaring and using

A **variable** is a named piece of storage that holds a value (usually text). You declare one like this:

```bash
name="Wesam"
```

⚠️ **Critical rule:** There must be **no spaces** around the `=` sign. `name = "Wesam"` is not an assignment at all — Bash would try to run a command literally called `name` with arguments `=` and `Wesam"`, and fail.

To *use* a variable's value, prefix it with a dollar sign:

```bash
echo $name
echo "${name}"
```

Both `$name` and `${name}` mean the same thing — retrieve the value stored in `name`. The curly-brace form `${name}` exists to remove ambiguity, especially when a variable name butts up against other text:

```bash
name="Wesam"
echo "$name_file.txt"     # Looks for a variable called "name_file" — probably empty!
echo "${name}_file.txt"   # Correctly inserts just $name, then the literal text "_file.txt"
```

💡 **Tip:** When in doubt, use `${name}` — it never hurts, and it prevents an entire class of "why is my variable empty" bugs.

### 7. Why quoting variables matters

This is one of the most important habits in this entire module. Compare:

```bash
file="my report.txt"
ls $file      # unquoted — dangerous
ls "$file"    # quoted — safe
```

Without quotes, Bash performs **word splitting**: it takes the variable's value and splits it into separate words wherever there's whitespace, then hands each word to the command as a *separate* argument. `ls $file` doesn't run `ls "my report.txt"` — it runs `ls "my" "report.txt"`, two separate (and probably nonexistent) filenames. Unquoted variables are also subject to **globbing** — if the value happens to contain `*` or `?`, Bash will try to expand it as a wildcard against real filenames.

✅ **Best Practice:** Always wrap variable references in double quotes — `"$name"` — unless you have a specific, deliberate reason not to (which is rare, and you'll know it when you need it). This single habit prevents the majority of real-world script bugs caused by filenames or input containing spaces.

### 8. Readonly variables and unsetting

- `readonly name="value"` creates a variable that **cannot be reassigned** later in the script — useful for constants you want to protect from accidental modification.
- `unset name` removes a variable entirely, as if it had never been set.

```bash
readonly MAX_RETRIES=3
MAX_RETRIES=5   # error: MAX_RETRIES: readonly variable

unset name
echo "${name:-not set}"   # prints "not set"
```

### 9. Shell variables vs. environment variables

This distinction confuses a lot of beginners, so let's be precise:

- A **shell variable** is only visible inside the current shell (or script). Any command or script it *launches* (a "child process") does **not** automatically see it.
- An **environment variable** is a shell variable that has been **exported** — marked so that every child process the shell launches inherits a copy of it.

```bash
greeting="hello"        # shell variable only
export greeting         # now it's also an environment variable

# or in one step:
export greeting="hello"
```

💡 **Analogy:** A shell variable is a sticky note on your own desk — only you can see it. Exporting it is like photocopying that sticky note and handing a copy to everyone you delegate a task to (every child process). They get their *own copy* — if they change it, your original note is unaffected.

Two commands let you inspect environment variables:

| Command | Shows |
|---|---|
| `env` | Every environment variable currently exported, as `NAME=value`, one per line |
| `printenv` | Same idea; `printenv NAME` prints just one variable's value |

🎯 **On the job:** This is exactly why a script might fail with "command not found" or "API key missing" when run from a cron job or CI pipeline, even though it works fine in your own terminal — your terminal has variables set (in `.bashrc`, etc.) that were never `export`-ed, or that only exist in your *interactive* shell's environment, not the child environment the automated job runs in.

### 10. Command substitution — capturing a command's output

You briefly met **command substitution** in Module 5. It lets you run a command and capture its *output* as a string, storing it in a variable or using it inline:

```bash
today=$(date +%F)
echo "Today is $today"
```

`$(...)` runs the command inside the parentheses, waits for it to finish, and substitutes its standard output right there in place of the whole `$(...)` expression. You'll use this constantly in scripts — for capturing timestamps, hostnames, command results, file contents, and more.

⚠️ **Note:** You'll also see the older backtick syntax `` `date +%F` `` in legacy scripts. It does the same thing but is harder to read and cannot be nested cleanly. Prefer `$(...)` in all new scripts.

### 11. Arithmetic in Bash

Bash can do integer arithmetic natively using **arithmetic expansion**: `$((...))`.

```bash
count=5
count=$((count + 1))
echo "$count"   # 6
```

Two older mechanisms exist and are worth recognizing but not preferring:

- `let count=count+1` — an older built-in for arithmetic assignment. Still works, but `$((...))` is clearer and more consistent with everything else in modern Bash.
- `expr` — a separate **external program** (not a shell built-in) that evaluates expressions, e.g. `count=$(expr $count + 1)`. This is a **legacy** tool from older, more limited shells. It's slower (it launches a whole separate process), fussier about spacing, and has been superseded by `$((...))` in virtually every case. You'll still see it in old scripts — recognize it, but don't write new code with it.

✅ **Best Practice:** Use `$((...))` for all new arithmetic. Reserve `let` and `expr` for reading legacy scripts you didn't write.

### 12. Positional parameters — the arguments passed to your script

When you run a script with arguments, like `./greet.sh Alice Bob`, Bash automatically populates a set of special variables:

| Variable | Holds |
|---|---|
| `$0` | The name of the script itself (as it was invoked) |
| `$1`, `$2`, `$3`, ... | The first, second, third, ... argument passed in |
| `$#` | The **count** of positional arguments passed in |
| `$@` | All positional arguments |
| `$*` | All positional arguments (subtly different when quoted — see below) |

`shift` moves every positional parameter down by one — `$2` becomes `$1`, `$3` becomes `$2`, and so on, while `$#` decreases by one. This is the standard way to loop through an unknown number of arguments one at a time.

### 13. `"$@"` vs `"$*"` — the difference that actually matters

This is a favorite interview question for a reason — get it wrong in a real script and you'll ship a subtle bug. Both `$@` and `$*` expand to "all the positional parameters" — but they behave **completely differently once you quote them**:

- `"$@"` expands to **each positional parameter as its own separate, quoted word** — exactly as they were originally passed in, spaces and all.
- `"$*"` expands to **a single word**: all positional parameters joined together into one string, separated by the first character of the `IFS` variable (a space, by default).

You'll see this demonstrated concretely with real broken vs. fixed output in the Practical Examples section below — it's much clearer to see it run than to read about it in the abstract.

✅ **Best Practice:** Use `"$@"` (always quoted) almost everywhere you want to "pass along all the arguments I received" — for example when one script forwards its arguments to another command. It's the only form of the two that reliably preserves each argument as a separate unit, including any that contain spaces.

### 14. Special variables: `$?`, `$$`, `$!`

| Variable | Holds |
|---|---|
| `$?` | The **exit status** of the last command that finished running |
| `$$` | The **process ID (PID)** of the current shell/script itself |
| `$!` | The PID of the most recently started **background** job (from Module 5's `&`) |

`$?` is one you will check constantly — see Exit Codes below.

### 15. Reading user input

The `read` built-in pauses a script and waits for the user to type something, storing it in a variable:

| Command | Behavior |
|---|---|
| `read name` | Waits for input, stores it in `name` |
| `read -p "Prompt: " name` | Prints `"Prompt: "` first, then waits for input, on the same line |
| `read -s -p "Password: " pass` | Same as above, but **silent** — input is not echoed to the screen, for passwords/secrets |

```bash
read -p "Enter your name: " user_name
echo "Hello, $user_name!"
```

### 16. Exit codes — how a script reports success or failure

Every command and every script, when it finishes, produces an **exit code** (also called exit status) — a number from 0 to 255 that reports how it went. By long-standing Unix convention:

- **`0` means success.**
- **Any non-zero number (1-255) means some kind of failure** (the specific number can carry meaning you define, e.g. `1` for a generic error, `2` for "bad arguments").

You control your own script's exit code with `exit`:

```bash
exit 0   # success
exit 1   # failure
```

If you don't call `exit` explicitly, a script's exit code is simply whatever the **last command it ran** returned. You can always check the exit code of the most recently finished command with `$?`:

```bash
grep "ERROR" logfile.txt
echo "$?"   # 0 if found, 1 if not found, 2+ if logfile.txt doesn't exist
```

🎯 **On the job:** This convention is the backbone of automation. A CI/CD pipeline, a cron job, or another script that calls yours doesn't read your output to figure out if it worked — it checks the exit code. `if my-deploy-script.sh; then echo "deployed"; fi` relies entirely on your script returning `0` for success and non-zero for failure. Get this wrong (e.g., always exiting `0` even after an error) and automated systems will confidently report a broken deployment as "successful."

### 17. Basic script structure and style conventions

A clean, professional Bash script conventionally starts like this:

```bash
#!/bin/bash
#
# deploy.sh — deploys the latest build to the given environment
# Usage: ./deploy.sh <environment>

# (script body starts here)
```

That's: a shebang, then a short comment block describing what the script does and how to use it, then the actual logic.

💡 **Tip — a preview of what's coming:** You will very often see production scripts start with a line like `set -euo pipefail` right after the shebang. This tells Bash to stop immediately on errors, treat unset variables as errors, and catch failures inside pipelines — it makes scripts dramatically safer. We are **not** diving into what each of those options does yet — that's a full topic of its own, covered in depth in Module 14. For now, just recognize the line when you see it in other people's scripts.

---

## Detailed Explanations

### Why the shebang isn't "just a comment"

Visually, `#!/bin/bash` looks like a comment (it starts with `#`), and Bash itself does treat it as a comment once the file is being interpreted. But its real power happens **before** Bash ever gets involved: when the Linux kernel is asked to execute a file directly (`./script.sh`), it inspects the first two bytes. If they're `#!`, the kernel reads the rest of that line as "the path to the real interpreter," and re-invokes *that* program, handing it your script file as its argument. This mechanism is part of the operating system itself, not a Bash feature — it's why shebangs work identically for Python scripts (`#!/usr/bin/env python3`), Perl scripts, and more.

### Why `./script.sh` needs execute permission but `bash script.sh` doesn't

`./script.sh` asks the **kernel** to execute the file directly as a program — and executing a file requires the execute permission bit, exactly as covered in Module 4. `bash script.sh`, by contrast, doesn't execute the file at all in that sense — it runs the `bash` program (which you already have permission to execute) and simply tells it "read this file as your input," the same way you could `cat` any readable file. That's why `bash script.sh` works even on a file with no execute permission at all — you're not executing the script, you're feeding it to an interpreter you're already allowed to run.

### The full picture: variables, quoting, and real command output

| Scenario | Command | What happens |
|---|---|---|
| Value has no spaces | `file=notes.txt; ls $file` | Works fine either way — no word splitting occurs because there's nothing to split |
| Value has spaces, unquoted | `file="my notes.txt"; ls $file` | Breaks — expands to `ls my notes.txt`, two separate arguments |
| Value has spaces, quoted | `file="my notes.txt"; ls "$file"` | Works correctly — expands to `ls "my notes.txt"`, one argument |
| Value is empty, unquoted | `name=""; echo $name end` | The empty value vanishes entirely — arguments shift unexpectedly |
| Value is empty, quoted | `name=""; echo "$name" end` | Correctly preserves an empty argument in place |

### Command substitution nesting and use in variables

Command substitution can be nested and combined with string concatenation:

```bash
backup_name="backup-$(date +%Y%m%d)-$(hostname).tar.gz"
```

This builds a filename like `backup-20260728-webserver01.tar.gz` in one line, by running `date` and `hostname` and splicing their output directly into the string.

### Arithmetic edge cases

`$((...))` only handles **integers** — no decimals. `$((7 / 2))` gives `3`, not `3.5` (integer division truncates). For floating-point math, Bash reaches for external tools like `bc`, which is outside this module's scope but worth knowing exists.

```bash
echo $((10 % 3))   # 1 — the modulo/remainder operator
echo $((2 ** 8))   # 256 — exponentiation
```

---

## Practical Examples

### Example 1 — Your first script, three ways to run it

Create a file called `hello.sh`:

```bash
#!/bin/bash
#
# hello.sh — prints a friendly greeting

echo "Hello from a real script!"
```

```bash
chmod +x hello.sh
./hello.sh
```

Expected output:
```
Hello from a real script!
```

```bash
bash hello.sh
sh hello.sh
```

Expected output (both):
```
Hello from a real script!
```

Line-by-line:
- `chmod +x hello.sh` grants the execute permission bit, required for `./hello.sh` to work at all.
- `./hello.sh` asks the kernel to run the file directly — it reads the shebang and hands the file to `/bin/bash`.
- `bash hello.sh` runs it via Bash explicitly, ignoring the shebang and not requiring execute permission.
- `sh hello.sh` runs it via `dash` on Ubuntu (since `/bin/sh` is symlinked there) — this simple script happens to work fine under `dash` too, but that won't always be true once you use Bash-specific features.

💡 **Tip:** If you forget `chmod +x` and try `./hello.sh`, you'll see `bash: ./hello.sh: Permission denied` — that error is your cue to go back to Module 4's permission model.

### Example 2 — Variables, and the quoting danger, made concrete

```bash
#!/bin/bash
#
# report-check.sh — demonstrates unquoted vs quoted variable expansion

touch "/tmp/monthly report.txt"

file="/tmp/monthly report.txt"

echo "--- Unquoted (broken) ---"
ls $file

echo "--- Quoted (correct) ---"
ls "$file"
```

```bash
chmod +x report-check.sh
./report-check.sh
```

Expected output:
```
--- Unquoted (broken) ---
ls: cannot access '/tmp/monthly': No such file or directory
ls: cannot access 'report.txt': No such file or directory
--- Quoted (correct) ---
/tmp/monthly report.txt
```

Line-by-line:
- `ls $file` (unquoted) undergoes word splitting: Bash splits the value on the space and hands `ls` two separate arguments, `/tmp/monthly` and `report.txt` — neither of which exists as a real filename.
- `ls "$file"` (quoted) passes the entire value, spaces included, as a single argument — exactly the file that actually exists.

⚠️ **Warning:** This is not a rare edge case. Filenames with spaces, usernames with spaces, and multi-word input from `read` are all completely ordinary in real systems. Unquoted variables are a live bug waiting for the right input to trigger it.

### Example 3 — Shell variables vs. environment variables

```bash
#!/bin/bash
#
# env-demo.sh — shows a child process only sees exported variables

shell_only="I stay local"
export exported_var="I get inherited"

echo "Inside this script:"
echo "  shell_only  = $shell_only"
echo "  exported_var = $exported_var"

echo "What a child process (bash -c) sees:"
bash -c 'echo "  shell_only  = $shell_only"; echo "  exported_var = $exported_var"'
```

```bash
chmod +x env-demo.sh
./env-demo.sh
```

Expected output:
```
Inside this script:
  shell_only  = I stay local
  exported_var = I get inherited
What a child process (bash -c) sees:
  shell_only  =
  exported_var = I get inherited
```

Line-by-line:
- `shell_only` was set but never exported — the child `bash -c` process (a separate process) has no idea it exists, so it prints empty.
- `exported_var` was exported — the child process inherits a copy and prints it correctly.

🎯 **On the job:** This is exactly the pattern behind bugs like "my script works interactively but fails in cron/CI" — a variable set in your `.bashrc` or terminal session was never exported, or exists only in an environment the automated job doesn't share.

### Example 4 — Command substitution and arithmetic together

```bash
#!/bin/bash
#
# backup-name.sh — builds a timestamped backup filename and reports its size in MB

timestamp=$(date +%Y%m%d-%H%M%S)
host=$(hostname)
filename="backup-${host}-${timestamp}.tar.gz"

size_bytes=204800
size_mb=$((size_bytes / 1024 / 1024))

echo "Backup filename: $filename"
echo "Approx size: ${size_mb} MB"
```

```bash
chmod +x backup-name.sh
./backup-name.sh
```

Expected output (values will differ on your machine):
```
Backup filename: backup-webserver01-20260728-143210.tar.gz
Approx size: 0 MB
```

Line-by-line:
- `$(date +%Y%m%d-%H%M%S)` and `$(hostname)` each run a real command and capture its printed output as a string.
- `$((size_bytes / 1024 / 1024))` performs pure integer arithmetic — note the result rounds down to `0` because `204800 / 1024 / 1024` isn't a whole number and Bash arithmetic has no decimals.

💡 **Tip:** If you need a legacy script's `expr $x + 1` translated to modern style, it becomes `$((x + 1))` — shorter, faster (no external process launched), and easier to read.

### Example 5 — Positional parameters and `shift`

```bash
#!/bin/bash
#
# args-demo.sh — shows positional parameters in action

echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "Total arguments: $#"

echo "Looping with shift:"
while [ "$#" -gt 0 ]; do
    echo "  Processing: $1"
    shift
done
```

```bash
chmod +x args-demo.sh
./args-demo.sh alpha beta gamma
```

Expected output:
```
Script name: ./args-demo.sh
First argument: alpha
Second argument: beta
Total arguments: 3
Looping with shift:
  Processing: alpha
  Processing: beta
  Processing: gamma
```

Line-by-line:
- `$0` is the script's own invocation name, `$1`/`$2` are the first two arguments, `$#` is the count (`3`).
- The `while [ "$#" -gt 0 ]` loop (`[ ]` test syntax and loops are covered fully in Module 7 — used here just to demonstrate `shift`) runs once per remaining argument. Each `shift` discards the current `$1` and renames everything down by one, so the loop naturally processes every argument without knowing in advance how many there are.

### Example 6 — `"$@"` vs `"$*"`: broken vs. fixed

This is the example to remember. Both scripts below simply loop over the arguments and print each one on its own line, wrapped in brackets so you can see exactly where each "word" begins and ends.

**The broken version, using `"$*"`:**

```bash
#!/bin/bash
#
# print-args-star.sh — BROKEN: uses "$*"

for arg in "$*"; do
    echo "[$arg]"
done
```

```bash
chmod +x print-args-star.sh
./print-args-star.sh "New York" Paris "Los Angeles"
```

Expected output:
```
[New York Paris Los Angeles]
```

**The fixed version, using `"$@"`:**

```bash
#!/bin/bash
#
# print-args-at.sh — FIXED: uses "$@"

for arg in "$@"; do
    echo "[$arg]"
done
```

```bash
chmod +x print-args-at.sh
./print-args-at.sh "New York" Paris "Los Angeles"
```

Expected output:
```
[New York]
[Paris]
[Los Angeles]
```

Line-by-line:
- Three arguments were passed in: `"New York"`, `Paris`, and `"Los Angeles"` — two of them contain a space.
- `"$*"` collapses **all** arguments into a **single string**, joined by a space (the first character of `IFS`). The `for` loop then sees only **one** "word" total — the whole joined string — so it runs its body exactly once, over the entire blob. That's why the whole thing prints as one bracketed line instead of three.
- `"$@"` expands to **three separate, individually-quoted words**, exactly matching what was originally passed in. The `for` loop correctly runs its body three times, once per original argument, spaces preserved intact within each one.

⚠️ **Warning:** This distinction **only** shows up when `$@`/`$*` are quoted (or used inside a `for` loop, which quotes them for you in the `"$@"` case implicitly in modern Bash — but always write it explicitly). Unquoted, both `$@` and `$*` behave the same (and both suffer the same word-splitting problems as any other unquoted variable). The safe, professional default is always `"$@"` when you mean "every argument, preserved exactly as given."

✅ **Best Practice:** Any time you write a wrapper script that just forwards its own arguments to another command (`my-wrapper.sh` calls `real-tool "$@"`), use `"$@"` — never `$@`, `$*`, or `"$*"` — or you will silently mangle any argument containing spaces.

### Example 7 — Reading user input, including passwords

```bash
#!/bin/bash
#
# login-demo.sh — demonstrates read, read -p, and read -s

read -p "Enter your username: " username
read -s -p "Enter your password: " password
echo    # move to a new line after the silent password prompt

echo "Welcome, $username! (password length: ${#password} characters)"
```

```bash
chmod +x login-demo.sh
./login-demo.sh
```

Expected interaction:
```
Enter your username: weki
Enter your password:
Welcome, weki! (password length: 8 characters)
```

Line-by-line:
- `read -p "Enter your username: " username` prints the prompt and waits, storing the typed text in `username`.
- `read -s -p "..." password` does the same, but `-s` ("silent") suppresses echoing the typed characters to the screen — essential for passwords or other secrets.
- The bare `echo` after it just prints a newline, since silent mode also suppresses the newline the user's Enter key would normally produce visually.
- `${#password}` (a length lookup, briefly previewed here) shows the password was captured correctly without ever displaying it.

⚠️ **Warning:** `read -s` hides the input on screen, but the value still lives in a plain variable in memory for the rest of the script. Never `echo` a password variable back out, and be careful about writing it to logs.

### Example 8 — Exit codes in practice

```bash
#!/bin/bash
#
# check-file.sh — reports success/failure via exit code, not just text

target="/etc/hostname"

if [ -f "$target" ]; then
    echo "Found: $target"
    exit 0
else
    echo "Missing: $target"
    exit 1
fi
```

```bash
chmod +x check-file.sh
./check-file.sh
echo "Exit code was: $?"
```

Expected output:
```
Found: /etc/hostname
Exit code was: 0
```

Line-by-line:
- The `if [ -f "$target" ]` test (fully covered in Module 7) checks whether the file exists.
- `exit 0` and `exit 1` explicitly set the script's own exit code — this is the number any calling script, CI pipeline, or `&&`/`||` chain will see.
- `$?` immediately after running the script captures its exit code — checked **immediately**, since `$?` is overwritten by the very next command that runs.

🎯 **On the job:** A deployment pipeline step might read `if ./check-file.sh; then ./deploy.sh; fi` — it never looks at the printed text at all, only at whether `check-file.sh` exited `0` or non-zero. Get the exit code wrong, and the pipeline makes the wrong decision regardless of what your `echo` statements said.

### Example 9 — Full annotated script: a mini deployment pre-check

This example ties every concept in this module together into one realistic, on-the-job script: it takes an environment name as an argument, confirms with the operator before proceeding, builds a log message with command substitution, and exits with a meaningful code.

```bash
#!/bin/bash
#
# predeploy-check.sh — verifies an environment name was given, confirms with
# the operator, and reports a clear exit code for automation to consume.
# Usage: ./predeploy-check.sh <environment>

# --- 1. Validate arguments (positional parameters + exit codes) ---
if [ "$#" -eq 0 ]; then
    echo "Usage: $0 <environment>" >&2
    exit 2   # 2 = bad usage
fi

environment="$1"
readonly environment

# --- 2. Build a timestamped log line (command substitution) ---
timestamp=$(date +"%Y-%m-%d %H:%M:%S")
log_line="[$timestamp] Pre-deploy check requested for: $environment"
echo "$log_line"

# --- 3. Track attempt count (arithmetic) ---
attempts=0
max_attempts=3
attempts=$((attempts + 1))
echo "Attempt $attempts of $max_attempts"

# --- 4. Confirm with the operator (read -p) ---
read -p "Deploy to '$environment'? Type 'yes' to continue: " confirmation

# --- 5. Decide and exit with a meaningful code ---
if [ "$confirmation" = "yes" ]; then
    echo "Confirmed. Proceeding with deployment to $environment."
    exit 0   # 0 = success, safe to proceed
else
    echo "Aborted by operator. No changes made."
    exit 1   # 1 = user declined
fi
```

```bash
chmod +x predeploy-check.sh
./predeploy-check.sh production
```

Expected interaction (operator confirms):
```
[2026-07-28 14:45:02] Pre-deploy check requested for: production
Attempt 1 of 3
Deploy to 'production'? Type 'yes' to continue: yes
Confirmed. Proceeding with deployment to production.
```

```bash
echo "Exit code: $?"
```
```
Exit code: 0
```

Running it with no argument:
```bash
./predeploy-check.sh
```
```
Usage: ./predeploy-check.sh <environment>
```
```bash
echo "Exit code: $?"
```
```
Exit code: 2
```

Line-by-line:
- Section 1 checks `$#` (the argument count) before touching `$1` at all — reading `$1` when no argument was given just gives an empty string rather than an error, so this guard is what actually catches misuse. It prints the usage message to `>&2` (standard error, from Module 5) since it's an error message, not normal output, and exits `2` to distinguish "bad usage" from other failure types.
- `environment="$1"` copies the first argument into a clearly-named, quoted variable — always prefer a descriptive name like `environment` over reusing `$1` throughout the rest of the script.
- `readonly environment` locks that variable so nothing later in the script can accidentally overwrite it.
- `$(date +"...")` (command substitution) captures the current timestamp as a string, spliced into `log_line`.
- `attempts=$((attempts + 1))` is arithmetic expansion — plain, fast integer math, no external process.
- `read -p "..." confirmation` pauses for operator input, with the prompt shown inline.
- The final `if` checks the typed confirmation and calls `exit 0` or `exit 1` — a real automation pipeline calling this script can now safely do `if ./predeploy-check.sh production; then ...` without parsing any text output at all.

---

## Common Pitfalls & Best Practices

- **Forgetting the shebang, or putting something before it.** Without a correct `#!/bin/bash` as the literal first line, you lose control over which interpreter runs your script — always make it line one, no exceptions.
- **Forgetting `chmod +x`.** Running `./script.sh` on a non-executable file gives `Permission denied`. This is one of the very first errors every beginner hits — check permissions with `ls -l` (Module 4) if it happens.
- **Putting spaces around `=` in an assignment.** `name = "value"` is not an assignment — Bash tries to run `name` as a command. Always write `name="value"` with zero spaces on either side of `=`.
- **Leaving variables unquoted.** `$file` instead of `"$file"` invites word splitting and globbing bugs the moment a value contains a space, `*`, or is empty. Quote by default.
- **Confusing `"$@"` and `"$*"`.** Use `"$@"` whenever you mean "every argument, individually, exactly as given." Reach for `"$*"` only in the rare case you deliberately want them joined into one string.
- **Assuming `sh script.sh` behaves like `bash script.sh` on Ubuntu.** `/bin/sh` is `dash`, a different, stricter shell — Bash-only syntax can silently misbehave or error out under it.
- **Reading a password without `-s`.** `read -p "Password: " pass` echoes every keystroke to the screen — always add `-s` for anything sensitive.
- **Not checking `$?` immediately.** `$?` is overwritten by the *next* command that runs — including something as innocuous as an `echo`. Capture it (`result=$?`) right after the command you care about if you need it later.
- **Forgetting that unset exit codes still exist.** If you never call `exit` explicitly, your script's exit code is simply whatever its last command returned — which might not be what you intend. Be deliberate about your final exit code in anything meant for automation.

---

## Hands-on Exercise

**Task:** Write a script called `welcome.sh` that:

1. Requires at least one name to be passed as an argument. If none is given, print a usage message to standard error and exit with code `2`.
2. Greets every name passed in, one per line (correctly handling names that might contain a space, like `"Mary Jane"`) — this means you need to think carefully about `"$@"` vs `"$*"`.
3. After greeting everyone, uses `read -p` to ask "Would you like a fun fact? (yes/no): ".
4. If the answer is `yes`, prints a fun fact and exits `0`. If `no` (or anything else), prints "Okay, maybe next time!" and exits `0` anyway (declining isn't an error).
5. Reports the exit code back with `$?` after you run it.

Try writing this yourself before reading the solution below.

### Solution

```bash
#!/bin/bash
#
# welcome.sh — greets one or more names, then optionally shares a fun fact.
# Usage: ./welcome.sh <name> [name2] [name3] ...

# 1. Require at least one argument
if [ "$#" -eq 0 ]; then
    echo "Usage: $0 <name> [name2] [name3] ..." >&2
    exit 2
fi

# 2. Greet every name individually — "$@" preserves each name, spaces and all
echo "Greeting $# guest(s):"
for guest in "$@"; do
    echo "  Hello, $guest! Welcome."
done

# 3. Ask if they want a fun fact
read -p "Would you like a fun fact? (yes/no): " answer

# 4. Respond and exit
if [ "$answer" = "yes" ]; then
    echo "Fun fact: the first Bash release was in 1989 — it's older than most of us!"
    exit 0
else
    echo "Okay, maybe next time!"
    exit 0
fi
```

```bash
chmod +x welcome.sh
./welcome.sh "Mary Jane" Alex
```

Expected interaction:
```
Greeting 2 guest(s):
  Hello, Mary Jane! Welcome.
  Hello, Alex! Welcome.
Would you like a fun fact? (yes/no): yes
Fun fact: the first Bash release was in 1989 — it's older than most of us!
```

```bash
echo "Exit code: $?"
```
```
Exit code: 0
```

Running with no arguments:
```bash
./welcome.sh
```
```
Usage: ./welcome.sh <name> [name2] [name3] ...
```
```bash
echo "Exit code: $?"
```
```
Exit code: 2
```

Explanation: I guarded against zero arguments first, before doing anything else, because reading `$1` with no arguments would just silently be empty rather than raising an error — the `$#` check is what actually catches the misuse and reports it clearly. I used `"$@"` (quoted, inside a `for` loop) specifically because the task calls for names that might contain a space, like `"Mary Jane"` — if I'd used `"$*"` or left it unquoted, that name would have been mangled or split. Finally, I made both branches of the fun-fact question exit `0`, since declining isn't a failure — only the missing-argument case represents a real error worth signaling with a non-zero code.

✅ Exercise complete — you've written a script with argument validation, correctly-quoted positional parameters, interactive input, and meaningful exit codes.

---

## ✅ Module Completion Checklist

- [ ] I can explain what a bash script is, how it differs from typing commands interactively, and why scripts matter on the job (repeatability, automation, version control)
- [ ] I can write a correct shebang line (`#!/bin/bash` or `#!/usr/bin/env bash`) and explain what it does and why it matters
- [ ] I can make a script executable with `chmod +x` and explain the difference between running it as `./script.sh`, `bash script.sh`, and `sh script.sh`
- [ ] I can declare and use variables correctly (`name="value"`, `$name`, `${name}`), and explain why quoting variables matters
- [ ] I can explain the difference between shell variables and environment variables, and use `export`, `env`, and `printenv`
- [ ] I can use command substitution `$(...)` and arithmetic expansion `$((...))` confidently, and explain why `expr` is considered legacy
- [ ] I can work with positional parameters (`$0`, `$1`, `$@`, `$*`, `$#`, `shift`) and explain precisely how `"$@"` differs from `"$*"`
- [ ] I can read user input with `read`, `read -p`, and `read -s`, and use exit codes (`exit 0`, `exit 1`, `$?`) so a script communicates success or failure to whatever runs it

## Next Step

Continue to [Module 7: Control Flow](../module7-control-flow/)
