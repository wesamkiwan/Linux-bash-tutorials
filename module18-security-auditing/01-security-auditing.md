# Module 18: Security Auditing Scripts 🔴

**Difficulty:** 🔴 Advanced
**Estimated Time:** 2.5 hours
**Prerequisites:** Modules 1-14 (builds directly on Modules 4 and 12)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Explain what a security audit script is, why it exists, and why it is a triage tool rather than a replacement for a real vulnerability scanner
- [ ] Find world-writable files and directories system-wide with `find`, explain exactly why a world-writable file is a security risk, and use `-xdev` to keep the search on one filesystem
- [ ] Find unexpected UID-0 (root-equivalent) accounts in `/etc/passwd` with `awk`, and explain why any line other than `root` is a critical finding
- [ ] Find SUID and SGID binaries system-wide with `find -perm`, and explain why an unrecognized SUID binary is one of the most serious findings an audit can turn up
- [ ] Audit `/etc/ssh/sshd_config` for risky settings — `PermitRootLogin`, `PasswordAuthentication`, and the historical `Protocol` directive — and know what a hardened value looks like for each
- [ ] Scan `/etc/cron.d/`, `/var/spool/cron/crontabs/`, and every user's crontab for unexpected scheduled jobs, tying this back to Module 15's coverage of cron
- [ ] Inspect listening ports and services with `ss -tulpn` (Module 12) and reason about which ones should — and should not — be reachable
- [ ] Check for outdated/vulnerable packages with `apt list --upgradable`, and explain what `debsecan` and `unattended-upgrades` each add on top of a manual check
- [ ] Scan `/var/log/auth.log` (or `journalctl`) for repeated failed SSH logins, count them by source IP with `awk`, and explain where `fail2ban` picks up where a read-only audit script stops
- [ ] Assemble multiple individual checks into a single script that prints a structured PASS/WARN/FAIL report, using functions and the hardened-script conventions from Module 14

---

## Module Goal

By the end of this module, you'll be able to sit down at an unfamiliar Ubuntu/Debian server — one you've just inherited, one about to go through a compliance audit, or one you're simply not sure has ever been looked at from a security angle — and, in a few minutes with nothing but bash and tools already on the box, produce a clear, structured answer to "what's actually wrong here, and how bad is it?"

🎯 **On the job:** Picture this: your manager tells you a customer's compliance auditor is showing up in three days to review your company's server fleet, and asks you to "just make sure everything looks okay first." You don't have budget approval for a commercial vulnerability scanner, and even if you did, provisioning it, getting it network access, and waiting for a scan to complete wouldn't happen in three days across every box. What you *do* have is SSH access and bash. This module is exactly that scenario: a fast, dependency-free, script-driven security health-check built entirely out of tools every Ubuntu/Debian server already has installed — `find`, `awk`, `grep`, `ss`, `apt` — assembled into a single script you can drop onto any server and run in seconds. This is a real, everyday skill for anyone doing systems administration, DevOps, or SRE work: a lightweight first pass that catches the obvious, common issues before anyone reaches for heavier tooling.

---

## Core Concepts

### 1. What is a security audit script?

A **security audit script** is a program — here, a bash script — that inspects a running system for known categories of security weakness and reports what it finds, without requiring any special software installation, network scanning, or external service. It doesn't *fix* anything by default; it **observes and reports**, the same way a smoke detector doesn't put out a fire — it tells you there's one so a human (or a separate, deliberate process) can act.

💡 **Analogy — the home inspector's checklist:** Think of a professional home inspector arriving at a house before a sale. They don't rebuild the house or replace the wiring themselves — they walk through it room by room with a checklist: is this window lock broken? Is there exposed wiring here? Is the smoke detector battery dead? At the end, they hand over a report: a list of items, each flagged as fine, needs attention, or urgent. The buyer (or the homeowner) then decides what to actually fix and in what order. A security audit script does exactly this for a Linux server — it walks through a checklist of well-known trouble spots (file permissions, unexpected privileged accounts, risky SSH settings, unexplained scheduled jobs, open ports, outdated packages, suspicious login attempts) and hands back a report. It doesn't rewire anything itself.

### 2. Why this matters even though "real" security tools exist

Dedicated security tools — vulnerability scanners, intrusion detection systems, commercial compliance platforms — exist and, for an organization with the budget and time to deploy them properly, are usually the *better* long-term answer (Module References points to some real ones, like Lynis). But a hand-rolled audit script earns its place for a different reason: it's **instant**. It requires no installation (every tool it uses ships on a stock Ubuntu/Debian install, or is one `apt install` away), no agents, no network access to an external scanning service, and no waiting. It's the fast first pass you run the moment you get SSH access to a server you don't fully trust yet — before deciding whether heavier tooling is even warranted.

