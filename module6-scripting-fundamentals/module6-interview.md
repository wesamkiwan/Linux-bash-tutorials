# 🎤 Module 6 Interview Prep — Bash Scripting Fundamentals

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a bash script, and how is it different from typing commands one at a time?**

> "A bash script is just a plain text file containing a sequence of shell commands, saved so the whole sequence can be run as a single unit instead of typed live at the prompt. The commands themselves are identical to what I'd type interactively — the difference is repeatability. A script runs the exact same steps in the exact same order every single time, it can be version-controlled in Git, reviewed like any other code, and triggered automatically by a schedule or a pipeline instead of requiring a person at the keyboard."

**Q2: What does the shebang line actually do?**

> "The shebang — `#!/bin/bash` as the literal first line of the file — tells the operating system which interpreter should execute the rest of the file. When I run a script directly with `./script.sh`, the kernel reads those first two bytes, sees `#!`, and reads the rest of that line as the path to the real program to hand the file to — in this case `/bin/bash`. It has to be the very first line, with nothing before it, or that mechanism doesn't kick in correctly."

**Q3: How do you make a script runnable, and what's the actual error if you skip that step?**

> "I run `chmod +x script.sh` to add the execute permission bit. If I skip it and try `./script.sh`, I get `Permission denied` — the kernel refuses to execute a file that doesn't have the execute bit set, which ties directly back to the permissions model from Module 4."

**Q4: What's wrong with writing `name = "value"` to assign a variable?**

> "The spaces around the equals sign break it. Bash doesn't see an assignment at all — it tries to run a command literally called `name`, passing `=` and `"value"` as arguments, and that fails since there's no command called `name`. The correct form is `name=\"value\"` with absolutely no spaces on either side of the `=`."

**Q5: What's the difference between `$name` and `${name}`?**

> "They both retrieve the same variable's value — `${}` just exists to remove ambiguity, particularly when the variable name sits right next to other text. `echo \"$name_file.txt\"` would actually look for a variable called `name_file`, which is probably not what I meant, while `echo \"${name}_file.txt\"` correctly isolates just `$name` and appends the literal text after it."

### 🟡 Intermediate

**Q6: Why do we need to quote variables like `"$file"` instead of just writing `$file`?**

> "Without quotes, Bash performs word splitting — it takes the variable's value and, if it contains whitespace, splits it into multiple separate words before handing them to the command, one argument per word. So if `file` holds `\"my report.txt\"`, an unquoted `ls $file` doesn't run `ls` on one file — it runs `ls my report.txt`, treating `my` and `report.txt` as two separate, nonexistent filenames. Quoting — `ls \"$file\"` — passes the entire value through as a single argument. Unquoted variables are also vulnerable to globbing if the value happens to contain wildcard characters. My default habit is to quote every variable expansion unless I have a specific reason not to."

**Q7: What's the difference between a shell variable and an environment variable?**

> "A shell variable only exists inside the current shell or script — no process it launches can see it. An environment variable is a shell variable that's been exported with `export`, which marks it to be copied into the environment of every child process that shell starts from then on. It's the difference between a private note only I can see and a photocopy I hand to every task I delegate — they get their own copy, and changes on their end don't feed back to mine."

**Q8: Precisely how does `\"$@\"` differ from `\"$*\"`?**

> "Both expand to 'all the positional parameters,' but quoting changes everything. `\"$@\"` expands to each argument as its own separate, individually-quoted word — exactly as they were originally passed in, including any spaces inside a single argument. `\"$*\"` expands to one single string — every argument joined together with the first character of `IFS`, a space by default. If I loop `for a in \"$@\"\"`, I get one iteration per original argument. If I loop over `\"$*\"`, I get exactly one iteration total, over the entire joined blob. The safe default for 'pass along every argument I received, intact' is always `\"$@\"`."

**Q9: What does command substitution do, and how is `$(...)` different from backticks?**

> "Command substitution runs the command inside `$(...)`, waits for it to finish, and replaces the whole expression with that command's standard output, so I can capture it in a variable or splice it into a string — like `today=$(date +%F)`. Backticks, `` `command` ``, do the exact same thing but are the older syntax; they're harder to read and painful to nest, since a backtick inside a backtick needs escaping. `$(...)` nests cleanly and is what you'll see in every modern script."

**Q10: Why is `expr` considered legacy, and what replaced it?**

