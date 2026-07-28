# Module 12: Networking Basics 🟡

**Difficulty:** 🟡 Intermediate
**Estimated Time:** 2 hours
**Prerequisites:** Modules 1-10 (Shell Fundamentals through Process Management)

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- [ ] Make and debug HTTP requests with `curl`, including GET and POST requests, custom headers, saving output to a file, following redirects, inspecting headers-only, and verbose debugging output
- [ ] Download files with `wget`, rename the saved file, resume an interrupted download, and explain when to reach for `wget` instead of `curl` (and vice versa)
- [ ] Explain what SSH is, connect to a remote host with `ssh user@host`, and explain the public/private key-pair authentication model conceptually
- [ ] Generate an SSH key pair with `ssh-keygen`, install a public key on a remote server with `ssh-copy-id`, and set up a `~/.ssh/config` host alias for fast, repeatable connections
- [ ] Explain what `known_hosts` is, and correctly interpret a "host key verification failed" warning — including why you investigate it instead of blindly bypassing it
- [ ] Copy files to and from a remote server with `scp`, and explain why `rsync -avz` is the more robust modern alternative for anything beyond a one-off copy
- [ ] Test host reachability with `ping`, read its output correctly, and explain what ICMP is at a basic level
- [ ] Inspect listening ports and active connections with `ss -tulpn` (and know `netstat` as its legacy predecessor), map local hostnames with `/etc/hosts`, and perform a basic DNS lookup with `dig`/`nslookup`/`host`

---

## Module Goal

By the end of this module, you'll be able to confidently talk to other machines from the command line — test whether they're even reachable, pull data from a web API, securely log into a remote server, move files back and forth, and figure out what's listening on which port when something isn't connecting the way it should.

🎯 **On the job:** Picture two everyday production scenarios. First: a colleague reports that "the app can't reach the payments API." You need to figure out, in order, whether the remote host is reachable at all (`ping`), whether the API itself responds the way you expect (`curl`), and whether the right port is even open and listening on your own box (`ss -tulpn`) — each tool eliminates one link in the chain. Second: you've just been handed a brand-new production server and need to connect to it securely, set up passwordless key-based login so you're not typing a password (or worse, sharing one) every time, and copy a deployment package onto it. This module builds both skill sets — diagnosing connectivity, and securely operating on remote machines — which are baseline, every-single-day skills for anyone who touches servers professionally.

---

## Core Concepts

### 1. What is networking, at the level a shell user needs?

Every time one computer talks to another — your laptop loading a web page, a script calling an API, you logging into a server — two computers are exchanging data over a network connection. As a shell user, you don't need to understand the deep plumbing of how packets physically travel; you need to know how to **initiate**, **test**, and **debug** that conversation from the command line. That's the entire scope of this module: a handful of tools that let you speak to, reach, and securely operate on other machines.

### 2. Ports — numbered doors into a building

A single server can run many different services at once — a web server, a database, an SSH login service — all reachable at the same IP address. So how does incoming traffic know which service it's meant for? Each service listens on a **port**: a numbered "door" into that machine, in the range 0–65535. An IP address gets you to the right *building*; a port number gets you to the right *door* inside it.

