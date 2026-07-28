# 🎤 Module 7 Interview Prep — Control Flow

## Conceptual Questions

### 🟢 Beginner

**Q1: What does an `if` statement actually check in Bash — is there a real "boolean" type?**

> "No, Bash has no boolean type at all. `if` runs whatever command follows it and looks at that command's exit status — `0` is treated as true, and any non-zero value is treated as false. `[ ]`, `[[ ]]`, and `test` are just commands that are specifically designed to evaluate a condition and return `0` or `1` accordingly, but `if` itself doesn't care what command it's given — it works the same way whether that's `[[ -f file ]]` or a totally ordinary command like `grep` or `ping`."

**Q2: What's the difference between `[ ]` and `test`?**

> "There isn't really a functional difference — `[` is literally another name for the same `test` command, with one quirk: it requires a matching `]` as its final argument. `[ -f file.txt ]` and `test -f file.txt` do exactly the same thing."

**Q3: What does `-z` check, and what's its opposite?**

> "`-z` checks whether a string is empty — zero length. Its opposite is `-n`, which checks that a string is *not* empty. `[[ -z "$name" ]]` is a very common guard for 'was this variable never set or left blank.'"

**Q4: What's the difference between `-f` and `-e` as file test operators?**

> "`-e` just checks that something exists at that path at all — file, directory, symlink, device, anything. `-f` is more specific: it checks that the path exists **and** is a regular file. If I'm checking for a config file specifically, I use `-f`, not `-e` — `-e` would happily say 'yes' even if a directory happened to have that same name."

**Q5: How do you write a `for` loop over three named servers?**

> "`for server in web01 web02 db01; do echo \"$server\"; done` — the loop runs once per word in that list, in order, with `$server` holding each one in turn."

### 🟡 Intermediate

**Q6: What's the actual difference between `[ ]` and `[[ ]]`, and why is `[[ ]]` generally preferred?**

> "`[ ]` is an ordinary command — really just another name for `test` — so its arguments get expanded by the shell exactly like any other command's arguments would, which means unquoted variables inside it are vulnerable to word splitting and globbing, the same bugs we saw with unquoted variables back in the scripting fundamentals module. `[[ ]]` is a Bash keyword, parsed specially by the shell itself, which sidesteps those problems — I can write `[[ $name == \"Mary Jane\" ]]` without worrying about it splitting on the space. `[[ ]]` also supports `&&`/`||` directly inside the brackets, plus glob pattern matching with `==` and full regex matching with `=~`, none of which `[ ]` can do at all. The one tradeoff is portability — `[[ ]]` is Bash-specific, so a script that must run under a strict POSIX shell like `dash` has to stick with `[ ]`/`test`. For a plain `#!/bin/bash` script, I default to `[[ ]]`."

**Q7: For vs. while vs. until — when do you reach for each one?**

> "I use a `for` loop when I have a known, iterable set of things up front — a fixed list, a glob of files, or the words in a command's output — because `for` naturally expresses 'once per item.' I use `while` when I'm repeating based on a condition that might change dynamically, like a retry counter or waiting for a service to respond, and the natural phrasing is 'keep going while this is true.' `until` is the mirror image of `while` — same mechanism, just running while the condition is *false* — and I reach for it only when that phrasing reads more naturally, like 'retry until the server responds' instead of 'while the server hasn't responded yet.' Functionally you can always rewrite one as the other by negating the condition; it's purely about which one reads clearer for the situation."

**Q8: Explain `-eq` vs `==` — why does it matter which one you pick?**

> "`==` (or `=` in `[ ]`) is a **string** comparison — it checks whether two sequences of characters are identical. `-eq` is a **numeric** comparison — it checks whether two values are mathematically equal. They usually agree, but not always: `[[ \"05\" == \"5\" ]]` is false, because as plain text those are two different strings, while `[[ \"05\" -eq 5 ]]` is true, because as numbers they're the same value. Using the wrong one is a classic real bug — comparing a zero-padded number, or a number typed with extra whitespace, with `==` instead of `-eq` and getting a false negative that looks completely inexplicable until you realize you're comparing text, not numbers."

**Q9: How does a `case` statement decide which block to run, and what's the role of the final `*)`?**

> "`case` compares one value against a series of glob-style patterns, top to bottom, and runs the first block whose pattern matches — then stops, unless you specifically use `;;&` to fall through and keep checking. The final `*)` is a catch-all pattern that matches literally anything, so it's used as a default case for any value that didn't match one of the earlier, more specific patterns. I always include one in production scripts — without it, an unexpected value just silently falls all the way through the `case` with no output and no error at all, which is a much harder failure to notice and debug than a clear 'unrecognized value' message."

**Q10: What does the `if command; then` pattern mean, and how is it different from `if [ ]; then`?**