🎯 **On the job:** This is genuinely how a lot of real incident response and onboarding work starts. Before you spin up Lynis or a full compliance scan, you SSH in and run a handful of quick checks to get oriented: is anything obviously, glaringly wrong? That triage step is exactly what this module teaches you to automate.

### 3. World-writable files — why "anyone can edit this" is dangerous

A file is **world-writable** when the "others" permission bit (Module 4's third permission category — anyone who isn't the owner and isn't in the owning group) includes write access. In octal terms, that's the `2` bit being set in the "others" digit — e.g. permissions like `666` or `777`.

The risk is direct: if *any* user account on the system — including a low-privileged one, or one an attacker only just barely compromised — can modify or completely replace the contents of a file, then that file can no longer be trusted to contain what its owner put there. If that file happens to be a script that runs automatically (a cron job, a service's startup script, something root eventually executes), a world-writable permission on it is effectively an open invitation for **privilege escalation** — a low-privileged attacker edits the file, waits for it to run as a more privileged user, and their injected code now runs with that higher privilege too.

⚠️ **Warning:** World-writable *directories* carry a related but distinct risk: anyone can create, delete, or rename files inside them (Module 4, Concept 5), which means an attacker could potentially delete a legitimate file and replace it with a malicious one of the same name, even if the file itself was never writable. The one common, *legitimate* exception is a directory like `/tmp`, which is world-writable by design — but it's also protected by the **sticky bit** (Module 4), which stops any user from deleting or replacing files they don't personally own even though the directory itself is world-writable. A world-writable directory **without** the sticky bit is a much bigger red flag than one with it.

### 4. `-xdev` — staying on one filesystem

`find`'s default behavior is to recurse into *every* directory it's given, including ones that are actually separate mounted filesystems — network shares, other mounted drives, `/proc`, `/sys` (Module 11's territory) — many of which are enormous, irrelevant to a local security audit, or simply not real disk-backed filesystems at all. The `-xdev` flag tells `find` to stay on the same filesystem the starting point (`/`) is on, and not descend into anything mounted from elsewhere.

This matters for two reasons: it's dramatically **faster** (you're not needlessly walking a mounted NFS share or a virtual `/proc` tree with millions of synthetic entries), and it keeps the results **relevant** — you're auditing this machine's own root filesystem, not somebody else's NFS export that happens to be mounted underneath it.

### 5. Unexpected UID 0 — more than one "root"

Every user account has a numeric **UID** (user ID), and by convention, UID `0` is reserved for exactly one account: `root`, the superuser (Module 4) that bypasses nearly all permission checks. Linux itself doesn't actually *enforce* that only one account can have UID 0 — `/etc/passwd` is just a text file, and nothing stops an entry like `backdoor:x:0:0::/root:/bin/bash` from existing alongside the real `root` line.

If it does exist, **any account with UID 0 has full root privileges**, indistinguishable from root itself, regardless of its username. This is a classic, well-documented technique for planting a hidden backdoor: create an innocuous-looking username with UID 0, and it has exactly as much power as `root` without the name "root" ever appearing suspicious in a casual glance at `who` or `ps`. Finding any line other than `root` with UID 0 is a **critical, stop-what-you're-doing finding**.

### 6. SUID and SGID binaries — why an unrecognized one is a red flag

Module 4 introduced **setuid** and **setgid**: a setuid executable runs with the privileges of the file's *owner* (often root) rather than the privileges of whoever launched it — this is exactly how an ordinary user can run `/usr/bin/passwd` to update their own password, even though that requires briefly writing to the normally root-only `/etc/shadow`.

That mechanism is legitimate and necessary for a small, well-known set of system binaries. The security concern is this: **every setuid-root binary is a potential path to root privilege for anyone who can execute it**, so the attack surface of a system grows with every additional SUID binary present, and a *new, unexpected* one that wasn't there before (or isn't part of the OS's standard install) is one of the strongest signals of a system that's been tampered with — either a legitimately installed but poorly-vetted program with its own vulnerabilities, or a deliberately planted backdoor (attackers who gain any foothold on a box will often copy a shell binary somewhere and `chmod` it setuid-root, so that running it later hands them a root shell instantly, without needing to re-exploit anything).

✅ **Best Practice:** An audit doesn't need to know every SUID binary is *bad* — most of what it finds will be completely normal, expected system binaries. What it needs to do is surface the **complete list**, so a human can compare it against what's expected and immediately spot anything that doesn't belong.

### 7. SSH hardening basics, revisited from an audit angle

Module 12 covered SSH as a tool for connecting to remote machines. This module looks at the *other side* of that connection: the **server's** configuration, in `/etc/ssh/sshd_config`, and specifically three settings an auditor checks first.

**`PermitRootLogin`** controls whether `root` can log in over SSH at all. A value of `yes` means an attacker who guesses or steals the root password (or a root key) can log straight in as the single most powerful account on the box. Hardened values are `no` (root can never SSH in directly — administrators use `sudo` after logging in as themselves instead, exactly the model Module 4 recommended) or `prohibit-password` (root can only log in with a pre-installed key, never a password — sometimes needed for specific automation, but still far safer than allowing a password).

**`PasswordAuthentication`** controls whether *any* account can log in with a plain password, as opposed to only a cryptographic key (Module 12's key-based auth). `yes` leaves every account vulnerable to online password-guessing (brute-force) attacks over the network. A hardened server sets this to `no` once key-based login is confirmed working for every account that needs access — exactly the recommendation Module 12 made from the client side, now viewed from the server's own configuration.

**`Protocol`** is a historical setting from an era when OpenSSH supported two entirely different, incompatible protocol versions: the original **SSHv1**, which had known, serious cryptographic weaknesses, and **SSHv2**, the redesigned, secure successor that's been the only protocol in real-world use for well over a decade. `Protocol 2` used to be an explicit, important hardening step to make sure a server wouldn't fall back to the broken SSHv1. On any modern OpenSSH version (which is all Ubuntu/Debian has shipped for many years), **SSHv1 support has been removed from the code entirely** — the `Protocol` directive is now a no-op if present at all, and modern `sshd_config` files typically don't mention it. It's still worth knowing about and checking for, both because you may encounter it in very old configs or documentation, and because "why doesn't this setting do anything anymore" is exactly the kind of gap in understanding that trips people up during an actual audit.

### 8. Cron jobs — the persistence mechanism auditors check

Module 15 covers `cron` as a scheduling tool. From a security angle, cron is also one of the most common places an attacker (or a well-meaning but undocumented internal script) establishes **persistence** — a scheduled job that keeps running their code at intervals, surviving reboots, without anyone needing to actively re-trigger it. An audit needs to enumerate *every* place a scheduled job could be hiding:

- **`/etc/cron.d/`** — a directory of system-wide cron job files, each one plain text, each line specifying a schedule and a command, typically installed by system packages or administrators directly.
- **`/var/spool/cron/crontabs/`** — where each user's *personal* crontab (edited with `crontab -e`) is actually stored on disk, one file per user who has one.
- **Every individual user's crontab**, checked with `crontab -u <username> -l` — looping over every account on the system, since a compromised low-privilege account could have its own hidden scheduled job that nobody but that account's owner would normally think to check.

### 9. Listening ports — what shouldn't be there

Module 12 introduced `ss -tulpn` as the tool for inspecting listening network services. From a security-audit perspective, the goal shifts slightly: instead of "what's listening" being purely diagnostic information, you're asking **"does anything here surprise me?"** A web server on 80/443, SSH on 22, and a database bound only to `127.0.0.1` are all expected on an app server. A service listening on a port nobody recognizes, or a database unexpectedly bound to `0.0.0.0` (reachable from *any* network, not just locally) instead of `127.0.0.1`, is exactly the kind of thing an audit exists to surface.

### 10. Outdated and vulnerable packages

Software has bugs, and some of those bugs are security vulnerabilities that get fixed in later versions. A package that hasn't been updated in a long time is a package that's potentially running with **publicly known, already-patched** vulnerabilities — often the easiest kind for an attacker to exploit, since the fix (and sometimes exploit code) is public knowledge. `apt list --upgradable` gives you the basic list of what's out of date on a Debian/Ubuntu system. Two tools go further:

- **`debsecan`** cross-references your installed package versions against a database of known security vulnerabilities (CVEs — Common Vulnerabilities and Exposures, the industry-standard naming scheme for publicly disclosed security flaws) and tells you specifically *which* installed packages have a known, named vulnerability — not just "there's a newer version," but "this specific version has this specific documented weakness."
- **`unattended-upgrades`** is the standard Ubuntu/Debian mechanism for *automatically* installing security updates on a schedule, without a human needing to remember to run `apt upgrade` regularly. An audit doesn't just ask "is everything updated right now" — it also asks "is there even a mechanism in place to keep this updated automatically going forward," and checks whether `unattended-upgrades` is installed and enabled.

### 11. Scanning logs for failed logins

`/var/log/auth.log` (or, on systems using `systemd`'s journal exclusively, `journalctl -u ssh`) records every authentication attempt against SSH, including every failed one. A handful of failed logins from a real user who mistyped their password is normal. **Dozens or hundreds of failed attempts from the same source IP** in a short window is the signature of a **brute-force attack** — an automated tool trying username/password combinations as fast as the network allows, hoping to eventually guess right.

A simple `grep`/`awk` count-by-IP over recent log entries is enough to surface this pattern instantly: sort the offending IPs by how many failed attempts they've racked up, and the worst offenders float straight to the top.

⚠️ **This is detection, not prevention.** A script that reads the log and reports "IP 203.0.113.55 failed 400 times in the last hour" has told you something is happening — it hasn't stopped anything. **`fail2ban`** is the production-grade tool that closes that gap: it watches log files continuously, and when it detects a pattern like this in real time, it automatically adds a firewall rule to temporarily (or permanently) block that IP address, with zero human intervention required. Think of the audit script's log check as the smoke detector; `fail2ban` is the sprinkler system that actually puts the fire out.

### 12. Assembling checks into one PASS/WARN/FAIL report

Running eight or nine separate one-off commands and eyeballing each output separately doesn't scale, and it's easy to miss something buried in a wall of text. The professional pattern is to wrap each check in its own **function** (Module 8), have each function decide independently whether its result is a **PASS** (nothing concerning found), a **WARN** (something worth a human's attention, but not necessarily an emergency), or a **FAIL** (a clear, serious problem), and print a consistently-formatted line for each — then tally the totals at the end so a reader gets both the detail and a one-line summary ("3 FAIL, 5 WARN, 12 PASS") without reading the whole report line by line.

---

## Detailed Explanations

### Why `-xdev` isn't just a performance nicety

It's tempting to think of `-xdev` as purely a speed optimization — and it is faster — but it also changes *what you're actually auditing*, which matters just as much. A server commonly has `/proc` and `/sys` mounted (Module 11) — these are **virtual filesystems** that don't represent real files on disk at all, but a live view into kernel and process state, and they can contain millions of synthetic entries that are meaningless to a permissions audit and can make `find` appear to hang. Servers also frequently have network shares (NFS, CIFS) or removable media mounted under various paths. Without `-xdev`, a single `find / -perm -0002` can wander into all of that, taking dramatically longer to finish and reporting on filesystems that may not even be under your organization's control to fix (an NFS share owned by another team, for instance). `-xdev` keeps the audit scoped to *this machine's own root filesystem* — the thing you're actually responsible for and auditing in the first place.

### Why an unexpected SUID binary specifically, and not just "any SUID binary," is the signal

A stock Ubuntu install ships with a well-known, fairly small, stable list of SUID binaries — things like `/usr/bin/passwd`, `/usr/bin/sudo`, `/usr/bin/su`, `/usr/bin/mount`, `/usr/bin/umount`, `/usr/bin/newgrp`, `/usr/bin/gpasswd`, `/usr/bin/chsh`, `/usr/bin/chfn`, and a few others tied to package managers or specific installed software (e.g. `/usr/lib/openssh/ssh-keysign`). None of these, by themselves, are a problem — they're deliberately designed, carefully maintained programs that each have a legitimate, narrow reason to run with elevated privilege. The actual audit value comes from **comparing today's list against yesterday's** (or against a known-good baseline for your organization's standard server image). A brand-new entry appearing in that list — especially something in an unusual location like `/tmp`, `/var/tmp`, or a user's home directory, or something with an unfamiliar name — is exactly what a real intrusion often looks like: an attacker who's gained *some* foothold copies `/bin/bash` (or a similar shell) somewhere, runs `chmod u+s` on it, and now has an instant, silent path back to a root shell any time they want it, without re-exploiting anything.

### Why `PermitRootLogin no` and `PasswordAuthentication no` are treated as a matched pair

These two settings are often discussed together because they close two different halves of the same door. `PermitRootLogin no` stops the single most powerful account from ever being a direct SSH target at all — even a perfectly-guessed root password (or leaked root key) does nothing if root can't log in over SSH in the first place; an administrator instead logs in as their own named account and uses `sudo`, which (Module 4) leaves a clear, per-command audit trail of exactly who did what. `PasswordAuthentication no` closes the second half: it removes the *weaker* authentication method (something you know, which can be guessed, phished, or brute-forced) from every account, not just root, leaving only the *stronger* method (something you cryptographically possess — a private key, per Module 12) as the way in. Setting only one of the two still leaves a real gap: `PermitRootLogin no` alone still permits brute-forcing passwords against every *other* account; `PasswordAuthentication no` alone still permits an attacker who somehow obtains a root key (or root's *own* stored key material) to log straight in as root. A genuinely hardened server sets both.

---

## Practical Examples

### Example 1 — Finding world-writable files and directories

```bash
find / -xdev -type f -perm -0002 2>/dev/null
```

Realistic output:
```
/var/www/html/uploads/config.php
/opt/legacy-app/run.sh
```

```bash
find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null
```

Realistic output:
```
/opt/legacy-app/scripts
```

Line-by-line:
- `-type f -perm -0002` matches regular files where the "others" write bit is set (Concept 3) — `-perm -0002` means "at least these bits are set," not "exactly these bits," so it correctly catches `666`, `777`, and anything else with that bit on.
- `2>/dev/null` discards the inevitable stream of `Permission denied` errors `find` produces when it tries to descend into directories your current user can't read — without a `sudo` (see Common Pitfalls), those errors would otherwise clutter every real result.
- The second command checks **directories**, but adds `! -perm -1000` — "and does NOT have the sticky bit set" (Module 4) — specifically to exclude legitimate cases like `/tmp`, which is *supposed* to be world-writable and is safe *because* of its sticky bit. What's left after that exclusion is the genuinely concerning case: a world-writable directory where any user could also delete or replace someone else's files inside it.
- Finding `/opt/legacy-app/run.sh` world-writable is a serious hit — if that script ever runs as a more privileged user (a cron job owned by root, for instance), any unprivileged user on the box could edit it and have their own code run with that higher privilege the next time it fires.

⚠️ **Warning:** Run this as `root` (or with `sudo`) for a complete result — as a regular user, `find` simply cannot see into directories it doesn't have read/execute access to, and will silently under-report, not over-report.

### Example 2 — Checking for unexpected UID 0 accounts

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd
```

Realistic output (healthy system):
```
root
```

Realistic output (compromised system):
```
root
sysmaint
```

Line-by-line:
- `-F:` tells `awk` to split each line of `/etc/passwd` on `:` — recall from Module 4 that `/etc/passwd` is colon-separated, with the UID as the third field.
- `$3 == 0` tests whether that third field (the UID) equals `0`; `{print $1}` prints the first field (the username) for every line that matches.
- A healthy result is **exactly one line: `root`**. Any additional line — like `sysmaint` here, an innocuous-looking name someone might easily overlook in a casual `cat /etc/passwd` — means a second account has full root-equivalent privilege. That's an immediate, critical finding: investigate that account's origin immediately, since it's a classic backdoor pattern (Concept 5).

🎯 **On the job:** This single-line check takes a fraction of a second to run and catches one of the most serious and most commonly *missed* findings on a compromised box, because nobody thinks to scroll through the whole of `/etc/passwd` checking every UID by eye.

### Example 3 — Finding SUID and SGID binaries system-wide

```bash
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -exec ls -la {} \; 2>/dev/null
```

Realistic output:
```
-rwsr-xr-x 1 root root   68208 Mar  1  2024 /usr/bin/passwd
-rwsr-xr-x 1 root root   88464 Mar  1  2024 /usr/bin/sudo
-rwxr-sr-x 1 root shadow  22984 Mar  1  2024 /usr/bin/chage
-rwsr-xr-x 1 root root   35112 Jul 28 09:41 /tmp/.hidden/bash
```

Line-by-line:
- `-perm -4000` matches the setuid bit; `-perm -2000` matches setgid (Module 4's octal special-bit values); `\( ... -o ... \)` is `find`'s grouped "or," so this single command catches both categories in one pass.
- `-exec ls -la {} \;` runs `ls -la` against every match found, so the audit output shows full permission strings, owner, and path together, rather than just bare filenames.
- The first three results — `passwd`, `sudo`, `chage` — are exactly the kind of well-known, expected system binaries described in the Detailed Explanations above; nothing alarming there.
- The **last line is the entire point of this check**: a setuid-root binary named `bash` sitting in a hidden directory (`/tmp/.hidden/`) that has no business containing an SUID binary at all. This is a textbook backdoor — running `/tmp/.hidden/bash` gets an attacker (or anyone who stumbles onto it) an instant root shell, no exploit required, because the setuid bit alone does all the work.

✅ **Best Practice:** Keep a saved baseline of this exact command's output from a known-clean install of your standard server image, and `diff` today's result against it — that turns "eyeball a list and hope something jumps out" into a precise, automatic answer to "what's new since last time."

### Example 4 — Auditing `sshd_config` for risky settings

```bash
grep -E "^\s*(PermitRootLogin|PasswordAuthentication|Protocol)\b" /etc/ssh/sshd_config
```

Realistic output (risky configuration):
```
PermitRootLogin yes
PasswordAuthentication yes
```

Realistic output (hardened configuration):
```
PermitRootLogin prohibit-password
PasswordAuthentication no
```

Line-by-line:
- `^\s*` anchors the match to the start of the line, allowing for leading whitespace (some configs indent directives) — this avoids accidentally matching the setting name inside a comment or later in a line.
- `(PermitRootLogin|PasswordAuthentication|Protocol)\b` matches any of the three directive names, with `\b` (a word boundary) making sure `Protocol` doesn't also accidentally match something like `ProtocolFamily` if that ever existed.
- The **risky** output shows `PermitRootLogin yes` (root can SSH in directly with a password) and `PasswordAuthentication yes` (any account can be brute-forced over the network) — both a FAIL by the standard set in Concept 7.
- The **hardened** output shows the two settings pulling in the right direction — `prohibit-password` for root (key-only, no root password login at all) and `no` for general password auth. Note that `Protocol` doesn't even appear in the hardened example — as Concept 7 explained, it's been a no-op on modern OpenSSH for years, and current best-practice configs simply omit it.

⚠️ **Warning:** If a directive doesn't appear in `sshd_config` at all, that does **not** automatically mean it's insecure — it means the compiled-in **default** applies, and you need to know what that default is for the OpenSSH version actually installed (`sshd -V` reports the version) rather than assuming. A truly thorough audit script should flag a *missing* line for a security-relevant setting as "WARN — verify default," not silently treat it as fine.

### Example 5 — Counting failed SSH logins by source IP

```bash
grep "Failed password" /var/log/auth.log \
  | awk '{for (i=1;i<=NF;i++) if ($i=="from") print $(i+1)}' \
  | sort | uniq -c | sort -rn | head -5
```

Realistic output:
```
    412 203.0.113.55
     37 198.51.100.12
      3 203.0.113.90
      1 192.0.2.44
```

Line-by-line:
- `grep "Failed password"` narrows `/var/log/auth.log` down to just failed authentication attempts, discarding every other log line (successful logins, session-open messages, unrelated services).
- The `awk` command loops over every field (`for (i=1;i<=NF;i++)`) on each matching line and prints whatever field comes immediately **after** the literal word `from`. This is more robust than counting fields from a fixed position, because SSH's actual log format shifts depending on whether the attempted username was valid or not (`Failed password for root from 203.0.113.55 port 22 ssh2` versus `Failed password for invalid user admin from 203.0.113.55 port 22 ssh2` — the IP sits in a different fixed column in each, but it's always the token right after `from`).
- `sort | uniq -c` counts how many times each unique IP appears; the second `sort -rn` re-sorts that count numerically, descending, so the worst offender is first; `head -5` keeps just the top five.
- `203.0.113.55` with **412** failed attempts is an unmistakable brute-force signature — a real user mistyping a password does not generate 412 failures. `198.51.100.12` at 37 is worth watching. The last two, at 3 and 1, are well within the range of ordinary human error and don't warrant concern on their own.

🎯 **On the job:** This exact pattern is the fastest way to answer "is someone actively trying to break in right now" during an audit — and it's also exactly the pattern `fail2ban` (Concept 11) automates continuously in production, so you never have to run this by hand to catch an attack as it's happening.

---

## Common Pitfalls & Best Practices

- **Running system-wide `find` without `-xdev` on a large or heavily-mounted server.** Without it, `find / -perm -0002` can wander into `/proc`, `/sys`, mounted network shares, and other filesystems, taking far longer than necessary and reporting on things outside the scope of the machine you're actually auditing. Always scope with `-xdev` (Concept 4) unless you have a specific, deliberate reason to cross filesystem boundaries.
- **Forgetting `sudo`, and mistaking silence for a clean result.** Several of these checks — especially the `find`-based ones and reading `/etc/shadow`-adjacent information — need elevated privileges to see everything on the system. Run as a regular user, `find` will silently skip directories it can't read and simply produce fewer results, which looks identical to "nothing was found" unless you're specifically watching for the `Permission denied` messages that `2>/dev/null` (Example 1) quietly hides. Run your audit script with `sudo` for a trustworthy result.
- **Treating a missing `sshd_config` directive as automatically safe.** As Example 4's warning notes, an absent line means "the compiled-in default applies" — not "this is configured securely." Know the actual default for your installed OpenSSH version, or explicitly set the directive so there's no ambiguity at all.
- **Slow, brute-force `find` invocations on genuinely huge filesystems.** Even with `-xdev`, a `find /` over a server with millions of files (a large fileserver, for instance) can take a meaningful amount of time. This is expected and acceptable for a periodic audit run — but don't schedule it every minute, and consider narrowing scope (e.g. auditing `/home`, `/opt`, `/var/www` specifically, rather than the entire root filesystem) if you need faster, more frequent checks.
- **Treating this script as a full replacement for real security tooling.** A hand-rolled bash audit catches well-known, common misconfigurations fast and for free — it does **not** perform deep vulnerability analysis, doesn't understand application-level flaws, and doesn't stay current with newly disclosed CVEs the way a dedicated tool like Lynis or a commercial scanner does. Use it as the fast first pass and daily/weekly sanity check it's good at, and reach for dedicated tools (Module References) for anything that needs to satisfy a real compliance framework or catch genuinely novel threats.
- **False positives on `find -perm` checks.** Not every world-writable file or every SUID binary is malicious — plenty of legitimate software installs its own SUID helper (screen, some VPN clients, certain games) or intentionally uses a world-writable file for interprocess communication. An audit's job is to **surface** these for a human to judge, not to assume every hit is automatically an incident. Investigate before you panic — but always investigate.

✅ **Best Practice — the "observe, don't fix" boundary:** A well-designed audit script only ever *reports*. It should never automatically delete a file, kill a process, or lock an account based on what it finds — a false positive turned into an automatic destructive action is its own kind of incident. Leave remediation to a human decision (or a separate, deliberately-reviewed automation) informed by the report.

---

## Hands-on Exercise

**Task:** Build a single script, `security-audit.sh`, that combines several of this module's checks into one PASS/WARN/FAIL report. Specifically, it must:

1. Use `set -euo pipefail` (Module 14) and define a small helper function to print a consistently-formatted `PASS`/`WARN`/`FAIL` line for each check, along with a running count of each.
2. Check for unexpected UID-0 accounts.
3. Check for world-writable files under `/etc` and `/var/www` (a narrower, faster scope than the whole filesystem, for a quick run).
4. Check `PermitRootLogin` and `PasswordAuthentication` in `/etc/ssh/sshd_config`.
5. Check for the top 3 IPs with the most failed SSH logins in `/var/log/auth.log`, flagging a WARN if any IP has more than 10 failures.
6. Print a final summary line: total PASS / WARN / FAIL counts.

Try building this yourself before reading the solution below.

### Solution

```bash
#!/bin/bash
set -euo pipefail

# --- counters -----------------------------------------------------------
PASS_COUNT=0
WARN_COUNT=0
FAIL_COUNT=0

# --- reporting helper -----------------------------------------------------
# report <STATUS> <message>   where STATUS is PASS, WARN, or FAIL
report() {
    local status="$1"
    local message="$2"
    case "$status" in
        PASS) PASS_COUNT=$((PASS_COUNT + 1)) ;;
        WARN) WARN_COUNT=$((WARN_COUNT + 1)) ;;
        FAIL) FAIL_COUNT=$((FAIL_COUNT + 1)) ;;
    esac
    printf "[%-4s] %s\n" "$status" "$message"
}

