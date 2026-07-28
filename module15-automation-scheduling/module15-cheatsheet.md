# 📋 Module 15 Cheat Sheet — Automation & Scheduling (cron/systemd)

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **`cron`** | A background daemon that wakes up every minute and launches commands whose scheduled time matches |
| **`crontab`** | Both the schedule-file format and the command used to view/edit a user's personal schedule |
| **user crontab** | A personal schedule, edited with `crontab -e`; every job runs as the owning user |
| **`/etc/crontab` / `/etc/cron.d/`** | System-wide crontab files; each entry has an **extra user field** naming who the job runs as |
| **`at`** | Schedules a command to run **once**, at a specific future time (not recurring) |
| **`systemd` timer** | A `.timer` unit that triggers a paired `.service` unit on a schedule — the modern alternative to cron |
| **`OnCalendar=`** | The directive in a `.timer` file that defines its schedule, using `systemd`'s own calendar syntax |
| **`journalctl`** | `systemd`'s logging system — captures every run's output automatically, no redirection needed |

## Crontab Commands

| Command | Effect |
|---|---|
| `crontab -e` | Edit your personal crontab |
| `crontab -l` | List your personal crontab (no editor opens) |
| `crontab -r` | **Delete** your entire personal crontab — no confirmation, no undo |
| `crontab -l > backup.txt` | Back up your crontab before making risky edits |
| `sudo crontab -e` | Edit the **root** user's crontab |
| `sudo crontab -u <user> -e` | Edit another specific user's crontab |

## Cron 5-Field Time Syntax

```
minute hour day-of-month month day-of-week   command
 0-59  0-23     1-31      1-12    0-7 (0 or 7 = Sunday)
```

| Symbol | Meaning | Example | Reads as |
|---|---|---|---|
| `*` | every value | `* * * * *` | every minute of every day |
| `,` | a list | `0 9,17 * * *` | 9 AM and 5 PM daily |
| `-` | an inclusive range | `0 9-17 * * *` | every hour, 9 AM–5 PM |
| `/` | a step | `*/15 * * * *` | every 15 minutes |
| combined | | `0 2 * * 1-5` | 2 AM, Monday–Friday |

⚠️ If **both** day-of-month and day-of-week are restricted (neither is `*`), cron runs when **either** matches, not only when both do.

## Cron Special Strings

| String | Equivalent | Meaning |
|---|---|---|
| `@reboot` | (none) | Once, at system startup |
| `@hourly` | `0 * * * *` | Top of every hour |
| `@daily` | `0 0 * * *` | Midnight every day |
| `@weekly` | `0 0 * * 0` | Midnight every Sunday |
| `@monthly` | `0 0 1 * *` | Midnight on the 1st of the month |
| `@yearly` / `@annually` | `0 0 1 1 *` | Midnight, January 1st |

## The Output-Redirection Pattern (never skip this)

```
0 2 * * * /path/to/script.sh >> /var/log/script.log 2>&1
```

