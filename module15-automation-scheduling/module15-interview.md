# 🎤 Module 15 Interview Prep — Automation & Scheduling (cron/systemd)

## Conceptual Questions

### 🟢 Beginner

**Q1: What is cron, and what does a cron entry actually do?**

> "Cron is a background daemon that wakes up once a minute, checks a set of schedule files, and runs any command whose scheduled time matches right now. A crontab entry is one line describing a schedule and the command to run at that schedule — five time fields (minute, hour, day-of-month, month, day-of-week), followed by the command itself. It's the classic Unix way to make a script run automatically without a human typing it in."

**Q2: Walk me through the 5-field cron time syntax.**

> "From left to right, the fields are minute (0-59), hour (0-23, 24-hour time), day-of-month (1-31), month (1-12), and day-of-week (0-7, where both 0 and 7 mean Sunday). Each field can be a specific number, a `*` meaning 'every value,' a comma-separated list, a range with a dash, or a step with a slash. So `30 2 * * 1-5` means minute 30, hour 2 — so 2:30 AM — every day of the month, every month, but only Monday through Friday."

**Q3: What are cron's special strings, and why would you use them?**

> "Shorthand for the most common schedules: `@reboot` runs once at startup, `@hourly` is `0 * * * *`, `@daily` is `0 0 * * *`, `@weekly` is `0 0 * * 0`, `@monthly` is `0 0 1 * *`, and `@yearly` is `0 0 1 1 *`. I'd use them whenever they match my actual intent, because `@daily` is instantly readable, while `0 0 * * *` requires the reader to mentally decode five fields to figure out the same thing."

**Q4: What's the difference between `crontab -l` and `crontab -e`?**

> "`crontab -l` lists your current personal crontab without opening an editor — read-only. `crontab -e` opens it in an editor so you can add, change, or remove entries. And there's a third one worth knowing, `crontab -r`, which deletes your entire crontab immediately with no confirmation prompt — it's easy to fat-finger `-r` when you meant `-l`, so I always back up with `crontab -l > backup.txt` before doing anything destructive."

**Q5: What is `at`, and how is it different from cron?**

> "`at` schedules a command to run once, at a specific future time — cron is built around recurring schedules, `at` is built around a single occurrence. I'd use `at` for something like 'send this notice at 10 PM tonight,' where setting up a whole recurring cron entry and then remembering to delete it afterward would be unnecessary overhead."

### 🟡 Intermediate

**Q6: Why did my cron job work perfectly when I ran it manually, but silently fail when cron ran it?**

> "This is almost always cron's minimal environment. When you're logged in interactively, your shell has read `.bashrc` and built up a rich `PATH`, plus whatever environment variables and aliases you've accumulated. Cron runs jobs with a stripped-down environment — often a bare `PATH` like `/usr/bin:/bin` — and no TTY at all. If your script calls a command by its bare name and that command lives somewhere only your interactive shell's `PATH` knows about — something under `~/.local/bin`, or installed via a version manager — cron simply can't find it, the command fails with 'not found,' and if you haven't redirected output to a log file, that failure message goes nowhere and you never see it. The fix is using absolute paths for every external command the script calls, and/or setting `PATH` explicitly at the top of the crontab."

**Q7: Compare cron and systemd timers. When would you choose each?**

> "Both schedule commands, but they differ in everything around the execution. Cron is simple, ubiquitous, and fast to set up — perfect for a personal schedule or a quick throwaway job — but its output defaults to email, which usually goes nowhere on a modern server unless you manually redirect it, and it has no built-in dependency management or missed-run recovery. Systemd timers pair a `.timer` (the schedule) with a `.service` (the command), and every run's output is automatically captured by `journalctl` with zero redirection required. Timers also support `Persistent=true`, which catches up on a run that was missed because the machine was off, and they integrate with systemd's dependency system, so you can say 'don't run this until the network is up.' For production services — anything another engineer needs to debug later — I default to systemd timers, specifically because of that built-in observability. For quick personal or dev-machine tasks, cron is still simpler and perfectly fine."

**Q8: Explain the difference between a user crontab and `/etc/crontab`/`/etc/cron.d/` entries.**

> "A user crontab, edited with `crontab -e`, is personal — every job in it runs as whoever owns that crontab, and there's no need to say so explicitly. `/etc/crontab` and files under `/etc/cron.d/` are system-wide, and because they don't have one implicit owner, each entry has one extra field, inserted right after the five time fields and before the command, naming exactly which user account the job should run as. So a user crontab line is `0 3 * * * /path/to/script.sh`, but the equivalent line in `/etc/crontab` is `0 3 * * * deploy /path/to/script.sh` — same five time fields, but now `deploy` explicitly says who it runs as."

**Q9: How does cron's minute-by-minute polling actually determine when to run a job?**

