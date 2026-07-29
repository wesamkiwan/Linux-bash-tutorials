# Capstone 1: Automated Backup & Log-Rotation Toolkit 🟡

**Difficulty:** 🟡 Intermediate | **Estimated Time:** 4-6h | **Prerequisites:** Modules 1-16

## Real-World Scenario

You're the only person who understands the infrastructure at a small company. Right now, "backups" means someone remembers to zip the app's data folder before a risky deploy. Last month, nobody remembered, and a bad migration wiped three days of customer uploads. Your manager asks for one thing: "make backups happen automatically, and make sure we can actually trust them when we need them."

This is one of the most common first automation tasks handed to a junior sysadmin or DevOps engineer — and it's a perfect showcase of everything from Modules 6, 14, 15, and 16: scripting fundamentals, error handling, scheduling, and production hardening, all in one script.

## What You'll Build

A `backup.sh` script that archives a data directory (and simulates a database dump) into a timestamped, checksummed, compressed archive; automatically deletes old backups beyond a retention count; logs everything; fails loudly and safely instead of silently; and runs unattended every night via cron or a systemd timer. Plus a `restore.sh` that proves the backups actually work.

## Requirements

Your solution must satisfy all of the following:

- [ ] Accept a configurable source directory (or list of directories) and a backup destination directory
- [ ] Create a single timestamped, compressed archive (`tar` + `gzip`) per run — filename must sort chronologically (e.g. `backup-20260315-020000.tar.gz`)
- [ ] Include a database-dump step in the same archive flow (a placeholder function is fine — show the real pattern with a comment showing where `mysqldump`/`pg_dump` would plug in)
- [ ] Generate a SHA-256 checksum file alongside every archive, and provide a `verify` function/mode that re-checks an archive against its checksum
- [ ] Rotation: keep only the last **N** backups (configurable), safely deleting anything older
- [ ] Log every action with timestamps to a log file via a `log()` helper
- [ ] Use `set -euo pipefail`, a `trap` that cleans up any partial/incomplete archive if the script dies mid-run, and a `die()` helper for fatal errors
- [ ] Be safe to re-run at any time (idempotent) — no crashes if the destination already has files, uses `mkdir -p`
- [ ] Include a `restore.sh` (or a `--restore` mode) that extracts a chosen backup after verifying its checksum
- [ ] Be schedulable — show **both** a crontab entry and a systemd `.timer` + `.service` pair

## Starter Guidance

Don't start from a blank file. Here's a skeleton to fill in — the hard parts (rotation math, trap cleanup, checksum verification) are for you to work out:

```bash
#!/usr/bin/env bash
set -euo pipefail

# ---- Configuration ----
SOURCE_DIRS=("/var/www/myapp/data")   # what to back up
BACKUP_ROOT="/var/backups/myapp"     # where archives live
RETENTION_COUNT=7                    # how many backups to keep
LOG_FILE="${BACKUP_ROOT}/backup.log"

# ---- Helpers ----
log() {
    # TODO: print "[timestamp] message" to both stdout and $LOG_FILE
    :
}

die() {
    # TODO: log the error, then exit 1
    :
}

cleanup_on_failure() {
    # TODO: this runs via `trap` on EXIT — if the archive is incomplete, remove it
    :
}

dump_database() {
    # TODO: placeholder for `mysqldump`/`pg_dump` — for now, just write a fake
    # dump file into a temp location that gets included in the tar archive
    :
}

create_archive() {
    # TODO: tar+gzip $SOURCE_DIRS plus the DB dump into one timestamped file
    # remember: generate the checksum AFTER the archive is fully written
    :
}

rotate_old_backups() {
    # TODO: list existing backups oldest-first, delete all but the newest
    # $RETENTION_COUNT
    :
}

verify_backup() {
    # TODO: given a backup file path, check it against its .sha256 file
    :
}

main() {
    trap cleanup_on_failure EXIT
    mkdir -p "$BACKUP_ROOT"
    log "Starting backup run"
    create_archive
    rotate_old_backups
    log "Backup run complete"
}

main "$@"
```

💡 **Hints:**
- Generate the checksum with `sha256sum "$archive_path" > "${archive_path}.sha256"` — do this only once the `tar` command has fully succeeded.
- For rotation, `ls -1t` (newest first) combined with `tail -n +$((RETENTION_COUNT + 1))` gives you exactly the files to delete.
- Your `trap` cleanup function needs to know whether the archive actually finished — a simple "did we reach the last line" flag variable works well here.

## Constraints & Assumptions

- Target is Ubuntu/Debian; `tar`, `gzip`, and `sha256sum` are part of the base install
- The database dump step is illustrative — you don't need a real database to complete this capstone, a placeholder function that writes a small text file is sufficient to prove the pattern
- Assume the script runs as a user with read access to the source directories and write access to the backup destination

## Stretch Goals

- Upload the archive to remote/offsite storage after local rotation (`rsync -avz` to another host, or `aws s3 cp` if you have an AWS account)
- Send a webhook notification on backup failure (reuse the pattern from Capstone 2)
- Encrypt the archive at rest with `gpg --symmetric` before it leaves the server
- Add a `--dry-run` flag that shows what would happen without actually deleting/writing anything

## 📋 How to Present This in a Portfolio/Interview

This project demonstrates **production-grade automation thinking**, not just "I can write a for loop." Specifically, it proves you understand:

- **Error handling that actually matters** — `set -euo pipefail` + `trap` cleanup means a half-written archive never gets mistaken for a good one
- **Data integrity practices** — checksums aren't decoration, they're how you catch silent corruption before you need the backup and discover it's useless
- **Operational hygiene** — retention/rotation, logging, and idempotency are the difference between a script and a *tool*

**Suggested portfolio description:**

> "Built an automated backup system in Bash for a multi-directory web application, including checksummed archives for integrity verification, configurable retention-based rotation, and a companion restore script. Hardened with `set -euo pipefail`, trap-based cleanup of partial failures, and full audit logging. Deployed via both cron and a systemd timer for reliability comparison."

Be ready to explain, out loud, *why* you generate the checksum after compression (not before), and what specifically your `trap` protects against — interviewers will probe exactly there.

---

**Reference solution:** [solution.md](solution.md)
