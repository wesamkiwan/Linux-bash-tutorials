# Capstone 3: Reference Solution — Security Audit Automation Script

This is one valid, complete solution. Your implementation doesn't need to match line-for-line — it needs to satisfy the requirements in [README.md](README.md).

## `audit.conf`

```bash
# Allowlisted SUID/SGID binaries (known-good, expected on this system)
ALLOWED_SUID=(
    "/usr/bin/sudo"
    "/usr/bin/passwd"
    "/usr/bin/su"
    "/usr/lib/openssh/ssh-keysign"
)

# Allowlisted listening ports (service: port)
ALLOWED_PORTS=(22 80 443)

# Failed-login thresholds (count of failures from a single IP in the log window)
FAILED_LOGIN_WARN=5
FAILED_LOGIN_CRITICAL=20
```

## `audit.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="${1:-./audit.conf}"
REPORTS_DIR="/var/log/security-audits"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
REPORT_FILE="${REPORTS_DIR}/audit-${TIMESTAMP}.txt"

[[ -f "$CONFIG_FILE" ]] || { echo "FATAL: config file not found: $CONFIG_FILE" >&2; exit 2; }
# shellcheck source=/dev/null
source "$CONFIG_FILE"

OVERALL_SEVERITY=0

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$REPORT_FILE"
}

die() {
    log "FATAL: $*"
    exit 2
}

report() {
    local level="$1" check_name="$2" detail="$3"
    local severity_value=0
    case "$level" in
        PASS)     severity_value=0 ;;
        WARN)     severity_value=1 ;;
        CRITICAL) severity_value=2 ;;
    esac
    log "[$level] $check_name: $detail"
    (( severity_value > OVERALL_SEVERITY )) && OVERALL_SEVERITY=$severity_value
}

require_root() {
    if [[ "$EUID" -ne 0 ]]; then
        report WARN "$1" "skipped — requires root/sudo to run completely"
        return 1
    fi
    return 0
}

check_world_writable() {
    require_root "world_writable" || return 0
    local hits
    hits=$(find / -xdev -type f -perm -0002 2>/dev/null | grep -v '^/proc' || true)
    if [[ -n "$hits" ]]; then
        report WARN "world_writable" "found world-writable file(s): $(echo "$hits" | tr '\n' ' ')"
    else
        report PASS "world_writable" "no world-writable files found"
    fi
}

check_uid_zero() {
    local hits
    hits=$(awk -F: '$3 == 0 { print $1 }' /etc/passwd | grep -v '^root$' || true)
    if [[ -n "$hits" ]]; then
        report CRITICAL "uid_zero" "non-root UID 0 account(s) found: $hits"
    else
        report PASS "uid_zero" "only root has UID 0"
    fi
}

