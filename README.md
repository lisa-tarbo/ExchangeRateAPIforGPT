# Exchange Rate API for GPT Actions

This POC project is a lightweight REST API that exposes two focused currency endpoints for use in a GPT Action:

- GBP to ZAR
- USD to ZAR

It is used to demonstrate a custom GPT Action calling an API for OpenAI GPT Actions and for Dimagi Open Chat Studio (OCS) Custom Actions (https://github.com/dimagi/open-chat-studio)

## What This Project Demonstrates

- Design: wraps a third-party exchange-rate provider behind an internal API.
- OpenAPI 3.1 specification for GPT Action compatibility. See [OpenAPI](#openapi-contract)
- Security: protected endpoints with HTTP Bearer token. See [Auth Design](#authentication-design)
- Easy deployment: hosted verification path on [PythonAnywhere](./tests/README.md#4-api-client-test-with-pythonanywhere).
- Testing layers: [see here](./tests/README.md)
- Pre-commit hooks: Ruff (lint and format), `detect-secrets` (secret scanning), YAML validation, debug statement checks, end-of-file and trailing-whitespace fixers, large-file check. See `.pre-commit-config.yaml`.
- Keys are loaded from environment variables or a local `.env` file (git-ignored). See [Local Setup](#local-setup-windows--vs-code)

## Architecture Overview

1. A Flask web-based API receives client requests.
2. For protected endpoints, the API first checks that `EXCHANGE_RATE_API_KEY` is configured (500 if not), then validates the Bearer token (401 if missing or wrong).
3. The service calls ExchangeRate-API as an upstream provider.
4. The service returns the ZAR exchange rate value as plain text.

### Why plain-text responses?

For this POC, returning a plain text value keeps the API simple for GPT Actions and quick manual tests.

## API Surface

- `GET /` health check endpoint.
- `GET /GBPRate` returns latest GBP to ZAR exchange rate (requires Bearer token).
- `GET /USDRate` returns latest USD to ZAR exchange rate (requires Bearer token).

## Authentication Design

The `/GBPRate` and `/USDRate` endpoints use HTTP Bearer token authentication.

- This is a deliberate security design for testing how GPT Actions and OCS custom actions (as [Authentication Providers](https://docs.openchatstudio.com/concepts/team/authentication_providers/)) handle security
- GPT Actions can work with this pattern when configured via OpenAPI.
- `SERVICE_AUTH_KEY` is loaded from environment variables, not hardcoded secrets.

Request example:

```http
GET /GBPRate
Authorization: Bearer <SERVICE_AUTH_KEY>
```

Auth failure response:

```http
401 Unauthorized
The authorization header is missing a Bearer token key or doesn't match the required key.
```

### Error responses

All responses are plain text. The protected endpoints (`/GBPRate`, `/USDRate`) can return:

| Status | When | Message |
|---|---|---|
| 200 | Success | The ZAR rate, e.g. `23.87` |
| 401 | `Authorization` header is missing or the Bearer token doesn't match `SERVICE_AUTH_KEY` | The authorization header is missing a Bearer token key or doesn't match the required key. |
| 500 | `EXCHANGE_RATE_API_KEY` is not set | Missing EXCHANGE_RATE_API_KEY environment variable. |
| 500 | `SERVICE_AUTH_KEY` is not set | Missing SERVICE_AUTH_KEY environment variable. |
| 502 | Upstream provider unreachable, or returned invalid JSON or a payload without `conversion_rates` / `ZAR` | `Failed to reach the exchange rate API: <detail>` or `Unexpected response from the exchange rate API: <detail>` |
| 504 | Upstream provider didn't respond within 10 seconds | The exchange rate API request timed out. Please try again later. |

The `EXCHANGE_RATE_API_KEY` check runs before the Bearer token check, so a missing upstream key returns 500 even if the caller's token is wrong.

## OpenAPI Contract

`openapi.yaml` defines the API using OpenAPI 3.1.0, including:

- server URL
- endpoint operations
- response formats
- Bearer token security scheme (`type: http`, `scheme: bearer`)

This makes the API easier to integrate with GPT Actions, OCS custom actions and tooling such as Postman.

> **Note:** the `servers` URL in `openapi.yaml` is hardcoded to the author's PythonAnywhere deployment (`https://lisajwa.pythonanywhere.com`). If you deploy your own copy, change it to your own host before using the file as a GPT Action schema.

## Prerequisites

- Python 3.10+
- Flask, requests and python-dotenv (used by `plugin.py`)
- pytest (to run the automated tests)
- ipykernel and the VS Code Jupyter extension (only to run the notebooks in `tests/`)
- ExchangeRate-API key: https://v6.exchangerate-api.com
- GPT Actions account if you want to demonstrate this: https://chatgpt.com/gpts
- Optional PythonAnywhere account for hosted testing: https://www.pythonanywhere.com

## Local Setup (Windows + VS Code)

```console
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip

pip install pre-commit
pre-commit install

pip install flask requests python-dotenv pytest ipykernel

$env:EXCHANGE_RATE_API_KEY = "your_exchange_rate_api_key" # pragma: allowlist secret
$env:SERVICE_AUTH_KEY = "your_service_auth_key" # pragma: allowlist secret
```

Then, with the virtual environment active and from the repository root:

```console
# Run the API locally at http://127.0.0.1:5000/
flask --app plugin run

# Run the automated tests
python -m pytest -q
```

See the [testing notes](./tests/README.md) for the other ways to test.

### Using a `.env` file instead

`plugin.py` loads a `.env` file from the project root at startup (via `python-dotenv`). Create one with:

```text
EXCHANGE_RATE_API_KEY=your_exchange_rate_api_key
SERVICE_AUTH_KEY=your_service_auth_key
```

`.env` is listed in `.gitignore`, so it is not committed. Variables already set in the environment (such as the `$env:` lines above) take precedence over values in `.env`.

## Testing Strategy

See [testing notes here](./tests/README.md)

## Ideas for improvements to POC

- Return structured JSON responses with metadata and timestamp.
- Add retry handling to upstream provider calls (a 10-second timeout is already in place and returns a 504 on expiry).
- Add logging and monitoring for failure visibility.
- Move from single shared token to a stronger auth strategy when scaling.
