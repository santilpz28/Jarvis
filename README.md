# JARVIS 🛡️

**Self-hosted AI security agent.** Runs on local LLMs so code-audit data never
leaves the machine. Connected to a hardened Linux VPS over an encrypted
Zero-Trust tunnel.

> Private repo material — this is the operational layer, not a tutorial.

---

## What it does

JARVIS is an OpenClaw-based agent wired to a production VPS. It watches a
Docker stack, reports on it, and delivers the results — autonomously.

```
┌──────────────┐   Zero-Trust tunnel    ┌─────────────────────┐
│  Local box   │◄───────────────────────►│  VPS (clon)         │
│  WSL2 + Brave│                        │  Apache + Docker    │
│  Ollama      │                        │  FastAPI + Postgres │
└──────────────┘                        └─────────────────────┘
```

| Layer | What runs |
|---|---|
| **Agent** | OpenClaw gateway, local model routing (Ollama + hosted fallbacks) |
| **Stack** | `chainguard_db` (Postgres) · `chainguard_cortex` (FastAPI) · `chainguard_ui` (nginx) |
| **Edge** | Apache reverse proxy, TLS via Let's Encrypt, fail2ban, UFW |
| **Delivery** | Telegram bot + weekly report generator + audio briefing |

---

## The scripts

Everything under `scripts/` is what actually runs in production.

| Script | Role |
|---|---|
| `chainguard_stack_health.sh` | **The health check.** Runs every 15 min. Restarts dead containers, verifies backend + UI, checks the gateway process, drains the pending-delivery queue. |
| `deliver_pending.py` | **The queue consumer.** Reads `pending_delivery.json`, sends report + audio over Telegram, deduplicates by `report_file`, marks items `delivered`. Supports `--dry-run` and `--limit`. |
| `chainguard_weekly_report.sh` | Builds the weekly ChainGuard report (text + audio) and enqueues it. |
| `jarvis_scout_weekly.sh` | Playwright-based scout that gathers live data. |
| `jarvis-clone-watchdog.sh` | Detects stale gateway locks (`active (exited)` ≠ running) and cleans them. |
| `jarvis-clone-precheck.sh` | Pre-flight check — refuses to start if credentials or the sandbox flag are wrong. |
| `chainguard_sub_issue_cert.sh` | Idempotent cert issuance for the API subdomain. Waits for DNS, then issues and wires the vhost. |

---

## Design rules

These are the constraints that came out of actual incidents, not theory.

### 1. A queue without a consumer is a silent black hole

`chainguard_weekly_report.sh` used to *write* to `pending_delivery.json` and
nothing ever read it. Eight weeks of reports were generated. Zero were
delivered. Nobody knew.

**Rule:** every queue producer needs a consumer in the same health check. If
the stack is green but the queue is growing, that is a red stack.

### 2. `active (exited)` is not `running`

A systemd `Type=oneshot` unit that daemonizes and exits with code 0 will lie
to you forever. `systemctl is-active` says `active`. Nothing is running.

**Rule:** verify with `pgrep -f "<real process>"`, never with `systemctl
is-active`.

### 3. `restart: unless-stopped` does not fight a manual stop

Docker honours the stop. The container stays down forever and the restart
policy quietly does nothing.

**Rule:** health checks are the safety net, not Docker's restart policy.

### 4. A fix that nobody executes is not a fix

The recovery scripts were written locally, in a temp directory, on the wrong
machine. They were never installed on the host that needed them. There was no
cron, no timer, no job. The infrastructure was orphaned code.

**Rule:** a script that must run on another machine gets `scp`'d there,
`chmod +x`'d, and registered in that machine's crontab — then verified with
`crontab -l` *on that host*.

### 5. Cross-validate before declaring anything down

From the WSL side, TCP ports would open and return `http=000`. From the host
itself: `HTTP/1.1 200`. The services were up the whole time. The network path
was lying.

**Rule:** two vantage points before you say "it's down".

### 6. The alert channel cannot depend on the failed host

The Telegram token lived on the VPS. When the VPS died, so did the only way to
report that the VPS died.

**Rule:** the alert path needs a route that does not traverse the failing
component.

### 7. A disabled service must not be resurrected by its own watchdog

The clone's watchdog cleaned the stale lock every 15 minutes. The health check
then saw "no gateway" and started it. The user had turned the bot off, and it
came back on its own — still polling, still burning tokens.

**Rule:** a user decision outranks auto-recovery. The watchdog and the health
check both check an explicit `disabled` flag first, and the start script
refuses to launch while it is set.

### 8. Never hardcode a server's address in a public repository

`chainguard_sub_issue_cert.sh` shipped the clon's public IP in the source.
Anyone scanning GitHub would have a working target for free.

**Rule:** read it from the environment (`CLONE_EXPECT_IP`) or auto-detect it at
runtime. A public repo is a search index.

---

## Configuration

`chainguard_sub_issue_cert.sh` needs to know the host's public IP to verify
that DNS already points at this machine. It reads it from the environment, and
falls back to auto-detection:

```bash
# explicit (recommended in cron/systemd)
export CLONE_EXPECT_IP="<host ip>"

# or let it self-detect — only meaningful when run ON the server
./chainguard_sub_issue_cert.sh
```

Auto-detection resolves to the *machine's own* public IP, so the script is
only correct when executed on the server itself. Everywhere else, set
`CLONE_EXPECT_IP` explicitly.

---

## Stack

- **Agent:** OpenClaw gateway, local LLM inference via Ollama
- **Backend:** Python 3.10 · FastAPI · PostgreSQL · Docker Compose
- **Edge:** Apache 2.4 · Let's Encrypt · UFW · fail2ban
- **Observability:** Prometheus · file-based state locks
- **Delivery:** Telegram Bot API (text + audio) · cron-driven scheduling

---

## License

MIT — see [LICENSE](LICENSE).

Code here is sanitized: no tokens, no keys, no private host details beyond
publicly reachable infrastructure.
