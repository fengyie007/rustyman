# CLAUDE.md

This file provides guidance to Claude Code and other agents when working with this repository.

## Project Overview

Rustyman is a high-performance MITM (Man-In-The-Middle) HTTP/HTTPS proxy written in Rust — the core infrastructure for **feedback loop driven AI-assisted engineering**.

Single Rust crate: lib (`src/lib.rs`) + bin (`src/main.rs`). Edition 2021, Rust 1.75+. The Web UI, REST API and SSE stream are the machine-facing control plane.

### Why Rustyman Exists

Rustyman is not "a faster mitmproxy." It is purpose-built infrastructure that enables AI agents to participate in a closed-loop engineering workflow: **configure → execute → observe → adjust**. The entire architecture serves this feedback loop:

- **REST API** — the control plane for AI agents to programmatically set up mock rules (Map Local), redirect traffic (Map Remote), rewrite headers, and query traffic records, all without human intervention.
- **SSE real-time event stream** (`/api/events`) — the observation channel. The primary consumer is not a human watching a browser, but an AI agent that needs structured, machine-readable feedback on HTTP traffic (request sent? response correct? status code? headers?) to decide its next action.
- **Single binary, zero dependencies** — designed for frictionless integration into CI/CD pipelines and automated test environments. An AI-driven workflow can spin up a proxy instance, run tests, collect feedback, and tear it down with no environment setup.
- **Declarative YAML configuration** — machine-writable, version-controllable, no imperative scripting required.

```
AI Agent
  │
  ├──▶ Configure rules (REST API)     ← Control plane
  ├──▶ Trigger tests                   ← Execution
  ├──◀ SSE real-time traffic events    ← Observation / Feedback
  └──▶ Adjust based on feedback        ← Closed-loop iteration
```

### How This Differs from mitmproxy

mitmproxy is designed for humans — TUI interaction, Python scripting, manual inspection. Rustyman is designed for **AI agents as first-class consumers**: machine-readable APIs, structured event streams, declarative configuration, and stateless deployment. This is the fundamental difference.

## Build Commands

```bash
cargo build                 # debug
cargo build --release
cargo test                  # all unit + integration tests
cargo test test_websocket_proxy        # single test by name
cargo test rules::                     # one module
cargo check
cargo clippy
cargo fmt                   # repo is NOT rustfmt-clean; `cargo fmt --check` fails on untouched files
cargo run -- --port 9090 --web-port 9091
```

`tests/proxy_integration.rs` is the only integration file. It starts a real proxy on a free port in-process with a tempdir CA — no network, no mocks, no external services. `src/**` unit tests use `#[cfg(test)] mod tests` at the bottom of each file.

## Project Architecture

```
src/
├── main.rs              # CLI entry point with clap argument parsing
├── lib.rs               # Library exports
├── cert/mod.rs          # CA certificate generation for MITM (rcgen)
├── config/mod.rs        # YAML configuration parsing (serde_yaml)
├── proxy/
│   ├── mod.rs           # ProxyState + ProxyServer orchestration
│   ├── handler.rs       # HTTP/HTTPS request handling (hyper)
│   └── tls.rs           # TLS acceptor for MITM (tokio-rustls)
├── rules/
│   ├── mod.rs           # Rule engine combining all rules
│   ├── map_remote.rs    # URL redirect rules with regex
│   ├── map_local.rs     # Local file serving rules
│   └── header.rs        # Header rewrite rules with regex
├── traffic/mod.rs       # Traffic recording & broadcast channel
└── web/
    ├── mod.rs           # Web UI server (axum) + route table
    └── api.rs           # REST API endpoints + SSE traffic streaming

static/                  # Web UI frontend files, embedded via include_str!
├── index.html
├── app.js
└── style.css
```

### Where changes go

- `src/proxy/handler.rs` — the entire request path in one ~1100-line file: CONNECT, MITM tunnel, pass-through, HTTP/2 upstream negotiation, WebSocket relay, rule application. Most behavior changes land here.
- `src/proxy/mod.rs` — `ProxyState` (config/rules `RwLock`, traffic store, cancel token) is shared by proxy and web server; both get the same `Arc`.
- `src/web/api.rs` + `src/web/mod.rs` — a new endpoint needs a handler in `api.rs` **and** a `.route()` in `mod.rs`.
- `src/rules/` — one file per rule type; adding one means `rules/mod.rs` + `config/mod.rs` struct + API handler.
- `static/*` is compiled in via `include_str!` (see `web/mod.rs`) — editing the UI requires a rebuild; no hot reload.
- `src/traffic/mod.rs` — in-memory `DashMap`, 10 000-entry cap (hardcoded in `impl Default for TrafficStore`), `broadcast` channel feeding `/api/events` SSE.