check_suid_binaries() {
    require_root "suid_binaries" || return 0
    local found unexpected=()
    mapfile -t found < <(find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null)
    for path in "${found[@]}"; do
        local allowed=0
        for ok in "${ALLOWED_SUID[@]}"; do
            [[ "$path" == "$ok" ]] && allowed=1 && break
        done
        [[ "$allowed" -eq 0 ]] && unexpected+=("$path")
    done
    if (( ${#unexpected[@]} > 0 )); then
        report WARN "suid_binaries" "unexpected SUID/SGID binaries: ${unexpected[*]}"
    else
        report PASS "suid_binaries" "all SUID/SGID binaries are allowlisted"
    fi
}

check_sshd_config() {
    local sshd_config="/etc/ssh/sshd_config"
    [[ -f "$sshd_config" ]] || { report WARN "sshd_config" "sshd_config not found, skipping"; return; }

    local permit_root pass_auth
    permit_root=$(grep -iE '^\s*PermitRootLogin' "$sshd_config" | awk '{print $2}' | tail -1)
    pass_auth=$(grep -iE '^\s*PasswordAuthentication' "$sshd_config" | awk '{print $2}' | tail -1)

    if [[ "${permit_root:-yes}" == "yes" ]]; then
        report CRITICAL "sshd_config" "PermitRootLogin is enabled (or unset/default)"
    else
        report PASS "sshd_config" "PermitRootLogin is disabled ($permit_root)"
    fi

    if [[ "${pass_auth:-yes}" == "yes" ]]; then
        report WARN "sshd_config" "PasswordAuthentication is enabled — key-only auth recommended"
    else
        report PASS "sshd_config" "PasswordAuthentication is disabled"
    fi
}

check_cron_jobs() {
    require_root "cron_jobs" || return 0
    local entries
    entries=$(cat /etc/cron.d/* /var/spool/cron/crontabs/* 2>/dev/null | grep -vE '^\s*#|^\s*$' || true)
    if [[ -n "$entries" ]]; then
        report WARN "cron_jobs" "review cron entries for anything unexpected: $(echo "$entries" | wc -l) active job(s)"
    else
        report PASS "cron_jobs" "no active cron entries found"
    fi
}

check_listening_ports() {
    local listening unexpected=()
    mapfile -t listening < <(ss -tulnH 2>/dev/null | awk '{print $5}' | awk -F: '{print $NF}' | sort -un)
    for port in "${listening[@]}"; do
        local allowed=0
        for ok in "${ALLOWED_PORTS[@]}"; do
            [[ "$port" == "$ok" ]] && allowed=1 && break
        done
        [[ "$allowed" -eq 0 ]] && unexpected+=("$port")
    done
    if (( ${#unexpected[@]} > 0 )); then
        report WARN "listening_ports" "unexpected listening port(s): ${unexpected[*]}"
    else
        report PASS "listening_ports" "all listening ports are allowlisted"
    fi
}

check_package_updates() {
    local count
    count=$(apt list --upgradable 2>/dev/null | grep -c -v '^Listing' || true)
    if (( count > 0 )); then
        report WARN "package_updates" "$count package(s) have available updates"
    else
        report PASS "package_updates" "system is fully patched"
    fi
}

check_failed_logins() {
    local auth_log="/var/log/auth.log"
    [[ -f "$auth_log" ]] || { report WARN "failed_logins" "auth.log not found, skipping"; return; }

    local max_count
    max_count=$(grep 'Failed password' "$auth_log" 2>/dev/null \
        | awk '{ for(i=1;i<=NF;i++) if ($i=="from") print $(i+1) }' \
        | sort | uniq -c | sort -rn | head -1 | awk '{print $1}')
    max_count="${max_count:-0}"

    if (( max_count >= FAILED_LOGIN_CRITICAL )); then
        report CRITICAL "failed_logins" "an IP has $max_count failed login attempts — possible brute-force in progress"
    elif (( max_count >= FAILED_LOGIN_WARN )); then
        report WARN "failed_logins" "an IP has $max_count failed login attempts"
    else
        report PASS "failed_logins" "no concerning failed-login patterns (max $max_count from a single IP)"
    fi
}

main() {
    mkdir -p "$REPORTS_DIR"
    log "=== Security Audit: $(hostname) — $(date) ==="
    [[ "$EUID" -ne 0 ]] && log "NOTE: not running as root — some checks will be incomplete"

    check_world_writable
    check_uid_zero
    check_suid_binaries
    check_sshd_config
    check_cron_jobs
    check_listening_ports
    check_package_updates
    check_failed_logins

    log "=== Overall severity: $OVERALL_SEVERITY (0=PASS 1=WARN 2=CRITICAL) ==="
    log "Report saved to: $REPORT_FILE"
    exit "$OVERALL_SEVERITY"
}

main "$@"
```

### Why these design choices?

- **`OVERALL_SEVERITY` only ever ratchets upward.** Checks run in any order, and a later PASS must never erase an earlier CRITICAL. Comparing and taking the max on every `report` call, instead of at the end, means you can't accidentally miss updating it.
- **`require_root` degrades to a WARN instead of crashing.** A script that dies the moment it's run without `sudo` gets abandoned by whoever forgets `sudo` once. Reporting "this check was skipped, here's why" keeps the tool usable and honest about what it actually verified.
- **Every allowlist check follows the same shape**: gather what exists, filter out what's expected, report only what's left. That symmetry is what makes it easy to add a fifth or sixth allowlisted category later without inventing a new pattern each time.
- **Exit code IS the interface for automation.** A human reads the report file; a CI pipeline reads `$?`. Keeping those two audiences served by the same run, without one blocking the other, is why the exit code is computed from the same `report` calls that write the log.

## Scheduling

```bash
# crontab -e — run nightly at 3 AM as root (required for full checks)
0 3 * * * /usr/local/bin/audit.sh /etc/security-audit/audit.conf
```

```ini
# /etc/systemd/system/security-audit.service
[Unit]
Description=Nightly security audit

[Service]
Type=oneshot
ExecStart=/usr/local/bin/audit.sh /etc/security-audit/audit.conf
```

```ini
# /etc/systemd/system/security-audit.timer
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Testing Your Solution

A clean run on a reasonably hardened box:

```
[2026-03-15 03:00:00] === Security Audit: web01 — Sat Mar 15 03:00:00 UTC 2026 ===
[2026-03-15 03:00:00] [PASS] world_writable: no world-writable files found
[2026-03-15 03:00:00] [PASS] uid_zero: only root has UID 0
[2026-03-15 03:00:01] [PASS] suid_binaries: all SUID/SGID binaries are allowlisted
[2026-03-15 03:00:01] [PASS] sshd_config: PermitRootLogin is disabled (no)
[2026-03-15 03:00:01] [PASS] sshd_config: PasswordAuthentication is disabled
[2026-03-15 03:00:01] [PASS] cron_jobs: no active cron entries found
[2026-03-15 03:00:01] [PASS] listening_ports: all listening ports are allowlisted
[2026-03-15 03:00:02] [WARN] package_updates: 4 package(s) have available updates
[2026-03-15 03:00:02] [PASS] failed_logins: no concerning failed-login patterns (max 2 from a single IP)
[2026-03-15 03:00:02] === Overall severity: 1 (0=PASS 1=WARN 2=CRITICAL) ===
```

`echo $?` after this run should print `1`. Now add an intentionally risky binary and re-run to confirm a CRITICAL-level finding correctly forces exit code `2`:

```bash
sudo cp /bin/cat /tmp/fake-cat
sudo chmod u+s /tmp/fake-cat
sudo ./audit.sh audit.conf; echo "Exit code: $?"
```

You should see `[WARN] suid_binaries: unexpected SUID/SGID binaries: /tmp/fake-cat` (or `[CRITICAL]` if you decide unexpected SUID binaries outside `/usr` should be treated as critical in your own severity mapping) and a non-zero exit code — confirming the tool would correctly fail a CI gate.
