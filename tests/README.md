# Testing the Exchange Rate API for GPT Actions

Multiple ways to test:
1. [Automated unit tests with pytest](#1-automated-unit-tests-with-pytest)
2. Directly to the 3rd party [Exchange Rate API using Jupyter Notebook](#2-provider-integration-test-jupyter-notebook)
3. Against locally running [Flask web app API](#3-flask-local-web-app-api-test-with-postman) using Postman
4. Against cloud hosted [web app API running on PythonAnywhere](#4-api-client-test-with-pythonanywhere)
5. Testing a GPT Action using the OpenAPI schema. See [GPT Action setup notes](#5-gpt-action-setup)


## 1) Automated unit tests with pytest

Run `python -m pytest -q` from the repository root. Command, gotchas and coverage are in [AGENTS.md](../AGENTS.md#testing).

The suite uses Flask's test client and mocks upstream provider calls, so it runs quickly with no live API calls.

## 2) Provider integration test Jupyter notebook

[`tests/test-exchange-rate-api.ipynb`](./test-exchange-rate-api.ipynb) calls ExchangeRate-API directly.

Required environment variable:

```console
setx EXCHANGE_RATE_API_KEY "your_exchange_rate_api_key"
```

If you use `setx`, restart VS Code before running notebook cells.

NOTE: see [AGENTS.md gotchas](../AGENTS.md#gotchas) on how notebooks read keys.

## 3) Flask local web app API test with Postman

Run the API locally with `flask --app plugin run --debug` and the keys set. See [AGENTS.md](../AGENTS.md#useful-commands).

Health check URL against locally running Flask web-based API:
```text
http://127.0.0.1:5000/
```

URLs for testing with Postman

- `http://127.0.0.1:5000/`
- `http://127.0.0.1:5000/GBPRate`
- `http://127.0.0.1:5000/USDRate`

Include `Authorization: Bearer <SERVICE_AUTH_KEY>` for protected endpoints.

## 4) API client test with PythonAnywhere

1. Login to your PythonAnywhere account
2. Follow these instructions for Flask on Python: https://help.pythonanywhere.com/pages/Flask

3. Upload these files to the "mysite" folder:
    - `plugin.py`
    - `.env` (holds `EXCHANGE_RATE_API_KEY` and `SERVICE_AUTH_KEY`)

    `openapi.yaml` is not uploaded. See [AGENTS.md](../AGENTS.md#runtime-configuration).

4. In a PythonAnywhere Bash console, install the required packages:

    ```console
    pip install flask requests python-dotenv
    ```

### 4.1) Test with Postman

Use Postman to call:

- `https://<your-pythonanywhere-domain>/`
- `https://<your-pythonanywhere-domain>/GBPRate`
- `https://<your-pythonanywhere-domain>/USDRate`

Include `Authorization: Bearer <SERVICE_AUTH_KEY>` for these protected endpoints.

### 4.2) Test with Jupyter Notebook
[`tests/test-exchange-rate-plugin.ipynb`](./test-exchange-rate-plugin.ipynb)

- Set `API_BASE_URL` and `SERVICE_AUTH_KEY` as described in [AGENTS.md gotchas](../AGENTS.md#gotchas).

--------------------------------------------------------------------------------------------

## 5) GPT Action Setup

Reference: https://help.openai.com/en/articles/8770868-gpt-builder

Use `openapi.yaml` as the action schema and configure Bearer authentication with `SERVICE_AUTH_KEY`. Change the hardcoded `servers` URL if you deployed your own copy (see [AGENTS.md](../AGENTS.md#gotchas)).

### POC Prompt/Instructions for GPT Action
You are a currency insights assistant focused on GBP/ZAR and USD/ZAR.

Primary behavior:

You are a GBP/ZAR and USD/ZAR currency insights assistant.

For current rates, always call the Actions API first.
Show the current rate(s) with retrieval date/time.
1) Explain movement in clear, non-technical language.
2) Use credible web sources for past month/year context and 1-week outlook.
3) Clearly label what comes from:
- Actions API
- Web research
- Your interpretation

If API fails:

1) Say the API is unavailable.
2) Provide best-effort web-based context.
3) Note reduced confidence.

Style:

Keep responses concise and executive-friendly.
Do not give financial advice.
Ask one clarifying question only if user intent is unclear.

Default response structure:

1) Current snapshot
2) Trend context (1 month, 1 year)
3) 1-week outlook
4) Key risks/watchouts
5) Source note (API timestamp + web sources used)
