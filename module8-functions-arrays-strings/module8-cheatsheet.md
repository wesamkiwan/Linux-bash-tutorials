# 📋 Module 8 Cheat Sheet — Functions, Arrays & String Manipulation

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Function** | A named, reusable block of code, called like any other command |
| **Local variable** | A variable scoped to the current function call only, via `local` |
| **Global variable** | A variable visible everywhere in the script (the default unless you use `local`) |
| **Return code** | The 0-255 exit status a function reports with `return` — success/failure only |
| **Indexed array** | An ordered list of values accessed by numeric position, starting at `0` |
| **Associative array** | A key-value map (Bash 4+), declared with `declare -A` |
| **Parameter expansion** | The `${...}` syntax family used for string length, substrings, replace, and more |

## Function Syntax

| Form | Example | Notes |
|---|---|---|
| POSIX-style (preferred) | `name() { ...; }` | Portable — also works in `sh`/`dash` |
| `function` keyword | `function name { ...; }` | Bash-only; recognize it, prefer the other form for new code |
| Calling | `name arg1 arg2` | Called exactly like any command |
| Parameters inside | `$1`, `$2`, `$@`, `$#` | Belong to **this function call only** — not the script's own `$1`/`$2` |
| Scoping a variable | `local varname="value"` | Confines the variable to this function call; **use it for every function-local variable** |
| Success/failure result | `return 0` / `return 1` | 0-255 only — exit-code style, checked with `if func; then ...` |
| Data result | `echo "value"` then `result=$(func)` | The only correct way to "return" a string, number >255, or anything non-exit-code |

## Function Definition & Call Order

```bash
# 1. Define first
my_function() {
    local x="$1"
    echo "Got: $x"
}

# 2. Call after it's defined
my_function "hello"   # Bash reads top-to-bottom — never call before defining
```

## The `return` Trap — Quick Reference

| You want to... | Use | Why |
|---|---|---|
| Signal success/failure | `return 0` / `return 1` | Native to `if`/`while`/`&&`/`\|\|` — but **0-255 only** |
| Hand back a string, number, or filename | `echo "$value"` + `result=$(func ...)` | `return` errors on non-numeric input, and silently wraps numbers >255 via `% 256` |
| Do both | `echo` the data, then `return 0`/`1` separately | A function can print data AND report success/failure |

## Indexed Arrays

| Syntax | Meaning |
|---|---|
| `arr=(a b c)` | Declare with 3 elements, indexes `0`, `1`, `2` |
| `${arr[0]}` | Element at index `0` (indexes start at **0**) |
| `${arr[@]}` | All elements, each a separate word when quoted |
| `${arr[*]}` | All elements joined into one string when quoted |
| `${#arr[@]}` | Count of elements |
| `arr+=(d)` | Append one or more elements |
| `${arr[@]:1:2}` | Slice: start at index 1, take 2 elements |
| `unset 'arr[1]'` | Remove one element (leaves a gap in indexes) |
| `for x in "${arr[@]}"; do ...; done` | Loop over every element — **always quote** |

## `${arr[@]}` vs `${arr[*]}` — same rule as `"$@"` vs `"$*"`

| Form | Quoted behavior |
|---|---|
| `"${arr[@]}"` | Each element as its own separate word — **use this** |
| `"${arr[*]}"` | All elements joined into a single string |

## Associative Arrays (Bash 4+)

| Syntax | Meaning |
|---|---|
| `declare -A arr` | Declare an associative array (**required** before assigning by key) |
| `arr[key]="value"` | Set a value under a named key |
| `declare -A arr=( [k1]="v1" [k2]="v2" )` | Declare and populate in one step |
| `${arr[key]}` | Look up the value for `key` |
| `${!arr[@]}` | All **keys** (the `!` means "keys/indexes, not values") |
| `${#arr[@]}` | Count of key-value pairs |
| `for k in "${!arr[@]}"; do echo "$k -> ${arr[$k]}"; done` | Standard loop-over-keys pattern |

⚠️ No guaranteed order when looping over associative array keys.

## String Manipulation — Parameter Expansion Reference