# --- check 1: unexpected UID 0 accounts -----------------------------------
check_uid_zero() {
    local extra_root_accounts
    extra_root_accounts=$(awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd)

    if [ -z "$extra_root_accounts" ]; then
        report PASS "Only 'root' has UID 0"
    else
        report FAIL "Unexpected UID 0 account(s) found: $extra_root_accounts"
    fi
}

# --- check 2: world-writable files in high-value locations ----------------
check_world_writable() {
    local hits
    hits=$(find /etc /var/www -xdev -type f -perm -0002 2>/dev/null)

    if [ -z "$hits" ]; then
        report PASS "No world-writable files under /etc or /var/www"
    else
        local count
        count=$(echo "$hits" | wc -l)
        report WARN "$count world-writable file(s) found under /etc or /var/www: $(echo "$hits" | tr '\n' ' ')"
    fi
}

# --- check 3: sshd_config risky settings -----------------------------------
check_sshd_config() {
    local sshd_conf="/etc/ssh/sshd_config"
    local root_login
    root_login=$(grep -E "^\s*PermitRootLogin" "$sshd_conf" | awk '{print $2}' || true)

    case "$root_login" in
        no|prohibit-password) report PASS "PermitRootLogin is '$root_login'" ;;
        yes) report FAIL "PermitRootLogin is 'yes' — root can log in with a password over SSH" ;;
        *) report WARN "PermitRootLogin not explicitly set — verify the compiled-in default for this OpenSSH version" ;;
    esac

    local pass_auth
    pass_auth=$(grep -E "^\s*PasswordAuthentication" "$sshd_conf" | awk '{print $2}' || true)

    case "$pass_auth" in
        no) report PASS "PasswordAuthentication is 'no'" ;;
        yes) report FAIL "PasswordAuthentication is 'yes' — accounts are brute-forceable over the network" ;;
        *) report WARN "PasswordAuthentication not explicitly set — verify the compiled-in default for this OpenSSH version" ;;
    esac
}

