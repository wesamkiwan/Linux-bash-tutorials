# 📋 Module 18 Cheat Sheet — Security Auditing Scripts

Fast reference for this module's scope only. See [master-cheatsheet.md](../master-cheatsheet.md) for the whole course.

## Core Vocabulary

| Term | Definition |
|---|---|
| **Security audit script** | A script that inspects a system for known weaknesses and reports findings, without fixing anything itself |
| **World-writable** | A file/directory where the "others" write bit is set — any user on the system can modify it |
| **UID 0** | Root-equivalent privilege; only the `root` account should ever have this |
| **SUID / SGID** | Special permission bits (Module 4) that make a program run as its owner/group rather than as the caller |
| **CVE** | Common Vulnerabilities and Exposures — the standard naming scheme for a publicly disclosed security flaw |
| **Brute-force attack** | Automated, repeated login attempts trying many credentials, hoping to eventually guess correctly |
| **Persistence** | A mechanism (e.g. a cron job) that lets an attacker's access/code survive reboots without re-exploiting anything |
| **PASS/WARN/FAIL** | The three-tier severity convention used to summarize an individual check's result |

## Security Audit One-Liners

| Purpose | Command |
|---|---|
| World-writable files, one filesystem | `find / -xdev -type f -perm -0002 2>/dev/null` |
| World-writable dirs, excluding sticky-bit ones | `find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null` |
| Unexpected UID 0 accounts | `awk -F: '$3 == 0 {print $1}' /etc/passwd` |
| SUID binaries, system-wide | `find / -xdev -type f -perm -4000 2>/dev/null` |
| SGID binaries, system-wide | `find / -xdev -type f -perm -2000 2>/dev/null` |
| SUID or SGID in one pass | `find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null` |
| Check `PermitRootLogin` | `grep -E "^\s*PermitRootLogin" /etc/ssh/sshd_config` |
| Check `PasswordAuthentication` | `grep -E "^\s*PasswordAuthentication" /etc/ssh/sshd_config` |
| Check legacy `Protocol` directive | `grep -E "^\s*Protocol" /etc/ssh/sshd_config` |
| List system-wide cron files | `cat /etc/cron.d/* 2>/dev/null` |
| List all personal crontab files | `ls -la /var/spool/cron/crontabs/` |
| Every user's crontab, looped | `for u in $(cut -f1 -d: /etc/passwd); do crontab -u "$u" -l 2>/dev/null; done` |
| Listening TCP/UDP ports + owning process | `sudo ss -tulpn` |
| Packages with available updates | `apt list --upgradable 2>/dev/null` |
| Security-specific updates only | `apt list --upgradable 2>/dev/null \| grep -i security` |
| Known-CVE scan of installed packages | `sudo debsecan` |
| Check auto-update mechanism is active | `dpkg -l unattended-upgrades \| grep ^ii` |
| Count failed SSH logins by source IP | `grep "Failed password" /var/log/auth.log \| awk '{for(i=1;i<=NF;i++) if($i=="from") print $(i+1)}' \| sort \| uniq -c \| sort -rn \| head` |
| Same, via journald instead of a flat file | `journalctl -u ssh --since "24 hours ago" \| grep "Failed password" \| awk '{for(i=1;i<=NF;i++) if($i=="from") print $(i+1)}' \| sort \| uniq -c \| sort -rn` |

## Risk-Severity Guide

