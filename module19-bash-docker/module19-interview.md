# 🎤 Module 19 Interview Prep — Bash + Docker (Entrypoints & CLI Scripting)

## Conceptual Questions

### 🟢 Beginner

**Q1: Why does a course on Bash have a whole module on Docker? Isn't containerization a separate skill?**

> "Containerizing an app doesn't remove the need for bash — it usually adds a thin layer of it. Almost every real container image has an entrypoint script, some setup/init logic, and CI/CD pipeline steps that are all just bash under the hood, even if the actual application is written in something completely different like Node or Java. That layer is small, but when it's wrong, it causes exactly the kind of hard-to-diagnose production problems this module is built around — containers that won't shut down cleanly, or that crash-loop on startup because they started before a dependency was ready."

**Q2: What is PID 1, and why is it special?**

> "PID 1 is the very first process in a process namespace — on a whole machine, that's normally `init` or `systemd`; inside a container, it's whatever process Docker launches as the `ENTRYPOINT`/`CMD`. The kernel treats PID 1 differently from every other process: it doesn't apply any default signal handling to it. For an ordinary process, an unhandled `SIGTERM` terminates it by default; for PID 1, an unhandled signal is just ignored, because PID 1 on a real machine is `init`, and the kernel doesn't want a stray signal accidentally taking down the whole system's init process."

**Q3: What does `exec` do in Bash, in one sentence?**

> "It replaces the currently running process with another program — same PID, no new process forked — rather than starting the new program as a child and waiting for it."

**Q4: Why does `exec` matter in a Docker entrypoint script?**

> "Because whatever your entrypoint script runs as its main process becomes PID 1 inside the container, and PID 1 gets no default signal handling from the kernel. If the entrypoint just runs the application as an ordinary foreground command, bash itself stays as PID 1 and forks the app as a child — so when `docker stop` sends `SIGTERM` to PID 1, it goes to bash, which has no handler for it and ignores it completely. The application child process never even hears about it. `exec "$@"` at the end of the entrypoint replaces bash with the application in-place, so the application itself becomes PID 1 and receives `SIGTERM` directly, letting it run its own graceful-shutdown logic."

**Q5: What happens to signals if you don't use `exec` in your entrypoint?**

> "They get silently swallowed. Bash stays as PID 1, has no default handler for something like `SIGTERM`, and simply ignores it — meanwhile the actual application is running as a child process that's never notified at all. From the outside, this looks like the container 'ignoring' `docker stop` — Docker waits out its grace period with no cooperation, then falls back to `SIGKILL`, which cannot be caught by anything, forcibly terminating the whole process tree with no chance for the app to finish in-flight work or shut down cleanly."

**Q6: What's a `HEALTHCHECK` in a Dockerfile, and what do the exit codes mean?**

> "It's an instruction telling Docker to periodically run a command inside the running container to judge whether the application is actually healthy, not just whether the process technically exists. The command's exit code is what Docker reads: `0` means healthy, `1` means unhealthy. A small bash script is a natural fit — check something real, like an HTTP endpoint responding, and translate the result into that exit code."

### 🟡 Intermediate

**Q7: How do you handle a container starting before its dependency is ready — say, a web app that depends on Postgres?**

> "The core insight is that a container being 'running' and the service inside it being 'ready' are two different moments — `docker run postgres` returns almost instantly, but Postgres itself still needs to run its own internal startup sequence before it can accept connections. The fix is a wait loop in the entrypoint or a preceding script: poll the dependency — a TCP port check with `nc -z`, an HTTP check with `curl`, or a purpose-built tool like `pg_isready` for Postgres specifically — retrying with a short sleep between attempts, and critically, bounded by a timeout so a genuinely broken dependency fails with a clear error instead of hanging forever. There are also existing tools that implement this — `wait-for-it.sh` and `dockerize` — which are perfectly reasonable to use in production instead of hand-rolling the loop."

**Q8: Walk me through exactly what happens, step by step, when `docker stop` is sent to a container whose entrypoint forgot `exec`.**

