# 🎤 Module 16 Interview Prep — Production Scripting & Security Hardening

## Conceptual Questions

### 🟢 Beginner

**Q1: What does it mean for a script to be "production-ready," beyond just working correctly on your own machine?**

> "It means the script can survive three pressures a laptop script never faces: input it didn't expect, an environment it doesn't fully control, and being run more than once by someone else, possibly unattended. On my own laptop I control every input and I'm watching it run. In production, arguments might come from a web form filled out by a stranger, the script might run on a shared server other people can log into, and a scheduler or a nervous engineer might trigger it twice in a row. Production-ready means I've deliberately thought through all three of those, not just the happy path I tested."

**Q2: What is input validation, and why does it matter even if the script 'seems to work' without it?**

> "Input validation is checking that a value is actually what I expect — the right type, the right shape, actually existing — before I use it for anything. It matters because 'the script ran without an error' isn't the same claim as 'the input was valid.' A script that gets handed garbage and just quietly does the wrong thing with it, instead of stopping and saying so, is arguably worse than one that crashes loudly, because nobody notices the problem until real damage is already done. I always validate a path exists before touching it, and confirm a value is genuinely numeric before doing arithmetic on it."

**Q3: How do you avoid command injection in Bash?**

> "The two habits that cover almost every case: quote every variable expansion — always `"$var"`, never a bare `$var` — and avoid `eval` entirely, building dynamic commands with arrays instead of concatenating strings together. Unquoted variables let the shell word-split and glob-expand data that was never supposed to be reinterpreted, and `eval` re-parses a string as actual shell syntax, so any shell metacharacter an attacker manages to get into that string — a semicolon, a pipe, backticks — runs as a real command. Quoting plus arrays means untrusted data stays inert, literal data, no matter what characters it contains."

**Q4: Why shouldn't you pass secrets as command-line arguments?**

> "Because command-line arguments aren't private on a Linux system — any user who can run `ps aux` can see the full command line, including every argument, of any running process on the machine. If I run `./script.sh --password mysecret`, anyone else logged into that box can run `ps aux | grep script.sh` and see `mysecret` in plain text for as long as the process is alive. That's not a misconfiguration, it's just how process listing has always worked on Unix-like systems. The safe alternatives are an environment variable, reading from stdin, or a file with restricted permissions — none of which show up in a process listing the same way."

**Q5: What does idempotent mean, and why does it matter for a script?**

> "Idempotent means running an operation once, or running it five times, produces the same end result — repeat runs don't pile duplicate effects on top of the first one. It matters because sooner or later something will cause a script to run more than once when only one run was intended — a scheduler retry, someone re-triggering a job that looked stuck, a network blip causing a retry. If the script isn't idempotent, that accidental second run can send a duplicate email, append a duplicate config line, or crash outright on something like 'directory already exists.' Designing for idempotency up front — checking before creating, using `mkdir -p`, checking if a service is already running before starting it — means a harmless accidental re-run stays harmless."

### 🟡 Intermediate

**Q6: Walk through exactly why `eval "grep $term $dir"` is exploitable, using a concrete malicious value.**

> "Say `term` is set to `TODO; rm -rf /home/alice`. Without `eval`, that whole string would just be one (weird, but inert) piece of data. But `eval` takes the fully-substituted string — `grep TODO; rm -rf /home/alice $dir` — and re-parses it as if it had been typed directly into the shell. At that point the semicolon isn't part of the search term anymore, it's a command separator, and `rm -rf /home/alice` runs as a completely independent command with whatever privileges the script has. The fix is to never use `eval` here at all — just call `grep -- "$term" "$dir"` directly, fully quoted, so the shell never gets a second chance to reinterpret the string as syntax."

**Q7: Explain the difference between validating input and sanitizing input, with an example of each.**

> "Validating means checking whether a value is acceptable and rejecting it outright if it isn't — for example, confirming a customer ID matches `^[0-9]+$` and calling `die` if it doesn't. Sanitizing means transforming a value into a safe form rather than rejecting it — for example, stripping or escaping characters from a filename before using it. In practice I lean toward validation-and-reject over sanitize-and-continue wherever possible, because sanitizing has a way of missing an edge case (a character nobody thought to strip), whereas a validation whitelist that says 'only these exact characters are allowed, reject everything else' is much harder to get subtly wrong."

