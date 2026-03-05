<h1 align="center">
  e s c o r t
  <br>
  <sub>API proxy for per-step memory retrieval</sub>
</h1>

A transparent proxy between Claude Code and the Anthropic API. escort intercepts every `POST /v1/messages` request — one per reasoning step — and injects relevant memories from [crib](https://github.com/bioneural/crib). Every step gets fresh context. A fail-open design means a broken retrieval never blocks an agent.

A single script. Ruby stdlib only. No gems, no API keys.

---

## The problem

Claude Code fires a `UserPromptSubmit` hook when a human types a message. Memory retrieval happens there — crib surfaces relevant context and a hook injects it. But during multi-step agentic loops, an agent reasons and calls tools across dozens of steps without human input. No new hook fires. No new memories surface. Context drifts from what the agent knows.

Claude Code has no per-agent-step hook event. But it supports `ANTHROPIC_BASE_URL` to redirect API calls through a proxy. escort uses this to intercept every reasoning step at the transport layer.

## How it works

escort binds a TCP server on `127.0.0.1:7710`. Claude Code sends API requests to escort instead of directly to the Anthropic API. For every `POST /v1/messages`:

1. **Extract a query** from the conversation (see query extraction below)
2. **Retrieve memories** by piping the query to `crib retrieve`
3. **Deduplicate** — skip if the same memory block was already injected this session
4. **Inject** — append crib's output as a new text content block on the last user message
5. **Forward** the modified request to the Anthropic API and stream the response back

Everything else passes through unmodified. If crib is unavailable, retrieval times out, or anything fails — the request forwards untouched.

### Query extraction

An API request's messages alternate between `user` and `assistant` roles. A user message contains a mix of content: human-typed text, `<system-reminder>` blocks from hooks, `<command-name>` tags, tool result blocks, and other system injections. The query extractor finds meaningful text among this noise using two signals, tried in order.

**Human text.** From the last user message, escort collects text blocks that are NOT entirely wrapped in a single XML element. If a block's entire content matches `<tag ...>...</tag>`, it is a system injection and is skipped. What remains is what the human typed. If found, the last 500 characters become the query.

**Agent reasoning.** During agentic loops the last user message is often just tool results and system-reminder XML — no human text. In this case, escort falls back to text content from the last assistant message. An assistant message contains the agent's reasoning: what it is currently thinking about, what it just decided to do. Using this as the query means memories surface based on what an agent is working on right now, not just what a human originally asked.

This two-signal approach means memories track an agent's evolving focus. When an agent reads a file about authentication, memories about authentication decisions surface. When it shifts to database queries, memories about database choices appear. When a human types a new question, memories relevant to that question appear instead.

### Injection

Memories are injected as a new text content block appended to the last user message's content array:

```json
[
  { "type": "text", "text": "...existing content..." },
  { "type": "text", "text": "<memory retrieved=\"...\" entries=\"...\">...</memory>" }
]
```

crib's output is passed through verbatim — escort does not parse, filter, or wrap it. crib owns the `<memory>` envelope, preamble, formatting, and metadata attributes.

### Per-session dedup

Each Claude Code session carries a unique identifier in a `metadata.user_id` field. escort hashes the entire crib response as one block and tracks which blocks have been injected per session. A memory block surfaced on step 3 is not re-injected on step 7 unless crib returns a different result (new memories added, different relevance scores). Parallel sessions maintain independent dedup sets. Stale sessions are cleaned after one hour.

### Zero-downtime restart

escort runs as a supervisor/worker pair. A supervisor process forks a worker and restarts it on any exit — with exponential backoff (0.5s base, caps at 30s, resets after 5 seconds of healthy runtime). A worker process runs the actual proxy server and watches its own source file for changes. When a code change is detected, the worker drains active connections and exits. The supervisor forks a fresh worker immediately — no downtime window.

Signal propagation: the supervisor traps SIGINT and SIGTERM and forwards them to the worker, so `Ctrl+C` or `kill` cleanly shuts down both processes.

---

## Usage

Start escort and point Claude Code at it:

```sh
bin/escort &
ANTHROPIC_BASE_URL=http://127.0.0.1:7710 claude
```

Logs go to stderr in key=value format, one line per request:

```
escort: messages=5 model=claude-opus-4-6 query=127 crib_ms=342 injected=856 action=injected
escort: messages=7 model=claude-opus-4-6 query=500 crib_ms=291 action=dedup
escort: messages=9 model=claude-opus-4-6 query=500 crib_ms=185 action=no_results
escort: messages=3 model=claude-opus-4-6 query=nil action=passthrough
```

| Field | Meaning |
|-------|---------|
| `messages` | Message count in the request |
| `model` | Model identifier |
| `query` | Query character count, or nil if no query was extracted |
| `crib_ms` | Retrieval latency in milliseconds |
| `injected` | Bytes of memory injected |
| `action` | Outcome: `injected`, `dedup`, `no_results`, or `passthrough` |

To tee logs to a file while keeping terminal output:

```sh
bin/escort 2>&1 | tee /tmp/escort.log &
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
| `ESCORT_CRIB` | sibling `../../crib/bin/crib` | Path to crib binary |
| `ESCORT_TOKEN_BUDGET` | `500` | Max tokens injected per step |
| `CRIB_DB` | crib's default | Path to crib database |

---

## Architecture

```
Claude Code ──POST /v1/messages──▶ localhost:7710 (escort) ──▶ api.anthropic.com
                                         │
                                   crib retrieve
                                   (local SQLite)
```

| Component | Purpose |
|-----------|---------|
| `DedupTracker` | Per-session block-level dedup, keyed on a session UUID from `metadata.user_id` |
| `MemoryRetriever` | Shells out to `crib retrieve` with a poll-based timeout (10s) |
| `RequestModifier` | Two-signal query extraction (human text / agent reasoning) and memory injection |
| `StreamForwarder` | Forwards to upstream via `Net::HTTP` with SSL, streams response chunks back |
| `EscortProxy` | TCPServer with a file watcher, connection tracking, drain-on-change, and lograge-style logging |

---

## Prerequisites

- Ruby 3.x (stdlib only — no gems)
- [crib](https://github.com/bioneural/crib) (sibling directory or `ESCORT_CRIB` env var)

---

## License

MIT
