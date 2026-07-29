# Capstone 3: Security Audit Automation Script 🔴

**Difficulty:** 🔴 Advanced | **Estimated Time:** 5-7h | **Prerequisites:** Modules 1-18 (especially 4, 12, 16, 18)

## Real-World Scenario

Module 18 taught you to check a system for world-writable files, rogue UID-0 users, unexpected SUID binaries, risky SSH settings, suspicious cron jobs, open ports, and failed logins — and to print a PASS/WARN/FAIL summary. That script works great on the one server you wrote it for. Now your security team wants to run it on **every** server, nightly, with results that don't require someone to read scrollback at 3 AM, and — critically — without it crying wolf every night about the one SUID binary that's *supposed* to be there.

This capstone turns Module 18's script into an actual tool: configurable, quiet about known-good exceptions, capable of gating a CI/CD pipeline with a real exit code, and keeping a paper trail of every run.

## What You'll Build

An `audit.sh` that runs all the Module 18 checks against a config file (thresholds + allowlists), writes a timestamped report to a reports directory, computes an overall severity (PASS/WARN/CRITICAL) with a matching exit code (0/1/2), and is schedulable nightly.

## Requirements

- [ ] Reuse/extend the Module 18 checks: world-writable files/dirs, non-root UID-0 users, SUID/SGID binaries, `sshd_config` risky settings, suspicious cron entries, listening ports, `apt list --upgradable`, failed SSH logins by IP
- [ ] Load configuration (thresholds, an **allowlist** of expected SUID binaries and expected listening ports) from a separate config file — not hardcoded in the script
- [ ] Every finding is categorized PASS / WARN / CRITICAL; the script computes an overall exit code: `0` = all PASS, `1` = at least one WARN (nothing CRITICAL), `2` = at least one CRITICAL
- [ ] Write a timestamped report file to a reports directory (don't overwrite previous runs — each run gets its own file so trends can be compared later)
- [ ] Full production hardening: `set -euo pipefail`, `trap`, `die()`, timestamped logging
- [ ] Be explicit about which checks require `sudo`/root and why (e.g. reading other users' crontabs, reading `/etc/shadow`-adjacent files)
- [ ] Schedulable nightly via cron or systemd timer

## Starter Guidance

The core architectural difference from Module 18 is a `run_check()` wrapper that standardizes how every check reports its severity, so the "compute overall exit code" logic only has to be written once:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG_FILE="./audit.conf"
REPORTS_DIR="/var/log/security-audits"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
REPORT_FILE="${REPORTS_DIR}/audit-${TIMESTAMP}.txt"

# shellcheck source=/dev/null
source "$CONFIG_FILE"   # defines ALLOWED_SUID=(...), ALLOWED_PORTS=(...), FAILED_LOGIN_WARN=, FAILED_LOGIN_CRITICAL=

OVERALL_SEVERITY=0   # 0=PASS 1=WARN 2=CRITICAL, only ever increases

log() { : ; }   # TODO: append to $REPORT_FILE with a timestamp
die() { : ; }   # TODO

report() {
    local severity_word="$1" check_name="$2" detail="$3"
    # TODO: write "[SEVERITY] check_name: detail" to $REPORT_FILE
    # TODO: bump $OVERALL_SEVERITY if this finding is worse than what's tracked so far
    :
}

check_world_writable() {
    # TODO: adapt from Module 18 — find world-writable files outside expected paths
    :
}

check_uid_zero() {
    # TODO: adapt from Module 18
    :
}

check_suid_binaries() {
    # TODO: adapt from Module 18, but skip anything in $ALLOWED_SUID
    :
}

check_sshd_config() {
    # TODO: adapt from Module 18
    :
}

check_cron_jobs() {
    # TODO: adapt from Module 18
    :
}

check_listening_ports() {
    # TODO: adapt from Module 18, but skip anything in $ALLOWED_PORTS
    :
}

check_package_updates() {
    # TODO: adapt from Module 18
    :
}

check_failed_logins() {
    # TODO: adapt from Module 18, comparing counts against
    # $FAILED_LOGIN_WARN / $FAILED_LOGIN_CRITICAL from config
    :
}

main() {
    mkdir -p "$REPORTS_DIR"
    log "=== Security Audit: $(hostname) — $(date) ==="
    check_world_writable
    check_uid_zero
    check_suid_binaries
    check_sshd_config
    check_cron_jobs
    check_listening_ports
    check_package_updates
    check_failed_logins
    log "=== Overall severity: $OVERALL_SEVERITY ==="
    exit "$OVERALL_SEVERITY"
}

main "$@"
```

💡 **Hints:**
- Keep `OVERALL_SEVERITY` monotonic — a WARN should never downgrade a CRITICAL that was already found. `(( severity_value > OVERALL_SEVERITY )) && OVERALL_SEVERITY=$severity_value` is the whole trick.
- The allowlist check for SUID binaries is just "is this path present in `$ALLOWED_SUID`?" — a simple loop or `printf '%s\n' "${ALLOWED_SUID[@]}" | grep -qxF "$path"` both work.
- Decide up front which checks genuinely need root (reading `/var/spool/cron/crontabs/*` for other users, for instance) and have the script warn clearly rather than silently produce incomplete results when run without `sudo`.

## Constraints & Assumptions

- Requires `sudo`/root for a subset of checks — the script should detect this and clearly flag which checks were skipped/incomplete if not run as root, rather than silently reporting a false PASS
- This is **not** a replacement for a real vulnerability scanner (Lynis, OpenVAS, Nessus) — it's a fast, zero-dependency first line of defense, and the README/report should say so explicitly
- Config file format is up to you — a sourced bash file (as shown above) is simplest; a `.conf` key=value file parsed manually is also fine

## Stretch Goals

- Generate an HTML report with color-coded severities instead of plain text
- Diff the current run's findings against the previous run's, and highlight only **new** findings prominently (this is what actually makes nightly reports readable long-term)
- Wire the exit code into an actual CI pipeline step (e.g. a GitHub Actions job that fails if `audit.sh` exits non-zero)
- Add a `--severity-threshold` flag so CI can gate on CRITICAL-only while a human still reviews WARN findings manually

## 📋 How to Present This in a Portfolio/Interview

This project demonstrates you can think about **security tooling as a product**, not a one-off script — configurability, false-positive management, and machine-readable results (exit codes) for pipeline integration are exactly what separates a script from a tool a team can actually adopt.

**Suggested portfolio description:**

> "Built a configurable Bash-based security audit tool covering file permissions, privileged accounts, SSH hardening, cron integrity, open ports, patch status, and brute-force login attempts. Designed with an allowlist system to eliminate false-positive fatigue and a severity-based exit code (0/1/2) enabling direct CI/CD pipeline gating. Generates timestamped, retained reports to support trend analysis over time."

Be ready to explain, out loud, why the exit code design matters for CI/CD (a human reading colored terminal output doesn't scale, but `if audit.sh; then deploy; fi` does), and how the allowlist prevents the tool from being ignored after the first week of false alarms — that's the actual lesson of this capstone.

---

**Reference solution:** [solution.md](solution.md)
