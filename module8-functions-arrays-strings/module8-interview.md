# 🎤 Module 8 Interview Prep — Functions, Arrays & String Manipulation

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a Bash function, and why would you use one instead of just repeating commands?**

> "A function is a named block of code I define once and can call as many times as I need, just like any other command. I use one instead of repeating commands because it collapses duplication — if I have the same five lines of logic needed for five different servers, I write it once as a function and call it five times with different arguments, instead of maintaining five copies. If I find a bug or need to change the logic, I fix it in exactly one place."

**Q2: What are the two ways to define a function in Bash, and which do you prefer?**

> "There's the `function name { ... }` form using the `function` keyword, and the POSIX-style `name() { ... }` form without it. They behave identically in Bash, but I prefer the `name() { }` form because it's portable — it also works in `sh`/`dash` and other POSIX shells that don't recognize the `function` keyword at all. Since scripts sometimes get run with `sh` instead of `bash` even when that's not intended, writing the more portable form by default is a good habit."

**Q3: If a function is called `greet` and the script also has a variable called `$1` from its own arguments, does calling `greet "Alice"` affect the script's own `$1`?**

> "No. Inside `greet`, `$1` refers to whatever was passed into *that specific call* — `\"Alice\"` — and it's completely separate from the script's own positional parameters. Each function call gets its own private `$1`, `$2`, `$@`, and `$#`, scoped just to that call."

**Q4: What's the difference between an indexed array and an associative array?**

> "An indexed array stores an ordered list of values accessed by numeric position, starting at index `0` — `arr=(a b c)`, and `${arr[0]}` is `a`. An associative array, available in Bash 4 and later, stores values under named keys instead of numbers, like a dictionary or map — `declare -A arr=( [name]=\"Alice\" )`, and `${arr[name]}` is `\"Alice\"`. I use indexed arrays for plain ordered lists — like a list of servers — and associative arrays when I need to look values up by a meaningful name, like mapping a hostname to its IP address."

**Q5: How do you find out how many elements are in an array?**

> "`${#arr[@]}` gives the count of elements — for either an indexed or an associative array. It's easy to confuse with `${#arr}` or `${#arr[0]}`, which instead give the character-length of just the first element treated as a string — the `[@]` inside is what asks for a count of *elements* rather than characters."

### 🟡 Intermediate

**Q6: How do you return a string from a Bash function?**

> "You don't use `return` for that — `return` in Bash is exit-code-only, a whole number from 0 to 255, meant to signal success or failure, not to carry data. To get a string (or any real value) back out of a function, the function `echo`s (or `printf`s) the value to standard output, and the caller captures that output using command substitution: `result=$(my_function args)`. That's the standard, correct pattern — `return` and `echo`-plus-`$()` solve two different problems, and mixing them up is one of the most common Bash mistakes."

**Q7: Explain local vs. global variables inside a function.**

> "By default, any variable you assign inside a Bash function is global — visible, and modifiable, from anywhere else in the script, both before and after the function runs. That means a function can silently overwrite a variable of the same name that already existed outside it, which is a real source of bugs, especially in larger scripts. The `local` keyword fixes this: `local varname=\"value\"` scopes that variable to just the current function call — it's created fresh each call and disappears the moment the function returns, with zero effect on any same-named variable outside. My habit is to declare every variable inside a function as `local` unless I have a deliberate, specific reason to modify something global."

**Q8: What's the quoting distinction between `${arr[@]}` and `${arr[*]}`, and why does it matter?**

> "It's the exact same distinction as `\"$@\"` versus `\"$*\"` for script arguments. Quoted, `\"${arr[@]}\"` expands to each array element as its own separate word — looping `for x in \"${arr[@]}\"` runs once per element. Quoted, `\"${arr[*]}\"` joins every element into a single combined string, separated by a space by default — looping over that runs exactly once, over the whole joined blob. It matters because if any element contains a space, using `[*]` where you meant `[@]` silently merges or mis-splits your data. My default is always `\"${arr[@]}\"`, quoted."

**Q9: How do you loop over the keys of an associative array?**

> "`${!arr[@]}` — the `!` in front means 'give me the keys (or indexes), not the values.' So `for key in \"${!arr[@]}\"; do echo \"$key -> ${arr[$key]}\"; done` loops over every key and looks up its value inside the loop. It's worth knowing there's no guaranteed order when iterating an associative array's keys — if a specific order matters, the keys need to be sorted explicitly."

**Q10: Walk me through what `${filename%%.*}` does.**

> "It removes the longest matching suffix from `filename` that matches the pattern `.*` — a literal dot followed by anything. Because it's the double-percent form, it's greedy: it matches as much as possible starting from the *first* dot in the string, effectively stripping every extension at once. So `report.tar.gz` becomes `report`. If I used the single-percent form, `${filename%.*}`, it would strip only the *shortest* matching suffix, which effectively means everything after the *last* dot — `report.tar.gz` would become `report.tar` instead."