💡 **Analogy:** Think of a server as an office building with one street address (its IP address). The building has many doors — the reception desk, the mailroom, the loading dock — each with its own door number (the port). Mail addressed to "Door 22" always goes to the same department (SSH, by convention), no matter which building you're visiting. Some door numbers are so consistently used for the same purpose across virtually every server in the world that they're treated as unofficial standards — these are called **well-known ports** (22 for SSH, 80 for HTTP, 443 for HTTPS, and so on — the full cheat sheet table in this module's cheat sheet lists the common ones).

### 3. `curl` — talking to web servers and APIs from the command line

`curl` ("Client URL") is a command-line tool for making HTTP(S) requests — the same kind of request your browser makes when it loads a web page, except `curl` prints the raw response and gives you precise control over every detail of the request. It's the standard tool for testing and debugging APIs from a terminal or a script.

A bare `curl` against a URL performs a **GET** request (the default HTTP method, meaning "give me this resource") and prints the response body straight to your terminal:

```bash
curl https://api.github.com/users/torvalds
```

### 4. HTTP methods and `curl -X`

HTTP requests carry a **method** — a verb describing what you want to do. `GET` (the default) means "retrieve data." `POST` means "create/submit data." Other common ones are `PUT` (replace a resource) and `DELETE` (remove one). You choose the method with `curl`'s `-X` flag, e.g. `curl -X POST ...`.

### 5. Sending data: `-d` / `--data`

Many API operations — logging in, creating a record, submitting a form — need to send a request **body**, not just a URL. `curl`'s `-d` (or its long form `--data`) flag attaches that body to the request, and using it automatically switches the method to `POST` even if you didn't specify `-X POST` yourself (though writing `-X POST` explicitly alongside it is common practice and makes your intent unambiguous to anyone reading the command later).

### 6. Custom headers: `-H`

An HTTP **header** is a small piece of metadata attached to a request or response — separate from the actual body — that describes something about the request itself: what format the body is in, what credentials are attached, what client is making the request. You add a header with `-H "Header-Name: value"`, and you can repeat `-H` as many times as you need. The most common one you'll type by hand is `-H "Content-Type: application/json"`, telling the server "the body I'm sending you is JSON."

### 7. Saving output: `-o` and `-O`

By default, `curl` prints the response body to your terminal. To save it to a file instead:

- `-o filename` — save the response to a file you name explicitly.
- `-O` (capital) — save the response using the **same filename** the URL itself ends in (e.g. `curl -O https://example.com/report.pdf` saves it as `report.pdf`).

### 8. Following redirects: `-L`

A server can respond with a **redirect** — "the thing you asked for has moved, go here instead" — rather than the actual content. By default, plain `curl` does **not** follow redirects; it just shows you the redirect response itself. Adding `-L` tells `curl` to automatically follow any redirect to its final destination, which is what you want the overwhelming majority of the time when you're actually trying to fetch content rather than inspect the redirect itself.

### 9. Headers only: `-I`

Sometimes you don't need the full response body at all — you just want to know *whether* a server responds, what status code it returns, and what headers it sends back. `curl -I` sends a **HEAD** request (like GET, but the server is asked to return only headers, no body) and prints just that — a fast way to sanity-check a URL without downloading its whole content.

### 10. Verbose/debug mode: `-v`

`curl -v` prints the entire conversation — the exact request `curl` sent (its headers included) and the exact response received — prefixed with `>` for outgoing lines and `<` for incoming ones. This is your primary debugging tool when a request isn't behaving the way you expect: wrong headers being sent, an unexpected redirect, a certificate problem, or an unexpected status code all become visible here.

### 11. `wget` — the download specialist

`wget` is another command-line tool for fetching content over HTTP(S) (and FTP), but its design center is different from `curl`'s: `wget` is built specifically for **downloading files** — including whole directory trees — reliably, with strong built-in support for resuming interrupted transfers. A bare `wget` saves the file to disk using the URL's own filename by default (the opposite of `curl`'s default of printing to your terminal).

```bash
wget https://example.com/ubuntu-22.04-server.iso
```

### 12. `wget -O` and `wget -c`

- `wget -O newname.iso` — save the download under a filename you choose, instead of the one implied by the URL.
- `wget -c` — **continue/resume** a partially-downloaded file rather than starting over from byte zero, provided the server supports resuming (most do). This matters enormously for large files (ISOs, archives, datasets) on a flaky connection.

### 13. `curl` vs. `wget` — when to use which

Both tools overlap heavily, but each has a clear home turf:

- Reach for **`curl`** when you're working with **APIs** — you need control over the HTTP method, custom headers, a request body, and you want to inspect or pipe the response (often JSON) into another program.
- Reach for **`wget`** when your goal is simply **"save this file to disk reliably"** — especially a large file where resuming an interrupted download matters, or recursively grabbing a whole set of files from a directory listing.

💡 **Tip:** If you only ever learn one tool deeply, learn `curl` — it can do almost everything `wget` can for a single file, plus everything API-testing requires that `wget` was never designed for.

### 14. SSH — an encrypted remote shell

**SSH** (Secure Shell) is a protocol and tool for logging into another computer over a network and getting an actual command-line shell on it, with the entire conversation **encrypted** end-to-end. Before SSH became standard, an older tool called `telnet` did the same job but sent everything — including your password — in plain text, readable by anyone who could observe the network traffic. SSH replaced it precisely because encryption prevents that.

```bash
ssh weki@203.0.113.10
```

This connects to the host `203.0.113.10` as user `weki`, and — after authenticating — drops you into a shell on that remote machine, exactly as if you'd sat down at its own keyboard.

### 15. Password auth vs. key-based auth — a shared secret vs. a lock and key

SSH supports two very different ways to prove you're allowed in:

- **Password authentication** — you type a password, the server checks it. Simple, but it's a **shared secret**: the password itself must travel (encrypted, but still transmitted) and must be remembered, typed, and kept safe by a human every single time.
- **Key-based authentication** — you hold a mathematically-linked **pair of keys**: a **private key** (which never leaves your machine, ever) and a **public key** (which you can safely hand out to any server you want to access). The server stores your public key; when you connect, a cryptographic challenge proves you hold the matching private key, without the private key itself ever being transmitted anywhere.

💡 **Analogy — a shared password vs. a lock-and-key pair:** A password is like a shared PIN code that both you and the door need to know — anyone who learns it can walk in, and if it leaks, you have to change it everywhere it's used. A key pair is like a physical lock-and-key: you keep the one and only key (your private key) in your pocket, and the door manufacturer installs a matching lock (your public key) on any door you want to be able to open. The lock can verify "yes, this is the correct key" without ever needing to know what the key looks like in advance from you re-typing it — and losing your key doesn't expose the shape of every other door's lock, unlike a leaked shared password.

### 16. `ssh-keygen` — generating your key pair

`ssh-keygen` creates a new key pair on your own machine:

```bash
ssh-keygen -t ed25519 -C "weki@work-laptop"
```

This produces two files, by default in `~/.ssh/`: a **private key** (e.g. `id_ed25519` — never share this, never copy it to a server, treat it like a physical house key) and a **public key** (e.g. `id_ed25519.pub` — safe to copy anywhere you want access).

### 17. `ssh-copy-id` — installing your public key on a server

Generating a key pair alone doesn't grant you access anywhere — the server needs a copy of your **public** key first. `ssh-copy-id` automates that:

```bash
ssh-copy-id weki@203.0.113.10
```

This logs in (using your password, one last time) and appends your public key to `~/.ssh/authorized_keys` on the remote server. From that point on, `ssh weki@203.0.113.10` can authenticate using your private key instead of a password.

### 18. `~/.ssh/config` — host aliases

Typing out `ssh weki@203.0.113.10 -p 2222 -i ~/.ssh/id_ed25519_prod` every time gets old fast, especially across many servers. `~/.ssh/config` lets you define a shortcut, called a **Host** block, for any connection details you use repeatedly:

```
Host prod-web
    HostName 203.0.113.10
    User weki
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_prod
```

After saving this, `ssh prod-web` alone connects using all of those settings — no more re-typing the full command, and `scp`/`rsync` (Concepts 20–21) can use the same alias too.

### 19. `known_hosts` and host key verification

The very first time you connect to a new host, SSH shows you that server's **host key fingerprint** and asks you to confirm you trust it. Once you accept, that fingerprint is stored locally in `~/.ssh/known_hosts`. On every future connection, SSH silently compares the server's current host key against what's stored — this is how SSH protects you not just from someone stealing your credentials, but from accidentally connecting to an **impostor** server pretending to be the real one (a "man-in-the-middle" attack).

If the server's host key ever changes and doesn't match what's stored, SSH refuses to connect and shows a loud, impossible-to-miss warning: **"WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!"** / "host key verification failed."

⚠️ **This warning is not a routine annoyance to click through — it is SSH doing exactly its job.** A host key can legitimately change for an innocent reason (the server was rebuilt/reimaged, or its OS was reinstalled), but it can *also* change because you're being redirected to a malicious server impersonating the real one, or because someone is intercepting your connection. The correct response is always to **investigate first**: confirm with whoever manages that server whether it was intentionally rebuilt, verify the new fingerprint out-of-band (e.g. a message from your infra team quoting the exact new fingerprint) if possible, and only then remove the old entry (with `ssh-keygen -R hostname`, which cleanly deletes just that one stale entry) before reconnecting. Never reflexively delete `known_hosts` wholesale or blindly type "yes" to bypass it just to make the error go away — that defeats the entire protection this mechanism exists to provide.

### 20. `scp` — copying files over SSH

`scp` ("secure copy") copies files between your machine and a remote one, reusing the exact same authentication (including your SSH keys and `~/.ssh/config` aliases) as `ssh` itself:

```bash
scp report.pdf weki@203.0.113.10:/home/weki/reports/
```

This copies `report.pdf` from your local machine into `/home/weki/reports/` on the remote host. The syntax reads as `source destination`, and either side can be local or remote — `scp weki@203.0.113.10:/var/log/app.log .` copies a remote file down into your current local directory instead.

### 21. `rsync -avz` — the more robust alternative

`scp` is simple and fine for a one-off file copy, but it has real limitations: if the transfer is interrupted partway, it starts over from scratch, and copying a large directory means re-sending every file even if only one changed. **`rsync`** solves both problems — it can **resume** interrupted transfers and, more importantly, it uses **delta transfer**: when copying a directory that partially already exists on the destination, it only sends the parts of files that actually changed, not the whole thing again.

```bash
rsync -avz ./build/ weki@203.0.113.10:/var/www/app/
```

The common flag combination `-avz` breaks down as:
- `-a` ("archive") — preserves permissions, timestamps, and symbolic links, and recurses into subdirectories — the sensible default bundle for "copy this whole thing faithfully."
- `-v` ("verbose") — prints what's being copied as it happens.
- `-z` ("compress") — compresses data in transit, speeding up transfers over slower connections.

✅ **Best Practice:** For anything beyond a single quick file — deploying a whole build directory, syncing a backup, repeatedly pushing updated files to the same server — reach for `rsync -avz` over `scp`. It's strictly more capable for that use case and is the tool actually used in most real-world deployment scripts.

### 22. `ping` — testing basic reachability

`ping` sends a small network message to a host and waits to see if it replies — the simplest possible test of "is this machine even reachable on the network at all?" It's almost always the very first tool you reach for when something can't connect.

```bash
ping -c 4 8.8.8.8
```

`-c 4` limits `ping` to sending exactly 4 requests and then stopping (without `-c`, plain `ping` runs forever until you press Ctrl+C).

### 23. Reading `ping` output, and what ICMP is

`ping` works using a network protocol called **ICMP** (Internet Control Message Protocol) — a lightweight protocol designed specifically for this kind of "are you there?" diagnostic traffic, separate from the protocols (like HTTP) that carry actual application data. Each line of `ping` output reports one reply:

```
64 bytes from 8.8.8.8: icmp_seq=1 ttl=115 time=12.4 ms
```

- **`icmp_seq`** — a sequence number, incrementing per request, letting you spot if any replies came back out of order or went missing entirely.
- **`ttl`** — "time to live," a countdown that decreases by one at every network hop the packet passes through; roughly indicates how many hops away the host is.
- **`time`** — the round-trip time in milliseconds — how long the reply took to come back. Low and consistent is healthy; high or wildly varying values suggest network trouble.

At the end, a summary line reports packet loss — `0% packet loss` is healthy; any nonzero percentage means some requests never got a reply at all, a strong signal of network problems between you and that host.

### 24. Checking listening ports and connections: `ss`

**`ss`** ("socket statistics") is the modern tool for inspecting network connections and listening ports on your own machine — which services are actively listening for incoming connections, and on which ports. The single most useful invocation to memorize:

```bash
ss -tulpn
```

- `-t` — show TCP sockets.
- `-u` — show UDP sockets.
- `-l` — show only **listening** sockets (services waiting for incoming connections), not every open connection.
- `-p` — show the **process** name/PID that owns each socket (may require `sudo` to see process details for services owned by other users).
- `-n` — show numeric addresses and ports instead of resolving them to names (faster, and avoids confusing DNS-lookup delays).

### 25. `netstat` — the legacy predecessor

**`netstat`** did the same job as `ss` for decades and is what you'll still see in a lot of older tutorials, scripts, and habits. It is considered **legacy** — `ss` is faster (it reads kernel data structures more directly) and is what modern distributions ship with by default. On a minimal Ubuntu/Debian installation, `netstat` may not even be installed out of the box; if you need it, it lives in the `net-tools` package:

```bash
sudo apt update
sudo apt install net-tools
```

✅ **Best Practice:** Learn `ss` as your default — it's the modern, actively-maintained tool. Recognize `netstat` commands when you encounter them in older documentation or scripts, but there's no need to reach for it yourself on a system where `ss` is available.

### 26. `/etc/hosts` — local hostname mapping

`/etc/hosts` is a plain text file that maps hostnames directly to IP addresses, checked **before** your system ever asks a DNS server (Concept 27). Each line is simply an IP address followed by one or more hostnames:

```
127.0.0.1   localhost
192.168.1.50   db-server.local
```

This is how you can make `ping db-server.local` or `curl http://db-server.local` work on your own machine without needing any real DNS entry — useful for local development, testing against a staging server by a friendly name, or working around a DNS problem temporarily.

### 27. DNS lookups: `dig`, `nslookup`, `host`

**DNS** (Domain Name System) is the system that translates human-friendly hostnames (like `example.com`) into the IP addresses computers actually use to route traffic — the internet's phone book. Three common command-line tools let you query it directly and see exactly what a name resolves to:

- **`dig example.com`** — the most detailed and commonly used by professionals; shows the full DNS response, including the record type and time-to-live.
- **`nslookup example.com`** — an older, simpler tool, still very common in muscle memory and quick checks.
- **`host example.com`** — the terse, one-line-of-output version, good for quick scripting.

🎯 **On the job:** If `curl` or `ping` fails and you're not sure whether it's a DNS problem or something else entirely, run `dig` (or `host`) against the same hostname — if it doesn't resolve to an IP address at all, you've isolated the problem to DNS before wasting time investigating anything else.

---

## Detailed Explanations

### The full SSH key-based authentication flow, step by step

It's worth walking through exactly what happens the moment you run `ssh prod-web` with keys already set up, since "it just works" hides genuinely useful detail:

1. Your SSH client contacts the server and the server presents its **host key** (Concept 19) — your client checks it against `~/.ssh/known_hosts` and proceeds only if it matches (or if this is truly the first-ever connection and you accept it).
2. The server tells your client which public keys it has on file for the account you're logging in as (its `~/.ssh/authorized_keys`, populated earlier by `ssh-copy-id`).
3. Your client selects a matching **private key** from your machine and uses it to cryptographically sign a challenge the server sends — proving mathematically that you possess the private key, **without ever transmitting the private key itself anywhere**.
4. The server verifies that signature against the public key it has stored. If it matches, you're authenticated — no password ever needed, ever exchanged.

This is why key-based auth is considered fundamentally more secure than passwords: there is no secret being transmitted or typed at all during authentication, only proof of possession. A stolen password can be reused by anyone who has it; a stolen *public* key is useless to an attacker (that's the whole point of it being public) — only the private key, which never leaves your machine and can itself be protected further with its own passphrase, would let someone impersonate you, and even then only if they also stole your actual laptop/key file.

### Why "host key verification failed" demands investigation, not a quick workaround

The entire point of `known_hosts` is to answer one specific question every single time you connect: "am I talking to the *same* server I successfully verified last time, or has something changed?" A changed host key means one of exactly two things happened — either the server itself was legitimately rebuilt with a fresh SSH installation (which generates a brand-new host key pair), or you are being routed to a **different machine entirely** that is pretending to be your server, potentially controlled by an attacker positioned to intercept your credentials and traffic. SSH has no way to tell these two cases apart on its own — that's exactly why it stops and asks you, the human, to make that judgment call using information SSH doesn't have (like "did our team reimage that box yesterday?"). Bypassing the warning without checking collapses this entire safety mechanism down to "always trust whatever key shows up," which is functionally identical to not having host verification at all. The safe process is always: pause, ask/verify out-of-band whether a legitimate change occurred, and only then clear the specific stale entry with `ssh-keygen -R <hostname>` before reconnecting.

### Why `ss` replaced `netstat`

`netstat` gathers its information by parsing files under `/proc/net/`, which becomes noticeably slower as the number of connections grows on a busy server. `ss` queries the kernel's socket information more directly, which is significantly faster on systems with large numbers of connections — a real, measurable difference on busy production servers, not just a stylistic preference. Ubuntu and most modern distributions ship `ss` (part of the `iproute2` package) by default, while `netstat` (part of `net-tools`) has been considered legacy/deprecated upstream for years, even though it remains extremely common in older documentation, forum answers, and long-standing scripts you'll still encounter on the job.

---

## Practical Examples

### Example 1 — `curl` GET and POST against a realistic API

Testing a JSON API is one of the most common daily uses of `curl`. Here's a GET, then a POST, against a fictional internal API:

```bash
curl -H "Accept: application/json" https://api.example.com/v1/users/42
```

Realistic output:
```json
{"id":42,"name":"Wesam K.","role":"engineer","active":true}
```

```bash
curl -X POST https://api.example.com/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"New Hire","role":"engineer"}'
```

Realistic output:
```json
{"id":137,"name":"New Hire","role":"engineer","active":true}
```

Line-by-line:
- The first command sends a GET (the default method) with an `Accept` header telling the server we want a JSON response; the API returns a single user record.
- `-X POST` explicitly sets the method to POST — creating a new record — even though `-d` alone would have implied POST anyway; writing it explicitly keeps the command self-documenting.
- `-H "Content-Type: application/json"` tells the server the body we're sending is JSON, so it parses it correctly instead of treating it as plain text or a web form.
- `-d '{"name":"New Hire","role":"engineer"}'` is the actual request body — the data being submitted to create the new user.
- The API responds with the newly created record, including a server-assigned `id` of `137`.

⚠️ **Warning:** Notice the JSON body is wrapped in single quotes. If you use double quotes in a typical shell, the shell will try to interpret the `"` characters inside the JSON itself, breaking the command. Always single-quote a JSON body on the command line.

### Example 2 — `curl -I`, `-L`, and `-v` for debugging

```bash
curl -I https://example.com
```

Realistic output:
```
HTTP/2 200
content-type: text/html; charset=UTF-8
content-length: 1256
date: Tue, 28 Jul 2026 14:02:11 GMT
```

```bash
curl -I http://example.com/old-page
```

Realistic output:
```
HTTP/1.1 301 Moved Permanently
Location: https://example.com/new-page
```

```bash
curl -L -v http://example.com/old-page 2>&1 | head -15
```

Realistic output (abbreviated):
```
> GET /old-page HTTP/1.1
> Host: example.com
>
< HTTP/1.1 301 Moved Permanently
< Location: https://example.com/new-page
<
* Ignoring the response-body
* Issue another request to this URL: 'https://example.com/new-page'
> GET /new-page HTTP/1.1
> Host: example.com
>
< HTTP/2 200
< content-type: text/html; charset=UTF-8
```

Line-by-line:
- `curl -I https://example.com` confirms the site is up and returns `200 OK` (success) without downloading the full page body — fast for a quick health check.
- The second `curl -I` reveals the old URL actually returns a `301 Moved Permanently` redirect, pointing to `/new-page` — but since we didn't pass `-L`, `curl` just shows us the redirect response itself rather than following it.
- The third command adds both `-L` (follow the redirect automatically) and `-v` (show the full conversation). The `>` lines are what `curl` sent; the `<` lines are what the server sent back. You can see the initial `301` response, then `curl` automatically issuing a second request to the new location, which finally returns `200`.

💡 **Tip:** `-I` alone is the fastest way to check "is this URL up, and what status code does it return?" without pulling down potentially megabytes of body content you don't need.

### Example 3 — `ssh-keygen`, `ssh-copy-id`, and `~/.ssh/config` walkthrough

```bash
ssh-keygen -t ed25519 -C "weki@work-laptop"
```

Realistic output:
```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/weki/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/weki/.ssh/id_ed25519
Your public key has been saved in /home/weki/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:9fK3jz... weki@work-laptop
```

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub weki@203.0.113.10
```

Realistic output:
```
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s)...
weki@203.0.113.10's password:
Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'weki@203.0.113.10'"
and check to make sure that only the key(s) you wanted were added.
```

Now add a `Host` alias to `~/.ssh/config`:

```
Host prod-web
    HostName 203.0.113.10
    User weki
    IdentityFile ~/.ssh/id_ed25519
```

```bash
ssh prod-web
```

Realistic output:
```
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-100-generic x86_64)
weki@prod-web:~$
```

Line-by-line:
- `ssh-keygen -t ed25519` generates a modern, fast, secure key pair (`ed25519` is the current recommended key type for new keys); `-C` attaches a comment (typically an email or a description) to help identify the key later. Choosing a passphrase here adds a second layer of protection on the private key file itself — recommended, though shown here as optional.
- `ssh-copy-id` logs in **using your password one last time**, specifically to append your public key to the server's `~/.ssh/authorized_keys` file. This is the one-time bootstrapping step.
- The `~/.ssh/config` block defines `prod-web` as a shortcut for the full connection details — host, user, and which key to use.
- `ssh prod-web` now connects with **zero password prompt** — key-based authentication happens silently and you land directly in a shell on the remote server.

✅ **Best Practice:** Give each key you generate a purpose-specific name (e.g. `id_ed25519_prod` vs. `id_ed25519_personal`) rather than always using the default filename, especially once you manage keys for multiple unrelated servers or accounts.

### Example 4 — `scp` and `rsync` in practice

```bash
scp ./deploy-package.tar.gz prod-web:/opt/releases/
```

Realistic output:
```
deploy-package.tar.gz             100%   842MB  38.2MB/s   00:22
```

```bash
rsync -avz ./build/ prod-web:/var/www/app/
```

Realistic output:
```
sending incremental file list
build/
build/index.html
build/assets/main.js
build/assets/style.css

sent 428,301 bytes  received 1,120 bytes  95,426.89 bytes/sec
total size is 1,842,003  speedup is 4.29
```

Line-by-line:
- `scp ./deploy-package.tar.gz prod-web:/opt/releases/` uses the `prod-web` alias from `~/.ssh/config` — notice we never re-typed the username, IP, or key path, because `scp` reuses the same config `ssh` does. It copies the whole archive in one shot with a progress indicator.
- `rsync -avz ./build/ prod-web:/var/www/app/` mirrors a local `build/` directory to the remote path. The trailing slash on `./build/` matters: it means "copy the *contents* of build/," not "create a build/ folder inside the destination."
- The `rsync` summary line's **"speedup is 4.29"** is the detail worth noticing: `rsync` compared what already existed on the remote side against the local files and only actually transferred the parts that differed, finishing roughly 4x faster than a full re-copy would have. Running this exact same command again after only editing `style.css` would transfer barely anything at all — just the changed bytes.

🎯 **On the job:** This is the "securely copying files to a production server" scenario from the Module Goal. `scp` is perfectly fine for the one-time archive drop; `rsync -avz` is what you'd actually use for repeatedly deploying an updated `build/` directory to the same server, since every subsequent deploy only pushes what changed.

### Example 5 — `ss -tulpn`: reading listening ports

```bash
sudo ss -tulpn
```

Realistic output:
```
Netid State   Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
udp   UNCONN  0      0      127.0.0.53%lo:53    0.0.0.0:*          users:(("systemd-resolve",pid=612,fd=13))
tcp   LISTEN  0      128    0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=901,fd=3))
tcp   LISTEN  0      511    127.0.0.1:5432        0.0.0.0:*          users:(("postgres",pid=1140,fd=5))
tcp   LISTEN  0      511    0.0.0.0:80            0.0.0.0:*          users:(("nginx",pid=2043,fd=6))
```

Line-by-line:
- `Netid` shows `tcp` or `udp` — the protocol.
- `State LISTEN` means this socket is actively waiting for incoming connections (that's what `-l` filtered for).
- `Local Address:Port` is the key column: `0.0.0.0:22` means SSH (`sshd`) is listening on port 22, on **every** network interface (`0.0.0.0` means "all addresses"). `127.0.0.1:5432` means PostgreSQL is listening on port 5432, but **only** on localhost — not reachable from outside this machine at all, a common and deliberate security choice for a database.
- `Process` (shown because of `-p`, and requiring `sudo` to see other users' process ownership) tells you exactly which running program owns that port — `sshd`, `postgres`, `nginx` — which is exactly what you need when you find an unexpected open port and need to know what's actually responsible for it.

🎯 **On the job:** This is the "is the right port even open" link in the Module Goal's troubleshooting chain. If an app can't reach a local database on port 5432, `ss -tulpn` immediately tells you whether Postgres is even listening there, and — critically — whether it's bound to `127.0.0.1` (only reachable from the same machine) versus `0.0.0.0` (reachable from other machines too), which is very often the actual root cause of "works locally, fails from another server."

---

## Common Pitfalls & Best Practices

- **Blindly accepting a changed host key warning.** Typing `yes` (or worse, deleting all of `known_hosts`) just to make the scary message go away defeats the entire purpose of host verification. Always investigate first — confirm with whoever manages the server whether a legitimate rebuild happened — then remove only the specific stale entry with `ssh-keygen -R <hostname>`.
- **Leaving password authentication enabled after key-based auth works.** If a server accepts both passwords and keys, it's only as secure as the weakest option an attacker can try — and passwords are brute-forceable in a way keys practically aren't. Once key-based login is confirmed working, disable `PasswordAuthentication` in the server's `/etc/ssh/sshd_config` (a server-side setting, applied by whoever administers that box) so keys become the only way in.
- **Forgetting to quote URLs with special characters.** A URL containing `&`, `?`, or spaces (common in query strings) will be misinterpreted by your shell if left unquoted — `&` in particular can cause the shell to background part of the command entirely. Always wrap URLs (and JSON bodies) in quotes: `curl "https://api.example.com/search?q=cats&limit=10"`.
- **Using `wget` for JSON APIs instead of `curl`.** `wget` has no convenient way to set an HTTP method, attach a JSON body, or set custom headers the way `curl` does — trying to force it to behave like an API client fights the tool's design. Reach for `curl` whenever the task is "talk to an API," and save `wget` for "download this file."
- **Copying a private key to a server, or ever transmitting it anywhere.** Only the **public** key (`.pub` file) should ever leave your machine. If a private key is ever copied to a shared or remote system, treat it as compromised and generate a fresh key pair.
- **Assuming `ping` working means an application will work too.** `ping` only proves basic network reachability via ICMP — plenty of servers deliberately block ICMP for security reasons while their actual services (web, API, database) work completely normally, and conversely, a host can respond to ping while the specific application port is down or firewalled. Use `ping` as one data point, not the final word — follow up with `curl` or `ss` for anything protocol-specific.

✅ **Best Practice — The layered troubleshooting mindset:** When something "can't connect," work outward to inward, one layer at a time: is the host reachable at all (`ping`)? Is the right port even open on the target (`ss -tulpn` on the target, if you have access)? Does the actual application respond correctly (`curl -v`)? Each tool isolates one specific layer, so you never waste time debugging the wrong one.

---

## Hands-on Exercise

**Task:**

1. Generate a new SSH key pair dedicated to practice, and install its public key onto a practice host you control (or a free online Linux sandbox that supports SSH).
2. Add a `Host` alias for that practice host in `~/.ssh/config`, and confirm you can connect with no password prompt.
3. Create a small local file and copy it to the practice host using both `scp` and `rsync -avz` — compare the two.
4. Use `curl` to make a GET request to the public test API `https://httpbin.org/get`, then a POST request with a JSON body to `https://httpbin.org/post`, and read the response each time.