**Q8: Why is a whitelist approach to input validation generally safer than a blacklist approach?**

> "A blacklist means listing the specific characters or patterns you consider dangerous and rejecting those — the problem is it's very easy to forget one. Someone lists semicolons and pipes and forgets backticks, or forgets that `$()` is also command substitution, or forgets a newline can act as a command separator too. A whitelist flips it: you define the narrow set of characters something is allowed to contain — say, letters, digits, underscore, dot, and hyphen for a filename component — and reject anything outside that set. It's much harder to accidentally leave a hole in 'only these five character classes are permitted' than in 'here's my best attempt at listing every dangerous character I can think of.'"

**Q9: How do you handle a secret that a script genuinely needs, from generation through use through cleanup?**

> "I avoid ever hardcoding it in the script or passing it as a CLI argument. I'd have it delivered via an environment variable — ideally from an actual secrets manager, or at minimum a file with `chmod 600` permissions that only the running user can read, sourced at the top of the script. Inside the script, I make sure it's never logged, never `echo`'d for debugging, and never handed to a child command as a literal `-p<secret>`-style flag if that command has its own safer option (like `MYSQL_PWD` instead of `mysql -p`). And I keep it out of shell history by never typing it directly at an interactive prompt — reading it from the environment or a file means it's never part of a command line Bash would save to `.bash_history` in the first place."

**Q10: What's the relationship between least privilege and the blast radius of a bug?**

> "Least privilege means a process only has the access it strictly needs — and the reason that matters isn't abstract security hygiene, it's that every other bug in the script inherits whatever privilege the script is running with. If a script runs entirely as root 'just in case,' then an injection vulnerability, an unvalidated path, or a typo'd `rm -rf` in that script doesn't just cause a limited, recoverable mistake — it causes a full-system one, because the buggy line has root's access to work with. If instead only the one line that genuinely needs elevation is wrapped in `sudo`, every other bug in the script is contained to whatever the unprivileged user could do anyway. Least privilege doesn't prevent bugs; it shrinks how much damage they can do when they happen."

### 🔴 Advanced

**Q11: A teammate says: "We can't have command injection, we never use `eval` anywhere in this codebase." Is that reassurance sufficient? Why or why not?**

> "No — `eval` is one specific way to get injection, but it's not the only one. Unquoted variable expansion alone is exploitable without `eval` anywhere in sight, purely through word-splitting and globbing: `cat $filename` with `filename` set to `/etc/passwd /etc/shadow` reads two files instead of one just from the unquoted space, and if `$filename` contained a glob character it would expand against real files in the current directory. More subtly, any place a script builds a command by string concatenation and eventually runs it — even without a literal `eval` keyword, for instance by writing the result to a temp script and executing that — reintroduces the same risk under a different name. The actual guarantee I'd want to see is 'every variable expansion is quoted, and every dynamic command is built with an array,' not just 'we don't call the `eval` builtin.'"

**Q12: You're reviewing a script that reads `DB_PASSWORD` from an environment variable — good — but then calls `psql "postgresql://user:$DB_PASSWORD@host/db"` to connect. Is this actually safe from the `ps aux` problem? Explain.**

> "It depends entirely on how `psql` itself handles that connection string internally, which is exactly the subtlety worth flagging: the goal was never 'get the secret out of my own script's arguments,' it's 'make sure the secret never becomes a literal argument to *any* process, including ones the script calls.' If `psql` takes that URL as a command-line argument — which this does, since it's passed positionally — then the password is embedded directly in `psql`'s own argument list, and `ps aux` would show the whole connection string, password included, regardless of the fact that it originated from an environment variable in the calling script. The genuinely safe version uses `psql`'s own environment-variable support (`PGPASSWORD`) so the password never appears as a literal argument to `psql` at all, the same fix pattern as preferring `MYSQL_PWD` over `mysql -p<password>`."

**Q13: Design the error-handling and idempotency strategy for a script that provisions a new user account, generates their SSH key, and registers it with a config management system — and must be safe to re-run if it fails partway through.**

