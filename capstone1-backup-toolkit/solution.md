# Capstone 1: Reference Solution — Automated Backup & Log-Rotation Toolkit

This is one valid, complete solution. Your implementation doesn't need to match line-for-line — it needs to satisfy the requirements in [README.md](README.md).

## `backup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# ---- Configuration ----
SOURCE_DIRS=("/var/www/myapp/data")
BACKUP_ROOT="/var/backups/myapp"
RETENTION_COUNT=7
LOG_FILE="${BACKUP_ROOT}/backup.log"

TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
ARCHIVE_NAME="backup-${TIMESTAMP}.tar.gz"
ARCHIVE_PATH="${BACKUP_ROOT}/${ARCHIVE_NAME}"
ARCHIVE_COMPLETE=0

# ---- Helpers ----
log() {
    local msg="[$(date '+%Y-%m-%d %H:%M:%S')] $*"
    echo "$msg" | tee -a "$LOG_FILE"
}

die() {
    log "FATAL: $*"
    exit 1
}

cleanup_on_failure() {
    local exit_code=$?
    if [[ "$ARCHIVE_COMPLETE" -eq 0 && -f "$ARCHIVE_PATH" ]]; then
        log "Backup did not complete successfully — removing partial archive: $ARCHIVE_PATH"
        rm -f "$ARCHIVE_PATH" "${ARCHIVE_PATH}.sha256"
    fi
    exit "$exit_code"
}

dump_database() {
    local dump_dir="$1"
    # Placeholder for a real database dump. Swap this line for:
    #   mysqldump -u backup_user -p"$DB_PASSWORD" myapp_db > "$dump_dir/db.sql"
    #   pg_dump -U backup_user myapp_db > "$dump_dir/db.sql"
    echo "-- placeholder dump generated at $(date)" > "$dump_dir/db.sql"
}

create_archive() {
    local staging_dir
    staging_dir="$(mktemp -d)"

    dump_database "$staging_dir"

    log "Creating archive: $ARCHIVE_PATH"
    tar -czf "$ARCHIVE_PATH" -C "$staging_dir" db.sql "${SOURCE_DIRS[@]}"

    log "Generating checksum"
    sha256sum "$ARCHIVE_PATH" > "${ARCHIVE_PATH}.sha256"

    rm -rf "$staging_dir"
    ARCHIVE_COMPLETE=1
    log "Archive complete: $ARCHIVE_PATH ($(du -h "$ARCHIVE_PATH" | cut -f1))"
}

rotate_old_backups() {
    log "Applying retention policy: keep last $RETENTION_COUNT backups"
    local to_delete
    to_delete=$(ls -1t "${BACKUP_ROOT}"/backup-*.tar.gz 2>/dev/null | tail -n +$((RETENTION_COUNT + 1)) || true)

    if [[ -z "$to_delete" ]]; then
        log "Nothing to rotate"
        return 0
    fi

    while IFS= read -r old_backup; do
        log "Removing old backup: $old_backup"
        rm -f "$old_backup" "${old_backup}.sha256"
    done <<< "$to_delete"
}

verify_backup() {
    local backup_file="$1"
    [[ -f "$backup_file" ]] || die "No such backup: $backup_file"
    [[ -f "${backup_file}.sha256" ]] || die "No checksum file for: $backup_file"

    log "Verifying $backup_file against its checksum"
    if (cd "$(dirname "$backup_file")" && sha256sum -c "$(basename "$backup_file").sha256"); then
        log "Verification PASSED"
    else
        die "Verification FAILED — backup may be corrupted"
    fi
}

main() {
    mkdir -p "$BACKUP_ROOT"
    trap cleanup_on_failure EXIT

    if [[ "${1:-}" == "--verify" ]]; then
        verify_backup "${2:?Usage: backup.sh --verify <path-to-archive>}"
        exit 0
    fi

    log "Starting backup run"
    create_archive
    verify_backup "$ARCHIVE_PATH"
    rotate_old_backups
    log "Backup run finished successfully"
}

