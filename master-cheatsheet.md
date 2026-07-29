# Master Cheat Sheet: Linux Bash — Zero to Hero

A single-page-per-topic quick reference for everything covered in this course. If you only keep one file from this course open while working, keep this one.

---

## 1. Navigation & File Operations (Modules 1-2)

| Command | Purpose |
|---|---|
| `pwd` | print working directory |
| `cd path`, `cd -`, `cd ~`, `cd ..` | change directory / previous / home / parent |
| `ls -la`, `ls -lh` | list all (incl. hidden) with details / human-readable sizes |
| `mkdir -p a/b/c` | create nested directories in one shot |
| `touch file` | create empty file / update timestamp |
| `cp -r src dst` | copy (recursive for directories) |
| `mv src dst` | move or rename |
| `rm -r dir`, `rm -rf dir` | remove recursively / force, no confirmation (⚠️ no undo) |
| `ln -s target link` | create a symbolic link |
| `du -sh dir`, `df -h` | directory size / filesystem free space |

**FHS quick map:** `/etc` configs · `/var` variable data & logs · `/home` user files · `/tmp` scratch space · `/usr/bin` most binaries · `/root` root's home.

---

## 2. Viewing & Finding Files (Module 3)

| Command | Purpose |
|---|---|
| `cat file`, `less file` | dump to stdout / paginated viewer (`q` quit, `/pat` search) |
| `head -n 20 file`, `tail -n 20 file` | first/last 20 lines |
| `tail -f file` | follow a growing file live (logs) |
| `wc -l file` | count lines |
| `grep -inr "pattern" .` | search recursively, case-insensitive, show line numbers |
| `find . -name "*.log" -mtime -1` | find files by name pattern modified in the last day |
| `find . -type f -size +100M` | find files over 100MB |
| `which cmd`, `whereis cmd` | locate an executable |

---

## 3. Permissions & Ownership (Module 4)

**Octal chart:** `4`=read `2`=write `1`=execute — sum them per owner/group/other.

| Octal | Meaning |
|---|---|
| `755` | owner rwx, group/other r-x (typical for scripts/dirs) |
| `644` | owner rw-, group/other r-- (typical for regular files) |
| `700` / `600` | owner-only access (secrets, private keys) |

| Command | Purpose |
|---|---|
| `chmod 755 file`, `chmod u+x file` | set permissions (octal / symbolic) |
| `chown user:group file` | change owner and group |
| `umask` | default permission mask for new files |
| SUID/SGID/sticky bit | `chmod u+s`, `g+s`, `+t` — special bits (see Module 4 for risk context) |
| `sudo cmd` vs `su` | run one command as root / switch user shells |

---

## 4. I/O Redirection & Pipes (Module 5)

| Operator | Meaning |
|---|---|
| `cmd > file` | overwrite stdout to file |
| `cmd >> file` | append stdout to file |
| `cmd < file` | feed file as stdin |
| `cmd 2> file` | redirect stderr only |
| `cmd > file 2>&1` | stdout AND stderr to the same file (order matters!) |
| `cmd 2>/dev/null` | discard errors |
| `cmd1 \| cmd2` | pipe stdout of cmd1 into stdin of cmd2 |
| `cmd \| tee file` | write to file AND pass through to stdout |
| `find . -name "*.tmp" \| xargs rm` | turn stdin lines into arguments |

---

## 5. Bash Scripting Fundamentals (Module 6)

```bash
#!/usr/bin/env bash
name="$1"                 # first positional argument, always quote when using
echo "Hello, ${name}"
echo "All args: $@"       # each arg stays separate — ALWAYS quote as "$@"
echo "Arg count: $#"
echo "Exit status of last command: $?"
```

| Symbol | Meaning |
|---|---|
| `$0 $1 $2 ...` | script name, positional args |
| `"$@"` vs `"$*"` | `"$@"` preserves each arg separately; `"$*"` joins into one string — use `"$@"` |
| `$#` | number of arguments |
| `$?` | exit code of last command |
| `$$` | PID of current script |
| `$(cmd)` | command substitution |
| `read -p "prompt: " var` | prompt and read user input |

---

## 6. Control Flow (Module 7)

```bash
if [[ -f "$file" ]]; then ...; elif [[ ... ]]; then ...; else ...; fi
for f in *.txt; do ...; done
for ((i=0; i<10; i++)); do ...; done
while read -r line; do ...; done < file
case "$env" in prod) ... ;; staging|dev) ... ;; *) ... ;; esac
```

**Prefer `[[ ]]` over `[ ]`/`test`** — no word-splitting surprises, supports `&&`/`||`/`=~` natively.

| Test | Meaning |
|---|---|
| `-f` / `-d` / `-e` | file exists / dir exists / anything exists |
| `-z` / `-n` | string is empty / non-empty |
| `-eq -ne -lt -le -gt -ge` | numeric comparisons |
| `= !=` | string comparison (numbers need `-eq` etc, not `=`) |

