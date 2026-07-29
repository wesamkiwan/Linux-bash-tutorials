# Master Interview Prep: Linux Bash — Zero to Hero

Consolidated interview questions spanning the entire course. Each module's own interview file goes deeper on that module's specifics — this file is your final-review pass before an interview, organized to mirror how these topics actually come up: conceptual understanding, hands-on coding, system-design judgment, and the "gotchas" experienced interviewers reach for.

---

## 🟢 Conceptual Questions — Beginner

**Q: What's the difference between the shell, the terminal, and Bash?**
A: "The terminal is the window/emulator I'm typing into. The shell is the program that reads my commands and runs them — Bash is one specific shell, the default on most Linux distros. It's like the terminal is the dashboard, the shell is the engine, and Bash is a particular engine model."

**Q: How do you check what a command does without running it?**
A: "`man command` for the full manual, `command --help` for a quick options summary, or `tldr command` for practical examples if it's installed. I reach for `tldr` first when I just need a quick example, and `man` when I need to understand every flag."

**Q: What's the difference between an absolute and a relative path?**
A: "An absolute path always starts from `/` and means the same thing no matter where I currently am — `/var/log/syslog`. A relative path is resolved from my current directory, so `../logs/app.log` only makes sense relative to where I'm standing when I run it."

**Q: Explain the difference between `>` and `>>`.**
A: "`>` overwrites the target file completely — if it already had content, that's gone. `>>` appends to the end, preserving what's there. Using `>` when you meant `>>` is a classic way to accidentally wipe a log file."

**Q: What does `chmod 755` actually do?**
A: "It sets permissions using the octal system — read=4, write=2, execute=1, summed per owner/group/other. 755 breaks down to owner=7 (rwx), group=5 (r-x), other=5 (r-x). It's the standard permission set for an executable script or a directory that others need to browse but not modify."

---

## 🟡 Conceptual Questions — Intermediate

**Q: Why should you prefer `[[ ]]` over `[ ]` in Bash conditionals?**
A: "`[ ]` is actually the `test` command, and it's subject to word-splitting and globbing on unquoted variables, which causes subtle bugs — an empty or multi-word variable can break the whole condition. `[[ ]]` is a Bash keyword that handles this safely, and it also supports `&&`/`||`/regex matching with `=~` directly inside the brackets."

**Q: What's the difference between `"$@"` and `"$*"`?**
A: "Quoted, `\"$@\"` expands to each positional argument as its own separate word — exactly what you want when looping over arguments or passing them through to another command. `\"$*\"` joins everything into a single string separated by the first character of `IFS`. If you loop over `\"$*\"` with multi-word arguments, you get one iteration instead of several — that's the bug this trips people into."

**Q: How do you return a string from a Bash function?**
A: "You don't use `return` for that — `return` only accepts an integer 0-255 and is meant for exit-code-style signaling. To get a string value out, the function echoes it, and the caller captures that with command substitution: `result=$(my_function)`."

**Q: Explain `2>&1` and why the order in `command > file 2>&1` matters.**
A: "File descriptor 1 is stdout, 2 is stderr. `> file` first redirects stdout to point at the file. Then `2>&1` says 'point descriptor 2 at wherever descriptor 1 currently points' — which is now the file. If you reversed the order, `2>&1` would duplicate stderr to wherever stdout was pointing *at that moment* — the terminal — before stdout gets redirected to the file, so you'd end up with stderr going to the terminal and only stdout in the file."

**Q: What does `set -euo pipefail` do, and what does it NOT catch?**
A: "`-e` exits on any command failure, `-u` errors on referencing an unset variable, and `pipefail` makes a pipeline's exit status reflect ANY stage failing, not just the last one. What it doesn't catch: a failing command used directly as an `if`/`while` condition (that's expected to potentially fail), and a failing command inside `&&`/`||` chains, since those are explicitly testing exit status already."

**Q: Why is cron notorious for jobs that work manually but fail when scheduled?**
A: "Cron runs jobs with a minimal environment — a bare-bones `PATH` that doesn't match your interactive login shell's `PATH`, no TTY, and a different working directory than you might assume. The fix is always using absolute paths for both the script and any commands it calls, and never relying on `PATH` picking things up the way it does interactively."

---

## 🔴 Conceptual Questions — Advanced

**Q: Walk me through how you'd avoid command injection in a Bash script that takes user input.**
A: "The core rule is: never build a command by interpolating unsanitized input into a string that gets evaluated, and never use `eval` on anything derived from input. If I need dynamic arguments, I build them as an array — `args=(--flag "$user_value")` — and pass the array, `\"${args[@]}\"`, rather than concatenating into a string. Quoting variables consistently also prevents a value like `; rm -rf /` from being interpreted as a second command."

**Q: Why shouldn't you pass secrets as command-line arguments?**
A: "Any user on the box can run `ps aux` (or read `/proc/<pid>/cmdline`) and see the full command line of any running process, including yours — arguments aren't private. Secrets should come from an environment variable, be piped via stdin, or be read from a file with restricted permissions, none of which show up in `ps` output or shell history the same way."