## Key Dependencies

- **tokio** - Async runtime
- **hyper** / **hyper-util** - HTTP client/server
- **axum** - Web framework for API
- **tokio-rustls** / **rustls** - TLS implementation
- **rcgen** - X.509 certificate generation
- **regex** - Pattern matching for rules
- **serde** / **serde_yaml** - Configuration serialization
- **clap** - Command-line argument parsing
- **tracing** - Logging and diagnostics

## Core Concepts

### MITM Flow
1. Client connects to proxy
2. For HTTPS (CONNECT method):
   - Proxy generates certificate for target domain signed by CA
   - Establishes TLS with client using generated cert
   - Forwards decrypted traffic to target server
3. Rules (Map Remote, Map Local, Header) are applied
4. Traffic is recorded and broadcast via SSE endpoint (`/api/events`)

### Rule Priority

Fixed order in `handler.rs::process_request`: **map local → map remote → header rewrite** (request, then response).

**Gotcha:** a Map Local match returns early, so header rules never run for mocked responses. Map Local has its own `headers` field for that reason — use it to attach extra response headers (e.g. CORS). Each rule can also suppress the `x-rustyman-map-local` debug header with `expose_debug_header: false` (default `true`). See the Map Local section in `README.md` for a CORS example.

### Configuration
- YAML-based configuration in `config.yaml`
- Runtime reloading via API (`POST /api/rules/reload`)
- CLI flags override config file values

## Testing

```bash
cargo test                    # all tests
cargo test cert::tests        # a module
cargo test test_map_local_custom_headers   # a single test
```

Integration tests live in `tests/proxy_integration.rs`; use `start_proxy_with(|config| { ... })` to inject rules into the fixture.

## Common Development Tasks

### Adding a New Rule Type
1. Create new file in `src/rules/`
2. Implement handler struct with `new()` and `match_url()` methods
3. Add to `RuleEngine` in `src/rules/mod.rs`
4. Add config struct in `src/config/mod.rs`
5. Add API endpoints in `src/web/api.rs` **and** the route in `src/web/mod.rs`

### Modifying Certificate Generation
- Certificate logic is in `src/cert/mod.rs`
- Uses rcgen 0.12 API with `Certificate::from_params()`
- Certificates are cached per-domain in memory

### Adding Web UI Features
- API endpoints in `src/web/api.rs`
- Frontend files in `static/` (embedded at compile time via `include_str!`)

## Gotchas

- **Config fields that are parsed but never used:** `proxy.timeout`, `proxy.max_connections`, `logging.file`, `log_traffic`. Only `proxy.max_websocket_messages` is read at runtime. Don't assume a config knob is enforced — grep for it first.
- **Nothing persists to disk.** API/rule changes mutate the in-memory `Config` only; `Config::save` exists but is never called. `/api/rules/reload` recompiles rules from the *in-memory* config, not from `config.yaml`.
- **`~` expansion uses `HOME` only** (`config/mod.rs::dirs_home`, `main.rs::expand_tilde`). On Windows, where `HOME` is often unset, the default `~/.rustyman/ca.crt` produces a literal `~` directory. Pass `--ca-cert`/`--ca-key` explicitly on Windows.
- Proxy host flag is `-H` (uppercase; `-h` is help). `--web-port 0` disables the Web UI. Missing config file is not an error — it silently falls back to `Config::default()`.
- Upstream TLS verifies against bundled `webpki-roots`, not the OS/browser store. Client-facing certs are generated per host and cached in `CertificateAuthority`.
- No CI runs tests or clippy. `.github/workflows/release.yml` is the only workflow: push a `v*` tag (bump `Cargo.toml` version first) to build 6 targets and publish the release + Homebrew formula.

## Notes

- CA certificate is auto-generated on first run at `~/.rustyman/ca.crt`
- Real-time traffic streaming via SSE endpoint: `GET /api/events`
- Web UI uses EventSource API for real-time traffic updates
- Traffic entries are stored in memory (10 000-entry cap, not configurable)
- The proxy handles both HTTP and HTTPS traffic on the same port

## Real-time Traffic Streaming

The `/api/events` SSE endpoint streams traffic events as JSON:

```bash
# Consume events via curl
curl -N http://localhost:8081/api/events

# Filter with jq
curl -N http://localhost:8081/api/events | grep "^data:" | jq -R 'fromjson?'
```

Event types:
- `request` - New HTTP request received
- `response` - Response received from server
- `completed` - Request/response cycle complete
- `cleared` - Traffic log cleared

Multiple consumers can connect simultaneously (broadcast channel).