> "First, I'd make every individual step independently idempotent rather than relying on an all-or-nothing wrapper: check whether the user account already exists before calling `useradd` (so a re-run doesn't error out on 'user already exists'), check whether an SSH key already exists at the expected path before generating a new one (so a re-run doesn't silently overwrite — and potentially orphan — a key that was already registered downstream), and check whether the config management system already has this user registered before registering again (many APIs error or duplicate on a second registration otherwise). Second, I'd combine `set -euo pipefail` with a `trap ... ERR` that logs exactly which step failed and at what line, so a partial failure produces a clear, actionable log entry instead of ambiguous silence. Third, I'd validate every input — username format, that the target host is reachable — before the first side-effecting command runs, since that's the point where Concept 2's validation-before-action principle matters most: nothing destructive should happen before every precondition is confirmed. The overall goal is that re-running the exact same script after a partial failure converges to the same correct end state, rather than needing a human to manually figure out which steps already happened."

---

## Practical/Coding Questions

**Q1: Fix this script so a value used in arithmetic can't crash or misbehave on bad input:**

```bash
#!/bin/bash
retry_count=$1
max_retries=$(( retry_count + 1 ))
echo "Will retry up to $max_retries times"
```

Solution:
```bash
#!/bin/bash
set -euo pipefail
retry_count="${1:-}"
[[ "$retry_count" =~ ^[0-9]+$ ]] || { echo "ERROR: retry_count must be a positive integer, got: '$retry_count'" >&2; exit 1; }
max_retries=$(( retry_count + 1 ))
echo "Will retry up to $max_retries times"
```
Explanation: the anchored regex `^[0-9]+$` confirms the whole argument is digits, start to end, before it's ever used in arithmetic expansion — rejecting empty strings, negative signs, decimals, and any embedded text up front, rather than letting `$(( ))` choke on or misinterpret something that was never validated.

**Q2: Rewrite this vulnerable snippet using an array instead of string concatenation + `eval`:**

```bash
cmd="tar czf"
[ "$verbose" = "true" ] && cmd="$cmd -v"
cmd="$cmd $archive_name.tar.gz $source_dir"
eval "$cmd"
```

Solution:
```bash
args=(tar czf)
[ "$verbose" = "true" ] && args+=(-v)
args+=("${archive_name}.tar.gz" "$source_dir")
"${args[@]}"
```
Explanation: each element of `args` stays a single, distinct argument regardless of what characters it contains — a `source_dir` with a space or a semicolon is passed to `tar` as one literal argument rather than being re-parsed as shell syntax. `"${args[@]}"` expands the array back into properly separated, individually-quoted words, and there's no `eval` anywhere for untrusted data to be reinterpreted by.

**Q3: Write a snippet that fails fast and clearly if a required secret environment variable is missing, without ever printing the secret's value.**

Solution:
```bash
: "${API_KEY:?API_KEY environment variable must be set}"
```
Explanation: `:` is the no-op builtin; its argument still gets parameter-expanded, so `${API_KEY:?message}` triggers Bash's built-in "print message and exit non-zero" behavior if `API_KEY` is unset or empty, without the script ever needing to reference — let alone print — the actual key value. This is the same fail-fast idiom used for `BUILD_DIR` in Module 14, applied here specifically to a secret.

**Q4: Given this script, identify why it isn't idempotent and fix it:**

```bash
#!/bin/bash
mkdir /opt/myapp/releases/v1
echo "export APP_ENV=production" >> /opt/myapp/config.env
```

Solution:
```bash
#!/bin/bash
set -euo pipefail
mkdir -p /opt/myapp/releases/v1

line="export APP_ENV=production"
config_file="/opt/myapp/config.env"
touch "$config_file"
grep -qxF "$line" "$config_file" || echo "$line" >> "$config_file"
```
Explanation: plain `mkdir` errors with "File exists" on a second run — which, under `set -e`, would kill the whole script — so `mkdir -p` is used instead, since it silently succeeds whether or not the directory already exists. The `echo ... >> config.env` line unconditionally appends every time it runs, so a second run would leave two identical `export APP_ENV=production` lines in the file; checking `grep -qxF "$line" "$config_file"` first (exact whole-line, literal-string match) means the line is only appended if it isn't already there.