> "The cron daemon wakes up once per minute and checks the current minute, hour, day-of-month, month, and day-of-week against every scheduled job's five fields. If all five match — remembering that if both day-of-month and day-of-week are restricted, it's actually an 'or' rather than an 'and' — it launches that command. It doesn't track sub-minute precision at all; the finest granularity cron itself offers is once per minute, which is why anything needing sub-minute scheduling needs a different tool entirely."

### 🔴 Advanced

**Q10: A nightly cron job worked fine for months, then failed exactly once, on the night clocks changed for Daylight Saving Time. What likely happened, and how would you prevent it?**

> "Cron schedules run according to the system's configured local timezone. On the specific night a DST transition happens, a clock time like 2:30 AM either doesn't exist at all (clocks spring forward past it) or occurs twice (clocks fall back through it), depending on the direction of the transition — so a job scheduled for that exact wall-clock time can run twice, not at all, or at a shifted moment relative to real elapsed time, purely because of that one night. The fix in production is to run the system on UTC rather than a DST-observing timezone — UTC never shifts, so a cron schedule means exactly the same thing, in terms of real elapsed time relative to other UTC-scheduled events, every single day of the year without exception. I'd check `timedatectl` to see the system's current timezone configuration, and treat 'not UTC' as something to flag on a production server unless there's a specific reason it needs local time."

**Q11: You need a scheduled job that must run even if the server was powered off exactly when it was scheduled — say, a monthly billing job that absolutely cannot be skipped. How would you guarantee that with the tools in this module?**

> "Cron has no answer for this at all — if the machine is off when a scheduled minute passes, that run is simply gone forever, with no catch-up mechanism. This is exactly the kind of case where I'd reach for a systemd timer instead, specifically for `Persistent=true` in the `.timer` unit's `[Timer]` section. With that set, systemd records the last time the timer fired, and if the machine was off during a scheduled run, it fires that missed run automatically the moment the system boots back up, rather than waiting for the next naturally-scheduled time. I'd pair that with `journalctl -u mybilling.service` so there's also a clear, permanent record of exactly when it ran — including the catch-up run — for anyone who needs to audit it later."

**Q12: How would you debug a systemd timer that `systemctl list-timers` shows as scheduled, but that never seems to actually produce results?**

> "First I'd check `journalctl -u <name>.service` to see whether the service is actually being triggered at all, and if so, what its own output says — this is the systemd equivalent of checking a cron log, except it requires no redirection setup to already exist. If the service unit shows failures, I'd look at the `ExecStart=` line for the same class of bug that breaks cron jobs — relative paths, commands that aren't on whatever minimal `PATH` the service inherits, or a `WorkingDirectory=` that doesn't match what the script assumes. If the timer genuinely never triggers at all, I'd double check that `daemon-reload` was actually run after the unit files were last edited — systemd caches unit definitions and won't notice an edited file on disk until it's told to re-read it — and confirm the timer is actually enabled and active with `systemctl status mytask.timer`, not just present on disk."

---

## Practical/Coding Questions

**Q1: Write a crontab entry that runs `/opt/scripts/cleanup.sh` every 15 minutes, only between 9 AM and 5 PM, Monday through Friday, with output safely logged.**

Solution:
```
*/15 9-17 * * 1-5 /opt/scripts/cleanup.sh >> /var/log/cleanup.log 2>&1
```
Explanation: `*/15` in the minute field steps every 15 minutes; `9-17` restricts the hour to 9 AM through 5 PM inclusive; `1-5` restricts the day-of-week to Monday through Friday. The output redirection ensures any failure is visible in the log rather than silently emailed or dropped.

**Q2: This script works when run by hand but fails silently under cron. Identify the bug and fix it.**

```bash
#!/bin/bash
set -euo pipefail
cd reports
python3 generate.py > summary.txt
```

Solution:
```bash
#!/bin/bash
set -euo pipefail
cd /home/deploy/reports || exit 1
/usr/bin/python3 /home/deploy/reports/generate.py > /home/deploy/reports/summary.txt
```
Explanation: `cd reports` is a relative path that assumes a particular working directory cron never guarantees — it should be an absolute path, or checked explicitly with `|| exit 1`. `python3` is called by bare name, which depends on cron's `PATH` including whatever directory it lives in; using the absolute path (`/usr/bin/python3`, confirm with `which python3` interactively) removes that dependency entirely. The output file `summary.txt` is also relative and should be absolute for the same reason.

**Q3: Create a systemd timer that runs a log-rotation service every Sunday at midnight, catching up on a missed run if the server was down.**