Try this yourself before reading the solution below.

### Solution

```bash
# 1. Generate a dedicated practice key pair
ssh-keygen -t ed25519 -C "practice-key" -f ~/.ssh/id_ed25519_practice
```
```
Generating public/private ed25519 key pair.
Your identification has been saved in /home/weki/.ssh/id_ed25519_practice
Your public key has been saved in /home/weki/.ssh/id_ed25519_practice.pub
```

```bash
# Install the public key on the practice host (replace with your real host/user)
ssh-copy-id -i ~/.ssh/id_ed25519_practice.pub practiceuser@203.0.113.20
```
```
Number of key(s) added: 1
```

```bash
# 2. Add a Host alias — append this block to ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'

Host practice
    HostName 203.0.113.20
    User practiceuser
    IdentityFile ~/.ssh/id_ed25519_practice
EOF

# Confirm passwordless login
ssh practice "echo connected successfully"
```
```
connected successfully
```

```bash
# 3. Create a test file and copy it two ways
echo "hello from my laptop" > note.txt

scp note.txt practice:/home/practiceuser/note-scp.txt
rsync -avz note.txt practice:/home/practiceuser/note-rsync.txt
```
```
note.txt                                     100%   22     1.1KB/s   00:00
sending incremental file list
note.txt

sent 126 bytes  received 35 bytes  322.00 bytes/sec
total size is 22  speedup is 0.14
```