**Q: What's the PID 1 problem in Docker, and why does `exec \"$@\"` at the end of an entrypoint script matter?**
A: "If bash itself stays running as PID 1 inside the container (instead of `exec`-ing into the real application), it doesn't automatically forward signals like `SIGTERM` to the app process the way an init system would. So `docker stop` sends `SIGTERM`, bash as PID 1 ignores it or doesn't relay it, Docker waits out its grace period, and eventually sends `SIGKILL` — the app never gets a chance to shut down cleanly. `exec` replaces the bash process image with the target program in-place, so it becomes PID 1 itself and receives signals directly."

**Q: How would you design alerting so it doesn't spam the same failure repeatedly?**
A: "I'd keep a small state file tracking the last time each specific check alerted. On a failure, I only send a new alert if either there's no prior recorded alert, or the cooldown window (say 30 minutes) has elapsed since the last one — otherwise I just log it locally. When a check that previously failed starts passing again, I check whether it has prior alert state; if so, I send a distinct 'recovered' notification and clear that state, so the next failure starts a fresh cooldown cycle."

**Q: What does idempotent mean in the context of a deployment or automation script, and why does it matter?**
A: "It means running the script twice (or twenty times) in a row produces the same end state as running it once — no duplicate resources, no crash on the second run because something already exists. It matters because automation gets re-run: a retried CI job, an accidentally-double-triggered cron job, a human re-running a script after a partial failure. A script that assumes it only ever runs once is a script that will eventually break something in production."

---

## 💻 Practical / Coding Questions

**Q1: Write a one-liner to find the 5 largest files under the current directory.**
```bash
find . -type f -printf '%s %p\n' | sort -rn | head -5
```

**Q2: Given `access.log` in Combined Log Format, count requests per status code, most frequent first.**
```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn
```

**Q3: Write a function that validates its single argument is a positive integer before using it.**
```bash
validate_positive_int() {
    local value="$1"
    if [[ ! "$value" =~ ^[0-9]+$ ]] || (( value <= 0 )); then
        echo "Invalid: '$value' is not a positive integer" >&2
        return 1
    fi
}
```

**Q4: Fix this script — it's supposed to loop over each filename argument, but breaks on names with spaces.**
```bash
# Broken:
for f in $*; do echo "$f"; done
# Fixed:
for f in "$@"; do echo "$f"; done
```
"The unquoted `$*` gets word-split on whitespace, so a single argument like `my file.txt` becomes two loop iterations. Quoting `\"$@\"` preserves each argument as originally passed."

**Q5: Write a safe `sed` in-place replacement workflow for changing a config value across many files.**
```bash
grep -rl "old_value" ./configs | while IFS= read -r f; do
    sed -i.bak "s/old_value/new_value/g" "$f"
done
# review with: for f in ./configs/*.bak; do diff "${f%.bak}" "$f"; done
# then: find ./configs -name '*.bak' -delete
```

---

## 🧩 Scenario / System-Design Questions

**Q: You've been asked to add automated nightly backups for a production database and app data directory. Walk me through your design.**
A: "I'd write a script that dumps the database and archives the data directory into a single timestamped, compressed file, generates a SHA-256 checksum of the finished archive (not before compression — I want to verify the actual artifact I'd restore from), and enforces a retention policy so old backups don't fill the disk. I'd harden it with `set -euo pipefail` and a `trap` that removes any partial archive if the script dies mid-run, so a failed backup never gets mistaken for a good one. I'd schedule it with a systemd timer rather than cron, since `Persistent=true` catches a missed run if the server was down at the scheduled time, and `journalctl` gives better failure visibility than cron's silent-by-default output handling. I'd also write a restore script and actually test it — a backup nobody has restored from is a backup you don't actually trust."

**Q: Design a deployment process for a single Docker host with zero downtime and safe rollback, without a full orchestrator like Kubernetes.**
A: "Blue/green on a single host: build the new image, start it under a temporary name alongside the still-running old container — the old one keeps serving traffic through the whole process. Poll the new container's healthcheck with a hard timeout. If it becomes healthy in time, only then do I stop the old container and promote the new one to the production name/port — that ordering is the entire point, since it guarantees the app was never down and never served from an unverified container. If the healthcheck times out, I remove the failed new container, leave the old one completely untouched, log clearly, and exit non-zero so CI marks the deploy as failed. I'd also keep a 'last known good' version recorded somewhere durable so a separate rollback script can revert quickly if a problem surfaces after cutover, not just during it."

**Q: A teammate says their monitoring script pages the on-call engineer every 5 minutes for a problem that's already being worked on. How would you fix this?**
A: "That's an alert-deduplication problem, not a monitoring-accuracy problem. I'd add a state file that records when each check last fired an alert, and only re-alert after a cooldown window has passed, even if the check keeps failing every run. I'd also add explicit recovery detection — when a previously-failing check starts passing again, send a distinct 'resolved' notification and clear its state, so the next time it fails, it's treated as a fresh incident rather than continuing an old cooldown."

