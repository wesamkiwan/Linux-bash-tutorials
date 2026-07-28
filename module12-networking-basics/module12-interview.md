# 🎤 Module 12 Interview Prep — Networking Basics

## Conceptual Questions

### 🟢 Beginner

**Q1: What is a port, and why do we need port numbers at all?**

> "A port is a numbered endpoint on a machine — think 0 to 65535 — that identifies which specific service on that machine a piece of network traffic is meant for. An IP address gets your traffic to the right machine; the port number gets it to the right service running on that machine, since a single server commonly runs many services at once — a web server, a database, SSH — all reachable at the same IP. Certain port numbers are so consistently used for the same purpose across the industry that they're treated as well-known standards — port 22 for SSH, 80 for HTTP, 443 for HTTPS."

**Q2: What's the difference between `curl` and `wget`, at a basic level?**

> "Both fetch content over HTTP, but they're built for different jobs. `curl` is designed for talking to APIs — it gives you fine control over the HTTP method, custom headers, and request bodies, and by default it prints the response straight to your terminal so you can inspect or pipe it. `wget` is built specifically for reliably downloading files to disk — it saves to a file by default, and it has strong built-in support for resuming an interrupted download, which `curl` doesn't do as naturally."

**Q3: What does `curl -I` do, and when would you use it?**

> "It sends a HEAD request instead of a GET — the server responds with just the headers and status code, no body at all. I'd use it when I only need to check whether a URL is up and what status code or headers it returns, without downloading potentially megabytes of content I don't actually need — a fast health check."

**Q4: What is SSH, in plain terms?**

> "SSH, Secure Shell, is a protocol and tool for logging into another machine's command line over a network, with the entire session encrypted end-to-end. It replaced older tools like `telnet` that sent everything, including your password, in plain text that anyone watching the network traffic could read."

**Q5: What is `ping` actually testing, and what protocol does it use?**

> "`ping` tests the most basic possible question — is this host reachable on the network at all? It works using ICMP, a lightweight protocol built specifically for this kind of diagnostic 'are you there' traffic, separate from the protocols like HTTP that carry real application data."

### 🟡 Intermediate

**Q6: How does SSH key-based authentication actually work?**

> "You generate a key pair — a private key that never leaves your machine, and a mathematically linked public key that you can safely hand out. You install the public key on any server you want access to, typically in that account's `~/.ssh/authorized_keys` file, often using `ssh-copy-id`. When you connect, the server sends a cryptographic challenge; your client uses the private key to sign it, proving you hold the matching private key, without the private key itself ever being transmitted anywhere. The server checks that signature against the public key it has on file, and if it matches, you're authenticated — no password ever exchanged. That's fundamentally more secure than a password, because there's no shared secret being transmitted or typed at all — only proof of possession, and a stolen public key is useless to an attacker by design."

**Q7: `curl` vs `wget` — walk me through exactly when you'd pick one over the other on the job.**

> "If I'm testing or scripting against an API — I need to set a specific HTTP method, attach a JSON body, set custom headers, or pipe the JSON response into something like `jq` — that's `curl`, every time; `wget` isn't really designed for any of that. If my actual goal is 'get this file onto disk reliably' — a large ISO, an archive, something that might get interrupted on a flaky connection — I reach for `wget`, specifically because of `-c` to resume a partial download, which is a much more natural fit than trying to force `curl` to do the same job."

**Q8: What does a 'host key verification failed' warning actually mean?**

> "SSH stores the host key fingerprint of every server you've successfully connected to and trusted, in `~/.ssh/known_hosts`. Every time you reconnect, it silently checks the server's current host key against what's stored. A 'host key verification failed' warning means those two don't match anymore — the server presenting itself now has a different key than the one you trusted before. That can mean the server was legitimately rebuilt or reimaged, generating a fresh host key — or it can mean you're being routed to a completely different, potentially malicious machine impersonating the real server, positioned to intercept your credentials and traffic. SSH has no way to distinguish those two cases on its own, which is exactly why it stops and forces a human decision instead of silently proceeding."

**Q9: Why is `rsync -avz` considered more robust than `scp` for deployments?**

> "`scp` does a straightforward, one-shot copy — if it's interrupted partway, you start over from scratch, and copying a directory means resending every file regardless of whether it changed. `rsync` supports resuming an interrupted transfer, and more importantly it does delta transfer — when the destination already has some version of the files, it compares them and only actually sends the parts that changed. For a one-off copy either works fine, but for repeatedly deploying an updated build directory to the same server, `rsync -avz` can be dramatically faster and safer against network interruptions."

**Q10: What's the practical difference between `ss` and `netstat`?**

