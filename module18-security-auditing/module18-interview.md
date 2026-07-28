# 🎤 Module 18 Interview Prep — Security Auditing Scripts

## Conceptual Questions

### 🟢 Beginner

**Q1: Why is a world-writable file a security risk?**

> "A world-writable file means the 'others' category — anyone on the system who isn't the owner and isn't in the owning group — can modify or completely overwrite its contents. That breaks the assumption that a file only ever contains what its legitimate owner put there. It becomes genuinely dangerous when the file is something that runs automatically with elevated privilege — a cron job, a startup script — because now any unprivileged user, or an attacker who's only gotten a low-privilege foothold, can edit that file and have their own code execute the next time it runs, potentially with root privileges they never had directly. That's a straightforward privilege-escalation path."

**Q2: Why does an audit check specifically for SUID binaries?**

> "SUID — setuid — makes a program run with the permissions of the file's owner, usually root, rather than whoever launched it. That's legitimate and necessary for a small set of well-known system programs, like `passwd`. But it also means every SUID-root binary is a potential path straight to root for anyone who can execute it. An audit isn't assuming every SUID binary found is malicious — most are completely normal — but it's surfacing the complete list so a human can compare it against what's expected. A brand-new one that wasn't there before, especially somewhere unusual like `/tmp` or a home directory, is one of the strongest signals of a compromised system, because copying a shell somewhere and making it setuid-root is a classic, simple way an attacker plants an instant backdoor."

**Q3: What should `PermitRootLogin` and `PasswordAuthentication` be set to on a hardened server, and why?**

> "`PermitRootLogin` should be `no`, or `prohibit-password` if there's a specific need for root to log in with a key only. Root is the single most powerful account, so removing it as a direct SSH target — forcing administrators to log in as themselves and use `sudo` instead — removes the biggest possible prize from anyone trying to brute-force or guess their way in. `PasswordAuthentication` should be `no` once key-based login is confirmed working for everyone who needs access, because passwords can be guessed or brute-forced over the network at scale, while a private key can't be practically guessed the same way. Together they close two different halves of the same door — one narrows *who* can be targeted, the other narrows *how* anyone can get in at all."

**Q4: What is a security audit script, in plain terms, and what does it not do?**

> "It's a script that walks through a checklist of well-known trouble spots on a system — file permissions, unexpected privileged accounts, risky SSH settings, scheduled jobs, open ports, outdated packages, suspicious login attempts — and reports what it finds as pass, warn, or fail. It doesn't fix anything itself; it observes and reports, the same way a smoke detector doesn't put out a fire, it tells you there's one so a human can decide what to do."

**Q5: Why do we care if a UID-0 account other than `root` exists?**

> "UID 0 is what actually grants root-equivalent privilege on Linux — the username `root` is just a convention, not what the kernel checks. Nothing stops a second `/etc/passwd` entry from also having UID 0 under a completely different, innocuous-looking name. Any account like that has full root power, indistinguishable from the real root account, and it's a classic way to hide a backdoor in plain sight — a quick glance at `who` or `ps` won't obviously flag a username that just looks like a normal service account."

### 🟡 Intermediate

**Q6: Why is `-xdev` important when running a system-wide `find` for world-writable files or SUID binaries?**

> "Without `-xdev`, `find /` will happily descend into every mounted filesystem it encounters — `/proc` and `/sys`, which are virtual filesystems with potentially millions of synthetic entries meaningless to a permissions audit, plus any mounted network shares or removable media. That's both slower — noticeably, on a real server — and it can report on filesystems outside your actual scope of responsibility, like an NFS share owned by a different team. `-xdev` keeps `find` on the same filesystem the starting point is on, so the audit stays scoped to the machine you're actually responsible for."

**Q7: Why does the module recommend checking for failed SSH logins with an `awk` pattern that searches for the literal word 'from', rather than a fixed field position?**