---

## 7. Functions, Arrays & Strings (Module 8)

```bash
greet() {
    local name="$1"        # always `local` inside functions
    echo "Hello, $name"    # "return" a string via echo + $(...)
}
result=$(greet "Ana")

arr=(a b c); arr+=(d)
echo "${arr[@]}"            # all elements
echo "${#arr[@]}"            # length

declare -A map=([env]=prod [region]=us-east)
echo "${map[env]}"
```

| Expansion | Meaning |
|---|---|
| `${#var}` | string length |
| `${var:offset:len}` | substring |
| `${var/old/new}` / `${var//old/new}` | replace first / all |
| `${var#pat}` / `${var##pat}` | strip shortest/longest matching prefix |
| `${var%pat}` / `${var%%pat}` | strip shortest/longest matching suffix |
| `${var^^}` / `${var,,}` | uppercase / lowercase |
| `${var:-default}` | use default if unset (doesn't assign) |
| `${var:=default}` | use default if unset AND assign it |

---

## 8. sed / awk / regex (Module 9)

| Task | Command |
|---|---|
| Replace first match per line | `sed 's/foo/bar/' file` |
| Replace all matches per line | `sed 's/foo/bar/g' file` |
| In-place edit with backup | `sed -i.bak 's/foo/bar/g' file` |
| Delete matching lines | `sed '/pattern/d' file` |
| Print column 2 | `awk '{print $2}' file` |
| Custom field separator | `awk -F',' '{print $1}' file.csv` |
| Sum a column | `awk '{sum+=$1} END {print sum}' file` |
| Count matches | `grep -c "pattern" file` |
| Extended regex | `grep -E "foo|bar"` |

**Regex quick reference:** `^` start · `$` end · `.` any char · `*` 0+ · `+` 1+ (ERE) · `?` 0-1 (ERE) · `[abc]` one of · `[^abc]` none of · `[a-z]` range.

---

## 9. Process Management (Module 10)

| Command | Purpose |
|---|---|
| `ps aux`, `pstree` | list all processes / process tree |
| `top`, `htop` | live resource monitor |
| `cmd &`, `jobs`, `fg`, `bg` | background a job / list / bring to foreground / resume in background |
| `nohup cmd &`, `disown` | survive terminal close |
| `kill PID`, `kill -9 PID` | SIGTERM (graceful) / SIGKILL (force, can't be caught) |
| `killall name`, `pkill name` | kill by process name |

---

## 10. Package Management & Monitoring (Module 11)

| Command | Purpose |
|---|---|
| `apt update` | refresh package index (do this first, always) |
| `apt upgrade` | actually install updates |
| `apt install/remove/purge pkg` | install / remove keeping configs / remove + configs |
| `apt autoremove` | clean up orphaned dependencies |
| `dpkg -L pkg`, `dpkg -S /path` | list files owned by package / find owning package |
| `uptime` | load average (compare to `nproc` core count) |
| `free -h` | memory usage |
| `df -h`, `du -sh dir` | disk free / directory size |

---

## 11. Networking (Module 12)

| Command | Purpose |
|---|---|
| `curl -X GET/POST -H "Header: v" -d 'body' url` | HTTP requests |
| `curl -O url`, `wget url` | download a file |
| `ssh user@host`, `ssh-keygen`, `ssh-copy-id user@host` | remote shell / generate keys / install public key |
| `scp file user@host:/path`, `rsync -avz src/ dst/` | copy over SSH / efficient sync |
| `ping host` | test reachability |
| `ss -tulpn` | list listening ports and owning processes |

---

## 12. Terminal Productivity (Module 13)

| tmux command | Purpose |
|---|---|
| `tmux new -s name` | start a named session |
| `Ctrl+b d` | detach (leaves session running) |
| `tmux attach -t name` | reattach |
| `Ctrl+b %` / `Ctrl+b "` | split pane vertically / horizontally |
| `Ctrl+b c` / `Ctrl+b n`/`p` | new window / next / previous |

**`.bashrc`** = every new interactive shell (aliases, functions, prompt). **`.bash_profile`/`.profile`** = login shells only (env vars that should exist once). When in doubt, put it in `.bashrc` and have `.bash_profile` source it.

---

## 13. Error Handling & Debugging (Module 14)

```bash
set -euo pipefail   # exit on error, error on unset var, fail pipelines on any stage failing
trap 'cleanup' EXIT # always run cleanup, success or failure
trap 'echo "error on line $LINENO"' ERR

die() { echo "FATAL: $*" >&2; exit 1; }

set -x   # print each command before running it (debug tracing)
```

**Known `set -e` gaps:** doesn't fire inside `if cmd; then`, doesn't fire on a command in `&&`/`||`, doesn't fire on a function used as a condition. `pipefail` is what catches failures inside a pipe that isn't the last command.

Check syntax without running: `bash -n script.sh`. Static analysis: `shellcheck script.sh`.

---

## 14. Automation & Scheduling (Module 15)

```
# crontab -e syntax:  min hour dom month dow command
0 2 * * *  /path/to/script.sh >> /var/log/script.log 2>&1
```

**Cron gotchas:** minimal `PATH` (use absolute paths), no TTY, output silently discarded unless redirected. **systemd timers** (`OnCalendar=`, paired `.timer`+`.service`, `systemctl list-timers`) are the modern preferred alternative — better logging via `journalctl`, `Persistent=true` catches missed runs.

---

## 15. Production Hardening (Module 16)

**Standard production script skeleton:** shebang → header comment → `set -euo pipefail` → `trap` cleanup → logging helper → input validation → functions → `main "$@"` at the bottom.

- Never build commands with unsanitized string interpolation or `eval` — use arrays for dynamic argument lists.
- Never pass secrets as CLI arguments (`ps aux` shows them to every user) — use env vars, stdin, or a restricted-permission file.
- Design for idempotency: `mkdir -p`, check-before-create, check-before-start.

---

## 16. Performance Tuning (Module 17)

- `time cmd` → `real` (wall clock) / `user` (CPU in your code) / `sys` (CPU in kernel calls).
- Avoid spawning an external process per loop iteration — do the whole job in one `awk`/`sed` pass instead.
- Avoid "Useless Use of Cat": `grep x file`, not `cat file | grep x`.
- Parallelize independent work: `xargs -P4`, or GNU `parallel`.

---

## 17. Security Auditing One-Liners (Module 18)

```bash
find / -xdev -type f -perm -0002                      # world-writable files
awk -F: '$3 == 0 {print $1}' /etc/passwd               # non-root UID 0 accounts
find / -xdev -type f \( -perm -4000 -o -perm -2000 \)  # SUID/SGID binaries
grep -iE '^PermitRootLogin|^PasswordAuthentication' /etc/ssh/sshd_config
ss -tulpn                                              # listening ports
apt list --upgradable                                  # patch status
grep 'Failed password' /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

---

## 18. Bash + Docker (Module 19)

```bash
#!/usr/bin/env bash
set -euo pipefail
: "${DATABASE_URL:?DATABASE_URL is required}"   # fail fast if unset

# wait-for-dependency loop, always with a timeout
until pg_isready -h "$DB_HOST" || [[ $SECONDS -gt 30 ]]; do sleep 1; done

exec "$@"   # <-- CRITICAL: hands PID 1 + signal handling to the real app
```

**Why `exec` matters:** without it, bash stays PID 1 and doesn't forward `SIGTERM`, so `docker stop` waits out the full timeout and force-kills instead of shutting down gracefully.

---

## 🔁 Key Workflows (memorize these)

**The Getting Unstuck Workflow** — confused by a command: `man cmd` → `cmd --help` → `tldr cmd` → search ExplainShell.com for a full pipeline.

**The Safe Delete Workflow** — before any `rm -rf`: (1) `echo` the glob/path first to see exactly what will match (2) run `ls` on it (3) only then swap `ls`/`echo` for `rm -rf` (4) never run it as a copy-pasted one-liner you haven't read.

**The Log Investigation Workflow** — chasing down an error: `tail -f` to watch live → `grep` to find the pattern → `sort | uniq -c | sort -rn` to rank by frequency → `awk` to extract just the fields you need.

**The New Script Bootstrap Workflow** — every new script: shebang → description comment → `chmod +x` → `set -euo pipefail` → `trap` cleanup → `main "$@"` at the bottom.

**The Safe `sed -i` Edit Workflow** — before editing in place: (1) test the command WITHOUT `-i` first (2) run with `-i.bak` (3) `diff file file.bak` to confirm the change is what you expected (4) delete the `.bak` once satisfied.

**The Graceful Shutdown Workflow** — stopping a process: (1) identify it with `ps`/`pgrep` (2) send `SIGTERM` first (3) wait a few seconds (4) escalate to `SIGKILL` only if it's still alive.

**The Production Promotion Checklist** — before a script touches production: `set -euo pipefail` + `trap` in place → no hardcoded secrets → inputs validated → idempotent re-run safe → `shellcheck` clean → logging in place.

**The New Cron Job Workflow** — adding a scheduled task: (1) test the script manually first (2) use absolute paths everywhere (3) redirect output to a log file (4) confirm the schedule on crontab.guru (5) consider a systemd timer instead for anything production-critical.

**The New Container Entrypoint Workflow** — writing `entrypoint.sh`: (1) validate required env vars (2) wait for dependencies with a timeout (3) do any one-time setup (4) `exec "$@"` as the very last line.
