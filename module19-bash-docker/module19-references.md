# 📚 Module 19 References — Bash + Docker (Entrypoints & CLI Scripting)

Curated resources for this module's scope: Docker entrypoint scripts, the PID 1 / `exec`/signal-forwarding problem, environment variable configuration patterns, `HEALTHCHECK` scripts, wait-for-dependency loops, Docker CLI scripting, and CI/CD bash glue code. Free resources are listed first in each category.

⚠️ **Note:** Links can change over time. If any link below is broken, search for the resource by name — these are all well-known, actively maintained resources as of this writing.

## 📺 YouTube Videos & Channels

- **[TechWorld with Nana](https://www.youtube.com/@TechWorldwithNana)** 🟢🟡 — Clear, widely-recommended Docker fundamentals content, including entrypoint scripts and container startup behavior, aimed at people moving from "using Docker" to "operating it well."
- **[NetworkChuck](https://www.youtube.com/@NetworkChuck)** 🟢 — Approachable, practical Docker walkthroughs, useful for reinforcing the CLI basics this module scripts against.
- **[Bret Fisher Docker and DevOps](https://www.youtube.com/@BretFisherDockerandDevOps)** 🟡🔴 — A Docker Captain's channel with deep, production-focused content specifically on entrypoints, healthchecks, and real-world container operations.

## 📖 Official Documentation

- **[Docker Docs — Dockerfile reference: `ENTRYPOINT`](https://docs.docker.com/reference/dockerfile/#entrypoint)** 🟡 — The authoritative specification of how `ENTRYPOINT` and `CMD` combine, exec form vs. shell form, and how arguments are passed through.
- **[Docker Docs — Dockerfile reference: `HEALTHCHECK`](https://docs.docker.com/reference/dockerfile/#healthcheck)** 🟡 — The authoritative reference for `HEALTHCHECK` syntax, options (`--interval`, `--timeout`, `--retries`), and how exit codes map to container health status.
- **[Docker Docs — "Foreground process must not daemonize" / signal handling in containers](https://docs.docker.com/engine/containers/multi-service_container/)** 🟡🔴 — Docker's own guidance on running a container's main process correctly, directly relevant to the PID 1/`exec` discussion in this module.
- **[Docker Docs — `docker stop` reference](https://docs.docker.com/reference/cli/docker/container/stop/)** 🟢🟡 — Explains exactly what `docker stop` does: sends `SIGTERM`, waits a grace period, then sends `SIGKILL`.
- **[Docker Docs — `docker exec`, `docker logs`, `docker ps` CLI reference](https://docs.docker.com/reference/cli/docker/)** 🟢 — The complete, authoritative flag reference for every Docker CLI subcommand used in this module's scripting examples.
- **[GNU Bash Manual — "Bourne Shell Builtins" (`exec`)](https://www.gnu.org/software/bash/manual/bash.html#Bourne-Shell-Builtins)** 🟡 — The precise, authoritative behavior of the `exec` builtin straight from GNU.
- **[GitHub Actions Docs — "Workflow syntax" (`run` steps)](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)** 🟡 — Official reference for how `run:` steps execute shell commands, relevant to this module's CI/CD tie-in.
- **[GitLab CI/CD Docs — `.gitlab-ci.yml` reference (`script`)](https://docs.gitlab.com/ee/ci/yaml/)** 🟡 — Official reference for how GitLab CI `script:` steps run as shell commands on a runner.

## 📝 Tutorials & Articles

- **["Understanding process behavior in Docker" — Docker Blog / Docs](https://docs.docker.com/engine/containers/multi-service_container/)** 🟡🔴 — Directly addresses why PID 1 matters and how to run a main process so signals forward correctly.
- **["Demystifying the init system (PID 1)" by Elton Stoneman / various container-focused authors](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)** 🔴 — A widely-cited deep dive into the PID 1 problem inside containers, covering both signal handling and the related "zombie process reaping" issue.
- **["12 Fractured Apps" / The Twelve-Factor App — "Config"](https://12factor.net/config)** 🟢🟡 — The canonical explanation of why configuration belongs in environment variables, not code — the philosophy behind this module's env-var validation patterns.
- **[vishnubob/wait-for-it — README](https://github.com/vishnubob/wait-for-it)** 🟢🟡 — The widely-used bash script for waiting on a TCP host:port with a timeout; read the README for usage patterns and how it composes with `exec`.
- **[jwilder/dockerize — README](https://github.com/jwilder/dockerize)** 🟡 — An alternative wait/config-templating tool, useful to see a different (Go-based) approach to the same wait-for-dependency problem.

## 🎓 Courses & Learning Portals

- **[Docker Docs — "Get started" guide](https://docs.docker.com/get-started/)** 🟢 — Free, official, and the most authoritative starting point for anyone shaky on container fundamentals before diving into entrypoint scripting specifically.
- **[KodeKloud — Docker for the Absolute Beginner](https://kodekloud.com/)** 🟢🟡 — A popular, hands-on paid course with labs covering Dockerfile authoring, healthchecks, and container lifecycle.
- **[freeCodeCamp — Docker course (YouTube/certification)](https://www.freecodecamp.org/news/tag/docker/)** 🟢🟡 — Free, long-form, project-based Docker content.

## 🌐 Websites & Interactive Platforms

- **[Play with Docker](https://labs.play-with-docker.com/)** 🟢🟡 — Free, disposable Docker playground in the browser — a safe place to test entrypoint scripts, kill signals, and healthchecks without touching a real machine.
- **[Katacoda-style labs / Killercoda — Docker scenarios](https://killercoda.com/)** 🟢🟡 — Free interactive terminals with Docker pre-installed, good for practicing the wait-loop and signal-forwarding examples in this module hands-on.
- **[explainshell.com](https://explainshell.com/)** 🟢🟡 — Paste any of this module's `curl`/`nc`/parameter-expansion lines and see every flag broken down individually.

## 📚 Books

- **["Docker Deep Dive" by Nigel Poulton](https://nigelpoulton.com/books)** 🟡🔴 — A widely-recommended, up-to-date book covering container internals, including process behavior, in more depth than official docs alone.
- **["The Linux Programming Interface" by Michael Kerrisk (No Starch Press)](https://man7.org/tlpi/)** 🔴 — The definitive systems-level reference on processes, signals, and PID 1 semantics, for anyone who wants the kernel-level "why" behind this module's central concept.
- **"Kubernetes Up & Running" by Brendan Burns, Joe Beda & Kelsey Hightower (O'Reilly)** 🟡🔴 — Not Docker-specific, but its coverage of readiness/liveness probes and graceful shutdown extends this module's concepts to orchestrated environments.

## 👥 Communities

- **[Docker Community Forums](https://forums.docker.com/)** 🟢🟡🔴 — Official community forum; searchable for entrypoint, healthcheck, and signal-handling questions with real production troubleshooting threads.
- **[r/docker](https://www.reddit.com/r/docker/)** 🟡🔴 — Active community for practical Docker questions, including real "why won't my container stop gracefully" war stories.
- **[Stack Overflow — `docker` and `dockerfile` tags](https://stackoverflow.com/questions/tagged/docker)** 🟢🟡🔴 — Enormous, searchable archive; questions like "container doesn't respond to SIGTERM" or "wait for database before starting" already have well-upvoted, detailed answers.
- **[CNCF Slack](https://slack.cncf.io/)** 🟡🔴 — Active channels covering containers and cloud-native tooling generally, useful once you're extending these patterns beyond plain Docker into orchestrated environments.