> "`docker stop` sends `SIGTERM` to the container's PID 1, which in this case is bash, since it never replaced itself with the application. Bash has registered no handler for `SIGTERM` — the kernel's rule for PID 1 is that unhandled signals are just ignored, not defaulted to 'terminate' like they would be for an ordinary process. So bash keeps running, and the actual application — a separate child process bash forked to run it — never receives anything at all. Docker then waits out its configured grace period, typically 10 seconds, sees the container still running, and sends `SIGKILL` instead, which cannot be caught or ignored by any process. The whole process tree is torn down abruptly, with the application having had zero opportunity to finish in-flight requests or close connections cleanly."

**Q9: What's the difference between `${VAR:-default}` and `${VAR:?message}`, and when would you use each in a container entrypoint?**

> "`${VAR:-default}` silently substitutes `default` if `VAR` is unset, and execution continues normally — that's the right tool for genuinely optional configuration, like a port number or a log level, where a sensible fallback exists. `${VAR:?message}` does the opposite: it causes the shell to exit immediately with `message` printed to stderr if `VAR` is unset or empty — that's for anything the application truly cannot run correctly without, like a database connection string or a secret key. Using `:-` where you meant `:?` is a real bug — it turns a missing required value into a silent empty string that fails confusingly somewhere deep inside the application, instead of a clear, immediate error at container startup."

**Q10: Why would you script `docker ps`/`docker logs`/`docker exec` from bash instead of just running them manually?**

> "Manually is fine for a one-off investigation, but the value of scripting them is repeatability and speed for things you do often — finding a container by name or label filter and immediately tailing its logs, or exec-ing a health check into every container matching a label in a loop. It's the same motivation as any other bash automation in this course: turning a multi-step manual process you'd otherwise have to remember and retype into a single, reliable command, especially useful when there could be several containers matching a pattern and you want the same check run against all of them consistently."

### 🔴 Advanced

**Q11: A team says "our container sometimes takes 10+ seconds to stop during every deploy, but we never see any errors." How would you investigate, and what's the most likely root cause?**

> "First thing I'd check is whether the application process inside the container is actually PID 1 — `docker exec <container> ps aux` and look at what's running as PID 1. If it's `bash` or `sh` instead of the application itself, that's almost certainly the root cause: the entrypoint is running the app as an ordinary foreground command instead of `exec`-ing it, so `docker stop`'s `SIGTERM` goes to bash, gets silently ignored per PID 1's signal rules, and the container sits there for the full grace period before Docker escalates to `SIGKILL`. The '10+ seconds, no errors' symptom is the textbook signature of this exact bug — the grace period is usually configured around 10 seconds by default, and there are no errors because nothing actually failed; the app was simply never asked to shut down in the first place. The fix is adding `exec` as the last line of the entrypoint script, ahead of the application command."

**Q12: You're reviewing a wait-for-dependency script and see it has no timeout. Why is that a real production risk, and how would you fix it while explaining the tradeoffs of the timeout value you choose?**

> "An unbounded wait loop — retrying forever with no exit condition — means a genuinely broken dependency (wrong hostname, firewall rule, the dependency container crash-looping itself) causes this container to hang indefinitely at startup, with no error message and no crash, which is actually worse than crashing because there's nothing for monitoring or alerting to catch; the container just looks like it's 'still starting' forever. I'd add a bounded loop — track elapsed time, and once it exceeds a timeout, exit with a clear error message naming exactly what wasn't reachable. Choosing the timeout value is a real tradeoff: too short, and a dependency that's simply a bit slow to start under normal conditions (cold start, a large migration running) causes false failures and unnecessary restart-loop churn; too long, and a genuinely broken deploy takes that long to surface as a visible failure. I'd generally start around 30-60 seconds, make it configurable via an environment variable so it can be tuned per environment without a rebuild, and rely on the orchestrator's own restart policy to retry a small number of times beyond that rather than making the wait loop itself absurdly generous."