| Finding | Typical severity | Why |
|---|---|---|
| A second account with UID 0 | 🔴 FAIL | Full root-equivalent access under a different, possibly innocuous name |
| Unrecognized SUID/SGID binary (esp. in `/tmp`, home dirs) | 🔴 FAIL | Classic root-shell backdoor pattern |
| `PermitRootLogin yes` | 🔴 FAIL | Root directly attackable over SSH with a password |
| `PasswordAuthentication yes` | 🟠 WARN–FAIL | Every account brute-forceable; severity depends on exposure (internet-facing vs. internal only) |
| World-writable script that runs as root/cron | 🔴 FAIL | Direct privilege-escalation path |
| World-writable data file, no execution path | 🟡 WARN | Integrity risk, but not immediate privilege escalation |
| World-writable dir with sticky bit (e.g. `/tmp`) | 🟢 PASS | Expected, safe-by-design pattern |
| Unexpected listening port/service | 🟠 WARN–FAIL | Depends entirely on what it is and whether it's bound to `0.0.0.0` |
| Database bound to `0.0.0.0` instead of `127.0.0.1` | 🟠 WARN | Often unintentional external exposure |
| Unrecognized cron job (system-wide or per-user) | 🟠 WARN–FAIL | Possible persistence mechanism |
| Outdated package, no known CVE | 🟡 WARN | Should be patched on a normal cadence, not an emergency |
| Outdated package with a known CVE (`debsecan` hit) | 🔴 FAIL | Publicly known, exploitable weakness present right now |
| `unattended-upgrades` not installed/enabled | 🟡 WARN | No safety net if manual patching lapses |
| A handful of failed SSH logins from varied IPs | 🟢 PASS | Normal human error |
| Hundreds of failed SSH logins from one IP | 🔴 FAIL | Active brute-force attack in progress |

## Expected SUID Baseline (stock Ubuntu, for comparison)

| Path | Purpose |
|---|---|
| `/usr/bin/passwd` | Lets any user update their own password (writes to root-only `/etc/shadow`) |
| `/usr/bin/sudo` | The `sudo` mechanism itself |
| `/usr/bin/su` | Switch-user shell |
| `/usr/bin/mount` / `/usr/bin/umount` | Mount/unmount filesystems |
| `/usr/bin/gpasswd` / `/usr/bin/newgrp` | Group membership/administration |
| `/usr/bin/chsh` / `/usr/bin/chfn` | Change your own shell/GECOS info |
| `/usr/lib/openssh/ssh-keysign` | SSH host-based authentication helper |

Anything **not** on a list like this, especially outside `/usr/bin`, `/usr/sbin`, or `/usr/lib`, deserves a closer look.

## `sshd_config` Hardened Values Quick Table

| Directive | Risky value | Hardened value |
|---|---|---|
| `PermitRootLogin` | `yes` | `no` or `prohibit-password` |
| `PasswordAuthentication` | `yes` | `no` (once key-based auth works for every needed account) |
| `Protocol` | *(historical — SSHv1 removed from modern OpenSSH entirely; a no-op today)* | typically omitted on modern configs |

## 🔁 The Quick Server Security Health-Check Workflow

Run through this on any new/unfamiliar server — take 5 minutes before you trust it:

1. **Check for extra root accounts** — `awk -F: '$3 == 0 {print $1}' /etc/passwd`. More than one line, stop and investigate immediately.
2. **Check SUID/SGID binaries** — `sudo find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null`. Scan for anything outside `/usr/bin`, `/usr/sbin`, `/usr/lib`.
3. **Check SSH hardening** — `grep -E "^\s*(PermitRootLogin|PasswordAuthentication)" /etc/ssh/sshd_config`. Both should be locked down.
4. **Check listening ports** — `sudo ss -tulpn`. Confirm every entry is something you recognize and expect.
5. **Check scheduled jobs** — `/etc/cron.d/*`, `/var/spool/cron/crontabs/`, and every user's `crontab -l`. Flag anything unfamiliar.
6. **Check for world-writable files in sensitive locations** — `sudo find /etc /var/www /opt -xdev -type f -perm -0002 2>/dev/null`.
7. **Check for pending security updates** — `apt list --upgradable 2>/dev/null | grep -i security`, and confirm `unattended-upgrades` is installed.
8. **Check failed login patterns** — count failed SSH logins by IP; anything in the hundreds means active brute-forcing right now.
9. **Write it all up as PASS/WARN/FAIL** and hand the report to whoever owns the decision on what to fix and in what order.

💡 **Tip:** Wrap all nine steps into one `security-audit.sh` script (this module's Hands-on Exercise builds exactly that) so this workflow becomes a single command, not nine things to remember under time pressure.