### 🔴 Advanced

**Q11: A function needs to both validate its input and, if valid, transform it into a new string. How would you structure that cleanly?**

> "I'd split the two responsibilities into two functions rather than cramming both into one. A `validate_x` function that only ever uses `return 0`/`return 1` to answer 'is this valid or not,' checked with `if validate_x \"$input\"; then ...`. And a separate `transform_x` function that assumes valid input and `echo`s the transformed result, captured with `result=$(transform_x \"$input\")`. Keeping them separate means each function has one clear job and one clear way of communicating its result — a yes/no through the exit code, or data through standard output — rather than a single function trying to overload `return` to somehow mean both 'it worked' and 'here's the value,' which isn't possible in Bash anyway since `return` can't carry a string."

**Q12: What actually happens if you `return 300` from a function, and why is that dangerous?**

> "It doesn't error. Bash exit codes are stored as an 8-bit value, so `return 300` silently wraps around via `300 modulo 256`, which comes out to `44`. There's no warning, no error message — the function just reports `44` as its exit status, which almost certainly isn't what the caller expects if they were hoping the actual number `300` would somehow come through. It's dangerous specifically because it fails silently rather than loudly — a caller checking `if my_func; then` would see a non-zero exit code and correctly treat it as failure, but anyone assuming `$?` after the call equals the literal number they passed to `return` would be wrong, and nothing in the output would tip them off. It's the clearest argument for never using `return` for anything except a plain, small, intentional success/failure signal."

**Q13: How would you design a set of functions and arrays to replace a script that currently has ten nearly-identical, copy-pasted blocks for ten different servers?**

> "First, I'd identify exactly what varies between the ten blocks — probably a hostname, maybe a role or an IP — and store that varying data in an array: an indexed array if it's just a flat list of hostnames, or an associative array if I need a hostname mapped to something like its role or IP. Then I'd extract the shared logic that's currently copy-pasted into a single function, parameterized on whatever varies, with every internal variable declared `local`. The function would use `return 0`/`return 1` if the calling code just needs a yes/no, or `echo`+`$()` if it needs to hand back a constructed value like a log line or filename. Finally, I'd replace the ten blocks with a single loop over the array that calls the function once per element. The result is that adding an eleventh server means adding one line to the array — the function itself, and the loop, never need to change."

---

## Practical/Coding Questions

**Q1: Write a function `is_positive` that takes a number and returns success (0) if it's positive, failure (1) otherwise, and use it in an `if`.**

Solution:
```bash
is_positive() {
    local n="$1"
    if (( n > 0 )); then
        return 0
    else
        return 1
    fi
}

if is_positive -5; then
    echo "positive"
else
    echo "not positive"
fi
```
Explanation: This is a correct use of `return` — a plain yes/no question, checked directly in an `if` condition, exactly the way `if some-command; then` works for any other command's exit status.

**Q2: Write a function `to_slug` that takes a string and "returns" a URL-friendly, lowercase, hyphen-separated version of it (e.g. `"Hello World"` → `"hello-world"`).**

Solution:
```bash
to_slug() {
    local input="$1"
    local slug

    slug="${input,,}"       # lowercase everything
    slug="${slug// /-}"     # replace every space with a hyphen

    echo "$slug"
}

result=$(to_slug "Hello World")
echo "$result"   # hello-world
```
Explanation: Since the function needs to hand back an actual transformed string, `echo` + `$()` is the correct mechanism, not `return`. `${input,,}` lowercases first, then `${slug// /-}` replaces every space (the `//` form for all matches, not just the first).

**Q3: Write a script that declares an indexed array of three filenames and prints each one's extension using parameter expansion (no external tools).**

Solution:
```bash
#!/bin/bash
files=("report.pdf" "archive.tar.gz" "notes.txt")

for f in "${files[@]}"; do
    ext="${f##*.}"
    echo "$f -> $ext"
done
```
Expected output:
```
report.pdf -> pdf
archive.tar.gz -> gz
notes.txt -> txt
```
Explanation: `${f##*.}` deletes the longest matching prefix ending in a dot, leaving just whatever comes after the *last* dot — the file extension. Looping with `"${files[@]}"` (quoted) processes each filename as its own word, safe even if a filename contained a space.

**Q4: Write a function `get_config` that uses an associative array to look up a value by key, and falls back to a default if the key doesn't exist.**

Solution:
```bash
#!/bin/bash
declare -A config=( [timeout]="30" [retries]="3" )

get_config() {
    local key="$1"
    local default="$2"
    echo "${config[$key]:-$default}"
}

echo "$(get_config timeout 10)"   # 30 — key exists
echo "$(get_config max_conn 100)" # 100 — key missing, falls back to default
```
Explanation: `${config[$key]:-$default}` combines an associative-array lookup with the `:-` default-value expansion in one expression — if the key isn't present (or is empty), the expansion falls back to `$default` instead. The function `echo`s the result so the caller captures it with `$()`.

