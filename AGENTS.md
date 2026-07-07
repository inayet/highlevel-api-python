# GoHighLevel Python SDK — Agent Guide

## Package identity
- PyPI: `gohighlevel-api-client` · Python import: `highlevel`
- Source under `src/highlevel/` (setuptools `package-dir = {"" = "src"}`)

## Commands
| Action | Command |
|--------|---------|
| Install deps | `uv sync` |
| Build | `uv build` |
| Lint / format | `uv run ruff check . && uv run ruff format .` |

The only dev dependency is `ruff` (v0.15.20). There are **zero tests** in the repo. No test framework, no CI, no typecheck scripts.

## Architecture
- **Entrypoint**: `HighLevel` class (`src/highlevel/highlevel.py:67`), imported as `from highlevel import HighLevel`
- **All service files are `# @generated` from the OpenAPI spec** — do not edit them directly. 43 service modules live under `src/highlevel/services/`.
- Each service is lazy-initialized as an attribute on `HighLevel` (e.g. `client.contacts`, `client.oauth`).
- Auth options (in priority order): `private_integration_token` > `agency_access_token` > `location_access_token` > OAuth tokens from session storage.
- Session storage: abstract `SessionStorage` base class with `MemorySessionStorage` (default) and `MongoDBSessionStorage` implementations.

## API quirks
- Base URL: `https://services.leadconnectorhq.com`, API version `v3`
- OAuth request bodies use **camelCase keys** (`clientId`, `grantType`, `code`, etc.)
- Token responses from the API are camelCase; the SDK normalizes them to snake_case via `SessionStorage.normalize_session_data()` — ether casing works for storage keys.
- Webhook signature verification: `WEBHOOK_SIGNATURE_PUBLIC_KEY` env var (Ed25519), legacy fallback `WEBHOOK_PUBLIC_KEY` (RSA-SHA256).
- Auto token refresh on 401 via `_handle_response` response hook.

## Generated-code constraint
`src/highlevel/highlevel.py` and every file under `src/highlevel/services/` is stamped `# @generated`. Edits to these files will be overwritten on the next codegen run. Only modify:
- `src/highlevel/__init__.py` (public API surface)
- `src/highlevel/storage/` (session storage implementations)
- `src/highlevel/webhook/`
- `src/highlevel/logging/`
- `src/highlevel/constants/`
- `src/highlevel/error.py`
- `src/highlevel/utils/`
- `pyproject.toml`, `setup.py`

## Python & tooling
- Requires Python >= 3.8
- `pyproject.toml` is the build-config source of truth; `setup.py` is a backward-compat shim
- Only runtime deps: httpx, pydantic, pymongo, cryptography, ty, ruff
