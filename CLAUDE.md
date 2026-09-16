# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

An MCP server for the Trading 212 public API (account, positions, orders, pies,
instrument metadata, history), built on the official MCP Python SDK 2.x.
Python 3.11–3.14; dependencies are managed with `uv` (`uv.lock` is authoritative).

## Commands

```sh
uv sync --frozen                     # install/refresh the venv
uv run --frozen ruff check .
uv run --frozen ruff format --check .
uv run --frozen mypy                 # strict, files = src/trading212_mcp
uv run --frozen pytest --cov --cov-report=term-missing
uv run --frozen pip-audit --local
uv build --no-build-isolation
```

Run a single test: `uv run --frozen pytest tests/test_client.py::test_name -x`.
`filterwarnings = ["error"]` is set — warnings fail the suite; keep it that way.

Run the server: `uv run --frozen trading212-mcp-server` (needs `.env`, see
`.env.example`). Inspector: `uv run --frozen mcp dev src/server.py` (needs Node/`npx`).

Dependency upgrades: `uv lock --upgrade && uv sync --frozen`, then regenerate
`uv export --frozen --no-dev --no-emit-project --output-file requirements.txt`.
Never hand-edit `requirements.txt` or pin transitives directly.

## Architecture

`src/trading212_mcp/` is the installed package; `src/server.py` is a thin
compatibility launcher (exports `mcp = create_server()`) kept for existing MCP
client configs and the Inspector — do not delete it.

Wiring: `server.create_server()` builds a `TradingServer` and calls
`tools.register` / `resources.register` / `prompts.register`, each of which
receives a `current_client` closure rather than a module-global client.
Importing the package opens no connection and reads no credentials; the client
is created lazily inside the lifespan. SSE opens one lifespan per connection, so
the lifespan refcounts (`active_lifespans`) and only closes the client on the
last exit.

Layers:
- `settings.py` — frozen pydantic `Settings`; `from_env()` loads dotenv **only**
  from `Path.cwd()/.env`, once at startup, without overriding real env vars.
- `client.py` — the sole HTTP boundary. Every endpoint is a method; all go
  through `_make_request` → `cache.request` → `_send`. `_normalise_path` rejects
  anything outside `/equity/` and any traversal/encoding trick.
- `cache.py` — Hishel adapter with credential-isolated SQLite storage.
- `models/` — `ApiModel` (`extra="ignore"`, tolerant of additive upstream
  fields) for responses; `RequestModel` (`extra="forbid"`, `allow_inf_nan=False`)
  for outbound bodies. Never flip those configs.
- `tools/`, `resources.py`, `prompts.py` — MCP handlers per domain.
- `registration.py` — `annotations(write=...)` for effect hints and
  `@safe_handler`, which maps `Trading212Error`/`ValidationError` to a generic
  `ResourceError`. Every handler gets both decorators; never let an upstream
  error message or credential reach the client.

Handlers are **synchronous** — the SDK runs them on worker threads. Keep client
methods sync; don't add background tasks or async client code.

### Freshness metadata

`TradingServer` overrides `call_tool`/`read_resource` to set the
`cache.observations` ContextVar, then attaches collected records to `_meta` under
`META_KEY` (`io.github.RohanAnandPandit.trading212/cache`). Adding a new code path
that bypasses `ResponseCache.request` silently drops that metadata.

### Cache invariants (`cache.py`)

These are security properties, not optimizations:
- Credentials are attached only by `httpx` in `client._send`. Hishel receives
  `Headers({})` and a response with a hardcoded allowlist. Authorization and
  cookies must never enter storage.
- The namespace directory is `sha256([base_url, authorization])`, isolating
  accounts, secret rotations, environments, and API versions.
- A per-namespace `filelock` covers the generation read, the request, **body
  consumption** (Hishel streams lazily), and post-mutation invalidation. Do not
  release the lock before `response.read()`.
- Any non-GET bumps the private generation in `state.sqlite` in a `finally`
  block — even on error, since the mutation may have applied. Metadata uses the
  fixed `"metadata"` generation.
- Only 200 GETs are cached (`SuccessfulResponse` filter); `NamespacedStorage.get_entries`
  re-checks TTL so a lowered config doesn't serve older entries.
- SQLite connections use `closing(...)` *and* the transaction context manager.

TTL routing is path-based: `/equity/metadata/` → `metadata_ttl`,
`/equity/history/` (except `/exports`) → `history_ttl`, everything else →
`account_ttl`. A TTL of 0 bypasses the cache entirely.

## Public contracts

Tool names, argument names, resource URIs, and response fields are frozen.
`tests/fixtures/mcp_v1.json` snapshots the discovery surface and
`model_fields_v1.json` the model fields/requiredness; tests compare against them.
Do not rewrite these snapshots to make a test pass. Deprecated aliases
(`fetch_account_info`, `fetch_all_open_positions`, `trading212://account/portfolio`,
etc.) stay. Negative order quantities mean sell — preserve the sign.

## Trading 212 schema drift

`https://docs.trading212.com/_bundle/api.yaml` is authoritative; `docs/api.json`
is only a checked-in snapshot and a green test run does **not** prove the server
is current. Before changing endpoints, models, auth, or pagination, follow
`.agents/skills/trading212-api-sync/SKILL.md`:

```sh
uv run --with PyYAML python .agents/skills/trading212-api-sync/scripts/sync_api_schema.py --check
uv run --with PyYAML python .agents/skills/trading212-api-sync/scripts/sync_api_schema.py --update
```

Review the diff rather than treating regeneration as sufficient. Audit every
changed enum — a new upstream value must not silently reject a valid response.
Do not invent request restrictions the public schema doesn't define.

## Testing rules

`tests/conftest.py` has an autouse `offline` fixture that chdirs to a tmp dir,
clears Trading 212 env vars, and monkeypatches `httpx.HTTPTransport.handle_request`
to fail the test. Never add live API calls or trading mutations to tests; use
mock transports and synthetic credentials. The schema check must never use
account credentials. Cross-process cache tests spawn real subprocesses so locks
and connections aren't inherited.

## Branches

Use `feature/`, `fix/`, `chore/`, `docs/`, or `refactor/` prefixes. Do not use a
`codex/` prefix in this repository.