**Q5: Write a function `require_var` that exits the whole script with an error if a given variable name (passed by value) is empty, using `${var:?message}`.**

Solution:
```bash
#!/bin/bash
api_key=""

: "${api_key:?ERROR: api_key must be set}"

echo "This line never runs if api_key was empty."
```
Expected output (exact wording/line number may vary slightly by Bash version):
```
bash: line 3: api_key: ERROR: api_key must be set
```
(and the script exits immediately with a non-zero status)

Explanation: `${api_key:?message}` triggers Bash's built-in "die if unset/empty" behavior — it prints `message` to standard error and terminates the **whole script** (not just a function), not merely the current expression. The leading `:` is Bash's no-op built-in command — it's there purely so the expansion has somewhere safe to happen without actually running anything else.

---

## Gotcha Questions

**Q1: "My function does `return $result` where `$result` is a string built by concatenating several values, and it fails with a cryptic error. What's going on?"**

> Trap: The candidate needs to recognize this as the return-exit-code trap, not a syntax typo. `return` in Bash strictly requires an integer from 0 to 255 — passing it any non-numeric string produces `return: <value>: numeric argument required` and the function's actual exit status ends up undefined/error rather than what was intended. The fix is to stop using `return` for this entirely: `echo "$result"` inside the function, and capture it at the call site with `captured=$(my_function ...)`. `return` communicates success/failure only; it was never meant to carry arbitrary data.

**Q2: "I looped over my array with `for item in ${arr[@]}` (no quotes) and it worked fine in testing, but broke the moment an element had a space in it, splitting into extra iterations. Isn't `[@]` supposed to keep elements separate?"**

> Trap: `[@]` only preserves multi-word elements as single units when it's **quoted** — `"${arr[@]}"`. Without quotes, `${arr[@]}` is subject to the exact same word-splitting as any other unquoted expansion: Bash re-splits every element on whitespace before handing them to the loop, so an element containing a space becomes two separate loop iterations instead of one. The fix, as always: quote it — `for item in \"${arr[@]}\"; do ... done`. This is identical in spirit to the `\"$@\"` vs `$@` distinction from Module 6 — the special handling only kicks in when quoted.

**Q3: "I forgot `local` on a variable inside a function, and much later in the script, a completely unrelated section started behaving incorrectly using a variable with the same name. How is that possible if I never touched that variable in the later section?"**

> Trap: This tests whether the candidate understands that Bash function variables are global **by default** — `local` is opt-in, not automatic, unlike many other languages where function-scoped variables are the default and you opt into globals. Without `local`, any variable a function assigns becomes (or overwrites) a global variable for the rest of the script's execution, even after the function has returned. If an unrelated part of the script later happens to use the same variable name — completely coincidentally — it silently inherits whatever value the earlier function left behind. The fix, and the correct default habit, is to declare every variable created inside a function as `local`, so it's scoped to that call and vanishes afterward, with zero chance of colliding with anything outside.

---

## Quick-Fire Rapid Review

- **Q: The two ways to define a Bash function?** A: `function name { }` and the preferred, portable `name() { }`.
- **Q: What do `$1`/`$2`/`$@` refer to inside a function?** A: That function's own arguments for this specific call — not the script's.
- **Q: What keyword scopes a variable to a function call only?** A: `local`.
- **Q: What range of values can `return` communicate?** A: Integers 0-255 only — an exit code, not data.
- **Q: How do you actually get a string back out of a function?** A: `echo` the value, capture with `result=$(func)`.
- **Q: What happens if you `return 300`?** A: It silently wraps to `44` (`300 % 256`) — no error.
- **Q: What index does a Bash array start at?** A: `0`.
- **Q: How do you get the number of elements in an array?** A: `${#arr[@]}`.
- **Q: How do you append to an array?** A: `arr+=(newvalue)`.
- **Q: What declares an associative array?** A: `declare -A arr`.
- **Q: How do you loop over an associative array's keys?** A: `for k in "${!arr[@]}"; do ... done`.
- **Q: `${var/x/y}` vs `${var//x/y}`?** A: First replaces only the first match; `//` replaces all matches.
- **Q: `${var#pattern}` vs `${var##pattern}`?** A: `#` removes the shortest matching prefix; `##` removes the longest.
- **Q: `${var%pattern}` vs `${var%%pattern}`?** A: `%` removes the shortest matching suffix; `%%` removes the longest.
- **Q: How do you uppercase an entire string?** A: `${var^^}`.
- **Q: How do you lowercase an entire string?** A: `${var,,}`.
- **Q: `${var:-default}` vs `${var:=default}`?** A: `:-` substitutes a default without changing `var`; `:=` also assigns it to `var`.
- **Q: What does `${var:?message}` do if `var` is empty?** A: Prints `message` to stderr and exits the script.
