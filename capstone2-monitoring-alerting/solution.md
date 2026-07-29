# Capstone 2: Reference Solution — Server Monitoring & Alerting System

This is one valid, complete solution. Your implementation doesn't need to match line-for-line — it needs to satisfy the requirements in [README.md](README.md).

## `monitor.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# ---- Configuration ----
LOAD_THRESHOLD_PER_CORE=1.5
MEM_THRESHOLD_PCT=90
DISK_THRESHOLD_PCT=85
DISK_PATH="/"
CRITICAL_PROCESSES=("nginx" "myapp")
WEBHOOK_URL="https://hooks.example.com/services/XXX/YYY/ZZZ"
COOLDOWN_SECONDS=1800
STATE_FILE="/var/lib/monitor/state.tsv"
LOG_FILE="/var/log/monitor.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

die() {
    log "FATAL: $*"
    exit 1
}

# ---- Checks: each prints a detail message and returns 0 (OK) or 1 (FAILING) ----

check_load() {
    local cores load_1m threshold
    cores=$(nproc)
    load_1m=$(awk '{print $1}' /proc/loadavg)
    threshold=$(awk -v c="$cores" -v t="$LOAD_THRESHOLD_PER_CORE" 'BEGIN { print c * t }')

    if awk -v l="$load_1m" -v th="$threshold" 'BEGIN { exit !(l > th) }'; then
        echo "load average $load_1m exceeds threshold $threshold ($cores cores)"
        return 1
    fi
    echo "load average $load_1m OK ($cores cores, threshold $threshold)"
    return 0
}

check_memory() {
    local pct
    pct=$(free | awk '/Mem:/ { printf "%.0f", ($2-$7)/$2 * 100 }')
    if (( pct >= MEM_THRESHOLD_PCT )); then
        echo "memory usage ${pct}% exceeds threshold ${MEM_THRESHOLD_PCT}%"
        return 1
    fi
    echo "memory usage ${pct}% OK"
    return 0
}

check_disk() {
    local pct
    pct=$(df -h "$DISK_PATH" | awk 'NR==2 { gsub("%","",$5); print $5 }')
    if (( pct >= DISK_THRESHOLD_PCT )); then
        echo "disk usage ${pct}% on $DISK_PATH exceeds threshold ${DISK_THRESHOLD_PCT}%"
        return 1
    fi
    echo "disk usage ${pct}% on $DISK_PATH OK"
    return 0
}

