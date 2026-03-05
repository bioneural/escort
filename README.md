<h1 align="center">
  e s c o r t
  <br>
  <sub>API proxy for per-step memory retrieval</sub>
</h1>

A transparent proxy that sits between Claude Code and the Anthropic API. escort intercepts every `POST /v1/messages` request — one per reasoning step — and injects relevant memories from [crib](https://github.com/bioneural/crib) before forwarding to upstream. Every step gets fresh context. Per-session dedup prevents re-injection. Fail-open design means a broken retrieval never blocks the agent.

A single script. Ruby stdlib only. No gems, no API keys.

---

## How it works

escort binds a TCP server on `127.0.0.1:7710`. Claude Code sends API requests to escort instead of directly to the Anthropic API. For every `POST /v1/messages` request, escort:

1. Parses the JSON body and extracts the last user message text
2. Pipes that text to [crib](https://github.com/bioneural/crib) retrieve (the memory store)
3. Filters out memories already injected in this session (per-session dedup)
4. Appends new memories as `<memory>` tags to the last user message
5. Forwards the modified request to the Anthropic API and streams the response back

Everything else passes through unmodified. If crib is unavailable, retrieval times out, or anything goes wrong — the request forwards untouched. Fail-open by design. A broken memory layer must not become a denial-of-service on the agent it serves.

### Per-session dedup

Each Claude Code session carries a unique identifier in `metadata.user_id`. escort tracks which memory lines have been injected per session. A memory surfaced on step 3 is not re-injected on step 7. Parallel sessions — different projects, different terminals — maintain independent dedup sets. Stale sessions are cleaned after one hour of inactivity.

### Zero-downtime restart

escort runs as a supervisor/worker pair. The supervisor forks a worker process and restarts it on any exit — with exponential backoff to handle crash loops (caps at 30 seconds, resets after 5 seconds of healthy runtime). The worker watches its own source file for changes. When a code change is detected, the worker closes its server socket (releasing the port), waits for active connections to drain, and exits. The supervisor immediately forks a new worker with fresh code. New requests are served without interruption.

---

## Usage

Start escort and point Claude Code at it:

```sh
bin/escort &
ANTHROPIC_BASE_URL=http://127.0.0.1:7710 claude
```

escort logs to stderr. Every intercepted request shows message count, model, session, and injection status:

```
escort: supervisor starting (pid 36966)
escort: listening on 127.0.0.1:7710
escort: upstream https://api.anthropic.com
escort: crib /path/to/crib/bin/crib
escort: [conn] POST /v1/messages?beta=true (92020 bytes)
escort: 5 msgs, model=claude-opus-4-6, session=94135728-d556-4b6c-b2b1-595aa2317551
escort: injected 39 new memory line(s) [session=94135728-d556-4b6c-b2b1-595aa2317551]
```

### Health check

```sh
curl http://127.0.0.1:7710/health
# {"status":"ok","port":7710,"upstream":"https://api.anthropic.com","crib":"/path/to/crib/bin/crib","crib_available":true}
```

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ESCORT_PORT` | `7710` | Listen port |
| `ESCORT_UPSTREAM` | `https://api.anthropic.com` | Upstream API URL |
| `ESCORT_CRIB` | sibling `../../crib/bin/crib` | Crib binary path |
| `ESCORT_TOKEN_BUDGET` | `500` | Max tokens injected per step |
| `CRIB_DB` | crib's default | Crib database path |

---

## Architecture

```
Claude Code ──POST /v1/messages──▶ localhost:7710 (escort) ──▶ api.anthropic.com
                                         │
                                   crib retrieve
                                   (local SQLite)
```

**Supervisor** — forks and monitors a worker process. Restarts on any exit with exponential backoff (0.5s base, doubles per consecutive fast failure, caps at 30s). Resets after the worker runs for more than 5 seconds.

**Worker** — runs the proxy server. Thread-per-connection. Watches its own source file for changes and drains active connections before exiting for a restart. The supervisor forks a fresh worker that binds the port immediately — no downtime window.

**Components:**

| Component | Purpose |
|-----------|---------|
| `DedupTracker` | Per-session memory dedup, keyed on session UUID from `metadata.user_id` |
| `MemoryRetriever` | Shells out to `crib retrieve` with poll-based timeout (10s) |
| `RequestModifier` | Extracts query text and injects `<memory>` tags into the request body |
| `StreamForwarder` | Forwards to upstream via `Net::HTTP` with SSL, streams response chunks back |
| `EscortProxy` | TCPServer with file watcher, connection tracking, and drain-on-change |

---

## Prerequisites

- Ruby 3.x (stdlib only — no gems)
- [crib](https://github.com/bioneural/crib) (sibling directory or `ESCORT_CRIB` env var)

---

## License

MIT
