# 📚 Module 16 References — Production Scripting & Security Hardening

Curated resources for this module's scope: the production script template, input validation, command injection and `eval`, secrets handling and the `ps aux` problem, least privilege and permissions, logging with `logger`/`logrotate`, and idempotency. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢🟡 — Practical, energetic videos on securing Linux systems and scripts, framed around real sysadmin scenarios.
- **[DistroTube](https://www.youtube.com/@DistroTube)** 🟡🔴 — Dedicated Bash scripting and Linux hardening content aimed at viewers past the fundamentals.
- **[The Cyber Mentor](https://www.youtube.com/@TCMSecurityAcademy)** 🟡🔴 — Security-focused channel with content covering command injection and related web/shell vulnerability classes from an attacker's-eye view, useful for understanding what a real exploit attempt looks like.

## 📖 Official Documentation

- **[OWASP — OS Command Injection](https://owasp.org/www-community/attacks/Command_Injection)** 🟡🔴 — The authoritative, vendor-neutral reference on what command injection is, how it's exploited, and standard mitigations; essential reading for this module's core topic.
- **[Bash Reference Manual — "Shell Builtin Commands" (`eval`)](https://www.gnu.org/software/bash/manual/bash.html#Bourne-Shell-Builtins)** 🟡 — The authoritative, exact specification of what `eval` does, straight from GNU.
- **[Bash Hackers Wiki — Security](https://web.archive.org/web/2023*/https://wiki.bash-hackers.org/scripting/security)** 🟡🔴 — A focused reference on Bash-specific security pitfalls: quoting, injection vectors, and `eval` dangers (archived link — the original wiki has gone offline at times, so use the Wayback Machine if the live site is unreachable).
- **[`man logger`](https://man7.org/linux/man-pages/man1/logger.1.html)** 🟢🟡 — The official manual page for sending messages to syslog from a script.
- **[`man logrotate`](https://man7.org/linux/man-pages/man8/logrotate.8.html)** 🟡 — The official manual page and configuration reference for rotating log files.
- **[`man ps`](https://man7.org/linux/man-pages/man1/ps.1.html)** 🟢 — Confirms exactly what `ps aux` displays, including full command lines — the basis of the CLI-argument secrets danger covered in this module.

## 📝 Tutorials & Articles

- **[Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)** 🟡🔴 — A widely-referenced professional style guide covering quoting discipline, avoiding `eval`, and defensive-scripting conventions used at scale; directly relevant to this module's injection-avoidance guidance.
- **[BashPitfalls — Greg's Wiki](https://mywiki.wooledge.org/BashPitfalls)** 🟡🔴 — An enormous, battle-tested catalog of common Bash mistakes, many of them exactly the unquoted-variable and word-splitting issues behind command injection.
- **[OWASP Command Injection — Testing Guide](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection)** 🟡🔴 — A deeper, testing-oriented companion to the main OWASP page, useful for understanding how injection is actually probed for in the wild.
- **["Bash Arrays" — Greg's Wiki](https://mywiki.wooledge.org/BashGuide/Arrays)** 🟡 — Detailed, correct treatment of Bash array syntax, the safe alternative to string-concatenated commands emphasized throughout this module.
- **[DigitalOcean — "How To Use Journalctl to View and Manipulate Systemd Logs"](https://www.digitalocean.com/community/tutorials/how-to-use-journalctl-to-view-and-manipulate-systemd-logs)** 🟢🟡 — Practical guide to reading back what `logger` sends to syslog on a modern systemd-based Ubuntu system.
- **[DigitalOcean — "How To Manage Logfiles with Logrotate on Ubuntu"](https://www.digitalocean.com/community/tutorials/how-to-manage-log-files-with-logrotate-on-ubuntu-20-04)** 🟢🟡 — A focused, example-driven walkthrough of configuring `logrotate` for a custom application logfile.

## 🎓 Courses & Learning Portals

- **[freeCodeCamp — Bash scripting content](https://www.freecodecamp.org/news/tag/bash/)** 🟢🟡 — Free, text-based articles that build toward defensive, production-grade scripting habits, including secrets and permissions hygiene.
- **[TryHackMe — "OWASP Top 10" room](https://tryhackme.com/)** 🟡🔴 — Free-tier, hands-on labs including practical, safe-to-practice command injection scenarios that reinforce this module's Detailed Explanations section.
- **[Udemy — "Bash Scripting and Shell Programming"](https://www.udemy.com/)** 🟡🔴 — A popular, affordable paid course with sections on production scripting practices and security; search the current catalog since specific titles/instructors rotate.

## 🌐 Websites & Interactive Platforms

- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any command — including an `eval`-based or array-based one — and see every part broken down individually; useful for confirming exactly how quoting changes a command's behavior.
- **[ShellCheck.net](https://www.shellcheck.net/)** 🟢🟡 — Paste a script directly into the browser to catch unquoted-variable and other injection-adjacent warnings without installing anything locally (same tool introduced in Module 14).
- **[Killercoda](https://killercoda.com/)** 🟢🟡 — Free, disposable Linux terminals in the browser — a safe place to actually run the `ps aux` secrets demo and injection examples from this module without risking a real machine.

## 📚 Books

- **["The Linux Command Line" by William Shotts](https://linuxcommand.org/tlcl.php)** 🟢🟡 — Free legal PDF; its scripting and permissions chapters build directly toward this module's least-privilege and file-permission guidance.
- **"Pro Bash Programming" by Chris Johnson & Jayant Varma (Apress)** 🟡🔴 — Covers defensive scripting patterns, including argument handling and quoting discipline, in more depth than an introductory text.
- **"Web Application Hacker's Handbook" by Dafydd Stuttard & Marcus Pinto** 🔴 — Not Bash-specific, but the definitive deep-dive on injection vulnerability classes in general, useful for understanding command injection as one member of a broader family (alongside SQL injection, etc.).

## 👥 Communities

- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** 🟢🟡🔴 — Enormous, searchable Q&A archive; questions like "why is `eval` dangerous" or "how do I pass a secret to a script safely" already have excellent, heavily upvoted answers here.
- **[r/bash](https://www.reddit.com/r/bash/)** 🟡🔴 — Active community for Bash scripting questions, including real production war stories about leaked secrets and injection incidents.
- **[r/netsec](https://www.reddit.com/r/netsec/)** 🟡🔴 — Broader security community where command-injection writeups and real-world incident postmortems are frequently discussed.
- **[OWASP Slack/Community](https://owasp.org/getinvolved/)** 🟡🔴 — The community behind the OWASP references above; a good place to ask deeper questions about injection classes and mitigations.