main "$@"
```

### Why these design choices?

- **Checksum comes after compression, not before.** A checksum's job is to prove *this exact file* wasn't corrupted or truncated. If you checksum the source files before archiving, you've verified the wrong thing — a truncated `tar` write would still "match" a pre-archive checksum. Always checksum the artifact you're actually going to trust later.
- **The `ARCHIVE_COMPLETE` flag protects against partial files, not partial trap logic.** `set -e` means any failed command inside `create_archive` jumps straight past the rest of the function to the `EXIT` trap. Without the flag, `cleanup_on_failure` couldn't tell "this archive finished fine" from "this archive died halfway through" — both look like "a file exists at `$ARCHIVE_PATH`."
- **Rotation counts from `ls -1t`, not from dates.** Counting the newest N files is simpler and more robust than date-math — it still works correctly if the server's clock skips, or if backups run more than once in a day.

## `restore.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_ROOT="/var/backups/myapp"
RESTORE_TARGET="/var/www/myapp/data-restored"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
die() { log "FATAL: $*"; exit 1; }

main() {
    local backup_file="${1:?Usage: restore.sh <path-to-archive>}"

    [[ -f "$backup_file" ]] || die "No such backup: $backup_file"
    [[ -f "${backup_file}.sha256" ]] || die "Missing checksum — refusing to restore an unverified backup"

    log "Verifying checksum before restore"
    (cd "$(dirname "$backup_file")" && sha256sum -c "$(basename "$backup_file").sha256") \
        || die "Checksum mismatch — backup is corrupted, aborting restore"

    mkdir -p "$RESTORE_TARGET"
    log "Extracting into $RESTORE_TARGET"
    tar -xzf "$backup_file" -C "$RESTORE_TARGET"

    log "Restore complete. Review $RESTORE_TARGET before promoting it to the live data directory."
}

main "$@"
```

Restoring into a separate `-restored` directory (rather than overwriting the live data path directly) is deliberate: it forces a human to review the restored data before it replaces anything live.

## Scheduling: crontab

```bash
# crontab -e
# Run nightly at 2:00 AM, log cron's own stderr in case the script never even starts
0 2 * * * /usr/local/bin/backup.sh >> /var/backups/myapp/cron.log 2>&1
```

## Scheduling: systemd timer (preferred for production)

`/etc/systemd/system/myapp-backup.service`:

```ini
[Unit]
Description=Nightly backup of myapp data

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

`/etc/systemd/system/myapp-backup.timer`:

```ini
[Unit]
Description=Run myapp-backup nightly at 2 AM

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp-backup.timer
systemctl list-timers myapp-backup.timer
```

`Persistent=true` means if the server was off at 2 AM (maintenance window, crash), the timer fires as soon as the system is back up — cron can't do that.

## Testing Your Solution

```bash
chmod +x backup.sh restore.sh
sudo mkdir -p /var/backups/myapp
sudo ./backup.sh
```

Expected log output:

```
[2026-03-15 02:00:00] Starting backup run
[2026-03-15 02:00:00] Creating archive: /var/backups/myapp/backup-20260315-020000.tar.gz
[2026-03-15 02:00:01] Generating checksum
[2026-03-15 02:00:01] Archive complete: /var/backups/myapp/backup-20260315-020000.tar.gz (4.2M)
[2026-03-15 02:00:01] Verifying /var/backups/myapp/backup-20260315-020000.tar.gz against its checksum
[2026-03-15 02:00:01] Verification PASSED
[2026-03-15 02:00:01] Applying retention policy: keep last 7 backups
[2026-03-15 02:00:01] Nothing to rotate
[2026-03-15 02:00:01] Backup run finished successfully
```

Run it 8 nights in a row (or just fake it by touching 8 archives with different timestamps) and confirm only 7 remain:

```bash
ls /var/backups/myapp/backup-*.tar.gz | wc -l   # should print 7
```

Then prove the restore path actually works:

```bash
./restore.sh /var/backups/myapp/backup-20260315-020000.tar.gz
ls /var/www/myapp/data-restored
```

If you want to see the failure path, corrupt a checksum file on purpose (`echo "garbage" >> backup-*.sha256`) and confirm `restore.sh` refuses to proceed.
