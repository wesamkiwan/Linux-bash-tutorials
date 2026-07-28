# 📋 Module 16 Cheat Sheet — Production Scripting & Security Hardening

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Input validation** | Checking that a value is what you expect (exists, right type, right shape) before using it |
| **Command injection** | Untrusted input getting re-interpreted as shell syntax instead of plain data |
| **`eval`** | Builtin that re-parses and runs a string as if typed directly into the shell — almost always avoid it |
| **Secret** | Any data whose exposure causes harm on its own (password, API key, token) |
| **Least privilege** | A process should have only the access it actually needs, nothing more |
| **Idempotent** | Running an operation once or many times produces the same end result |
| **`logger`** | Command that sends a message to syslog, viewable via `journalctl` |
| **`logrotate`** | System utility that archives/compresses/deletes old log files on a schedule |

## The Production Script Template (copy-paste skeleton)

```bash
#!/bin/bash
#
# script-name.sh — one-line description of what this does.
# Usage: script-name.sh --flag <value>
#
# Exit codes:
#   1 - invalid/missing arguments
#   2 - dependency missing
#   3 - operation failed
#
set -euo pipefail

LOG_FILE="/var/log/script-name.log"
log() { printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1" "$2" | tee -a "$LOG_FILE"; }
die() { log "ERROR" "$1"; exit "${2:-1}"; }

WORKDIR=""
cleanup() { [ -n "$WORKDIR" ] && rm -rf "$WORKDIR"; }
trap cleanup EXIT

parse_args() {
    while [ $# -gt 0 ]; do
        case "$1" in
            --flag) FLAG_VALUE="${2:-}"; shift 2 ;;
            -h|--help) echo "Usage: $0 --flag <value>" >&2; exit 1 ;;
            *) die "unknown argument: $1" ;;
        esac
    done
}

validate_inputs() {
    [ -n "${FLAG_VALUE:-}" ] || die "missing --flag" 1
}

main_logic() {
    : # real work goes here, in functions
}

main() {
    parse_args "$@"
    validate_inputs
    main_logic
    log "INFO" "Done."
}

main "$@"
```

## Security Checklist — Do / Don't

| Category | ✅ Do | ❌ Don't |
|---|---|---|
| Injection | Quote every variable: `"$var"` | Leave variables unquoted: `$var` |
| Injection | Build dynamic commands with arrays: `args=(...); cmd "${args[@]}"` | Concatenate command strings and `eval` them |
| Injection | Whitelist allowed characters with `[[ "$x" =~ ^[A-Za-z0-9_.-]+$ ]]` | Trust input just because it was supplied |
| Secrets | Read secrets from env vars, stdin, or `chmod 600` files | Hardcode secrets in the script |
| Secrets | Use a tool's own env-var support (e.g. `MYSQL_PWD`) | Pass a secret as a `-p<secret>` CLI flag |
| Secrets | Keep secrets out of `log()`/`echo` calls | `echo "password is $SECRET"` "just to debug" |
| Secrets | Be aware raw commands with secrets land in `.bash_history` | Type a raw password directly at the interactive prompt |
| Privilege | `sudo` only the one line that needs it | Run the whole script as root "just in case" |
| Permissions | `chmod 700` scripts with sensitive logic, `chmod 600` secret files | Leave sensitive scripts/configs world-readable |
| Idempotency | Check before create; `mkdir -p`; check service state before starting | Assume the script only ever runs once |

## Input Validation Snippets

```bash
# Path exists and is readable
[ -e "$path" ] || { echo "ERROR: not found: $path" >&2; exit 1; }
[ -r "$path" ] || { echo "ERROR: not readable: $path" >&2; exit 1; }

# Value is a positive integer, before doing arithmetic on it
[[ "$value" =~ ^[0-9]+$ ]] || { echo "ERROR: not a number: $value" >&2; exit 1; }

# Whitelist a "safe" string (letters, digits, underscore, dot, hyphen only)
[[ "$name" =~ ^[A-Za-z0-9_.-]+$ ]] || { echo "ERROR: invalid name: $name" >&2; exit 1; }
```

## Command Injection: Vulnerable vs. Fixed