> "`if [ ]; then` and `if [[ ]]; then` are really just special cases of the same general rule — `if` runs whatever follows it and branches on its exit status. `if command; then` makes that explicit by putting a completely ordinary command there instead of a bracket test — something like `if grep -q \"ERROR\" file.log; then` or `if ping -c 1 host > /dev/null; then`. It's testing 'did this real operation actually succeed,' not a string/number/file condition. It's an extremely common pattern for things like 'did this dependency check succeed,' 'was this pattern found,' or 'did my own script run cleanly' — `if my-script.sh; then ...` relies on exactly this."

### 🔴 Advanced

**Q11: You have a nested loop — an outer loop over servers, an inner loop over services on each server. One particular service failure means 'abandon this entire server, move to the next one.' How do you express that, and why doesn't plain `continue` work?**

> "Plain `continue` (with no number) only ever affects the **innermost** loop — it would skip to the next service, but it would still keep checking the remaining services for that same troubled server, and 'abandon this whole server' wouldn't actually happen. I'd use `continue 2` instead, which tells Bash to skip the rest of the current iteration two loop levels out — meaning both the rest of the inner service loop *and* the rest of the outer server loop's current iteration are abandoned, jumping straight to the next server. If instead I wanted to stop checking *every* server entirely the moment this happens, I'd use `break 2` instead, which exits both loops completely rather than moving on to the outer loop's next iteration."

**Q12: Why can `[ $a == $b ]` behave completely differently from `[[ $a == $b ]]` when `$a` or `$b` is empty or contains spaces?**

> "`[ ]` is a plain command, so Bash expands `$a` and `$b` before `[` ever sees them, exactly like arguments to any other command — word splitting and globbing both apply. If `$a` is empty, `[ $a == $b ]` can literally collapse down to `[ == $b ]`, which is a syntax error because `[` sees a missing first operand. If `$a` contains a space, it can split into multiple arguments and again confuse `[`'s argument parsing. `[[ ]]`, being a Bash keyword rather than a regular command, treats `$a` and `$b` as single fields even when unquoted, so an empty or space-containing value doesn't break the comparison. This is one of the concrete, reproducible reasons `[[ ]]` is preferred — not just a style preference."

**Q13: A production script uses `while [[ condition ]]; do ... done` and, under a rare set of inputs, never terminates. How would you audit the script for this class of bug, and what would a safer version look like?**

> "I'd trace exactly which variable(s) the condition depends on, and verify every single code path through the loop body actually updates at least one of them before looping back — a common real cause is an early `continue` that skips over the line that increments a counter, so some iterations silently never advance the loop toward its exit condition. I'd also check whether the condition depends on external state (a file appearing, a service responding) that might simply never become true under some real-world failure mode, in which case the loop is 'correct' in isolation but unsafe in production. The fix is almost always to add an explicit upper bound — a `max_attempts` counter checked in the *same* condition (`while [[ condition && attempts -lt max_attempts ]]`) — so the loop is provably guaranteed to terminate even if the underlying condition never resolves, plus a clear failure message and non-zero exit code once that bound is hit."

---

## Practical/Coding Questions

**Q1: Write a script that reports whether a given path is a readable file, a directory, or missing, using `if`/`elif`/`else`.**

Solution:
```bash
#!/bin/bash
path="$1"

if [[ -f "$path" && -r "$path" ]]; then
    echo "$path is a readable file."
elif [[ -d "$path" ]]; then
    echo "$path is a directory."
else
    echo "$path is missing or not readable."
fi
```
Explanation: The first branch combines two file tests with `&&` inside a single `[[ ]]`. `elif` only runs if that combined condition was false, letting me check a second, distinct possibility (directory) before falling back to a generic `else`.

**Q2: Write a `case` statement that classifies a file by its extension (`.txt`, `.sh`, `.tar.gz`, or unknown).**

Solution:
```bash
#!/bin/bash
filename="$1"

case "$filename" in
    *.tar.gz)
        echo "Compressed archive"
        ;;
    *.txt)
        echo "Plain text file"
        ;;
    *.sh)
        echo "Shell script"
        ;;
    *)
        echo "Unknown file type"
        ;;
esac
```
Explanation: `*.tar.gz` is checked before `*.txt`/`*.sh` deliberately — pattern order matters in `case`, and a more specific double-extension pattern needs to be checked before any pattern that might otherwise seem to apply. The `*)` catch-all guarantees every input gets some response.

**Q3: Write a script using a C-style `for` loop that prints only the even numbers from 1 to 10.**

