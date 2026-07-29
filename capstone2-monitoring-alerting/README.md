# Capstone 2: Server Monitoring & Alerting System 🔴

**Difficulty:** 🔴 Advanced | **Estimated Time:** 5-7h | **Prerequisites:** Modules 1-16 (especially 10, 11, 12, 14, 15)

## Real-World Scenario

Your company runs a handful of production servers but doesn't have budget for Datadog or Prometheus+Grafana yet. You're on call this week, and last time a disk filled up silently, nobody noticed until customers started complaining. Your manager wants a lightweight bash-based monitor that checks the essentials and pages the team the moment something's wrong — but *without* flooding the Slack channel with the same alert every five minutes while the problem is being fixed.

This capstone is where Modules 10 (process management), 11 (system monitoring), 12 (webhooks via curl), 14 (error handling), and 15 (scheduling) come together into a single real tool — and it's the kind of project that separates "can write a bash script" from "understands operational software design."

## What You'll Build

A `monitor.sh` script that checks load average, memory usage, disk usage, and whether specific critical processes are running; sends a webhook alert when any check fails; tracks state so it doesn't re-alert on every run; and sends a "recovery" notification when a problem clears. Scheduled to run every 5 minutes.

## Requirements

- [ ] Check system load average against a threshold, normalized to CPU core count (`nproc`) — a load of 4 means something different on a 2-core box than a 32-core box
- [ ] Check memory usage percentage against a threshold
- [ ] Check disk usage percentage (for a configurable filesystem, e.g. `/`) against a threshold
- [ ] Check that a configurable list of critical process names are running (via `pgrep`), alerting if any are missing
- [ ] On ANY failed check, send a webhook alert (`curl -X POST` with a JSON body) containing hostname, timestamp, and which check(s) failed
- [ ] **Alert deduplication**: don't re-send the same alert every run — use a state file to track when each check last alerted, and only re-alert after a cooldown period (e.g. 30 minutes) even if the script runs every 5 minutes
- [ ] **Recovery detection**: when a previously-failing check returns to normal, send a distinct "recovered" notification and clear that check's alert state
- [ ] `set -euo pipefail`, a `die()` pattern, and full timestamped logging
- [ ] Schedulable every 5 minutes via cron or a systemd timer

## Starter Guidance

The state-file logic is the genuinely hard part of this capstone — everything else is straightforward once you've built the individual checks. Start here:

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
COOLDOWN_SECONDS=1800   # 30 minutes
STATE_FILE="/var/lib/monitor/state.tsv"
LOG_FILE="/var/log/monitor.log"

log() { : ; }   # TODO
die() { : ; }   # TODO

check_load() {
    # TODO: compare 1-min load average (from `uptime` or /proc/loadavg)
    # against $LOAD_THRESHOLD_PER_CORE * $(nproc)
    # return 0 = OK, 1 = FAILING
    :
}

check_memory() {
    # TODO: parse `free` to get used/total percentage
    :
}

check_disk() {
    # TODO: parse `df -h $DISK_PATH`
    :
}

check_processes() {
    # TODO: pgrep each name in $CRITICAL_PROCESSES, report which are missing
    :
}

get_last_alert_time() {
    # TODO: look up $1 (check name) in $STATE_FILE, print its timestamp or 0
    :
}

set_alert_state() {
    # TODO: update (or insert) $1 (check name) with current timestamp in $STATE_FILE
    :
}

clear_alert_state() {
    # TODO: remove $1 (check name) from $STATE_FILE — used on recovery
    :
}

send_alert() {
    # TODO: curl -X POST $WEBHOOK_URL with a JSON payload; $1 = check name,
    # $2 = "FAILING" or "RECOVERED", $3 = detail message
    :
}

evaluate_check() {
    local check_name="$1" check_result="$2" detail="$3"
    # TODO: this is the core state machine:
    #  - if check_result is FAILING and cooldown has elapsed -> send_alert + set_alert_state
    #  - if check_result is FAILING and still in cooldown -> log only, no alert
    #  - if check_result is OK and there was a prior alert state -> send RECOVERED alert + clear_alert_state
    #  - if check_result is OK and no prior alert -> nothing to do
    :
}

main() {
    log "Starting monitoring run"
    # TODO: call each check_*, feed the result into evaluate_check
}

main "$@"
```

💡 **Hints:**
- A simple state file format like `check_name<TAB>unix_timestamp` works well and is trivially parsed with `awk -F'\t'`.
- `$(( $(date +%s) - last_alert_time ))` gives you elapsed seconds — compare against `$COOLDOWN_SECONDS`.
- Test your state machine logic with a fake check that you can force to fail/pass on demand, rather than waiting for real load/memory/disk conditions to change.

## Constraints & Assumptions

- Target is Ubuntu/Debian; assumes `bc` or pure bash integer arithmetic for the load-average comparison (load averages are floats — decide how you'll compare them without a full float library)
- The webhook URL is a placeholder — the same `curl -X POST` pattern works for Slack, Discord, Microsoft Teams, or any incoming-webhook-style endpoint; the JSON shape differs slightly per platform (check their docs), but the mechanism is identical
- No real paging/on-call platform (PagerDuty, Opsgenie) integration is required — a webhook alert is sufficient to demonstrate the pattern

## Stretch Goals

- Write a `status.txt`/`status.json` file each run showing the current state of every check, so a simple `cat` or a tiny web page can show current health at a glance
- Add exponential backoff to the cooldown (alert again after 30 min, then 1h, then 2h) instead of a fixed cooldown
- Support multiple webhook destinations (e.g. Slack for warnings, PagerDuty-style urgent webhook for critical failures)
- Add a `--dry-run` mode that logs what alerts *would* fire without actually sending them

## 📋 How to Present This in a Portfolio/Interview

This project's real value isn't "I can check disk space" — it's that you understood **alert fatigue is a real production problem** and designed state management to solve it. That's a genuinely differentiating detail most junior candidates miss entirely.

**Suggested portfolio description:**

> "Designed and built a lightweight bash-based server monitoring system for environments without a full observability stack. Monitors load average (normalized to core count), memory, disk, and critical process health, alerting via webhook. Implemented stateful alert deduplication with a cooldown window and automatic recovery notifications to prevent alert fatigue — a pattern typically associated with dedicated monitoring platforms, built from first principles in bash."

Be ready to explain, out loud, exactly how your state file prevents duplicate alerts and how recovery detection works — this is the part interviewers will dig into, because it's the part that proves real understanding versus a copy-pasted check script.

---

**Reference solution:** [solution.md](solution.md)