**Q: How would you safely roll out a change to `/etc/ssh/sshd_config` across a fleet of servers without risking locking yourself out?**
A: "I'd never restart the SSH service blind. My checklist: keep an active SSH session open on a test server while making the change, validate the new config with `sshd -t` before restarting/reloading, restart (not just edit) on that one server first and confirm a *new* connection still works from a separate session before closing the original one, then roll out to the rest of the fleet — ideally through the same automation/security-audit tooling that checks `PermitRootLogin`/`PasswordAuthentication`, so drift gets caught automatically afterward."

---

## ⚠️ Gotcha Questions

**Q: What's wrong with `rm -rf $DIR/*` if `$DIR` happens to be unset?**
A: "It silently becomes `rm -rf /*` — wiping the root filesystem's contents (permissions/system protections permitting). This is exactly what `set -u` guards against: with it enabled, referencing an unset `$DIR` errors out immediately instead of quietly expanding to nothing."

**Q: Does `set -e` guarantee your script stops on any error?**
A: "No — it has well-known gaps. It won't fire if the failing command is directly the condition of an `if`/`while`, won't fire on a command as part of `&&`/`||`, and by default won't catch a failure in the middle of a pipeline (that's what `pipefail` is specifically for). `set -euo pipefail` closes most of the common gaps, but it's a strong default, not a guarantee against every failure mode."

**Q: If a background job was started with `command &` in an SSH session and you close the terminal, does it keep running?**
A: "Not by default — closing the session sends `SIGHUP` to the job, which typically kills it. You need `nohup command &` (ignores SIGHUP) or `disown` after backgrounding it, or better, run it inside `tmux`/`screen` so the whole session, not just one command, survives the disconnect."

**Q: Why can `kill -9` (SIGKILL) be dangerous for a database or file-writing process?**
A: "SIGKILL can't be caught, blocked, or handled — the process is terminated immediately with no chance to flush buffers, close file handles cleanly, or finish a write. That can leave data files in a corrupted or inconsistent state. Always try SIGTERM first and give the process a chance to shut down gracefully; reach for SIGKILL only when it's genuinely unresponsive."

**Q: What happens if you forget `-xdev` when running `find /` for a security audit?**
A: "`find` will cross into every mounted filesystem it encounters — network mounts, other disks, `/proc`, `/sys` — which is both extremely slow and can produce misleading or nonsensical results (permission bits on procfs entries don't mean the same thing as on a real disk). `-xdev` keeps `find` on the starting filesystem only."

---

## ⚡ Quick-Fire Rapid Review

- **Q: `>` vs `>>`?** A: Overwrite vs append.
- **Q: Prefer `[[ ]]` or `[ ]`?** A: `[[ ]]` — safer with unquoted/empty variables.
- **Q: How do you "return" a string from a function?** A: `echo` it, capture with `$(...)`.
- **Q: `"$@"` or `"$*"` in a for loop?** A: `"$@"` — keeps each argument separate.
- **Q: What flag makes a pipeline fail if any stage fails, not just the last?** A: `pipefail`.
- **Q: SIGTERM vs SIGKILL?** A: Graceful, catchable vs immediate, uncatchable.
- **Q: What does `nohup` protect against?** A: The job dying from SIGHUP when the terminal closes.
- **Q: `apt update` vs `apt upgrade`?** A: Refresh the index vs actually install updates.
- **Q: Best practice for passing secrets to a script?** A: Env var or file, never a CLI argument.
- **Q: Why checksum a backup after compression, not before?** A: You need to verify the actual artifact you'll restore from.
- **Q: What makes cron jobs fail that work fine manually?** A: Minimal PATH/environment — always use absolute paths.
- **Q: Why does `exec "$@"` matter in a Docker entrypoint?** A: So the app becomes PID 1 and receives signals directly.
- **Q: `time` command's three outputs?** A: `real` (wall clock), `user` (CPU in your code), `sys` (CPU in kernel).
- **Q: What's "Useless Use of Cat"?** A: `cat file | grep x` instead of `grep x file`.
- **Q: `-i` vs `-i.bak` with `sed`?** A: `-i` edits in place with no backup; `-i.bak` keeps the original as `.bak` first — always test without `-i` first.
- **Q: What command checks a script's syntax without running it?** A: `bash -n script.sh`.
- **Q: Static analysis tool for shell scripts?** A: `shellcheck`.
- **Q: `NR` and `NF` in awk?** A: Current record (line) number, and number of fields in the current record.
- **Q: Idempotent script — why does it matter?** A: Safe to re-run without side effects from duplication.
- **Q: `ss` vs `netstat`?** A: `ss` is the modern replacement, generally faster and always available.