**Q13: How does the discipline you'd apply to a CI/CD pipeline script relate to what you learned about `set -euo pipefail` and cron jobs earlier in this course?**

> "It's the same discipline, applied to another kind of unattended-execution environment. A CI runner executing a `script:` step is conceptually identical to a cron job: nobody is watching it run live, and if a step fails silently but the pipeline still reports success, a broken build or a bad deploy ships with nobody the wiser — exactly the same failure mode as a cron job that silently stops working and nobody notices until someone asks why last night's report didn't run. So the same fixes apply directly: `set -euo pipefail` at the top of any real script step so a failure actually stops the step instead of limping forward, meaningful and distinct exit codes so the calling system can react correctly, and fail-fast validation — a `die()`-style check — for missing configuration like a deployment secret, so the pipeline fails immediately and legibly at the point of the actual problem instead of surfacing a confusing downstream error several steps later."

---

## Practical/Coding Questions

**Q1: Here's an entrypoint script. Identify the bug and fix it.**

```bash
#!/bin/bash
set -euo pipefail
echo "Starting up..."
"$@"
```

Solution:
```bash
#!/bin/bash
set -euo pipefail
echo "Starting up..."
exec "$@"
```
Explanation: without `exec`, `"$@"` runs as an ordinary foreground command — bash forks a child process for it and stays as PID 1 itself. Since bash has no default `SIGTERM` handler, `docker stop` can't gracefully signal the real application at all; it has to wait out the grace period and force-kill. Adding `exec` replaces bash with the application in the same PID slot, so signals reach it directly.

**Q2: Write an env-var validation block for a script that requires `API_KEY` and `DATABASE_URL`, and defaults `PORT` to `8080` and `LOG_LEVEL` to `info`.**

Solution:
```bash
: "${API_KEY:?API_KEY must be set}"
: "${DATABASE_URL:?DATABASE_URL must be set}"
PORT="${PORT:-8080}"
LOG_LEVEL="${LOG_LEVEL:-info}"
```
Explanation: `${VAR:?message}` exits immediately with a clear message if `API_KEY` or `DATABASE_URL` is unset or empty — these are required, so failing fast and loud at startup is correct. `${VAR:-default}` silently fills in a fallback for `PORT`/`LOG_LEVEL` since the app can reasonably run without them being explicitly set.

**Q3: Write a healthcheck script that checks whether a local web server on port `5000` responds successfully at `/status`, with a 2-second timeout, returning the exit code Docker's `HEALTHCHECK` expects.**

Solution:
```bash
#!/bin/bash
set -euo pipefail
if curl -fs -o /dev/null --max-time 2 "http://localhost:5000/status"; then
    exit 0
else
    exit 1
fi
```
Explanation: `-f` makes `curl` itself return non-zero on an HTTP error response (without it, a 500 page would still count as a "successful" curl); `--max-time 2` bounds how long the check itself can take so a hung app doesn't also hang the healthcheck. The `if`/`else` maps success/failure directly onto exit `0`/`1`, which is exactly what Docker's `HEALTHCHECK CMD` reads to decide `healthy`/`unhealthy`.

**Q4: Write a bash wait loop that waits for a Redis server at `$REDIS_HOST:$REDIS_PORT` (default port `6379`), retrying every 2 seconds, failing after 20 seconds total.**

Solution:
```bash
REDIS_HOST="${REDIS_HOST:?REDIS_HOST must be set}"
REDIS_PORT="${REDIS_PORT:-6379}"
TIMEOUT=20
elapsed=0
until nc -z "$REDIS_HOST" "$REDIS_PORT" 2>/dev/null; do
    elapsed=$((elapsed + 2))
    if [ "$elapsed" -ge "$TIMEOUT" ]; then
        echo "ERROR: timed out after ${TIMEOUT}s waiting for Redis at ${REDIS_HOST}:${REDIS_PORT}" >&2
        exit 1
    fi
    sleep 2
done
echo "Redis is reachable"
```
Explanation: `nc -z` checks whether the TCP port is open without sending any real data. The loop increments `elapsed` by the same interval as the `sleep`, and bails out with a clear, specific error message the moment `elapsed` reaches the timeout — never retrying forever.

