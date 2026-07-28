# Module 8: Functions, Arrays & String Manipulation 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2.5 hours
**Prerequisites:** Modules 1-7 (Shell Fundamentals through Control Flow — you should already be comfortable with `if`, `case`, `for`, `while`, `until`, variables, quoting, and exit codes)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Define a function using both the `function name { }` and `name() { }` forms, and call it like any other command
- [ ] Explain that `$1`, `$2`, and `$@` inside a function refer to *that function's own arguments*, not the script's, and pass arguments into a function correctly
- [ ] Use the `local` keyword to scope a variable to a function, and explain why forgetting it causes real bugs
- [ ] Explain precisely why `return` in Bash is exit-code-only (0-255) and cannot "return" a string — and use `echo` + command substitution `$()` as the correct pattern for getting data back out of a function
- [ ] Declare, populate, access, loop over, append to, and slice an **indexed array**, and explain the `${arr[@]}` vs `${arr[*]}` quoting distinction
- [ ] Declare and use a **Bash 4+ associative array** with `declare -A`, and loop over its keys with `${!arr[@]}`
- [ ] Manipulate strings using parameter expansion: length, substrings, search-and-replace, prefix/suffix removal, case conversion, and default-value handling
- [ ] Refactor a repetitive, copy-pasted script into clean, reusable functions that operate over arrays of real-world data (like a list of servers or filenames)

---

## Module Goal

By the end of this module, you'll be able to take a script that repeats the same five lines of logic for every server, every file, or every user — and collapse it into one small function you call in a loop.

🎯 **On the job:** Picture a deploy script that was copy-pasted for `web01`, then `web02`, then `web03` — each block doing the same SSH-connect-check-restart dance with the hostname hardcoded. It works, until someone adds `web04` and forgets to update one of the five copy-pasted blocks, or a bug is found and has to be fixed in four places instead of one. This module teaches you to write that logic **once**, as a function, store your server names in an **array**, loop over the array calling the function for each one, and use **string manipulation** to build hostnames, log lines, and file names on the fly. This is the exact moment a script stops being "long and repetitive" and starts being "short and maintainable" — one of the clearest signals of a professional script versus a beginner one.

---

## Core Concepts

### 1. What is a function?

A **function** is a named block of code you define once and can call — run — as many times as you like, anywhere in your script, just by typing its name like any other command.

💡 **Analogy — a mini-program inside your program:** Think of your script as a small factory. Without functions, if the same five-step assembly process is needed in six places, you'd build six separate assembly lines, each one identical. A function is building that assembly line **once** and routing every product through it. Change the process once, and every use of it is instantly fixed — no hunting for five copies you might have missed.

Functions are also called **subroutines** in older programming terminology — a smaller routine "called" from within a larger one.

### 2. Two ways to define a function

Bash supports two equivalent syntaxes:

```bash
# Form 1: the "function" keyword
function greet {
    echo "Hello!"
}

# Form 2: parentheses, no keyword (POSIX-style — more portable)
greet() {
    echo "Hello!"
}
```

Both define a function called `greet` that does exactly the same thing. They behave identically in Bash.

✅ **Best Practice:** Prefer `name() { }` (Form 2). It's the POSIX-standard form, meaning it also works in `sh`/`dash` and other shells that don't understand the `function` keyword at all. Since Module 6 taught you that `/bin/sh` on Ubuntu is `dash`, writing the more portable form is a habit that pays off the moment a script needs to run somewhere other than Bash. This course will use `name() { }` throughout, but you should recognize the `function` keyword form when you see it in other people's scripts.

⚠️ **Warning:** You can technically combine both (`function greet() { ... }`), but this is redundant and non-portable — pick one style and stick with it.

### 3. Calling a function

Once defined, you call a function exactly like you'd run any command — by name:

```bash
greet         # calls the function, prints "Hello!"
```

⚠️ **Critical rule — order matters:** Bash reads a script top to bottom. A function must be **defined before** the point in the script where you call it. Calling `greet` on line 2 when it's not defined until line 10 will fail with "command not found," because as far as Bash is concerned at line 2, no such function exists yet.

✅ **Best Practice:** Put all your function definitions near the top of the script, followed by the "main" logic that calls them, so the whole script reads top-to-bottom like a table of contents followed by the story.

### 4. Parameters inside a function — they're the function's own, not the script's

This is one of the most important (and most commonly misunderstood) facts in this whole module: **when you call a function with arguments, that function gets its own private `$1`, `$2`, `$@`, and `$#`** — completely separate from the script's own positional parameters.