> "Because the log line's format shifts depending on whether the attempted username was valid or not. `Failed password for root from 1.2.3.4 port 22 ssh2` and `Failed password for invalid user admin from 1.2.3.4 port 22 ssh2` put the IP address at different fixed column positions — there's an extra 'invalid user X' clause in one but not the other. Looping over every field and printing whatever comes right after the literal token 'from' works correctly regardless of which format the line is in, instead of hardcoding a field number that only works for one of the two cases."

**Q8: What's the difference between what `apt list --upgradable` tells you and what `debsecan` tells you?**

> "`apt list --upgradable` just tells you a newer version of a package exists in your configured repositories — useful, but it doesn't tell you *why* the update matters, or whether the update is even security-related versus just a feature bump. `debsecan` cross-references your actually-installed package versions against a database of known, named CVEs and tells you specifically which installed packages have a documented, public security vulnerability right now. It turns 'there's a newer version available' into 'this specific version has this specific known weakness' — much more actionable for prioritizing what to patch first."

**Q9: Why would you still run a manual failed-login check if `fail2ban` already exists?**

> "`fail2ban` is the production answer for *ongoing, automatic* protection — it watches logs continuously and actively blocks offending IPs in real time, which a one-off audit script can never do. But when you're doing a point-in-time audit of a server — especially one you've just inherited or aren't sure has ever been looked at — you want to know what's *already* happened and whether `fail2ban` (or an equivalent) is even installed and running in the first place. The manual check is a diagnostic snapshot; `fail2ban` is the ongoing defense. A good audit checks both: are there signs of an attack, and is there a mechanism in place to actually stop one automatically?"

**Q10: Why does the module check both `/etc/cron.d/` and every individual user's crontab, instead of just one or the other?**

> "They're separate places a scheduled job can live, and each has a different threat model. `/etc/cron.d/` holds system-wide jobs, usually installed by packages or an administrator directly — if something unexpected shows up there, it likely required root or sudo access to place. A user's personal crontab, stored under `/var/spool/cron/crontabs/`, could be modified by that user alone — including a low-privileged account that's been compromised, with nobody else routinely checking it. An audit that only checks one location has a real blind spot; persistence mechanisms specifically exploit whichever spot nobody's looking at."

### 🔴 Advanced

**Q11: You find an SGID binary owned by a service account, in a directory outside the usual system paths. Walk through how you'd decide whether it's a real problem.**

> "First, I wouldn't assume it's malicious just because it's unusual — plenty of legitimate software installs its own setgid helper outside the standard system directories, especially third-party applications. I'd start by identifying what installed it: check the package manager (`dpkg -S /path/to/binary`) to see if it belongs to a known, installed package, and if so, whether that's expected on this particular server. If it's not owned by any package, I'd look at its modification time relative to other files nearby, and relative to any known incident window, and I'd inspect what group it's setgid to — does that group grant access to something sensitive, like `shadow` or a database's data directory? I'd also check whether the binary itself looks legitimate: is it a real, recognizable program, or does it look like a renamed shell or a stripped-down binary with no obvious purpose? Only after ruling out a legitimate explanation would I treat it as a likely compromise indicator and escalate."

**Q12: A colleague argues that since `set -euo pipefail` (from Module 14) is good practice, an audit script should also `set -e` immediately if any single check fails. Why is that the wrong design for this specific kind of script?**

> "The whole point of an audit script is to run every check and report a complete picture, even when several checks fail — a FAIL on one check shouldn't prevent the script from also telling you about the other seven. If the script used `set -e` and let an individual check's failing condition propagate as a script-ending error, you'd only ever see the first problem found, and you'd have to fix it and re-run the whole thing just to discover the second one. That's the opposite of what an audit is for. `set -euo pipefail` is still valuable for *unexpected* failures — a required command that isn't installed, a typo — but each individual security check's 'bad' result needs to be captured and reported as data (a FAIL line), not treated as a script-ending error condition. That's why each check function tests its own condition and calls `report()` rather than letting a failing test exit the whole script."

**Q13: Why is `PasswordAuthentication no` alone not sufficient if `PermitRootLogin` is still `yes` — or vice versa? What does having only one of the two actually leave open?**