Solution:
```ini
# /etc/systemd/system/rotate-logs.service
[Unit]
Description=Weekly log rotation

[Service]
Type=oneshot
ExecStart=/opt/scripts/rotate-logs.sh
```
```ini
# /etc/systemd/system/rotate-logs.timer
[Unit]
Description=Run rotate-logs.service every Sunday at midnight

[Timer]
OnCalendar=Sun *-*-* 00:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rotate-logs.timer
```
Explanation: `OnCalendar=Sun *-*-* 00:00:00` fires every Sunday at midnight. `Persistent=true` guarantees a missed run (server down at the scheduled time) fires automatically on the next boot instead of being lost entirely — something a plain cron `0 0 * * 0` entry has no equivalent for.

**Q4: Schedule a one-time job with `at` to restart a service at 3 AM tonight, then show how you'd cancel it if plans changed.**

Solution:
```bash
echo "sudo systemctl restart myapp.service" | at 3am
atq
```
```
1    Wed Jul 29 03:00:00 2026 a deploy
```
```bash
atrm 1
```
Explanation: piping the command into `at 3am` queues a single, one-time execution. `atq` lists every pending job with its job number; `atrm 1` cancels job 1 before it ever runs, using the job number `atq` reported.

---

## Gotcha Questions

**Q1: "My script uses `aws`, `jq`, and `python3` and works flawlessly by hand, but every single one of them 'fails' under cron. Is cron broken?"**

> Trap: Cron isn't broken — this is the classic minimal-`PATH` gotcha, just hitting three different commands at once instead of one. Cron runs jobs with a bare, minimal environment that usually doesn't include whatever directories your interactive shell's `.bashrc` has added to `PATH` over time (version managers, `~/.local/bin`, manually-installed tools). The fix is absolute paths for every external command the script calls, or an explicit `PATH=` line at the top of the crontab — not troubleshooting cron itself, which is behaving exactly as documented.

**Q2: "I scheduled a cron job and it's been running every night for a month with zero errors reported. Doesn't that mean it's definitely working correctly?"**

> Trap: "No errors reported" and "working correctly" are not the same claim if output was never redirected anywhere. By default, cron tries to email a job's stdout/stderr, and on most modern servers with no mail system configured, that output simply vanishes — there's no error report because there's nowhere for one to go, not because nothing went wrong. The only way to actually know is to check that the job's expected output (a file, a database row, an uploaded backup) genuinely exists, or — much better — to have redirected output to a log file from the very start so you have a real record to check instead of an absence of complaints.

**Q3: "I scheduled my job for 2:30 AM with cron, and on one specific night it either ran twice or didn't run at all. What's going on?"**

> Trap: This is a Daylight Saving Time transition, not a cron bug. Cron schedules by local wall-clock time, and on the night clocks spring forward, 2:30 AM might not exist that day at all (skipped straight from 1:59 to 3:00); on the night clocks fall back, 2:30 AM happens twice. Cron has no special handling for this — it just checks the clock every minute and does what the schedule says, and the DST transition genuinely changes what the clock says that one night. The professional fix is running production systems on UTC, which never observes DST, so this ambiguity structurally can't happen.

---

## Quick-Fire Rapid Review

- **Q: What are the 5 fields in a cron time expression, in order?** A: Minute, hour, day-of-month, month, day-of-week.
- **Q: What do 0 and 7 both mean in the day-of-week field?** A: Sunday.
- **Q: What does `*/15` in the minute field mean?** A: Every 15 minutes.
- **Q: What does `@daily` expand to?** A: `0 0 * * *` — midnight every day.
- **Q: What command lists your crontab without opening an editor?** A: `crontab -l`.
- **Q: What command deletes your entire crontab with no confirmation?** A: `crontab -r`.
- **Q: What extra field do `/etc/crontab` and `/etc/cron.d/` entries have that a user crontab doesn't?** A: A user field, naming who the job runs as.
- **Q: Why does a script that works manually often fail under cron?** A: Cron's minimal environment/`PATH` doesn't match your interactive shell's `PATH`.
- **Q: What must you always add to a cron entry to avoid silently losing output?** A: Output redirection, e.g. `>> logfile 2>&1`.
- **Q: What command schedules a one-time future job?** A: `at`.
- **Q: What commands manage the `at` queue?** A: `atq` to list, `atrm` to cancel.
- **Q: What two files make up a systemd scheduled task?** A: A `.timer` file (when) and a paired `.service` file (what).
- **Q: What directive sets a systemd timer's schedule?** A: `OnCalendar=`.
- **Q: What must you run after creating/editing a systemd unit file?** A: `sudo systemctl daemon-reload`.
- **Q: What command shows every active systemd timer and its next run?** A: `systemctl list-timers`.
- **Q: What captures a systemd service's output automatically, with no redirection needed?** A: `journalctl -u <service-name>`.
- **Q: What `.timer` setting catches up on a run missed while the machine was off?** A: `Persistent=true`.
- **Q: Why do production systems typically run on UTC instead of local time for scheduling?** A: UTC never observes Daylight Saving Time, so scheduled times stay unambiguous year-round.