```bash
greet() {
    echo "Hello, $1!"   # this $1 belongs to greet, not to the script
}

greet "Wesam"           # $1 inside greet is "Wesam" for this call only
```

💡 **Analogy:** Think of calling a function like handing a sealed envelope to a coworker who has their own desk. Whatever's in *their* envelope (their own `$1`, `$2`) has nothing to do with what's sitting on *your* desk (the script's own `$1`, `$2`) — even if you happen to call your variable the same name.

🎯 **On the job:** This is exactly why functions are reusable. `greet "Wesam"` and, later in the same script, `greet "Delta"` each get their own fresh `$1` for that specific call — the function doesn't care what was passed in last time.

### 5. The `local` keyword and why it matters

By default, a variable you create *inside* a function is **global** — visible everywhere in the script, both before and after the function runs, and capable of silently overwriting a variable of the same name that already existed outside the function.

The `local` keyword fixes this: it declares a variable that only exists **inside the current function call**, and disappears the moment the function returns.

```bash
count=100

update_count() {
    local count=5      # this "count" only exists inside update_count
    echo "Inside function: $count"
}

update_count
echo "Outside function: $count"
```

Expected output:
```
Inside function: 5
Outside function: 100
```

Without `local`, the second `echo` would print `5` — the function would have silently overwritten the script's own `count` variable.

💡 **Analogy — scope is a fence:** `local` puts a fence around a variable so it only exists inside the function's own yard. Without the fence, every variable you create in a function leaks out into the shared yard of the whole script, where it can collide with (and overwrite) anything of the same name.

✅ **Best Practice:** Declare **every** variable you create inside a function as `local`, unless you have a specific, deliberate reason to modify a global variable (which is rare and should be a conscious choice, not an accident).

### 6. Getting a value back out of a function: `return` vs `echo`

Bash gives you two completely different mechanisms for a function to communicate something back to whatever called it, and conflating them is the single most common function-related bug beginners write.

- **`return N`** — sets the function's **exit status**, a whole number from **0 to 255**, exactly like a script's `exit` code (Module 6). It answers "did this succeed?", not "what's the answer?"
- **`echo` (or `printf`) + command substitution `$(...)`** — the function *prints* its result to standard output, and the caller captures that printed text using `$(function_name ...)`, the same command substitution you already know from Module 6. This is how a function "returns" a **string or number of any size**.

```bash
# return: reports success/failure only (0-255)
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0   # success = "yes, it's even"
    else
        return 1   # failure = "no, it's not"
    fi
}

if is_even 4; then
    echo "4 is even"
fi

# echo + $(): "returns" actual data
double() {
    echo $(( $1 * 2 ))   # prints the result — doesn't "return" it
}

result=$(double 21)
echo "Doubled: $result"   # Doubled: 42
```

### 7. The `return` exit-code trap — read this twice

This deserves its own numbered concept because it trips up almost every beginner at least once: **`return` can only hand back a whole number from 0 to 255.** It is not a general-purpose "give me back any value" mechanism — it is specifically, only, an exit code.

```bash
get_name() {
    return "Wesam"   # THIS IS BROKEN
}
```

Try to `return` a string, and Bash will throw an error like `return: Wesam: numeric argument required`, because `return` expects an integer, full stop. Even if you *do* pass a number, if it's outside 0-255, Bash silently **wraps it around** using modulo 256 — so `return 300` doesn't fail loudly, it quietly becomes `44` (`300 - 256`), which can produce a bug that's very hard to track down because nothing "errors" — it just silently gives you the wrong number.

⚠️ **Warning:** This wraparound behavior is a classic interview "gotcha" question precisely because it fails silently. If a function needs to report anything other than "did it succeed" (a plain yes/no, true/false), `return` is the wrong tool — reach for `echo` + `$()` instead.

✅ **Best Practice — the rule to memorize:**
- Use `return` (0-255) only to signal **success or failure**, so the function can be used directly in an `if`, `while`, or `&&`/`||` condition.
- Use `echo` + `$()` any time you need the function to hand back **actual data** — a string, a number outside 0-255, a filename, anything at all.
- A function can do **both**: print data with `echo` for the caller to capture, *and* separately `return 0`/`return 1` to indicate whether it succeeded.

### 8. Recursion — a brief mention

A function that calls **itself** is called **recursive**. Bash supports this, but it's used sparingly in real shell scripts — most everyday tasks (looping over servers, processing files) are handled more clearly and safely with the loops you already know from Module 7.

```bash
countdown() {
    local n="$1"
    if (( n <= 0 )); then
        echo "Liftoff!"
        return
    fi
    echo "$n..."
    countdown $(( n - 1 ))   # the function calls itself
}

countdown 3
```

🎯 **On the job:** You'll occasionally see recursion for things like walking a directory tree yourself, but in practice, most Bash scripts prefer plain loops — they're easier to read, easier to debug, and don't risk hitting Bash's function-call-depth limits. Know it exists; don't reach for it by default.

### 9. Indexed arrays — labeled storage bins

An **indexed array** is a variable that holds an ordered **list** of values, each accessible by a numeric position (its **index**), starting at `0`.

💡 **Analogy:** A plain variable is a single sticky note with one value on it. An **array** is a labeled shelf of numbered bins — bin `0`, bin `1`, bin `2`, and so on — where each bin holds one value, and you can grab any bin by its number, or grab everything on the shelf at once.

You declare one like this:

```bash
servers=(web01 web02 web03)
```

### 10. Accessing array elements

| Syntax | Meaning |
|---|---|
| `${servers[0]}` | The element at index `0` — `web01` (indexes start at **0**, not 1) |
| `${servers[@]}` | **Every** element, each preserved as its own separate word |
| `${servers[*]}` | **Every** element, joined into a **single** string when quoted |

This is exactly the same distinction you learned for `"$@"` vs `"$*"` back in Module 6 — and it matters for exactly the same reason:

```bash
for s in "${servers[@]}"; do echo "[$s]"; done   # one iteration per element — correct
for s in "${servers[*]}"; do echo "[$s]"; done   # one iteration total, over a joined blob
```

✅ **Best Practice:** Always quote array expansions — `"${servers[@]}"` — and default to `[@]` over `[*]`, for the same reasons you always quoted `"$@"` over `"$*"`.

### 11. Array length

```bash
echo "${#servers[@]}"   # 3 — the number of elements
```

⚠️ **Warning:** Don't confuse `${#servers[@]}` (count of elements) with `${#servers}` (length of just the *first* element, `servers[0]`, treated as a string) or `${#servers[0]}` (also the length of just element 0). The `[@]` inside the `#...[ ]` matters.

### 12. Appending to an array

```bash
servers+=(web04)          # adds one element to the end
servers+=(web05 web06)    # adds two more at once
```

### 13. Slicing an array

You can pull out a sub-range of elements using `${arr[@]:offset:length}` — start at index `offset`, take `length` elements:

```bash
subset=("${servers[@]:1:2}")   # starting at index 1, take 2 elements → web02 web03
```

### 14. Looping over an array

```bash
for server in "${servers[@]}"; do
    echo "Checking $server..."
done
```

This is the pattern you'll use constantly: build a list once, then loop over it with a `for` loop (from Module 7) to run the same logic against every element.

### 15. Associative arrays (Bash 4+) — key-value storage

An **associative array** (sometimes called a "map," "dictionary," or "hash" in other languages) stores values under **named keys** instead of numeric indexes. It requires Bash 4.0 or later (Ubuntu's default Bash has supported this for years, so you're safe on any current Ubuntu/Debian system).

You must **declare** it explicitly before assigning to it:

```bash
declare -A server_ips
server_ips[web01]="10.0.0.1"
server_ips[web02]="10.0.0.2"

# or all at once:
declare -A server_ips=( [web01]="10.0.0.1" [web02]="10.0.0.2" )
```

⚠️ **Warning:** Skipping `declare -A` and just writing `server_ips[web01]=...` creates a regular **indexed** array instead, where Bash tries (and typically fails, or misbehaves) to treat `web01` as a numeric index. Always `declare -A` first.

### 16. Looping over an associative array's keys

Because there's no guaranteed numeric order, you loop over an associative array using `${!server_ips[@]}` — the `!` here means "give me the **keys**, not the values":

```bash
for hostname in "${!server_ips[@]}"; do
    echo "$hostname -> ${server_ips[$hostname]}"
done
```

🎯 **On the job:** Associative arrays are perfect for configuration lookups — mapping environment names to URLs, server names to IPs, or status codes to human-readable messages — anywhere you'd otherwise write a long `case` statement (Module 7) just to look up a value.

### 17. String manipulation via parameter expansion

Bash can slice, search, replace, and reshape strings **without launching external tools** like `sed` or `awk` (those come in Module 9) — using special `${...}` syntax called **parameter expansion**. This matters on the job because it's faster (no external process) and works everywhere Bash does.

The core operations, introduced one at a time:

**Length:**
```bash
name="production"
echo "${#name}"        # 10 — number of characters
```

**Substring — `${var:offset:length}`:**
```bash
echo "${name:0:4}"     # "prod" — start at index 0, take 4 characters
echo "${name:4}"       # "uction" — start at index 4, take the rest
```

**Replace — first vs. all occurrences:**
```bash
path="/usr/local/bin"
echo "${path/local/opt}"    # /usr/opt/bin      — replaces the FIRST match only
echo "${path//\//-}"        # -usr-local-bin    — // replaces ALL matches (here, every /)
```

**Delete a prefix or suffix (pattern matching, not exact text):**

| Syntax | Removes | Greediness |
|---|---|---|
| `${var#pattern}` | Shortest matching **prefix** | non-greedy |
| `${var##pattern}` | Longest matching **prefix** | greedy |
| `${var%pattern}` | Shortest matching **suffix** | non-greedy |
| `${var%%pattern}` | Longest matching **suffix** | greedy |

```bash
file="archive.tar.gz"
echo "${file%.gz}"      # archive.tar   — strips shortest matching suffix ".gz"
echo "${file%%.*}"      # archive       — strips longest matching suffix starting at the first "."
```

💡 **Memory trick:** `#` is on the left of the keyboard's number row relative to `%` in the "prefix vs suffix" sense — but the real trick that sticks: **`#` looks like it's pointing at the front (prefix)**, **`%` sits at the visual "end" of a percentage figure (suffix)**. Doubling the symbol (`##`, `%%`) always means "be greedy — take as much as possible."

**Case conversion (Bash 4+):**
```bash
word="Hello"
echo "${word^^}"    # HELLO — uppercase everything
echo "${word,,}"    # hello — lowercase everything
echo "${word^}"     # Hello — uppercase just the first character
```

**Default values and "assign if unset":**

| Syntax | Behavior |
|---|---|
| `${var:-default}` | If `var` is unset/empty, **use** `default` for this expression — doesn't change `var` itself |
| `${var:=default}` | If `var` is unset/empty, **assign** `default` to `var` (changes it) and use that value |
| `${var:?error message}` | If `var` is unset/empty, print `error message` to stderr and **exit** the script |

```bash
echo "${environment:-staging}"     # prints "staging" if $environment is unset, without setting it
: "${environment:=staging}"        # sets $environment to "staging" if it was unset
: "${api_key:?API key is required}"   # exits with an error if $api_key is empty or unset
```

🎯 **On the job:** `${var:-default}` is everywhere in real deployment scripts — `region="${AWS_REGION:-us-east-1}"` means "use whatever's configured, or fall back to a sane default" in a single line.

---

## Detailed Explanations

### Why `return` really can't return a string — the mechanism underneath

A function's `return` value occupies the exact same slot as a script's `exit` code — it's stored by the shell as `$?`, an 8-bit unsigned integer. That's a hardware/OS-level convention across all of Unix, not a Bash design choice: every process, and every function inside a shell, communicates "how did it go?" through the same narrow 0-255 channel. There is no separate "return a string" channel — it simply doesn't exist. `echo`/`printf` + `$()`, by contrast, uses an entirely different pipe: standard output, which can hold arbitrarily long text. That's why "returning a value" from a Bash function always really means "printing a value and having the caller capture it" — you're routing data through the same channel any command uses to produce output, not through the exit-code channel at all.

### `local` scope and nested function calls

`local` scopes a variable to the current function **call**, which matters if a function calls another function:

```bash
outer() {
    local x="outer value"
    inner
    echo "outer sees: $x"   # still "outer value" — inner's local x didn't leak out
}

inner() {
    local x="inner value"
    echo "inner sees: $x"
}

outer
```

Each function call gets its own `local` scope layer — `inner`'s `local x` doesn't touch `outer`'s `x`, even though they share the same name.

### `${arr[@]}` / `${arr[*]}` and word-splitting, side by side with `${!arr[@]}`

| Expansion | Gives you |
|---|---|
| `${arr[@]}` | All **values**, each a separate word when quoted |
| `${arr[*]}` | All **values**, joined into one string when quoted |
| `${!arr[@]}` | All **indexes** (indexed array) or **keys** (associative array) |
| `${#arr[@]}` | The **count** of elements |

Note the `!` and `#` are doing different jobs here even though they both sit right before the array name: `!` means "give me the keys/indexes instead of the values," and `#` means "give me a count," and they can even combine in less common expressions — but for this module, knowing these four forms covers the overwhelming majority of real-world array usage.

### Pattern matching in `#`/`##`/`%`/`%%` is glob-style, not regex

The "pattern" in `${var#pattern}` and friends uses the same simple wildcard rules as filename globbing (`*`, `?`, `[...]` — familiar from earlier modules), **not** full regular expressions. `${file%%.*}` works because `.*` here means "a literal dot, followed by anything" as a glob, not a regex. True regex matching in Bash comes later, in Module 9 and beyond.

---

## Practical Examples

### Example 1 — Defining and calling functions, with parameters and `local`

```bash
#!/bin/bash
#
# greet-demo.sh — shows both function definition forms, parameters, and local scope

# Form 2 (preferred): name() { }
greet_server() {
    local hostname="$1"
    local role="$2"
    echo "Preparing $hostname (role: $role)..."
}

greet_server "web01" "frontend"
greet_server "db01" "database"
```

```bash
chmod +x greet-demo.sh
./greet-demo.sh
```

Expected output:
```
Preparing web01 (role: frontend)...
Preparing db01 (role: database)...
```

Line-by-line:
- `greet_server() { ... }` defines the function once, using the preferred POSIX-style form.
- `local hostname="$1"` and `local role="$2"` copy the function's *own* positional parameters into clearly-named local variables — notice `$1` and `$2` here belong to `greet_server`, not to `greet-demo.sh` itself.
- Each call, `greet_server "web01" "frontend"`, hands that specific call its own fresh `$1`/`$2` — the second call's values don't leak into or get confused with the first.

💡 **Tip:** Naming your local variables (`hostname`, `role`) instead of using `$1`/`$2` throughout the function body makes the function dramatically easier to read once it grows past a couple of lines — exactly the same habit you learned for script arguments in Module 6.

### Example 2 — The `return` trap, broken then fixed

**Broken — trying to `return` a string:**

```bash
#!/bin/bash
#
# broken-return.sh — demonstrates the return exit-code trap

build_filename() {
    local env="$1"
    return "backup-${env}.tar.gz"   # BROKEN — return only accepts 0-255
}

build_filename "production"
```

```bash
./broken-return.sh
```

Expected output:
```
./broken-return.sh: line 7: return: backup-production.tar.gz: numeric argument required
```

**Fixed — using `echo` + command substitution:**

```bash
#!/bin/bash
#
# fixed-return.sh — "returns" a string the correct way

build_filename() {
    local env="$1"
    echo "backup-${env}.tar.gz"   # prints the result; doesn't "return" it
}

filename=$(build_filename "production")
echo "Built filename: $filename"
```

```bash
chmod +x fixed-return.sh
./fixed-return.sh
```

Expected output:
```
Built filename: backup-production.tar.gz
```

Line-by-line:
- The broken version tries to hand a whole string to `return`, which only understands integers 0-255 — Bash errors immediately.
- The fixed version has `build_filename` **print** its result with `echo`, and the caller wraps the call in `$()` (command substitution, from Module 6) to **capture** that printed text into the `filename` variable.

⚠️ **Warning:** If `build_filename` had also printed something else along the way (like a debug message with a plain `echo "checking env..."`), that text would get captured into `filename` too — anything a function prints to standard output while its result is being captured ends up mixed into that captured string. Keep functions that "return" data via `echo` free of any other unrelated `echo` output.

### Example 3 — `return` used correctly: success/failure only

```bash
#!/bin/bash
#
# validate-env.sh — return used the way it's meant to be used: yes/no

is_valid_environment() {
    local env="$1"
    case "$env" in
        production|staging|development)
            return 0   # success — it's valid
            ;;
        *)
            return 1   # failure — it's not
            ;;
    esac
}

if is_valid_environment "staging"; then
    echo "staging is valid"
else
    echo "staging is NOT valid"
fi

if is_valid_environment "banana"; then
    echo "banana is valid"
else
    echo "banana is NOT valid"
fi
```

```bash
chmod +x validate-env.sh
./validate-env.sh
```

Expected output:
```
staging is valid
banana is NOT valid
```

Line-by-line:
- `is_valid_environment` never needs to "return data" — it only needs to answer a yes/no question, which is exactly what `return 0` / `return 1` is for.
- `if is_valid_environment "staging"; then ...` works directly, the same way `if some-command; then ...` worked back in Module 6 and 7 — Bash treats any command (including a function call) that `return`s `0` as "true" in an `if` condition.

✅ **Best Practice:** If you ever find yourself wanting to `return` anything other than a plain success/failure signal, that's your cue to switch to the `echo` + `$()` pattern from Example 2 instead.

### Example 4 — Looping over an indexed array

```bash
#!/bin/bash
#
# server-loop.sh — declares an indexed array and loops over it

servers=(web01 web02 web03)

echo "Total servers: ${#servers[@]}"

for server in "${servers[@]}"; do
    echo "Checking $server..."
done

# append a new one, then loop again
servers+=(web04)
echo "After adding web04, total: ${#servers[@]}"
```

```bash
chmod +x server-loop.sh
./server-loop.sh
```

Expected output:
```
Total servers: 3
Checking web01...
Checking web02...
Checking web03...
After adding web04, total: 4
```

Line-by-line:
- `servers=(web01 web02 web03)` declares an indexed array with three elements, indexes `0`, `1`, `2`.
- `${#servers[@]}` gives the element **count**, `3`.
- `for server in "${servers[@]}"` — quoted, using `[@]` — loops once per element, exactly like `"$@"` looped once per script argument in Module 6.
- `servers+=(web04)` appends a fourth element without disturbing the first three.

### Example 5 — Associative array for a config lookup

```bash
#!/bin/bash
#
# server-ips.sh — an associative array mapping hostnames to IPs

declare -A server_ips=(
    [web01]="10.0.0.1"
    [web02]="10.0.0.2"
    [db01]="10.0.0.10"
)

for hostname in "${!server_ips[@]}"; do
    echo "$hostname -> ${server_ips[$hostname]}"
done

echo "---"
echo "Looking up just web02: ${server_ips[web02]}"
```

```bash
chmod +x server-ips.sh
./server-ips.sh
```

Expected output (order of keys is not guaranteed):
```
web01 -> 10.0.0.1
db01 -> 10.0.0.10
web02 -> 10.0.0.2
---
Looking up just web02: 10.0.0.2
```

Line-by-line:
- `declare -A server_ips=(...)` declares an **associative** array and populates it with three key-value pairs in one step.
- `"${!server_ips[@]}"` expands to the **keys** (`web01`, `web02`, `db01`) — the `!` is what asks for keys instead of values.
- Inside the loop, `${server_ips[$hostname]}` looks up the value for whatever key the loop is currently on.

⚠️ **Warning:** Unlike indexed arrays, associative arrays have **no guaranteed order** when you loop over `${!server_ips[@]}`. If you need a specific, predictable order, sort the keys explicitly (a technique for Module 9, once you have `sort` available) rather than assuming insertion order.

### Example 6 — String manipulation cheat example: several expansions combined

```bash
#!/bin/bash
#
# string-toolbox.sh — combines several parameter-expansion operations at once

filename="Backup_Report_FINAL.TAR.GZ"

echo "Original:        $filename"
echo "Length:          ${#filename}"
echo "Lowercase:       ${filename,,}"
echo "Strip .GZ:       ${filename%.GZ}"
echo "Strip all ext:   ${filename%%.*}"
echo "Replace _ with -: ${filename//_/-}"

# defaulting example
region="${AWS_REGION:-us-east-1}"
echo "Region in use:   $region"
```

```bash
chmod +x string-toolbox.sh
./string-toolbox.sh
```

Expected output:
```
Original:        Backup_Report_FINAL.TAR.GZ
Length:          26
Lowercase:       backup_report_final.tar.gz
Strip .GZ:       Backup_Report_FINAL.TAR
Strip all ext:   Backup_Report_FINAL
Replace _ with -: Backup-Report-FINAL.TAR.GZ
Region in use:   us-east-1
```

Line-by-line:
- `${#filename}` counts every character, including underscores and dots.
- `${filename,,}` lowercases the entire string.
- `${filename%.GZ}` removes only the shortest matching suffix `.GZ` from the end, leaving `.TAR` in place.
- `${filename%%.*}` removes the **longest** matching suffix starting from the first `.` — stripping every extension at once, down to just the base name.
- `${filename//_/-}` replaces **every** underscore with a hyphen (`//` = all occurrences).
- `${AWS_REGION:-us-east-1}` falls back to `us-east-1` since `AWS_REGION` was never set in this script — without changing `AWS_REGION` itself.

### Example 7 — Full refactor: the deploy-script scenario from the Module Goal

This ties functions, both array types, and string manipulation together into the realistic scenario this module opened with: turning repetitive, copy-pasted logic into a clean, reusable script.

```bash
#!/bin/bash
#
# deploy-check.sh — refactored: one function, called once per server,
# instead of five copy-pasted blocks.

declare -A server_roles=(
    [web01]="frontend"
    [web02]="frontend"
    [db01]="database"
)

check_server() {
    local hostname="$1"
    local role="$2"
    local log_line

    log_line="[check] ${hostname} (${role^^})"   # ${role^^} uppercases the role

    if [[ "$role" == "database" ]]; then
        log_line="${log_line} — requires maintenance window"
    fi

    echo "$log_line"
    return 0
}

echo "Starting pre-deploy checks for ${#server_roles[@]} servers..."
echo

for hostname in "${!server_roles[@]}"; do
    check_server "$hostname" "${server_roles[$hostname]}"
done

echo
echo "All checks complete."
```

```bash
chmod +x deploy-check.sh
./deploy-check.sh
```

Expected output (order of servers is not guaranteed since it's an associative array):
```
Starting pre-deploy checks for 3 servers...

[check] web01 (FRONTEND)
[check] web02 (FRONTEND)
[check] db01 (DATABASE) — requires maintenance window

All checks complete.
```

Line-by-line:
- The associative array `server_roles` replaces what would otherwise be three (or, in a real fleet, dozens of) hardcoded, copy-pasted blocks — it's a single source of truth for "which server has which role."
- `check_server` is written **once** and holds all the shared logic — change the logic once, and every server benefits.
- `${role^^}` (case conversion) and the `[[ "$role" == "database" ]]` conditional (`[[ ]]`, covered in Module 7) combine to customize the message per role.
- The `for hostname in "${!server_roles[@]}"` loop calls `check_server` once per key, passing the hostname and its looked-up role as that call's own `$1`/`$2`.

🎯 **On the job:** This is the actual shape of most real infrastructure scripts you'll write or maintain — a small data structure (the array) describing *what* to act on, and a small function describing *how* to act on each one. Adding a fourth server means adding one line to `server_roles`; you never touch `check_server` at all.

---

## Common Pitfalls & Best Practices

- **Forgetting `local`.** Any variable set inside a function without `local` is global by default — it can silently overwrite a same-named variable elsewhere in the script, causing bugs that only show up much later, far from the function that actually caused them. Make `local` a reflex for every variable inside every function.
- **Trying to `return` a string, or a number outside 0-255.** `return` is exit-code-only, 0-255. A string causes an immediate "numeric argument required" error; a number like `300` silently wraps around (`300 % 256 = 44`) with no error at all — arguably worse, because nothing looks wrong. Use `echo` + `$()` for any real data.
- **Off-by-one array indexing.** Bash indexed arrays start at `0`, not `1`. `${servers[1]}` is the **second** element, not the first. This trips up people coming from languages or spreadsheets that start counting at 1.
- **Unquoted array expansion.** `${servers[@]}` without quotes is subject to the exact same word-splitting and globbing dangers as an unquoted `$file` from Module 6. Always write `"${servers[@]}"`.
- **Confusing `${arr[@]}` and `${arr[*]}`.** Same rule as `"$@"` vs `"$*"` — use `[@]`, quoted, unless you deliberately want everything joined into one string.
- **Assigning to an associative array key without `declare -A` first.** Skipping the declaration silently creates (or corrupts) a regular indexed array instead, since Bash tries to treat your text key as a number. Always `declare -A name` before the first assignment.
- **Calling a function before it's defined.** Bash reads top to bottom — define your functions first, call them later, never the reverse.
- **Assuming `${var:-default}` changes `var`.** It doesn't — it only substitutes `default` for *this one expression*. If you actually want to set `var` for the rest of the script, you need `${var:=default}` instead.

---

## Hands-on Exercise

**Task:** Write a script called `rename-logs.sh` that:

1. Defines a function `validate_input` that takes one argument (a filename) and, using `return`, reports success (`0`) if the filename is non-empty and failure (`1`) if it's empty — used purely as a yes/no check, the correct use of `return`.
2. Defines a function `clean_filename` that takes one filename, and using string manipulation:
   - Converts it to lowercase.
   - Replaces every space with an underscore.
   - Strips any `.tmp` suffix if present.
   - "Returns" the cleaned name the correct way (`echo` + `$()` from the caller).
3. Declares an indexed array of raw filenames: `("Report Draft.TMP" "Server Log.txt" "")`.
4. Loops over the array. For each filename:
   - Calls `validate_input`. If it fails (empty filename), print a warning and `continue` to the next one (skip it).
   - Otherwise, calls `clean_filename`, captures the result, and prints `"<original>" -> "<cleaned>"`.

Try writing this yourself before reading the solution below.

### Solution

```bash
#!/bin/bash
#
# rename-logs.sh — validates and cleans a list of raw filenames using
# functions, local scope, return-as-yes/no, and string manipulation.

# 1. Yes/no validation — correct use of return
validate_input() {
    local name="$1"
    if [[ -z "$name" ]]; then
        return 1   # failure — empty filename
    fi
    return 0       # success — non-empty
}

# 2. Cleans a filename and "returns" it via echo + $()
clean_filename() {
    local name="$1"
    local cleaned

    cleaned="${name,,}"          # lowercase everything
    cleaned="${cleaned// /_}"    # replace every space with an underscore
    cleaned="${cleaned%.tmp}"    # strip a trailing .tmp, if present

    echo "$cleaned"
}

# 3. Raw filenames, including one deliberately empty entry
raw_files=("Report Draft.TMP" "Server Log.txt" "")

# 4. Process each one
for original in "${raw_files[@]}"; do
    if ! validate_input "$original"; then
        echo "Warning: skipping empty filename"
        continue
    fi

    cleaned=$(clean_filename "$original")
    echo "\"$original\" -> \"$cleaned\""
done
```

```bash
chmod +x rename-logs.sh
./rename-logs.sh
```

Expected output:
```
"Report Draft.TMP" -> "report_draft"
"Server Log.txt" -> "server_log.txt"
Warning: skipping empty filename
```

Explanation: `validate_input` only ever needs to answer "is this usable or not," which is exactly what `return 0`/`return 1` is for — I check it with `if ! validate_input "$original"; then` (negating it with `!`, from Module 7's conditionals) to catch the failure case and `continue` past it. `clean_filename`, by contrast, needs to hand back an actual transformed **string**, so it uses `echo` and the caller captures it with `cleaned=$(clean_filename "$original")` — trying to `return` that string instead would have failed immediately, exactly like the broken example earlier in this module. Notice `${name,,}` lowercases first (I only need lowercase, not uppercase-first-letter, so `^^`/`^` weren't the right tool), then `${cleaned// /_}` replaces **every** space (not just the first, which is why I used `//` and not `/`), and finally `${cleaned%.tmp}` strips a trailing `.tmp` — note this only matches lowercase `.tmp`, which works here because I lowercased the string first.

💡 **Tip:** Notice the case-conversion step happened *before* the `.tmp` suffix-strip step, specifically so `%.tmp` (lowercase) would actually match what used to be `.TMP`. Order of operations matters when chaining string manipulations — think through what each step's input actually looks like.

✅ Exercise complete — you've written functions with correctly-scoped locals, used `return` for yes/no logic and `echo`+`$()` for real data, and chained several string-manipulation expansions together against a real array of input.

---

## ✅ Module Completion Checklist

- [ ] I can define a function using both the `function name { }` and `name() { }` forms, and call it like any other command
- [ ] I can explain that `$1`, `$2`, and `$@` inside a function refer to that function's own arguments, not the script's, and pass arguments into a function correctly
- [ ] I can use the `local` keyword to scope a variable to a function, and explain why forgetting it causes real bugs
- [ ] I can explain precisely why `return` in Bash is exit-code-only (0-255) and cannot "return" a string — and use `echo` + command substitution `$()` as the correct pattern for getting data back out of a function
- [ ] I can declare, populate, access, loop over, append to, and slice an indexed array, and explain the `${arr[@]}` vs `${arr[*]}` quoting distinction
- [ ] I can declare and use a Bash 4+ associative array with `declare -A`, and loop over its keys with `${!arr[@]}`
- [ ] I can manipulate strings using parameter expansion: length, substrings, search-and-replace, prefix/suffix removal, case conversion, and default-value handling
- [ ] I can refactor a repetitive, copy-pasted script into clean, reusable functions that operate over arrays of real-world data

## Next Step

Continue to [Module 9: Text Processing Power Tools (sed/awk/regex)](../module9-sed-awk-regex/)