# --- check 4: failed SSH login attempts by IP ------------------------------
check_failed_logins() {
    local logfile="/var/log/auth.log"

    if [ ! -r "$logfile" ]; then
        report WARN "$logfile not readable — run as root/sudo, or check journalctl instead"
        return
    fi

    local top_offender
    top_offender=$(grep "Failed password" "$logfile" \
        | awk '{for (i=1;i<=NF;i++) if ($i=="from") print $(i+1)}' \
        | sort | uniq -c | sort -rn | head -1)

    if [ -z "$top_offender" ]; then
        report PASS "No failed SSH login attempts found in $logfile"
        return
    fi

    local top_count
    top_count=$(echo "$top_offender" | awk '{print $1}')
    local top_ip
    top_ip=$(echo "$top_offender" | awk '{print $2}')

    if [ "$top_count" -gt 10 ]; then
        report WARN "IP $top_ip has $top_count failed SSH login attempts — possible brute-force"
    else
        report PASS "No IP exceeds 10 failed SSH attempts (worst: $top_ip with $top_count)"
    fi
}

# --- run every check, then print the summary -------------------------------
echo "=== Security Audit Report ==="
check_uid_zero
check_world_writable
check_sshd_config
check_failed_logins
echo "=============================="
echo "Summary: $PASS_COUNT PASS, $WARN_COUNT WARN, $FAIL_COUNT FAIL"