> "They answer the same question — what's listening on which ports, and what's connected to what — but `ss` is the modern tool, reading kernel socket information more directly and performing noticeably faster on busy systems with many connections. `netstat` is the legacy predecessor; it's not installed by default on a lot of minimal Ubuntu/Debian systems anymore — you'd need to install `net-tools` to get it. I'd default to `ss -tulpn` day to day, but I'd still recognize `netstat` commands in older scripts or documentation without needing to relearn anything."

### 🔴 Advanced

**Q11: You SSH into a server you've connected to dozens of times before, and today you get a host key verification failure. Walk me through exactly what you'd do.**

> "First, I would not bypass it or delete `known_hosts` wholesale just to get past the warning — that's exactly the scenario this mechanism exists to catch. I'd pause and investigate: check with whoever administers that server whether it was intentionally rebuilt, reimaged, or had its SSH host keys regenerated recently — a legitimate infra change would explain it immediately. If I can get the new fingerprint confirmed through a trusted channel — a message from the team that did the rebuild, quoting the exact new fingerprint — I'd compare it against what SSH is now presenting. Only once I've confirmed it's legitimate would I clear just that one specific entry with `ssh-keygen -R hostname` and reconnect. If I can't get independent confirmation, I would not connect at all, since proceeding blind defeats the entire point of host verification and could mean handing credentials or data to an attacker-controlled machine."

**Q12: A teammate can't reach an internal API and asks you to help debug it. What's your actual command-by-command approach?**

> "I'd work outward-in, isolating one layer at a time rather than guessing. First `ping -c 4 <host>` to confirm basic network reachability — if that fails entirely, I know it's a network-level problem before even looking at the application. Next, if the target is addressed by hostname, `dig <hostname>` to confirm DNS actually resolves it to the IP I expect — a surprising number of 'can't connect' issues are actually DNS misconfigurations. Then, if I have access to the target machine, `ss -tulpn` to confirm the API's process is actually `LISTEN`ing on the expected port, and specifically whether it's bound to `0.0.0.0` (reachable externally) or only `127.0.0.1` (localhost only) — binding to localhost when external access is needed is an extremely common root cause. Finally, `curl -v` against the actual API endpoint to see the full request/response conversation — this surfaces things like TLS certificate problems, unexpected redirects, or authentication failures that ping and ss can't tell you anything about. Each tool rules out exactly one layer, so I never waste time debugging the wrong one."

**Q13: Why would you disable password authentication on a server even after key-based auth is already working?**

> "If a server accepts both passwords and keys, its real security is only as strong as the weaker of the two options an attacker can try — and passwords are brute-forceable at scale in a way a private key practically isn't, especially over the open internet where automated bots constantly probe port 22 with credential-stuffing attempts. Once I've confirmed key-based login genuinely works for every account that needs access, disabling `PasswordAuthentication` in `/etc/ssh/sshd_config` removes that weaker option entirely, so a stolen or guessed password simply can't be used to log in at all — key possession becomes the only way in."

---

## Practical/Coding Questions

**Q1: Write a `curl` command that sends a POST request with a JSON body to `https://api.example.com/v1/orders`, setting the correct content-type header, and saves the response to a file called `response.json`.**