---

## Gotcha Questions

**Q1: "I removed the hardcoded password and now read it from an environment variable instead. We're good on secrets now, right?"**

> Trap: Getting the secret out of the script file is real progress, but it's not the whole story. The candidate needs to check what the script *does* with that environment variable next — if it hands the value to another command as a literal CLI argument (`some-cli --password "$PASSWORD"`), the secret is now exposed via `ps aux` on that child process's argument list, regardless of how cleanly it was sourced into the script itself. The actual requirement is that the secret never becomes a literal argument to *any* process anywhere in the chain — check every downstream command the script calls, not just the script's own argument list.

**Q2: "My script only uses `sudo` on one line, so it already follows least privilege, correct?"**

> Trap: Using `sudo` sparingly is good, but least privilege is broader than just where `sudo` appears. The candidate should also check: does the script itself, or any file it reads secrets from, have overly permissive file permissions (world-readable when it shouldn't be)? Does the *unprivileged* portion of the script still have more access than it needs — for example, read access to files unrelated to its job? A single well-placed `sudo` call is one piece of least privilege, not the entire practice — permissions on the script and its config files (Module 4's `chmod`) matter just as much.

**Q3: "I quoted every variable in my script, so I'm safe from command injection now, right?"**

> Trap: Quoting closes the word-splitting/globbing class of injection, which is the most common one — but it doesn't cover every case. If the script still uses `eval` anywhere, quoting the variable being passed *into* `eval` doesn't help, because `eval` re-parses its argument as shell syntax regardless of how carefully it was quoted going in — quoting prevents the shell from reinterpreting a variable during normal expansion, but `eval`'s entire job is to deliberately force a second round of interpretation on top of that. The fix isn't "quote it more carefully before handing it to eval" — it's removing the `eval` call entirely and using an array instead. The candidate should recognize that quoting and avoiding `eval` are two separate, both-necessary habits, not one solving the other.

---

## Quick-Fire Rapid Review

- **Q: What are the three pressures that make a script "production-ready"?** A: Unexpected input, an environment you don't fully control, and being run more than once.
- **Q: What's the single biggest cause of command injection?** A: Unquoted variables and/or `eval` re-parsing untrusted data as shell syntax.
- **Q: What should you use instead of `eval` for building dynamic commands?** A: Arrays (`args=(...); cmd "${args[@]}"`).
- **Q: Why is a CLI argument a bad place to put a secret?** A: Any user on the machine can see it in `ps aux` for the life of the process.
- **Q: Where should secrets come from instead?** A: Environment variables, stdin, or a file with `chmod 600` permissions.
- **Q: What does idempotent mean?** A: Running an operation once or many times produces the same end result.
- **Q: What's the idempotent alternative to plain `mkdir`?** A: `mkdir -p` — succeeds silently if the directory already exists.
- **Q: How do you check if a config line is already present before appending it?** A: `grep -qxF "$line" "$file" || echo "$line" >> "$file"`.
- **Q: What does least privilege mean for `sudo` usage in a script?** A: Wrap only the specific line(s) that need elevation, not the whole script.
- **Q: What permission mode should a script with sensitive logic have?** A: `chmod 700` (owner: rwx, nobody else: anything).
- **Q: What permission mode should a file holding secrets have?** A: `chmod 600` (owner: rw, nobody else: anything).
- **Q: What command sends a message to syslog from a script?** A: `logger`.
- **Q: What utility prevents a growing logfile from filling the disk over time?** A: `logrotate`.
- **Q: What's the safest way to validate a value is a positive integer before arithmetic?** A: `[[ "$value" =~ ^[0-9]+$ ]]`.
- **Q: Why is whitelisting generally safer than blacklisting for input validation?** A: It's easy to forget a dangerous character in a blacklist; a whitelist rejects everything not explicitly allowed.
- **Q: What's the fail-fast idiom for a required secret environment variable?** A: `: "${SECRET:?SECRET must be set}"`.