if [ "$FAIL_COUNT" -gt 0 ]; then
    exit 2
elif [ "$WARN_COUNT" -gt 0 ]; then
    exit 1
fi
exit 0
```

Realistic output, run with `sudo ./security-audit.sh`:
```
=== Security Audit Report ===
[FAIL] Unexpected UID 0 account(s) found: sysmaint
[PASS] No world-writable files under /etc or /var/www
[FAIL] PermitRootLogin is 'yes' — root can log in with a password over SSH
[FAIL] PasswordAuthentication is 'yes' — accounts are brute-forceable over the network
[WARN] IP 203.0.113.55 has 412 failed SSH login attempts — possible brute-force
==============================
Summary: 1 PASS, 1 WARN, 3 FAIL
```

Explanation: `set -euo pipefail` (Module 14) means an unexpected failure inside the script — a command that isn't installed, a typo — stops the script loudly instead of silently producing a misleading report; note the deliberate `|| true` after commands whose "nothing found" result is expected and should not trigger `set -e` (this is Module 14's conditional-gotcha pattern, applied on purpose). The `report()` function centralizes both the printed format and the counters, so every check "speaks the same language" regardless of which specific test it ran — exactly the same motivation as Module 14's `die()` function, just adapted to report three outcomes instead of failing immediately on the first one. Each `check_*` function is self-contained and independently testable — you could run `check_uid_zero` alone while developing it, without needing the whole script to work first. The script's own exit code (2 for any FAIL, 1 for WARN-only, 0 for a fully clean report) means it can be wired into a monitoring system or CI pipeline exactly like any other pass/fail check from Module 14, not just read by a human.

✅ Exercise complete — you've built a real, extensible security-audit script using functions, a consistent PASS/WARN/FAIL reporting pattern, and the hardened-script conventions from Module 14, covering four of this module's core checks end to end.

---

## ✅ Module Completion Checklist

- [ ] I can explain what a security audit script is, why it exists, and why it is a triage tool rather than a replacement for a real vulnerability scanner
- [ ] I can find world-writable files and directories system-wide with `find`, explain exactly why a world-writable file is a security risk, and use `-xdev` to keep the search on one filesystem
- [ ] I can find unexpected UID-0 (root-equivalent) accounts in `/etc/passwd` with `awk`, and explain why any line other than `root` is a critical finding
- [ ] I can find SUID and SGID binaries system-wide with `find -perm`, and explain why an unrecognized SUID binary is one of the most serious findings an audit can turn up
- [ ] I can audit `/etc/ssh/sshd_config` for risky settings — `PermitRootLogin`, `PasswordAuthentication`, and the historical `Protocol` directive — and know what a hardened value looks like for each
- [ ] I can scan `/etc/cron.d/`, `/var/spool/cron/crontabs/`, and every user's crontab for unexpected scheduled jobs
- [ ] I can inspect listening ports and services with `ss -tulpn` and reason about which ones should — and should not — be reachable
- [ ] I can check for outdated/vulnerable packages with `apt list --upgradable`, and explain what `debsecan` and `unattended-upgrades` each add on top of a manual check
- [ ] I can scan `/var/log/auth.log` (or `journalctl`) for repeated failed SSH logins, count them by source IP with `awk`, and explain where `fail2ban` picks up where a read-only audit script stops
- [ ] I can assemble multiple individual checks into a single script that prints a structured PASS/WARN/FAIL report, using functions and the hardened-script conventions from Module 14

## Next Step

Continue to [Module 19: Bash + Docker](../module19-bash-docker/)