The single most important table in this module — memorize this.

| Syntax | Operation | Example (`var="Hello World"`) | Result |
|---|---|---|---|
| `${#var}` | Length in characters | `${#var}` | `11` |
| `${var:offset}` | Substring from `offset` to end | `${var:6}` | `World` |
| `${var:offset:length}` | Substring, `length` chars from `offset` | `${var:0:5}` | `Hello` |
| `${var/pattern/repl}` | Replace **first** match | `${var/o/0}` | `Hell0 World` |
| `${var//pattern/repl}` | Replace **all** matches | `${var//o/0}` | `Hell0 W0rld` |
| `${var#pattern}` | Delete shortest matching **prefix** | `file="a.b.c"; ${file#*.}` | `b.c` |
| `${var##pattern}` | Delete longest matching **prefix** | `file="a.b.c"; ${file##*.}` | `c` |
| `${var%pattern}` | Delete shortest matching **suffix** | `file="a.tar.gz"; ${file%.*}` | `a.tar` |
| `${var%%pattern}` | Delete longest matching **suffix** | `file="a.tar.gz"; ${file%%.*}` | `a` |
| `${var^^}` | Uppercase everything | `${var^^}` | `HELLO WORLD` |
| `${var,,}` | Lowercase everything | `${var,,}` | `hello world` |
| `${var^}` | Uppercase first character only | `${var^}` | `Hello World` (unchanged here) |
| `${var,}` | Lowercase first character only | `${var,}` | `hello World` |
| `${var:-default}` | Use `default` if unset/empty — **doesn't change `var`** | `${missing:-fallback}` | `fallback` |
| `${var:=default}` | **Assign** `default` to `var` if unset/empty, then use it | `${missing:=fallback}` | sets and returns `fallback` |
| `${var:?message}` | Print `message` to stderr and **exit** if unset/empty | `${required:?must be set}` | exits if `required` is empty |
| `${var:+altvalue}` | Use `altvalue` **only if** `var` IS set/non-empty (opposite of `:-`) | `${set_var:+yes}` | `yes` if `set_var` has a value |

💡 **Memory aid:** `#` = prefix (front), `%` = suffix (end). Doubling the symbol (`##`, `%%`) = greedy (longest match).

## Common Pattern Recipes

```bash
# Get a file's extension
ext="${filename##*.}"

# Get a file's name without extension
base="${filename%.*}"

# Get a file's directory-free name (like basename)
name="${path##*/}"

# Get a directory path (like dirname)
dir="${path%/*}"

# Default a variable, without changing it
port="${PORT:-8080}"

# Require a variable to be set, or fail loudly
: "${API_KEY:?API_KEY must be set}"
```

## 🔁 The Function Refactor Workflow

Do this every time a script repeats the same logic 3+ times:

1. **Spot the repetition** — copy-pasted blocks that differ only by a hostname, filename, or a couple of values.
2. **Extract the shared logic into a function**, replacing the differing values with parameters (`$1`, `$2`, ...).
3. **Declare every variable inside the function as `local`** — no exceptions without a reason.
4. **Decide what the function needs to communicate**: a plain success/failure (`return 0`/`1`) or actual data (`echo` + `$()`) — never mix the two meanings into one `return`.
5. **Store the varying inputs in an array** (indexed for a plain list, associative for key-value data).
6. **Loop over the array**, calling the function once per element.
7. **Quote every array and parameter expansion** — `"${arr[@]}"`, `"$1"`, `"$hostname"`.
8. **Test with an edge case** — an empty element, a value with spaces — to catch quoting bugs early.

## 🔁 The String-Cleanup Workflow

Do this any time you need to normalize user input or filenames:

1. Lowercase or uppercase first if the transform is order-sensitive (e.g. before stripping an extension that might be mixed-case).
2. Strip unwanted characters/prefixes/suffixes with `#`/`##`/`%`/`%%`.
3. Replace remaining unwanted characters with `//`.
4. Validate the result isn't empty before using it (`${var:?message}` or an explicit `if [[ -z "$var" ]]`).