> "`expr` is a separate external program, not a shell built-in, so every call to it launches a whole new process just to do simple arithmetic — that's slow and it's fussy about spacing around operators. Modern Bash arithmetic expansion, `$((expression))`, does the same integer math natively inside the shell, with no extra process, cleaner syntax, and support for things like modulo and exponentiation. I'd recognize `expr` when reading an old script, but I wouldn't write new code with it."

### 🔴 Advanced

**Q11: Why can a script that works fine when you run it interactively fail when triggered by cron or a CI pipeline?**

> "The most common cause is environment variables that were never exported. Your interactive shell often has variables set through `.bashrc` or similar startup files, and some of those might only ever be set as plain shell variables, or might rely on an interactive login environment that cron and most CI runners don't replicate. If a script depends on a variable like an API key or a modified `PATH` that was never explicitly `export`-ed — or that only gets set in an interactive-shell startup file cron doesn't read — the script's child processes simply won't see it, and things that worked perfectly at your own prompt fail silently or loudly somewhere else."

**Q12: A script exits with code 0 even though something clearly went wrong inside it. What are the likely causes, and how would you fix it?**

> "If the script never calls `exit` explicitly, its final exit code is just whatever its very last command returned — so if that last command happens to succeed (like a harmless final `echo`) even though an earlier, important command failed, the script reports success anyway. I'd fix this by checking `$?` immediately after any command whose success actually matters, and calling `exit` with a deliberate, meaningful non-zero code the moment I detect a real failure, rather than letting the script fall through to whatever its last line happens to return. This exact problem is also part of why `set -e`-style options exist — covered in depth later in the course — but even without them, being deliberate about `exit` codes at every path through a script is the core discipline."

**Q13: How would you design a script's exit codes so that other automation calling it can make good decisions?**

> "I'd reserve `0` strictly for success, and assign distinct, documented non-zero codes to distinct failure categories instead of collapsing everything into a generic `1` — for example `2` for bad usage/arguments, `3` for a missing dependency, `4` for a failed remote connection. I'd document that mapping in the script's header comment. That way, a calling pipeline doesn't have to parse my printed text output at all — it can branch on `$?` and know exactly what category of problem occurred, which is far more reliable than string-matching log output that might change wording later."

---

## Practical/Coding Questions

**Q1: Write a script `first-arg.sh` that safely prints the first argument passed to it, or a clear error if none was given.**

Solution:
```bash
#!/bin/bash
if [ "$#" -eq 0 ]; then
    echo "Usage: $0 <argument>" >&2
    exit 1
fi
echo "First argument: $1"
```
Explanation: I check `$#` before touching `$1` — reading `$1` with no arguments doesn't error, it's just silently empty, so the `$#` guard is what actually catches the misuse. The usage message goes to `>&2` because it's an error, and the script exits non-zero to signal that to anything calling it.

**Q2: Write a script that safely deletes a file passed as an argument, but only after confirming with the user, handling filenames that contain spaces.**

Solution:
```bash
#!/bin/bash
target="$1"

if [ -z "$target" ]; then
    echo "Usage: $0 <file>" >&2
    exit 1
fi

read -p "Really delete '$target'? (yes/no): " confirm
if [ "$confirm" = "yes" ]; then
    rm "$target"
    echo "Deleted."
    exit 0
else
    echo "Cancelled."
    exit 0
fi
```
Explanation: `"$target"` is quoted every single time it's used — in the prompt, and critically in the `rm` call — so a filename like `my report.txt` is treated as one argument throughout, never split into two.

**Q3: Write a script that forwards all of its own arguments to `echo`, preserving each one exactly, even ones with spaces.**

Solution:
```bash
#!/bin/bash
echo "$@"
```
Explanation: `"$@"` (quoted) is the only form that reliably preserves each original argument as a distinct unit when forwarded onward. `$@`, `$*`, and `"$*"` would each either re-split on whitespace or collapse everything into one joined string, losing the original argument boundaries.

**Q4: Write a script that builds a log filename using the current date and the hostname, then reports whether it successfully created that file.**

Solution:
```bash
#!/bin/bash
logfile="log-$(hostname)-$(date +%Y%m%d).txt"

if touch "$logfile"; then
    echo "Created: $logfile"
    exit 0
else
    echo "Failed to create: $logfile" >&2
    exit 1
fi
```
Explanation: Two command substitutions build the filename in one line. `if touch "$logfile"; then ... else ... fi` uses `touch`'s own exit code directly in the `if` condition — the idiomatic way to branch on success/failure, rather than running the command and separately checking `$?` afterward.

**Q5: Write a script that reads a numeric retry count from the user and increments it three times, printing the value each time.**

