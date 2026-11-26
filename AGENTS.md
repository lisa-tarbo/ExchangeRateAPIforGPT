# Exchange Rate API

Flask POC API (`plugin.py`) exposing GBP to ZAR and USD to ZAR rates for GPT Actions and Open Chat Studio custom actions. Contract: `openapi.yaml`.

## Useful commands

Run from the repository root with the virtual environment active.

- `flask --app plugin run`: run the API locally at http://127.0.0.1:5000/
- `flask --app plugin run --debug`: same, with the reloader and debugger
- `python -m pytest -q`: run the automated tests
- `pre-commit run --all-files`: run all pre-commit hooks

Notes:

- Always use `python -m pytest`, not bare `pytest`. `python -m` adds the current directory to the import path so tests can import `plugin.py`. Bare `pytest` fails with a collection error.
- Pre-commit hooks (see `.pre-commit-config.yaml`): Ruff lint and format, `detect-secrets`, YAML validation, debug statement checks, end-of-file and trailing-whitespace fixers, large-file check.
- Placeholder keys in docs and snippets need a `# pragma: allowlist secret` comment, or `detect-secrets` blocks the commit.

## Runtime configuration

- `EXCHANGE_RATE_API_KEY`: key for the upstream ExchangeRate-API (https://v6.exchangerate-api.com)
- `SERVICE_AUTH_KEY`: Bearer token callers must send to `/GBPRate` and `/USDRate`
- `plugin.py` also loads a `.env` file from the project root at startup (`python-dotenv`). Variables already set in the environment take precedence over `.env`.
- `.env` is git-ignored.
- On PythonAnywhere, upload `plugin.py` and `.env` only. `openapi.yaml` is never served by Flask, so it is not uploaded. It is pasted into the GPT builder.

## Testing

- `tests/test_plugin.py` uses Flask's test client and mocks all upstream provider calls. Tests must not make live API calls.
- Covered: health check, GBP and USD success, auth failures (missing header or wrong token, 401), missing `EXCHANGE_RATE_API_KEY` or `SERVICE_AUTH_KEY` (500), upstream timeout (504), upstream request failure, missing `conversion_rates` and invalid JSON (502).
- Notebooks in `tests/` are manual checks against the real provider and the hosted API. They are not part of the pytest suite.
- See [tests/README.md](tests/README.md) for the manual test routes (Postman, PythonAnywhere, GPT Action).

## Gotchas

- The notebooks read keys from the OS environment only (`os.getenv`). They do not load `.env`, so a key that exists only in `.env` is not found.
- After `setx`, restart VS Code before running notebook cells.
- `openapi.yaml` `servers` URL is hardcoded to `https://lisajwa.pythonanywhere.com`. Change it when deploying a different copy.
- `API_BASE_URL` in `tests/test-exchange-rate-plugin.ipynb` is hardcoded to the same host.

## Security boundaries

- Never hardcode secrets. Keys come from environment variables or the git-ignored `.env`.
- Never commit `.env`, and never print or log key values.
- Protected endpoints use a single shared Bearer token. This is a deliberate POC design for testing GPT Action and OCS auth, not a pattern for production. Do not weaken or remove the check.