Solution:
```bash
#!/bin/bash
for ((i = 1; i <= 10; i++)); do
    if [[ $((i % 2)) -eq 0 ]]; then
        echo "$i"
    fi
done
```
Explanation: `$((i % 2))` (Module 6's arithmetic expansion, modulo operator) gives the remainder of `i` divided by 2 — `0` for even numbers. `-eq 0` is a numeric comparison, since we're testing an arithmetic result, not a string.

**Q4: Write a retry loop using `while` that tries a command up to 3 times and reports final success/failure with an appropriate exit code.**

Solution:
```bash
#!/bin/bash
max_attempts=3
attempt=1
succeeded=false

while [[ "$attempt" -le "$max_attempts" ]]; do
    if some-flaky-command; then
        succeeded=true
        break
    fi
    echo "Attempt $attempt failed, retrying..."
    attempt=$((attempt + 1))
done

if [[ "$succeeded" == "true" ]]; then
    echo "Succeeded on attempt $attempt."
    exit 0
else
    echo "Failed after $max_attempts attempts." >&2
    exit 1
fi
```
Explanation: `if some-flaky-command; then` is the `if command; then` pattern — testing the real command's exit status directly rather than wrapping it in brackets. `break` stops retrying the moment it succeeds; the counter increments only on failure, and the loop condition itself provides the upper bound so this can never retry forever.

---

## Gotcha Questions

**Q1: "My script has `if [$x -eq 5]; then` and it just errors out with something about a missing command. What's wrong?"**

> Trap: Missing spaces. `[$x` and `5]` are each parsed as one single "word" — Bash is looking for a command literally named `[$x` (which doesn't exist), not the `[` command with separate arguments. The fix is `[ $x -eq 5 ]` (or, preferably, `[[ $x -eq 5 ]]`) with a space immediately after the opening bracket and immediately before the closing one. This is one of the most common very first errors people hit with conditionals, and it's purely a spacing issue, not a logic issue.

**Q2: "I wrote `if [ $count == 10 ]` to check a loop counter and it never matches, even though I can see the counter clearly reaches 10 when I echo it. Why?"**

> Trap: `==`/`=` are **string** comparisons, not numeric ones. If `$count` ever holds something like `" 10"` with leading whitespace, or is being compared against a value with different formatting, the strings won't match even though they represent the 'same number' to a human eye — and more generally, using a string operator for what's conceptually a numeric comparison invites exactly this kind of subtle mismatch. The fix is `-eq` for numeric comparisons: `[ "$count" -eq 10 ]`. A good habit is to ask "am I comparing text or numbers?" before choosing the operator, every single time.

**Q3: "I removed the increment line from inside my `while` loop during a refactor because it looked redundant, and now the script just hangs forever when I run it. What happened?"**

> Trap: A classic infinite loop. `while` re-checks its condition every iteration — if nothing inside the loop body actually changes the value(s) that condition depends on, the condition never becomes false and the loop runs forever. The "redundant-looking" increment (`attempt=$((attempt + 1))`, or whatever advances the loop toward its exit condition) was in fact essential. The fix is restoring it, and more generally: before removing any line inside a loop body, check whether it's the thing that eventually makes the loop's condition go false.

**Q4: "I have `case \"$answer\" in yes) ...;; *) ...;; esac` and typing `Yes` (capital Y) falls into my catch-all `*)` branch instead of matching. Is `case` broken?"**

> Trap: `case` pattern matching is exact and case-sensitive by default — `yes` and `Yes` are different strings as far as glob-style pattern matching is concerned, and there's no built-in case-insensitivity. This isn't a bug — the candidate should recognize this and either normalize the input first (e.g., convert `$answer` to lowercase before the `case`, a technique covered with string manipulation next module) or explicitly list every accepted casing as alternatives: `yes|Yes|YES)`.

---

## Quick-Fire Rapid Review

- **Q: What does `if` actually check — a boolean, or something else?** A: A command's exit status (`0` = true, non-zero = false).
- **Q: Is `[ ]` a command or a keyword?** A: A command — literally another name for `test`.
- **Q: Is `[[ ]]` a command or a keyword?** A: A Bash keyword, parsed specially by the shell.
- **Q: Why is `[[ ]]` generally preferred over `[ ]` in new Bash scripts?** A: No word-splitting/globbing issues on unquoted variables, plus native `&&`/`||` and pattern/regex matching.
- **Q: What checks "string is empty"?** A: `-z`.
- **Q: What checks "path exists and is a regular file" specifically (not just any path)?** A: `-f`.
- **Q: `-eq` compares what kind of values?** A: Numbers, not strings.
- **Q: What must every `case` statement end with?** A: `esac`.
- **Q: What's the recommended catch-all pattern in a `case` statement?** A: `*)`, placed last.
- **Q: What's the difference between `while` and `until`?** A: `while` loops while the condition is true; `until` loops while it's false (the mirror image).
- **Q: What does plain `break` (no number) affect in a nested loop?** A: Only the innermost loop.
- **Q: How do you make `break`/`continue` affect an outer loop?** A: Give it a level number — `break 2`, `continue 2`.
- **Q: What's the #1 cause of an infinite `while` loop?** A: The condition's underlying variable is never updated inside the loop body.
- **Q: What pattern lets you branch directly on a real command's success, with no brackets at all?** A: `if command; then ... fi`.
- **Q: Does `case` matching support regular expressions?** A: No — glob patterns only (`*`, `?`, `[...]`, `|` for alternatives); `[[ ]]`'s `=~` is for regex.
