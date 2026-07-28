# Module 15: Automation & Scheduling (cron/systemd) 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-14 (Shell Fundamentals through Error Handling, Traps & Debugging). Module 14's `set -euo pipefail`, `trap`, `die()`, and `shellcheck` habits are assumed to already be part of every script you schedule in this module — an unattended script needs them more, not less, than one you run by hand.

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain why scheduling matters on the job, and what "unattended" really implies about how a script must be written
- [ ] Manage a user's personal schedule with `crontab -e`, `crontab -l`, and `crontab -r`
- [ ] Read and write the 5-field cron time syntax (`minute hour day-of-month month day-of-week`), including ranges, steps, and lists
- [ ] Use cron's special strings (`@reboot`, `@daily`, `@hourly`, `@weekly`) and know when they're appropriate
- [ ] Explain the difference between a user's personal crontab and the system-wide `/etc/crontab` / `/etc/cron.d/` files, including the extra user field
- [ ] Diagnose and fix the classic "it works when I run it myself, but fails under cron" problem, caused by cron's minimal environment (`PATH`, working directory, no TTY)
- [ ] Redirect a scheduled job's output to a log file so failures are never silently emailed or dropped
- [ ] Schedule a one-time future job with `at`, and manage the queue with `atq`/`atrm`
- [ ] Build and enable a `systemd` timer (`.timer` + `.service` pair) as the modern, production-grade alternative to cron
- [ ] Decide, for a given real-world task, whether `cron` or a `systemd` timer is the better tool

---

## Module Goal

By the end of this module, you'll be able to take a script that only ever runs because a human remembers to type its name, and turn it into one that runs itself — reliably, on schedule, whether or not anyone is watching, and in a way that tells you loudly if it ever fails instead of failing in silence.

🎯 **On the job:** Picture a small team running a production web app. Every night, a database backup needs to happen. Every week, old log files need to be rotated and compressed so the disk doesn't fill up. Every five minutes, a health-check script needs to hit an internal endpoint and page someone if it doesn't respond. None of this can depend on a human remembering to SSH in and run a command — people go on vacation, forget, get pulled into other fires, or simply go home at 6pm while the servers keep running all night. The entire point of this module is the difference between "we have a backup script" (which is worthless if nobody runs it) and "we have a backup that happens" (which only exists because something schedules and executes that script automatically, on time, every time). That "something" is either `cron`, `at`, or a `systemd` timer — the three tools this module covers.

---

## Core Concepts

### 1. Why scheduling is its own discipline, not just "running a script later"

Every script in Modules 6-14 assumed a human was there to type the command, watch the output, and react if something went wrong. Scheduling removes the human from that loop entirely. A **scheduled job** is a command that runs automatically, at a time or interval you define in advance, with nobody present to notice its output, catch its errors, or confirm it even started.

💡 **Analogy — cron as an alarm clock:** Think of `cron` as an alarm clock you set once: "ring every day at 2:00 AM." The alarm clock doesn't know or care *why* you wanted to wake up then, and it doesn't check whether you're actually in bed to hear it — it just rings, unconditionally, at the time you configured, forever, until you change or remove the setting. Cron behaves the same way toward your script: it launches the exact same command, over and over, at the times you configured, with zero awareness of whether the script succeeded last time, whether it's even still a good idea to run, or whether anyone will ever look at what happened. That unconditional, unattended reliability is exactly what makes it useful — and exactly why the script it launches has to be self-sufficient, since nothing is standing by to help it if something goes wrong.

### 2. `cron` and `crontab` — the classic Linux scheduler