check_processes() {
    local missing=()
    for proc in "${CRITICAL_PROCESSES[@]}"; do
        pgrep -x "$proc" > /dev/null || missing+=("$proc")
    done
    if (( ${#missing[@]} > 0 )); then
        echo "missing critical process(es): ${missing[*]}"
        return 1
    fi
    echo "all critical processes running"
    return 0
}

# ---- State management ----

get_last_alert_time() {
    local check_name="$1"
    [[ -f "$STATE_FILE" ]] || { echo 0; return; }
    awk -F'\t' -v name="$check_name" '$1 == name { print $2; found=1 } END { if (!found) print 0 }' "$STATE_FILE"
}

set_alert_state() {
    local check_name="$1" now="$2"
    mkdir -p "$(dirname "$STATE_FILE")"
    touch "$STATE_FILE"
    grep -v "^${check_name}	" "$STATE_FILE" > "${STATE_FILE}.tmp" 2>/dev/null || true
    echo -e "${check_name}\t${now}" >> "${STATE_FILE}.tmp"
    mv "${STATE_FILE}.tmp" "$STATE_FILE"
}

clear_alert_state() {
    local check_name="$1"
    [[ -f "$STATE_FILE" ]] || return 0
    grep -v "^${check_name}	" "$STATE_FILE" > "${STATE_FILE}.tmp" 2>/dev/null || true
    mv "${STATE_FILE}.tmp" "$STATE_FILE"
}

# ---- Alerting ----

send_alert() {
    local check_name="$1" status="$2" detail="$3"
    local hostname payload
    hostname="$(hostname)"
    payload=$(cat <<EOF
{"text": "[${status}] ${hostname} — ${check_name}: ${detail} (at $(date -u '+%Y-%m-%dT%H:%M:%SZ'))"}
EOF
)
    if curl -sf -X POST -H 'Content-Type: application/json' -d "$payload" "$WEBHOOK_URL" > /dev/null; then
        log "Alert sent for ${check_name} (${status})"
    else
        log "WARNING: failed to send webhook alert for ${check_name} — check will still be tracked locally"
    fi
}

# ---- The core state machine ----

evaluate_check() {
    local check_name="$1" check_result="$2" detail="$3"
    local now last_alert elapsed

    now=$(date +%s)
    last_alert=$(get_last_alert_time "$check_name")

    if [[ "$check_result" -ne 0 ]]; then
        elapsed=$(( now - last_alert ))
        if [[ "$last_alert" -eq 0 || "$elapsed" -ge "$COOLDOWN_SECONDS" ]]; then
            log "FAILING: ${check_name} — ${detail}"
            send_alert "$check_name" "FAILING" "$detail"
            set_alert_state "$check_name" "$now"
        else
            log "FAILING (in cooldown, ${elapsed}s/${COOLDOWN_SECONDS}s): ${check_name} — ${detail}"
        fi
    else
        if [[ "$last_alert" -ne 0 ]]; then
            log "RECOVERED: ${check_name} — ${detail}"
            send_alert "$check_name" "RECOVERED" "$detail"
            clear_alert_state "$check_name"
        else
            log "OK: ${check_name} — ${detail}"
        fi
    fi
}

run_check() {
    local check_name="$1" check_fn="$2"
    local detail result=0
    detail=$("$check_fn") || result=$?
    evaluate_check "$check_name" "$result" "$detail"
}

main() {
    log "Starting monitoring run"
    run_check "load"      check_load
    run_check "memory"    check_memory
    run_check "disk"      check_disk
    run_check "processes" check_processes
    log "Monitoring run complete"
}

main "$@"
```

### Why these design choices?

- **The state file only ever contains checks that are currently *failing* and already alerted.** A check that's healthy simply doesn't appear in the file. This makes `clear_alert_state` and the "was there a prior alert?" test (`last_alert -ne 0`) trivially simple — presence in the file *is* the alert state.
- **Load average uses `awk` for the float comparison, not bash arithmetic.** Bash's `(( ))` only does integer math — `4.2 > 3` would silently misbehave. `awk` handles the comparison correctly and cheaply without needing `bc`.
- **`run_check` captures the check's stdout as the detail message *and* its exit code as the result**, using `|| result=$?` specifically because `set -e` would otherwise kill the whole script the instant a check function returns non-zero. This is the same "set -e doesn't play nicely with intentional non-zero returns" gotcha from Module 14 — the fix is the same: capture the exit status explicitly instead of letting it propagate.
- **Recovery detection falls directly out of the same state file**, rather than needing separate tracking — if a check is now passing (`result -eq 0`) but was previously in the state file, that's exactly the definition of "recovered."

## Scheduling

### crontab

```bash
# crontab -e
*/5 * * * * /usr/local/bin/monitor.sh >> /var/log/monitor-cron.log 2>&1
```

### systemd timer

`/etc/systemd/system/server-monitor.service`:
```ini
[Unit]
Description=Server health check and alerting

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitor.sh
```

`/etc/systemd/system/server-monitor.timer`:
```ini
[Unit]
Description=Run server-monitor every 5 minutes

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

## Testing Your Solution

Force a failure by temporarily lowering a threshold to something guaranteed to trip:

```bash
DISK_THRESHOLD_PCT=1 ./monitor.sh   # almost any disk will exceed 1%
```

Expected first-run output (fresh state, no prior alert):

```
[2026-03-15 14:00:00] Starting monitoring run
[2026-03-15 14:00:00] OK: load — load average 0.42 OK (4 cores, threshold 6)
[2026-03-15 14:00:00] OK: memory — memory usage 61% OK
[2026-03-15 14:00:00] FAILING: disk — disk usage 47% on / exceeds threshold 1%
[2026-03-15 14:00:00] Alert sent for disk (FAILING)
[2026-03-15 14:00:00] OK: processes — all critical processes running
[2026-03-15 14:00:00] Monitoring run complete
```

Run it again within 30 minutes — the disk check should now be silent about re-alerting:

```
[2026-03-15 14:05:00] FAILING (in cooldown, 300s/1800s): disk — disk usage 47% on / exceeds threshold 1%
```

Now fix the threshold back and run once more — you should see the recovery path fire:

```
[2026-03-15 14:10:00] RECOVERED: disk — disk usage 47% on / OK
[2026-03-15 14:10:00] Alert sent for disk (RECOVERED)
```
