# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
This file should always be kept up to date with changes if needed.

## Project Overview

Python tool to fetch and track fixed electricity price offerings (hintakiinnitykset) from multiple providers. Currently supports **Lumme Energia** and **PKS (Pohjois-Karjalan Sähkö)**. Prices change daily and are stored for historical analysis.

## Commands

- **Show prices:** `uv run -m hintakiinnitys`
- **Store to DB:** `uv run -m hintakiinnitys store`
- **HTML report:** `uv run -m hintakiinnitys report > report.html`
- **Backfill PKS history:** `uv run -m hintakiinnitys backfill-pks`
- **Add dependency:** `uv add <package>`

## Project Structure

Uses `src` layout with `uv_build` backend:

```
src/hintakiinnitys/
  __main__.py   — CLI entry point
  client.py     — Lumme Energia API client (fetch_offers)
  pks_client.py — PKS API client (fetch_pks_offers, fetch_pks_history)
  db.py         — SQLite storage (store_offers, store_pks_history, load_all_offers, init_db)
  models.py     — dataclasses (PriceFixOffer)
  report.py     — HTML report generator with interactive charts and provider toggle
```

## APIs

### Lumme Energia

Open and unauthenticated:

- **Base URL:** `https://api-omalumme-prod.azure-api.net/FA-OmaLumme-Prod`
- `GET /v1/priceFixOffers` — quarterly fixed price offerings (c/kWh, VAT info, date range)

The API only returns the latest snapshot (`Export_date` is usually the previous day). There is no history: query parameters (e.g. `date`, `from`/`to`) are ignored, resource paths like `/v1/priceFixOffers/{id}` return 404, and the OmaLumme web app itself calls this endpoint with no parameters. Days the daily job misses can't be backfilled. For example, 17.–22.11.2026 are missing because of the PKS outage, and the Wayback Machine has no snapshots either.

### PKS (Pohjois-Karjalan Sähkö)

**Currently broken (since Tue 17.11.2026; last stored data 16.11.2026):** PKS removed the public demo site. `live.pks.fi` is now a CNAME to `pkslive.pks.fi` (CloudFront, whose cert doesn't cover `live.pks.fi`, so TLS handshakes fail), which serves a new login-only customer portal (Akamon, Cognito auth + GraphQL). The `/demo` session and `/Api/...` endpoints below no longer exist. Because of this, the PKS fetch is non-fatal: on failure `store` saves Lumme data, prints a warning (a `::warning` annotation in GitHub Actions) and exits successfully. TLS errors are not retried. `backfill-pks` doesn't work until a new data source is found. Historical PKS data stays in `prices.db` and the report.

Was accessed via public demo session on old ASP.NET MVC site:

- **Base URL:** `https://live.pks.fi`
- `GET /demo` — creates demo session (sets auth cookies)
- `GET /Api/Periods/Available` — quarterly periods with metadata (Id, Name, Season, Start/Stop)
- `GET /Api/Prices/Period/Graph/{periodId}/24/{useVat}/{tubeType}/` — daily price history
- `GET /Api/Periods/VatRates` — VAT rate history

Season-to-quarter mapping: winter=Q1, spring=Q2, summer=Q3, autumn=Q4. Filter to `PeriodType==1` for quarterly periods.

Offerings and time periods are dynamic — never hardcode them.

## Conventions

- Python >=3.12, managed with `uv`
- `httpx` for HTTP requests
- Standard `dataclasses` for data models
- SQLite via Python's built-in `sqlite3` — DB file at project root (`prices.db`)
- One row per offer per day per provider — PK is `(fetch_date, price_id, provider)`
- `provider` column distinguishes data sources (`"lumme"` or `"pks"`)
- `prices.db` is tracked in git (committed by CI daily)

## CI/CD

GitHub Actions workflow (`.github/workflows/daily.yml`):
- Scheduled daily at 05:00 UTC (~8 AM Finnish time) + manual trigger. GitHub starts scheduled runs late: in November 2026 they started 7–10 hours after that
- `update` job: fetches prices from both providers, commits `prices.db`, generates the report and uploads it as the `github-pages` artifact. It checks out the branch tip (`ref: ${{ github.ref }}`), not the triggering commit, so a re-run doesn't get its push rejected
- `deploy` job: runs `actions/deploy-pages`, waits 60 s and retries once if that fails. It is a separate job because deploy-pages finds the artifact by name and that listing can lag the upload ([actions/deploy-pages#451](https://github.com/actions/deploy-pages/issues/451)); with upload and deploy in one job it intermittently failed with "No artifacts named github-pages"
- If `deploy` fails, use "Re-run failed jobs". "Re-run all jobs" uploads a second `github-pages` artifact to the same run, and the lookup then fails with "Multiple artifacts named github-pages"
- Requires GitHub Pages source set to "GitHub Actions" in repo settings