| Piece | Purpose |
|---|---|
| `>>` | Append stdout to a log file (doesn't erase prior runs) |
| `2>&1` | Send stderr to the same place stdout is going |

⚠️ Without this, cron emails output (often to nowhere, if no mail system is configured) or drops it silently. A failing job with no redirection produces **zero visible symptoms** beyond the missing result.

## The PATH/Environment Gotcha

| Interactive shell | Cron |
|---|---|
| Reads `.bashrc` etc. | Reads **nothing** — minimal environment |
| Rich `PATH` (includes `~/.local/bin`, version-manager dirs, etc.) | Bare `PATH`, often just `/usr/bin:/bin` |
| Has a TTY | **No TTY** — no interactivity possible |
| Working directory = wherever you `cd`'d to | Working directory is **not guaranteed** to be the script's directory |

Fixes:
```bash
# Option 1: absolute paths inside the script (most robust)
/usr/local/bin/aws s3 cp ...

# Option 2: set PATH explicitly at the top of the crontab (fixes every job in it)
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Option 3: cd explicitly instead of relying on relative paths
cd /home/deploy/scripts || exit 1
```

Simulate cron's stripped environment before scheduling:
```bash
env -i /bin/bash /path/to/script.sh
```

## `at` — One-Time Scheduling

| Command | Effect |
|---|---|
| `echo "cmd" \| at 5pm` | Queue `cmd` to run once at 5 PM |
| `at 5pm` then type commands, `Ctrl-D` | Interactive job entry |
| `atq` | List all queued one-time jobs (with job numbers) |
| `at -c <job-number>` | Show the full command/environment for a queued job |
| `atrm <job-number>` | Cancel a queued job |
| `sudo apt install at` | Install `at` on Ubuntu/Debian if missing |

## systemd Timer Unit Reference

Minimal pair:

```ini
# /etc/systemd/system/mytask.service
[Unit]
Description=What this job does

[Service]
Type=oneshot
User=deploy
WorkingDirectory=/home/deploy/scripts
ExecStart=/home/deploy/scripts/mytask.sh
```

```ini
# /etc/systemd/system/mytask.timer
[Unit]
Description=When this job runs

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

### `OnCalendar=` Syntax Examples

| Expression | Meaning |
|---|---|
| `*-*-* 02:30:00` | Every day at 2:30 AM |
| `daily` | Shorthand for midnight every day |
| `hourly` | Shorthand for the top of every hour |
| `weekly` | Shorthand for midnight every Monday |
| `Mon *-*-* 09:00:00` | Every Monday at 9:00 AM |
| `*-*-01 00:00:00` | Midnight on the 1st of every month |
| `Mon..Fri *-*-* 08:00:00` | Every weekday at 8:00 AM |
| `*:0/15` | Every 15 minutes |

### Managing Timers

| Command | Effect |
|---|---|
| `sudo systemctl daemon-reload` | Re-read unit files after creating/editing one — **required every time** |
| `sudo systemctl enable --now mytask.timer` | Enable at boot **and** start immediately |
| `systemctl list-timers` | Show every active timer, next run, and last run |
| `systemctl status mytask.timer` | Show one timer's current state |
| `journalctl -u mytask.service` | See every past run's captured output, no redirection needed |
| `journalctl -u mytask.service --since today` | Filter run history to today |
| `sudo systemctl disable --now mytask.timer` | Stop and disable a timer |

## Cron vs. systemd Timers — Decision Table

| Situation | Use |
|---|---|
| Quick personal/dev-machine schedule | `cron` |
| Need it running in 30 seconds, simplicity matters most | `cron` |
| Production service another engineer must debug later | `systemd` timer |
| Need automatic logging with no manual redirection | `systemd` timer |
| Need to catch up on a run missed during downtime | `systemd` timer (`Persistent=true`) |
| Need dependency management ("run after network is up") | `systemd` timer |
| One-time future task, not recurring | `at` |
| Ubiquitous across nearly every Unix-like system, minimal setup | `cron` |

## 🔁 The New Cron Job Workflow

Do this every time you add a cron job:

1. **Test the script manually first** — confirm it works standalone before scheduling anything.
2. **Use absolute paths** for the script itself and every external command it calls.
3. **Redirect output to a log file** — `>> /var/log/yourjob.log 2>&1`, no exceptions.
4. **`cd` explicitly** or use absolute paths — never assume a particular working directory.
5. **Add a comment above the entry** in the crontab explaining what it does and why.
6. **Back up the crontab** (`crontab -l > backup.txt`) before editing an existing one.
7. **Check the log after the first scheduled run** — don't assume silence means success.

## 🔁 The New systemd Timer Workflow

Do this every time you add a systemd timer:

1. **Write and test the script manually first**, exactly as with cron.
2. **Create the `.service` file** — what to run, as which user, from which directory.
3. **Create the paired `.timer` file** — when to run it, with `OnCalendar=` and `Persistent=true`.
4. **Run `sudo systemctl daemon-reload`** — required after every create/edit.
5. **Enable and start it**: `sudo systemctl enable --now mytask.timer`.
6. **Confirm the schedule**: `systemctl list-timers mytask.timer`.
7. **Check the first run's log**: `journalctl -u mytask.service --since today`.
