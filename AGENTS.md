# AGENTS.md — GoHighLevel Python SDK

## Quick start

```bash
uv sync                          # editable install from source (creates .venv)
```

Python >= 3.14 required. Package `src/` layout — package name is `highlevel`.

```python
from highlevel import HighLevel
client = HighLevel(client_id="...", client_secret="...")
```

## LSP / editable install

`uv sync` creates an **editable install** via `.venv` — the package `highlevel` resolves to the real `src/highlevel/` directory, not a wheel. This means your LSP (pyright, pylance, jedi) follows imports to the actual `.py` files in this repo, giving you accurate type info, go-to-definition, and autocompletion for all services.

## Scripts

Put your learning/testing code in a **separate project** outside this repo. The SDK is consumed via editable install so LSP resolves to `src/highlevel/` source files.

```bash
mkdir ~/my-ghl-google-ads
cd ~/my-ghl-google-ads
uv init
uv add --editable /path/to/_highlevel-api-python
uv run python google_ads_demo.py
```

## OpenAPI spec repo

The `# @generated` service files (`src/highlevel/services/*`) are generated from the GoHighLevel OpenAPI spec. To regenerate or inspect the spec alongside the code:

```bash
git submodule add <spec-repo-url> spec
```

The published API reference: https://marketplace.gohighlevel.com/docs/

## Architecture

- **Entrypoint**: `src/highlevel/highlevel.py:67` — `HighLevel` class. All 40+ services are instantiated as attributes in `_initialize_services()`.
- **Generated code**: `services/*` directories and `highlevel.py` are `# @generated` from OpenAPI spec. Hand-editing generated service files may be overwritten.
- **Non-generated**: `storage/`, `logging/`, `utils/`, `webhook/`, `constants/`, `error.py`, `__init__.py`
- **All API methods are async** (httpx.AsyncClient). SDK is fully async-first.
- **API base**: `https://services.leadconnectorhq.com` (not documented in README), Version `v3`.

## Auth (3 modes)

1. **Private integration token** — set via `private_integration_token` param; highest priority
2. **Direct access tokens** — `agency_access_token` or `location_access_token` params
3. **OAuth with session storage** — requires `client_id` + `client_secret` + `session_storage`

Token resolution priority in `get_auth_token()`: private_integration_token > agency_access_token > location_access_token > storage lookup (by company/location ID).

Automatic token refresh on 401 responses (`_handle_response` at `highlevel.py:603`).

## OAuth key casing

**v3 API uses camelCase.** The OAuth `get_access_token()` request body uses camelCase keys (`clientId`, `grantType`, `code`). The response also returns camelCase (`accessToken`, `refreshToken`, `expiresIn`). The SDK normalizes via `SessionStorage.normalize_session_data()` which maps camelCase -> snake_case automatically. You can pass either casing to `set_session()`.

## Webhook signature verification

Two schemes:
- **Ed25519** (preferred) — `x-ghl-signature` header + `WEBHOOK_SIGNATURE_PUBLIC_KEY` env var
- **RSA-SHA256** (legacy) — `x-wh-signature` header + `WEBHOOK_PUBLIC_KEY` env var

If neither is configured, webhooks are skipped with a warning. `CLIENT_ID` env var must also be set (matched against `appId` in webhook payload).

## Key dev commands

```bash
uv sync                         # install from source
uv add <pkg>                    # add dependency
uv lock                         # lock after dependency changes
ruff check .                    # lint (if ruff config exists — currently none in pyproject.toml)
```

There are **no tests** in this repo. No CI workflows. No lint/format/typecheck configuration exists beyond `ruff` being listed as a project dependency.

## Notable quirks

- `.gitignore` excludes `uv.lock` — do **not** add it back without asking; this is intentional
- `ruff` and `buff` are listed as runtime `dependencies` in `pyproject.toml` (likely misplaced — should be dev dependencies)
- Logging uses a custom `Logger` class (`src/highlevel/logging/`), not Python's `logging` module. Levels: `none`, `error`, `warn` (default), `info`, `debug`
- Session storage keys are `"{application_id}:{resource_id}"` where `application_id = client_id.split("-")[0]`
- Services follow a consistent pattern: `param_defs -> extract_params -> config dict -> get_auth_token -> build_request -> client.send -> response.json()`
- Every request sets `Version: v3` header and `GHL-SDK-Version: python/{version}` header