> "They close two different gaps, and leaving either one open still leaves a real attack path. If `PasswordAuthentication` is `no` but `PermitRootLogin` is still `yes`, an attacker who somehow gets hold of a valid root private key — through a leaked backup, a poorly-secured workstation, or a supply-chain issue — can still log straight in as root; disabling passwords doesn't touch key-based root access at all. If `PermitRootLogin` is `no` but `PasswordAuthentication` is still `yes`, root itself can't be targeted directly, but every *other* account on the box is still brute-forceable over the network with nothing but guessed or leaked passwords — and depending on `sudo` group membership, compromising even one of those accounts might still grant effective root access anyway. A genuinely hardened configuration needs both settings locked down together; each one alone only closes half the actual risk."

---

## Practical/Coding Questions

**Q1: Write a command that finds every SUID binary on the system, restricted to the local filesystem, and prints full permission details for each.**

Solution:
```bash
find / -xdev -type f -perm -4000 -exec ls -la {} \; 2>/dev/null
```
Explanation: `-xdev` keeps the search on one filesystem (avoiding `/proc`, `/sys`, and any mounted shares); `-perm -4000` matches files with the setuid bit set; `-exec ls -la {} \;` runs `ls -la` on every match so the output shows the full permission string and owner alongside the path, instead of just a bare filename list. `2>/dev/null` discards `Permission denied` noise from directories the current user can't read — run with `sudo` for a complete result.

**Q2: Write an `awk` one-liner against `/etc/passwd` that lists every account with UID 0 other than `root` itself.**

Solution:
```bash
awk -F: '$3 == 0 && $1 != "root" {print $1}' /etc/passwd
```
Explanation: `-F:` splits each line on the colon delimiter used by `/etc/passwd`; `$3 == 0` filters to lines where the UID field equals zero; `$1 != "root"` additionally excludes the legitimate `root` entry itself, so this prints **only** the unexpected extras — an empty result here is the healthy outcome.

**Q3: Write a check that reads `/etc/ssh/sshd_config`, and prints `FAIL` if `PasswordAuthentication` is `yes`, `PASS` if it's `no`, and `WARN` if the setting isn't present in the file at all.**