Solution:
```bash
#!/bin/bash
read -p "Starting retry count: " count

for i in 1 2 3; do
    count=$((count + 1))
    echo "After increment $i: $count"
done
```
Explanation: `$((count + 1))` performs integer arithmetic expansion — the modern, preferred way over `let` or `expr`. Reading numeric input from `read` still gives a string, but Bash arithmetic expansion automatically treats numeric-looking strings as numbers inside `$((...))`.

---

## Gotcha Questions

**Q1: "My script ran `rm $file` and deleted the wrong things — the variable definitely held the right filename when I echoed it. What happened?"**

> Trap: The candidate should recognize this as unquoted variable expansion combined with word splitting (and possibly globbing). If `$file` held something with a space, or characters that happen to match existing filenames when treated as a glob pattern, an unquoted `rm $file` can expand into multiple separate arguments — deleting more, or different, things than intended. The fix is always the same: quote it, `rm \"$file\"`. Seeing the right value with `echo $file` doesn't prove anything — `echo` itself doesn't care about word splitting the same way most commands' argument handling does, so it can display correctly while still misbehaving once unquoted in a different command.

**Q2: "I wrote `for arg in $*; do ... done` to loop over my script's arguments, and it looked fine in testing — until someone passed an argument with a space in it, and now it silently split into two iterations. Why didn't quoting `\"$*\"` fix it?"**

> Trap: This tests whether the candidate actually understands the mechanism, not just the "use `$@`" rule by memory. Unquoted `$*` and unquoted `$@` behave identically — both are subject to word splitting, so an argument with a space becomes two words either way; quoting `\"$*\"` doesn't fix that case at all, because `\"$*\"` still joins everything into one single combined string (correctly preserving the original spaces within it, but merging all arguments into one). The only form that both preserves internal spaces *and* keeps each original argument separate is `\"$@\"`, quoted. The lesson: memorize `\"$@\"` specifically, quoted, not just "quote $* or $@ and it'll be fine."

**Q3: "I set `API_KEY=abc123` at the top of my script and my script itself uses it fine, but a helper script I call from inside it says the variable is empty. Why?"**

> Trap: Candidates who don't fully grasp the shell-vs-environment-variable distinction assume any variable set in a script is automatically visible to anything that script calls. It isn't — a plain assignment like `API_KEY=abc123` is a shell variable, local to the current shell only. The helper script runs as a separate child process and simply never receives it unless it was exported with `export API_KEY=abc123` (or `export API_KEY` after the fact). The fix is to export any variable that needs to survive into a child process.

**Q4: "My script checked `$?` after a whole block of several commands and it only ever seems to reflect the very last one, even when an earlier command clearly failed. Is `$?` broken?"**

> Trap: `$?` isn't broken — it's working exactly as designed: it always holds the exit status of the single most recently completed command, and it gets overwritten every single time another command runs, including something as innocuous as an `echo` inserted for a debug message. If several commands run in a row and only the last one's status is checked, everything earlier is invisible by the time `$?` is read. The fix is to check (and typically capture, e.g. `result=$?`) immediately after the specific command whose status actually matters, not after a whole block of unrelated commands have already run afterward.

---

## Quick-Fire Rapid Review

- **Q: What must the very first line of a script be, with nothing above it?** A: The shebang, e.g. `#!/bin/bash`.
- **Q: Command to make a script executable?** A: `chmod +x script.sh`.
- **Q: What shell does `/bin/sh` point to on Ubuntu?** A: `dash`, not Bash.
- **Q: Is `name = "value"` a valid assignment?** A: No — no spaces are allowed around `=`.
- **Q: What does an unquoted `$var` risk if its value has spaces?** A: Word splitting into multiple arguments.
- **Q: What marks a shell variable as inherited by child processes?** A: `export`.
- **Q: What does `$(command)` do?** A: Command substitution — captures the command's output as a string.
- **Q: Is `expr` the modern way to do arithmetic?** A: No — it's legacy; use `$((...))`.
- **Q: What does `$#` hold?** A: The count of positional arguments.
- **Q: What does `shift` do?** A: Drops `$1` and renumbers every remaining positional parameter down by one.
- **Q: Quoted, how does `"$@"` differ from `"$*"`?** A: `"$@"` keeps each argument separate; `"$*"` joins them all into one string.
- **Q: Which flag makes `read` hide typed input?** A: `-s`.
- **Q: What exit code means success by convention?** A: `0`.
- **Q: What does `$?` hold?** A: The exit status of the last completed command.
- **Q: What does `$$` hold?** A: The PID of the current shell/script.