---

## Gotcha Questions

**Q1: "My entrypoint script has `set -euo pipefail` at the top, so it must be production-ready. What am I missing?"**

> Trap: `set -euo pipefail` handles error propagation *within* the entrypoint script's own execution — it has nothing to do with what happens to the process *after* the script hands off to the application. A perfectly strict-mode script can still forget `exec "$@"` at the end, leaving bash as PID 1 and silently swallowing `SIGTERM` from `docker stop`. Strict mode and correct signal forwarding are two entirely separate concerns; you need both.

**Q2: "I added a wait loop for my database dependency, tested it locally, and it worked great — the database was always up before my app container. Ship it?"**

> Trap: The candidate needs to notice there's no timeout mentioned. It "working great" locally, where the dependency is reliably fast, tells you nothing about what happens when the dependency is slow, misconfigured, or genuinely down — which is exactly the scenario the wait loop exists for in the first place. Without a timeout, that exact failure case turns into an indefinite hang with no error message at all, which is often harder to diagnose than an outright crash, because nothing looks obviously broken — the container just never finishes starting.

**Q3: "I used `exec "$@"` in my entrypoint, but I also wanted to log something after the app exits, like 'application has stopped.' Why doesn't that log line ever print?"**

> Trap: This is a fundamental, unavoidable property of `exec`, not a bug to work around by tweaking the script. `exec` replaces the current process in place — after it succeeds, the bash script's own code no longer exists in that process at all, so there is no "after" for it to return to and run more lines. If you genuinely need something to run after the application exits, `exec` is the wrong tool for that specific line — you'd need to *not* exec, and accept the resulting signal-forwarding cost, or handle that logging need a different way entirely (e.g., have the orchestrator's own lifecycle hooks log the container's exit, rather than the entrypoint script itself).

---

## Quick-Fire Rapid Review

- **Q: What becomes PID 1 inside a Docker container?** A: Whatever process is launched as the `ENTRYPOINT`/`CMD`.
- **Q: Does the kernel apply a default signal handler to PID 1?** A: No — an unhandled signal on PID 1 is simply ignored, unlike an ordinary process.
- **Q: What does `exec` do?** A: Replaces the current process with another command — same PID, no child forked.
- **Q: What's the one-line fix for a container that ignores `docker stop`?** A: Add `exec "$@"` as the last line of the entrypoint.
- **Q: What signal does `docker stop` send first, and what follows if the process doesn't exit?** A: `SIGTERM` first, then `SIGKILL` after the grace period.
- **Q: Can `SIGKILL` be caught or ignored?** A: No — by any process, ever.
- **Q: What's the syntax for a required env var that fails fast if missing?** A: `${VAR:?message}`.
- **Q: What's the syntax for an optional env var with a default?** A: `${VAR:-default}`.
- **Q: What exit codes does Docker's `HEALTHCHECK` expect?** A: `0` for healthy, `1` for unhealthy.
- **Q: Why doesn't a running container guarantee the service inside is ready?** A: The process inside may still be running its own internal startup sequence after the container itself has started.
- **Q: What must a wait-for-dependency loop always include?** A: A timeout.
- **Q: Name two off-the-shelf tools that implement the wait-for-dependency pattern.** A: `wait-for-it.sh` and `dockerize`.
- **Q: What Docker CLI command lists running containers, filterable by name/label?** A: `docker ps --filter`.
- **Q: What Docker CLI command streams a container's stdout/stderr?** A: `docker logs -f`.
- **Q: What Docker CLI command runs a command inside an already-running container?** A: `docker exec`.
- **Q: Why should you avoid hardcoding hostnames in container scripts?** A: The same image needs to run unmodified across dev/staging/production with different networking — use environment variables instead.
- **Q: How is a CI/CD runner similar to a cron job?** A: Both are unattended-execution environments — a silent failure ships or breaks something with nobody watching live.