```bash
# 4. curl against httpbin.org — GET then POST
curl https://httpbin.org/get
```
```json
{
  "args": {},
  "headers": {"Host": "httpbin.org", "User-Agent": "curl/8.5.0"},
  "url": "https://httpbin.org/get"
}
```

```bash
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"exercise":"module12","status":"complete"}'
```
```json
{
  "args": {},
  "data": "{\"exercise\":\"module12\",\"status\":\"complete\"}",
  "headers": {"Content-Type": "application/json", "Host": "httpbin.org"},
  "json": {"exercise": "module12", "status": "complete"},
  "url": "https://httpbin.org/post"
}
```

Explanation: The key pair was generated with a distinct filename (`-f`) so it doesn't collide with any other key already sitting in `~/.ssh/`. `ssh-copy-id` handled installing the public key using the one-time password login. The `Host practice` block in `~/.ssh/config` means every subsequent command — `ssh`, `scp`, and `rsync` alike — can just say `practice` instead of repeating the full user/host/key details, and the passwordless `ssh practice "echo ..."` confirms key-based auth actually took effect. `scp` and `rsync` both moved the same tiny file successfully; on a real, larger file, running the same `rsync` command a second time (say, after a small edit) would demonstrate the delta-transfer speedup that `scp` can't offer. Finally, `httpbin.org` is a public API purpose-built for testing HTTP clients — it echoes back exactly what it received, which is why the POST response shows the JSON body right back in its own `"json"` field, confirming the request was built correctly.