Solution:
```bash
value=$(grep -E "^\s*PasswordAuthentication" /etc/ssh/sshd_config | awk '{print $2}' || true)

case "$value" in
    no)  echo "PASS: PasswordAuthentication is 'no'" ;;
    yes) echo "FAIL: PasswordAuthentication is 'yes'" ;;
    *)   echo "WARN: PasswordAuthentication not explicitly set — check the compiled-in default" ;;
esac
```
Explanation: the `grep` pulls the matching line (if any); `awk '{print $2}'` extracts just the value token after the directive name. The `case` statement handles three outcomes explicitly: an exact `no` passes, an exact `yes` fails, and anything else (including an empty string, meaning the directive wasn't found at all) falls to the wildcard `*` branch as a WARN rather than being silently treated as safe — the `|| true` keeps `set -e`-style strict mode (Module 14) from treating "grep found nothing" as a script-ending error.

**Q4: Write the `awk` pipeline that counts failed SSH login attempts by source IP from `/var/log/auth.log`, sorted with the worst offender first, and explain why it doesn't rely on a fixed column number.**

Solution:
```bash
grep "Failed password" /var/log/auth.log \
  | awk '{for (i=1;i<=NF;i++) if ($i=="from") print $(i+1)}' \
  | sort | uniq -c | sort -rn
```
Explanation: rather than assuming the IP address always sits at, say, field 11, the loop scans every field on the line and prints whatever token immediately follows the literal word `from` — this works correctly whether the log line is a normal failed-password attempt or one that includes the extra `invalid user X` clause, since the column position of the IP shifts between those two cases but its position relative to the word `from` never does. `sort | uniq -c` tallies occurrences per unique IP, and `sort -rn` orders that tally numerically, largest count first.

---

## Gotcha Questions

**Q1: "I ran `find / -perm -0002` as my regular (non-root) user and it came back completely clean — no world-writable files at all. Great, the system's secure, right?"**

> Trap: a clean result from a regular, unprivileged user is not trustworthy — `find` silently skips any directory it doesn't have read/execute permission to enter, and simply produces *fewer* results, which looks identical to "nothing was found." Without `sudo`, you have no way to distinguish "genuinely nothing there" from "couldn't see large parts of the filesystem." Always run a permissions/SUID audit with `sudo` (or as root) for a result you can actually trust.

**Q2: "This server has a `find / -perm -4000` audit step that runs every five minutes as a monitoring check, and it's noticeably slowing the whole box down. What's the likely cause, and what's the fix?"**

> Trap: the likely cause isn't that `find` is inherently slow — it's that the check is walking the *entire* filesystem, every five minutes, probably without `-xdev`, potentially crossing into `/proc`, `/sys`, or mounted network shares each time, none of which need to be re-scanned that frequently or at all. The fix is twofold: add `-xdev` so it stays on the local root filesystem, and reconsider the interval — a system-wide SUID/permissions sweep is reasonable to run hourly or daily as a background job, not every five minutes; if faster detection is genuinely needed, narrow the scope to the specific high-value directories that actually change (e.g. `/opt`, `/var/www`, `/home`) rather than sweeping the whole tree that often.

**Q3: "I found a world-writable file during an audit and immediately flagged it as a critical incident. My lead pushed back and said not so fast — why might that pushback be reasonable?"**

> Trap: not every world-writable file is a security incident by itself — some legitimate applications intentionally use a world-writable file for interprocess communication, logging, or a shared cache, and a data file with no execution path attached to it (nothing runs it as a privileged user) is a much lower-severity finding than a world-writable *script* that a cron job runs as root. The audit's job is to *surface* the finding for a human to evaluate in context — what the file actually is, whether anything privileged executes it, whether it's expected for the software installed — not to declare every hit an automatic critical incident. Escalating without that context erodes trust in the audit process the next time a real finding shows up.

---

## Quick-Fire Rapid Review

- **Q: What `find` flag keeps a search on one filesystem?** A: `-xdev`.
- **Q: What octal permission value indicates a world-writable file/dir via `find -perm`?** A: `-0002`.
- **Q: What UID is reserved for root?** A: `0`.
- **Q: What command lists every UID-0 account in one line?** A: `awk -F: '$3 == 0 {print $1}' /etc/passwd`.
- **Q: What `find -perm` value matches setuid binaries?** A: `-4000`.
- **Q: What `find -perm` value matches setgid binaries?** A: `-2000`.
- **Q: What sshd_config setting controls whether root can SSH in?** A: `PermitRootLogin`.
- **Q: What sshd_config setting controls whether passwords (not just keys) are accepted?** A: `PasswordAuthentication`.
- **Q: What historical sshd_config directive is now a no-op on modern OpenSSH?** A: `Protocol` (SSHv1 support has been removed entirely).
- **Q: Where are system-wide cron job files stored?** A: `/etc/cron.d/`.
- **Q: Where are individual users' personal crontabs stored on disk?** A: `/var/spool/cron/crontabs/`.
- **Q: What command inspects listening TCP/UDP ports with owning process info?** A: `ss -tulpn`.
- **Q: What command lists packages with an available update?** A: `apt list --upgradable`.
- **Q: What tool cross-references installed packages against known CVEs?** A: `debsecan`.
- **Q: What mechanism automatically installs security updates on a schedule?** A: `unattended-upgrades`.
- **Q: What log file records SSH authentication attempts on a traditional (non-journald-only) Ubuntu system?** A: `/var/log/auth.log`.
- **Q: What tool actively blocks brute-forcing IPs in real time, rather than just reporting on them?** A: `fail2ban`.
- **Q: What does a security audit script do when it finds a problem — fix it, or report it?** A: Report it only; remediation is a separate, deliberate human decision.
- **Q: Why should you run these audit checks with `sudo`?** A: Without it, permission-restricted directories/files are silently skipped, producing an incomplete, falsely-clean result.
- **Q: What's the standard three-tier severity convention used to summarize each check's result?** A: PASS / WARN / FAIL.