Solution:
```bash
curl -X POST https://api.example.com/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"item":"widget","qty":3}' \
  -o response.json
```
Explanation: `-X POST` sets the method explicitly, `-H` sets the content-type so the server parses the body as JSON, `-d` supplies the JSON body itself (single-quoted so the shell doesn't try to interpret the internal double quotes), and `-o response.json` writes the server's response to a file instead of printing it to the terminal.

**Q2: Write the commands to generate a new SSH key pair named specifically for a "staging" server, and install it there.**

Solution:
```bash
ssh-keygen -t ed25519 -C "staging-access" -f ~/.ssh/id_ed25519_staging
ssh-copy-id -i ~/.ssh/id_ed25519_staging.pub deploy@staging.example.com
```
Explanation: `-f` gives the key pair a purpose-specific filename so it doesn't collide with or get confused with any other keys already in `~/.ssh/`. `ssh-copy-id -i` then installs specifically that public key (not whatever the default key happens to be) onto the staging server's `authorized_keys`, authenticating with a password one final time to do so.

**Q3: Write a `~/.ssh/config` entry for a host at `10.0.4.15`, username `ubuntu`, using a custom port `2222` and a specific identity file, aliased as `bastion`. Then show the `rsync` command that would sync a local `dist/` folder to `/srv/app/` on that host using the alias.**

Solution:
```
Host bastion
    HostName 10.0.4.15
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_bastion
```
```bash
rsync -avz ./dist/ bastion:/srv/app/
```
Explanation: The `Host` block bundles the IP, user, non-default port, and specific key under one memorable alias. Because `rsync` (like `scp`) reuses the system's SSH configuration, `rsync -avz ./dist/ bastion:/srv/app/` automatically picks up the port and identity file from the config — nothing needs to be repeated on the command line.

**Q4: You need to find out which process is listening on port 5432 on the current machine. Write the command and explain how you'd read the result.**

Solution:
```bash
sudo ss -tulpn | grep ':5432'
```
Realistic output:
```
tcp   LISTEN  0  511  127.0.0.1:5432   0.0.0.0:*   users:(("postgres",pid=1140,fd=5))
```
Explanation: `ss -tulpn` lists all listening TCP/UDP sockets with owning process info; piping through `grep ':5432'` narrows it to just that port. The result shows PostgreSQL (`postgres`, PID 1140) is listening, bound only to `127.0.0.1` — meaning it's reachable from this machine only, not from the network — which is exactly the kind of detail that would explain "the app on another server can't reach the database" even though the database process itself is running fine.

---

## Gotcha Questions

**Q1: "I ran `curl "https://api.example.com/search?q=cats&limit=10"` without quotes and it seemed to hang / only searched for 'cats' with no limit. What happened?"**

> Trap: without quotes, the shell interprets `&` as "run the preceding command in the background," splitting the URL at the `&` — so `curl` only ever receives `https://api.example.com/search?q=cats`, and the shell tries to run `limit=10` as a separate (nonsensical) background command. This is why URLs with query strings containing `&`, `?`, spaces, or other shell-special characters must always be quoted.

**Q2: "SSH gave me a host key verification failure, so I just deleted my entire `~/.ssh/known_hosts` file to make it go away and reconnected — problem solved, right?"**

> Trap: deleting the whole file doesn't "solve" anything — it silences the warning by throwing away *every* trust record you had, for *every* server you've ever verified, not just the one that changed. The next connection to any of those hosts will silently re-accept whatever key is presented with no comparison at all, since there's nothing left to compare against. If the specific host that triggered the warning really was compromised or impersonated, this makes it worse, not better — you've now trained yourself (and possibly your tooling) to blindly accept whatever's presented. The correct move is investigating that one host and removing only its specific stale entry with `ssh-keygen -R hostname`.

**Q3: "`ping google.com` works fine, but my app still can't reach the API on that same server. Doesn't `ping` prove the network connection is fine?"**

> Trap: `ping` only proves basic ICMP reachability — it says nothing about whether any particular application port is open, listening, or working correctly. Plenty of servers block ICMP outright for security reasons while their real services work fine (the opposite of this scenario), and just as easily a server can respond to ping while the specific application port is down, firewalled off, or the app itself has crashed. `ping` succeeding rules out "the network path to this host is completely broken" — nothing more. You still need `curl`/`ss` to say anything about the actual application.

---

## Quick-Fire Rapid Review

- **Q: What HTTP method does plain `curl <url>` send?** A: GET.
- **Q: Which `curl` flag sets a custom HTTP method?** A: `-X` (e.g. `-X POST`).
- **Q: Which `curl` flag attaches a request body?** A: `-d` / `--data`.
- **Q: Which `curl` flag makes it follow redirects?** A: `-L`.
- **Q: Which `curl` flag sends a headers-only HEAD request?** A: `-I`.
- **Q: Which `wget` flag resumes an interrupted download?** A: `-c`.
- **Q: What never leaves your machine in SSH key-based auth?** A: The private key.
- **Q: What file stores trusted host key fingerprints?** A: `~/.ssh/known_hosts`.
- **Q: What command installs your public key on a remote server?** A: `ssh-copy-id`.
- **Q: What file lets you define SSH host aliases?** A: `~/.ssh/config`.
- **Q: What's the modern replacement for `netstat`?** A: `ss`.
- **Q: What flags does `ss -tulpn` combine, and roughly what does each mean?** A: TCP, UDP, listening-only, show process, numeric.
- **Q: What protocol does `ping` use?** A: ICMP.
- **Q: What file maps local hostnames to IPs before DNS is even consulted?** A: `/etc/hosts`.
- **Q: What are three tools for doing a manual DNS lookup?** A: `dig`, `nslookup`, `host`.
- **Q: What flag combination does `rsync` commonly use, and what does each letter mean?** A: `-avz` — archive, verbose, compress.
- **Q: Why is `rsync` often preferred over `scp` for repeated deployments?** A: It resumes interrupted transfers and only sends changed data (delta transfer).
- **Q: What port does SSH use by default?** A: 22.