✅ Exercise complete — you've set up passwordless SSH key auth end-to-end, moved a file both ways SSH supports, and exercised both a GET and a POST with `curl` against a real API.

---

## ✅ Module Completion Checklist

- [ ] I can make and debug HTTP requests with `curl`, including GET and POST requests, custom headers, saving output to a file, following redirects, inspecting headers-only, and verbose debugging output
- [ ] I can download files with `wget`, rename the saved file, resume an interrupted download, and explain when to reach for `wget` instead of `curl` (and vice versa)
- [ ] I can explain what SSH is, connect to a remote host with `ssh user@host`, and explain the public/private key-pair authentication model conceptually
- [ ] I can generate an SSH key pair with `ssh-keygen`, install a public key on a remote server with `ssh-copy-id`, and set up a `~/.ssh/config` host alias for fast, repeatable connections
- [ ] I can explain what `known_hosts` is, and correctly interpret a "host key verification failed" warning — including why I investigate it instead of blindly bypassing it
- [ ] I can copy files to and from a remote server with `scp`, and explain why `rsync -avz` is the more robust modern alternative for anything beyond a one-off copy
- [ ] I can test host reachability with `ping`, read its output correctly, and explain what ICMP is at a basic level
- [ ] I can inspect listening ports and active connections with `ss -tulpn` (and know `netstat` as its legacy predecessor), map local hostnames with `/etc/hosts`, and perform a basic DNS lookup with `dig`/`nslookup`/`host`

## Next Step

Continue to [Module 13: Terminal Productivity](../module13-terminal-productivity/)