| Vulnerable | Why it's dangerous | Fixed |
|---|---|---|
| `eval "grep $term $dir"` | `term`/`dir` re-parsed as shell syntax; `;`, `\|`, backticks execute | `grep -- "$term" "$dir"` |
| `cat $filename` | Unquoted → word-splitting + globbing | `cat -- "$filename"` |
| `cmd="$cmd --opt $val"` then `eval "$cmd"` | String concatenation + `eval` | Build an array: `args+=(--opt "$val")`; run `cmd "${args[@]}"` |

## Secrets: How to Pass Them

| Method | Visible via `ps aux`? | Visible in shell history? | Use it? |
|---|---|---|---|
| CLI argument (`--password xyz`) | ✅ Yes — plain text, for the process's whole life | ✅ Yes, if typed interactively | ❌ Never |
| Environment variable | ❌ No (unless the child process re-exposes it as its own arg) | ❌ No (if set via a sourced file, not typed inline) | ✅ Preferred |
| stdin | ❌ No | ❌ No | ✅ Preferred |
| File with `chmod 600` | ❌ No | ❌ No | ✅ Preferred |

```bash
# ps aux danger, demonstrated
sleep 300 --my-fake-password="SuperSecret123"   # terminal 1
ps aux | grep sleep                              # terminal 2 — password visible in plain text

# Safe pattern: fail fast if a required secret env var is missing
: "${API_KEY:?API_KEY environment variable must be set}"
```

## Least Privilege

```bash
# Bad: whole script needs root
sudo ./deploy.sh

# Good: only the one privileged line uses sudo
some_normal_command
sudo systemctl restart myapp   # only this needs elevation
another_normal_command
```

```bash
chmod 700 deploy.sh        # owner: rwx, nobody else: anything
chmod 600 secrets.env      # owner: rw, nobody else: anything (no execute)
```

## Logging Patterns

```bash
# Timestamped logfile helper
log() { printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$1" "$2" | tee -a "$LOG_FILE"; }
log "INFO" "Starting deployment"
log "ERROR" "Deployment failed"

# Send to syslog instead of / in addition to a file
logger -t myscript "Deployment started"
logger -t myscript -p user.err "Deployment failed"

# Read syslog entries back (systemd-based systems)
journalctl -t myscript
```

`logrotate` config snippet (`/etc/logrotate.d/myscript`) so a custom logfile doesn't grow forever:

```
/var/log/myscript.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}
```

## Idempotency Patterns

```bash
# Check before create
[ -d "$dir" ] || mkdir "$dir"
mkdir -p "$dir"                     # simpler: succeeds silently if it already exists

# Check before appending a config line
grep -qxF "$line" "$file" || echo "$line" >> "$file"

# Check before starting a service
systemctl is-active --quiet myapp || sudo systemctl start myapp

# Fresh working directory every run (no leftover state from a prior run)
rm -rf "$WORKDIR" && mkdir -p "$WORKDIR"
```

## 🔁 The Production Promotion Checklist Workflow

Run through this before any script touches production:

1. **Template check** — does it start with shebang, header comment, `set -euo pipefail`, and end with `main "$@"`?
2. **Validate every input** — every argument, env var, and file path checked before use, not assumed valid.
3. **Hunt for injection** — search the script for `eval` and any unquoted `$variable`; fix every one with quoting or arrays.
4. **Hunt for secrets** — search for hardcoded passwords/keys/tokens; move them to env vars, stdin, or a `chmod 600` file. Confirm no secret is ever passed as a literal CLI argument to any command, including ones the script calls internally.
5. **Check privilege scope** — is `sudo` used only on the specific lines that need it, not wrapping the whole script?
6. **Check permissions** — `chmod 700` on the script if it holds sensitive logic; `chmod 600` on any file holding secrets.
7. **Check logging** — does every meaningful step log a timestamped message? Is there a `logrotate` entry so the logfile won't grow forever?
8. **Check idempotency** — mentally (or actually) run it twice in a row. Does anything break, duplicate, or corrupt state?
9. **Run `shellcheck`** (Module 14) one more time — clean output before it ships.