**`cron`** is a background daemon (a long-running process with no controlling terminal, first introduced in Module 10's process concepts) that wakes up once a minute, checks a set of schedule files, and launches any command whose scheduled time matches right now.

**`crontab`** ("cron table") is both the *file format* that describes a schedule, and the *command-line tool* you use to view and edit your own personal schedule. Every user on the system can have their own crontab.

Three commands you'll use constantly:

| Command | What it does |
|---|---|
| `crontab -e` | Open **your** crontab in an editor to add/change/remove scheduled jobs |
| `crontab -l` | List (print) your current crontab, without opening an editor |
| `crontab -r` | **Remove** your entire crontab — no confirmation prompt, no undo |

⚠️ **Warning:** `crontab -r` deletes your whole personal schedule immediately, with no "are you sure?" prompt. It's easy to type `-r` when you meant `-l`. Before ever running `-r` on a machine that matters, run `crontab -l > my-crontab-backup.txt` first, so you have something to restore from.

### 3. The 5-field cron time syntax

Every line in a crontab that isn't a comment or a special string follows this shape:

```
minute hour day-of-month month day-of-week   command-to-run
  0-59  0-23     1-31      1-12     0-7
```

- **`minute`** — 0 through 59.
- **`hour`** — 0 through 23, in 24-hour time (no AM/PM).
- **`day-of-month`** — 1 through 31.
- **`month`** — 1 through 12 (or the first three letters of the name: `jan`, `feb`, ...).
- **`day-of-week`** — 0 through 7, where **both 0 and 7 mean Sunday** (or the first three letters: `sun`, `mon`, ...).

Each field also accepts these operators:

| Symbol | Meaning | Example |
|---|---|---|
| `*` | "every value" — no restriction | `*` in the minute field means every minute |
| `,` | a list of specific values | `1,15` in day-of-month means the 1st and the 15th |
| `-` | an inclusive range | `9-17` in hour means 9 AM through 5 PM |
| `/` | a step within a range or `*` | `*/15` in minute means every 15 minutes (0, 15, 30, 45) |

**Worked example — breaking down a real entry:**

```
30 2 * * 1-5 /home/deploy/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Reading each field left to right:

1. `30` → at minute 30
2. `2` → of hour 2 (2 AM)
3. `*` → every day of the month
4. `*` → every month
5. `1-5` → Monday through Friday only

Put together: **"run `/home/deploy/scripts/backup.sh` at 2:30 AM, Monday through Friday, every month, regardless of the day of the month."** This is exactly the kind of schedule you'd use for a nightly backup you don't want running on weekends when nobody's around to notice if it fails.

💡 **Tip:** When `day-of-month` and `day-of-week` are **both** restricted (neither is `*`), cron runs the job if **either** condition matches — not only when both match. This is a well-known point of confusion; if you need "the 1st, but only if it's also a Monday," you can't express that with the 5-field syntax alone.

### 4. Cron's special strings

For the most common schedules, cron accepts a handful of shorthand strings in place of the 5 fields entirely:

| Special string | Equivalent to | Meaning |
|---|---|---|
| `@reboot` | (none) | Run once, at system startup |
| `@hourly` | `0 * * * *` | Run once, at the top of every hour |
| `@daily` | `0 0 * * *` | Run once, at midnight every day |
| `@weekly` | `0 0 * * 0` | Run once, at midnight every Sunday |
| `@monthly` | `0 0 1 * *` | Run once, at midnight on the 1st of the month |
| `@yearly` (or `@annually`) | `0 0 1 1 *` | Run once, at midnight on January 1st |

```
@daily /home/deploy/scripts/rotate-logs.sh >> /var/log/rotate-logs.log 2>&1
@reboot /home/deploy/scripts/start-health-monitor.sh >> /var/log/health-monitor.log 2>&1
```

✅ **Best Practice:** Use the special strings whenever they match your actual intent — `@daily` is more readable at a glance than `0 0 * * *`, and readability matters enormously for a schedule someone else will have to understand at 3 AM during an incident.

### 5. User crontabs vs. system-wide crontabs

So far, everything shown has been a **user crontab** — edited with `crontab -e`, stored per-user, and every job in it runs **as that user**. There's a second, system-wide mechanism:

- **`/etc/crontab`** — a single system-wide file, edited directly (not through `crontab -e`).
- **`/etc/cron.d/`** — a directory where packages and administrators can drop additional system-wide crontab-style files, one per package or purpose, without editing a shared file.

The critical difference: entries in `/etc/crontab` and `/etc/cron.d/*` have **one extra field** — the user to run the command as — inserted right after the 5 time fields and before the command:

```
# user crontab (crontab -e) — 5 fields, then the command:
0 3 * * * /home/deploy/scripts/backup.sh

# /etc/crontab or /etc/cron.d/* — 5 fields, THEN a user field, THEN the command:
0 3 * * * deploy /home/deploy/scripts/backup.sh
```

💡 **Why this exists:** A user crontab only ever needs to say *what* to run and *when* — it already knows *who*, because it's inherently tied to whoever ran `crontab -e`. The system-wide files have no single implicit owner, so they must say explicitly which user account each job should run as.

✅ **Best Practice:** For a personal or single-application schedule, `crontab -e` is almost always the right tool. Reach for `/etc/cron.d/` when you're packaging software for multiple users/systems, or when you want a job's schedule to live in version control alongside the application it belongs to, rather than only inside one user's private crontab.

### 6. The cron environment gotcha — a minimal, different environment

This is the single most important thing to understand before scheduling anything: **cron does not run your commands the same way your interactive login shell does.**

When you open a terminal and log in, your shell reads configuration files (`.bashrc`, `.bash_profile`, and so on, from Module 1) that set up a rich `PATH` (Module 2), environment variables, aliases, and more. Cron does **none of this**. It launches jobs with a **minimal, stripped-down environment** — often just a bare `PATH` like `/usr/bin:/bin`, no aliases, and none of the customizations your interactive shell has accumulated. There is also **no TTY** (no controlling terminal) and **no interactivity** — a command that would normally prompt you for input just hangs or fails, since there's no terminal attached for you to type into.

⚠️ **This is precisely why a script that works perfectly when you run it by hand can fail — often completely silently — the moment cron runs it instead.** If your script calls a command by its bare name (`node`, `aws`, `psql`) and that command only exists in a directory your interactive shell's `PATH` includes (like something installed under your home directory, or via a version manager), cron's minimal `PATH` may not include that directory at all — and the command simply isn't found.

### 7. Where cron's output goes — and why you must redirect it

By default, cron captures anything a job prints to stdout or stderr and tries to **email it** to the crontab's owner, using the local mail system. On most modern servers, **no mail system is configured at all**, so that output — including every error message — simply **vanishes**. Nobody sees it. The job can fail every single night for a month, and the only symptom is that the thing the job was supposed to produce (a backup, a report) quietly stops appearing.

✅ **Best Practice — always redirect a scheduled job's output to a log file, without exception:**

```
0 2 * * * /home/deploy/scripts/backup.sh >> /var/log/backup.log 2>&1
```

- `>> /var/log/backup.log` appends stdout to a log file (Module 5's redirection, `>>` rather than `>` so each run's output doesn't erase the last).
- `2>&1` redirects stderr into the same place stdout is going, so **both** the normal output and any error messages land in the one log file you can check.

🎯 **On the job:** "Redirect output to a log file" isn't optional polish — it's the difference between finding out your backup has been silently failing for three weeks when you desperately need last night's backup and it doesn't exist, versus finding out the very next morning when you glance at the log.

### 8. `at` — scheduling a one-time future job

**`at`** schedules a command to run **once**, at a specific future time — unlike cron, which is built around *recurring* schedules. Install it if it isn't already present:

```bash
sudo apt update
sudo apt install at
```

Basic usage:

```bash
echo "/home/deploy/scripts/send-report.sh" | at 5pm
```

or interactively:

```bash
at 5pm
at> /home/deploy/scripts/send-report.sh
at> <Ctrl-D>
```

Managing the queue of pending one-time jobs:

| Command | What it does |
|---|---|
| `atq` | List every job currently queued (with a job number) |
| `at -c <job-number>` | Print the full command/environment that will run for a queued job |
| `atrm <job-number>` | Cancel a queued job before it runs |

🎯 **On the job:** `at` is the right tool for "run this one specific thing later today" — deploying a change at a scheduled maintenance window, or sending a one-off reminder — where cron's recurring model would be overkill and you'd have to remember to remove the crontab entry afterward anyway.

### 9. `systemd` timers — the modern alternative

**`systemd`** is the init system and service manager used by Ubuntu, Debian, and most modern Linux distributions — the same system that starts and supervises long-running services (introduced conceptually back in Module 10/11's process and service management). A **`systemd` timer** is a `systemd` unit that triggers another unit (almost always a `.service` unit) on a schedule, instead of cron doing the triggering.

A timer always comes in a **pair** of files with the same base name:

- **`mytask.service`** — describes *what* to run (the command, working directory, user, environment).
- **`mytask.timer`** — describes *when* to run it (the schedule), and which `.service` it triggers.

`mytask.service`:

```ini
[Unit]
Description=Nightly database backup

[Service]
Type=oneshot
WorkingDirectory=/home/deploy/scripts
ExecStart=/home/deploy/scripts/backup.sh
```

`mytask.timer`:

```ini
[Unit]
Description=Run backup.service every night at 2:30 AM

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

- **`Type=oneshot`** means the service is expected to run once and exit, rather than staying running in the background (as a normal long-lived service would) — exactly the shape a scheduled script has.
- **`OnCalendar=`** uses `systemd`'s own calendar-event syntax (`*-*-* 02:30:00` means "every day, at 02:30:00"; Section on `OnCalendar` syntax has more examples below).
- **`Persistent=true`** tells `systemd` to run the job immediately on boot if the machine was off when a scheduled run was missed — something cron has no equivalent for at all.

Managing timers:

```bash
sudo systemctl enable --now mytask.timer   # enable at boot AND start it now
systemctl list-timers                      # see every active timer and its next run time
journalctl -u mytask.service               # see every past run's full logged output
```

💡 **Analogy:** If cron is a simple alarm clock, a `systemd` timer is a smart alarm clock connected to a logbook that automatically records exactly what happened every time it went off — whether it rang successfully, whether whoever it woke up actually got up, and what excuse they gave if they didn't. `journalctl` is that logbook, always available, with no separate log-file redirection required.

### 10. Cron vs. systemd timers — choosing the right tool

Both schedule commands. The real difference is in what happens **around** the running of the command:

- With cron, output is emailed-or-dropped by default, there's no dependency management (you can't easily say "don't run this until the network is up" or "only run this if the previous run finished"), and a failure produces no automatic record beyond whatever you manually redirected to a log file.
- With `systemd` timers, every run's output is automatically captured by `journalctl` (the `systemd` logging system) with no redirection needed, timers can declare dependencies on other units, `Persistent=true` catches up on missed runs after downtime, and `systemctl status`/`list-timers` give you first-class tooling to inspect what ran, what's scheduled next, and what failed — all without you building any of that yourself.

✅ **Best Practice:** For quick, personal, throwaway schedules — "remind me," a one-off script on your own dev machine — `crontab -e` is simpler and gets you there in ten seconds. For anything running in production, anything another engineer will need to debug at 2 AM, or anything where a silent failure is genuinely costly, a `systemd` timer's built-in logging and observability make it the professional default in modern Linux environments.

---

## Detailed Explanations

### The PATH gotcha, broken then fixed

**Broken version.** Imagine this script, `backup.sh`, works flawlessly every time you run it by hand:

```bash
#!/bin/bash
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
aws s3 cp /var/backups/db.sql "s3://my-backups/db-${TIMESTAMP}.sql"
echo "Backup uploaded successfully"
```

You add it to your crontab:

```
0 2 * * * /home/deploy/scripts/backup.sh
```

The next morning, no backup exists in the S3 bucket. There's no error you can see anywhere — cron's default mail-based output has nowhere to go, so it's simply gone (Concept 7). Running the exact same script by hand, right now, works perfectly.

**What actually happened:** `aws` (the AWS CLI) was installed under something like `/home/deploy/.local/bin/aws`, and your interactive login shell's `PATH` includes `~/.local/bin` because your `.bashrc` adds it. Cron's minimal environment (Concept 6) uses a bare `PATH` like `/usr/bin:/bin` — which does **not** include `~/.local/bin` — so when cron's shell tries to run `aws`, it gets **"aws: command not found"**, the script exits, and since output isn't being redirected anywhere you can see, that error message vanishes along with everything else.

**Fixed version** — two changes, both defensive and both worth doing together:

```bash
#!/bin/bash
set -euo pipefail

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
/home/deploy/.local/bin/aws s3 cp /var/backups/db.sql "s3://my-backups/db-${TIMESTAMP}.sql"
echo "Backup uploaded successfully"
```

```
0 2 * * * /home/deploy/scripts/backup.sh >> /var/log/backup.log 2>&1
```

- Using the **absolute path** to `aws` (`/home/deploy/.local/bin/aws`) means the script no longer depends on cron's `PATH` containing the right directory at all — it doesn't matter what `PATH` cron uses, because the script never asks the shell to search for `aws` in the first place.
- Adding `>> /var/log/backup.log 2>&1` means that if this ever fails again for a *different* reason, the error message lands somewhere you'll actually see it, instead of disappearing.

💡 **Tip:** An alternative (or complementary) fix is setting `PATH` explicitly at the very top of the crontab itself:

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/deploy/.local/bin
0 2 * * * /home/deploy/scripts/backup.sh >> /var/log/backup.log 2>&1
```

This fixes the problem for *every* job in that crontab at once, but absolute paths inside the script itself are more robust still — they work correctly even if someone runs the script from a completely different context (another crontab, a `systemd` timer, manually by a different user) where you don't control the `PATH` line at all.

### Why the working directory matters too

Cron doesn't just use a different `PATH` — it also doesn't `cd` into your home directory or wherever you happen to keep your scripts. A cron job's working directory is typically the crontab owner's home directory by default, but you should **never rely on that assumption**. A script that does `cp ./config.ini /etc/myapp/` assuming "the current directory is wherever my script lives" will silently look in the wrong place the moment it's run by something other than you, by hand, from inside that exact folder.

✅ **Best Practice:** Either `cd` explicitly to a known directory at the top of the script, or — more robustly — avoid relative paths entirely and reference everything by absolute path, exactly the same discipline as the `PATH` fix above.

```bash
#!/bin/bash
set -euo pipefail
cd /home/deploy/scripts || exit 1
```

---

## Practical Examples

### Example 1 — A full crontab entry, time-syntax breakdown

```
# Run the weekly report generator every Monday at 6:15 AM
15 6 * * 1 /home/deploy/scripts/weekly-report.sh >> /var/log/weekly-report.log 2>&1
```

Field-by-field:
- `15` — minute 15
- `6` — hour 6 (6 AM)
- `*` — any day of the month
- `*` — any month
- `1` — Monday (day-of-week: 0/7 = Sunday, 1 = Monday)

Realistic contents of `/var/log/weekly-report.log` after it runs:
```
Generating weekly report for week of 2026-07-20...
Report written to /home/deploy/reports/2026-07-27.pdf
Report emailed to team@example.com
```

💡 **Tip:** Comments in a crontab start with `#`, exactly like a Bash script — always leave one directly above each entry explaining what it does and why. A crontab with a dozen bare, unexplained schedule lines is a maintenance nightmare for whoever inherits it.

### Example 2 — The "why did my cron job silently fail" PATH-gotcha demo

Given this crontab entry:

```
*/5 * * * * /home/deploy/scripts/health-check.sh >> /var/log/health-check.log 2>&1
```

and `health-check.sh`:

```bash
#!/bin/bash
set -euo pipefail
STATUS=$(curl -s https://internal.example.com/health | jq -r .status)
echo "$(date): status is $STATUS"
```

Realistic contents of `/var/log/health-check.log` after cron starts running it:
```
/home/deploy/scripts/health-check.sh: line 3: jq: command not found
```

Line-by-line:
- `*/5 * * * *` means "every 5 minutes" (Concept 3's step operator) — the job is running exactly as scheduled.
- The problem isn't the schedule at all — it's that `jq` (a JSON-processing tool) was installed by hand under `/usr/local/bin/jq` on this box, which is present in the admin's interactive-shell `PATH` but **not** in cron's minimal `PATH`.
- Because `>> /var/log/health-check.log 2>&1` was already in place (Concept 7's best practice), the failure is **visible** — this is exactly the difference redirecting output makes: instead of a silent, invisible gap in monitoring, there's a clear, greppable error sitting in the log the moment you go looking.

The fix:

```bash
#!/bin/bash
set -euo pipefail
STATUS=$(curl -s https://internal.example.com/health | /usr/local/bin/jq -r .status)
echo "$(date): status is $STATUS"
```

Realistic contents of the log after the fix:
```
Wed Jul 29 00:05:01 UTC 2026: status is ok
Wed Jul 29 00:10:01 UTC 2026: status is ok
```

✅ **Best Practice:** When a script that works interactively fails under cron, the very first thing to suspect is `PATH` — confirm it by running `env -i /bin/bash /home/deploy/scripts/health-check.sh` interactively, which strips your environment down to nearly nothing before running the script, closely simulating cron's minimal environment without waiting for the next scheduled run.

### Example 3 — A one-time job with `at`

```bash
echo "/home/deploy/scripts/send-maintenance-notice.sh" | at 22:00
```

Realistic output:
```
warning: commands will be executed using /bin/sh
job 3 at Tue Jul 28 22:00:00 2026
```

Checking the queue:

```bash
atq
```

Realistic output:
```
3       Tue Jul 28 22:00:00 2026 a deploy
```

Canceling it if plans change:

```bash
atrm 3
```

Line-by-line:
- `at 22:00` schedules the piped-in command for 10 PM **today** (or tomorrow, if 22:00 has already passed) — a one-time event, not a recurring rule.
- `atq`'s output shows the job number (`3`), the exact scheduled time, the queue letter (`a`, the default queue), and the owning user (`deploy`) — everything needed to identify and manage it.
- `atrm 3` removes job 3 from the queue entirely; it will simply never run.

🎯 **On the job:** This is the right tool for "send this one maintenance notice tonight" — a cron entry for a single occurrence would need to be manually removed afterward anyway, so `at` is both simpler and self-cleaning.

### Example 4 — A full `systemd` timer + service pair

Creating the service unit, `/etc/systemd/system/backup.service`:

```ini
[Unit]
Description=Nightly database backup

[Service]
Type=oneshot
User=deploy
WorkingDirectory=/home/deploy/scripts
ExecStart=/home/deploy/scripts/backup.sh
StandardOutput=append:/var/log/backup.log
StandardError=append:/var/log/backup.log
```

Creating the timer unit, `/etc/systemd/system/backup.timer`:

```ini
[Unit]
Description=Run backup.service nightly at 2:30 AM

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enabling it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
```

Realistic output:
```
Created symlink /etc/systemd/system/timers.target.wants/backup.timer → /etc/systemd/system/backup.timer.
```

Checking it:

```bash
systemctl list-timers backup.timer
```

Realistic output:
```
NEXT                         LEFT     LAST  PASSED  UNIT          ACTIVATES
Wed 2026-07-29 02:30:00 UTC  6h left  n/a   n/a     backup.timer  backup.service
```

Viewing its execution history:

```bash
journalctl -u backup.service --since today
```

Realistic output:
```
Jul 29 02:30:00 web01 systemd[1]: Starting Nightly database backup...
Jul 29 02:30:03 web01 backup.sh[41213]: Backup uploaded successfully
Jul 29 02:30:03 web01 systemd[1]: backup.service: Deactivated successfully.
Jul 29 02:30:03 web01 systemd[1]: Finished Nightly database backup.
```

Line-by-line:
- `User=deploy` in the service file means the job runs as the `deploy` user without needing a separate personal crontab at all — the identity is declared explicitly in the unit file, right next to everything else about how the job runs.
- `StandardOutput=append:/var/log/backup.log` / `StandardError=append:/var/log/backup.log` is the `systemd` equivalent of cron's `>> file 2>&1` — though in practice you often don't even need this, since `journalctl -u backup.service` already captures everything automatically without any redirection at all.
- `systemctl daemon-reload` tells `systemd` to re-read unit files from disk — required any time you create or edit one, before `systemd` will recognize the change.
- `enable --now` does two things in one command: `enable` makes the timer start automatically on every future boot, and `--now` also starts it immediately, without waiting for the next reboot.
- `systemctl list-timers` shows exactly when this timer will next fire and how long ago it last ran — direct, built-in observability cron simply doesn't have.

✅ **Best Practice:** Always run `sudo systemctl daemon-reload` after creating or editing any unit file — `systemd` won't pick up the change otherwise, and the resulting confusion ("I edited the timer and nothing changed!") is a very common early mistake.

---

## Common Pitfalls & Best Practices

- **Using bare command names instead of absolute paths.** As Concept 6 and the PATH-gotcha example showed, cron's minimal `PATH` is the single most common reason a script that works by hand fails under cron. Use absolute paths for every external command a scheduled script calls, or explicitly set `PATH` at the top of the crontab.
- **Not redirecting output to a log file.** Without `>> logfile 2>&1` (cron) or relying on `journalctl` (`systemd`), a failing job's only symptom is that whatever it was supposed to produce silently stops appearing — with no error message anywhere a human will ever see.
- **Never testing the script manually before scheduling it.** A script that has never been run at all, on its own, from a clean shell, is not ready to be handed to an unattended scheduler. Test it manually first — ideally under conditions that resemble cron's minimal environment (Example 2's `env -i` trick) — before it ever goes into a crontab or timer.
- **Assuming the working directory is what you expect.** Relative paths (`./config.ini`, `../logs/`) inside a scheduled script are fragile, because the working directory a scheduler uses is not guaranteed to be the directory the script itself lives in. `cd` explicitly, or use absolute paths throughout.
- **DST and timezone surprises.** Cron schedules run according to the system's configured timezone (check with `timedatectl`), and a job scheduled for, say, 2:30 AM can behave unexpectedly on the specific night a Daylight Saving Time transition happens — that clock time might not exist, or might occur twice, depending on the transition direction. Production systems typically run on UTC for exactly this reason — it never observes DST, so a schedule means the same thing every day of the year, without exception.
- **Forgetting `crontab -e` is per-user.** Editing your own crontab has zero effect on jobs that need to run as a different user or with root privileges; those belong in `/etc/cron.d/` (with the extra user field) or a root crontab (`sudo crontab -e`), not your personal one.
- **Forgetting `daemon-reload` after editing a `systemd` unit file.** `systemd` reads unit files once and caches them; editing `backup.timer` on disk does nothing until you tell `systemd` to re-read it.

✅ **Best Practice — the "would a stranger understand this at 3 AM" test:** Before scheduling anything, imagine a different engineer, unfamiliar with this specific script, getting paged at 3 AM because it silently stopped working. Would they be able to find out *why*, from the log file or `journalctl` alone, without needing to SSH in and manually re-run the script to reproduce the failure? If not, add more logging before you schedule it, not after it's already failed in production.

---

## Hands-on Exercise

**Task:** You have a backup script that works fine when you run it manually. Schedule it to run every night with proper logging using cron, then recreate the exact same schedule as a `systemd` timer.

Starting script, `/home/deploy/scripts/nightly-backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

BACKUP_SRC="/var/lib/myapp/data"
BACKUP_DEST="/var/backups/myapp"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

[ -d "$BACKUP_SRC" ] || die "source directory missing: $BACKUP_SRC" 2

mkdir -p "$BACKUP_DEST"
tar czf "${BACKUP_DEST}/backup-${TIMESTAMP}.tar.gz" -C "$BACKUP_SRC" .

echo "Backup complete: ${BACKUP_DEST}/backup-${TIMESTAMP}.tar.gz"
```

Your job:
1. Test the script manually first, and confirm it works.
2. Schedule it via `crontab -e` to run every night at 1:00 AM, with output redirected to a log file.
3. Recreate the same schedule as a `systemd` timer + service pair, and enable it.
4. Verify both approaches with `crontab -l` and `systemctl list-timers` respectively.

Try it yourself before reading the solution.

### Solution

**Step 1 — test manually first:**

```bash
chmod +x /home/deploy/scripts/nightly-backup.sh
/home/deploy/scripts/nightly-backup.sh
```

```
Backup complete: /var/backups/myapp/backup-20260728-193000.tar.gz
```

✅ Confirmed working before it's ever handed to a scheduler — exactly the discipline Common Pitfalls called out.

**Step 2 — cron entry:**

```bash
crontab -e
```

Add this line:

```
0 1 * * * /home/deploy/scripts/nightly-backup.sh >> /var/log/nightly-backup.log 2>&1
```

Verify:

```bash
crontab -l
```

```
0 1 * * * /home/deploy/scripts/nightly-backup.sh >> /var/log/nightly-backup.log 2>&1
```

Field breakdown: `0` minute, `1` hour (1 AM), `*` every day-of-month, `*` every month, `*` every day-of-week — "every night at 1:00 AM." Output is appended to `/var/log/nightly-backup.log` with stderr folded into the same stream, so any failure (a missing source directory, a full disk) is captured, not dropped.

**Step 3 — the equivalent `systemd` timer + service:**

`/etc/systemd/system/nightly-backup.service`:

```ini
[Unit]
Description=Nightly backup of myapp data

[Service]
Type=oneshot
User=deploy
WorkingDirectory=/home/deploy/scripts
ExecStart=/home/deploy/scripts/nightly-backup.sh
```

`/etc/systemd/system/nightly-backup.timer`:

```ini
[Unit]
Description=Run nightly-backup.service every night at 1:00 AM

[Timer]
OnCalendar=*-*-* 01:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nightly-backup.timer
```

**Step 4 — verify both:**

```bash
crontab -l
systemctl list-timers nightly-backup.timer
```

Realistic output for the timer:
```
NEXT                         LEFT     LAST  PASSED  UNIT                   ACTIVATES
Wed 2026-07-29 01:00:00 UTC  5h left  n/a   n/a     nightly-backup.timer   nightly-backup.service
```

Checking a run's history (after it's fired at least once):

```bash
journalctl -u nightly-backup.service --since today
```

```
Jul 29 01:00:00 web01 systemd[1]: Starting Nightly backup of myapp data...
Jul 29 01:00:01 web01 nightly-backup.sh[52011]: Backup complete: /var/backups/myapp/backup-20260729-010001.tar.gz
Jul 29 01:00:01 web01 systemd[1]: Finished Nightly backup of myapp data.
```

Explanation of why both are equivalent-but-different: both run the identical script at the identical time. The cron version needs its own explicit `>> logfile 2>&1` to avoid silently dropped output (Concept 7); the `systemd` version gets that for free through `journalctl`, with no redirection syntax required at all. The `systemd` version also adds `Persistent=true`, meaning if the server happened to be powered off at 1:00 AM, the backup runs automatically the moment it boots back up — cron has no equivalent for this; a missed cron minute is simply gone forever.

✅ Exercise complete — you've scheduled the same real task two different ways, and can now explain concretely why the `systemd` version gives you more for the same amount of configuration effort.

---

## ✅ Module Completion Checklist

- [ ] I can explain why scheduling matters on the job, and what "unattended" implies about how a script must be written
- [ ] I can manage a user's personal schedule with `crontab -e`, `crontab -l`, and `crontab -r`
- [ ] I can read and write the 5-field cron time syntax, including ranges, steps, and lists
- [ ] I can use cron's special strings (`@reboot`, `@daily`, `@hourly`, `@weekly`) appropriately
- [ ] I can explain the difference between a user crontab and `/etc/crontab`/`/etc/cron.d/`, including the extra user field
- [ ] I can diagnose and fix the "works manually, fails under cron" problem caused by cron's minimal `PATH`/environment
- [ ] I can redirect a scheduled job's output to a log file so failures are never silently emailed or dropped
- [ ] I can schedule a one-time future job with `at`, and manage the queue with `atq`/`atrm`
- [ ] I can build and enable a `systemd` timer (`.timer` + `.service` pair)
- [ ] I can decide, for a given real-world task, whether cron or a `systemd` timer is the better tool

## Next Step

Continue to [Module 16: Production Scripting & Security Hardening](../module16-production-security-hardening/)
