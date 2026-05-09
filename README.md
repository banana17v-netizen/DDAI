# Real-time Flight Price Tracker

A service-based system for tracking domestic Vietnam airline prices and aviation fuel costs in near real-time. Instead of crawling HTML, this project uses an **API-first approach**: capture the actual browser request with DevTools, then have a Python worker replay that request on a schedule — no browser automation required.

---

## Architecture

The system runs as four Docker Compose services:

| Service | Description |
|---|---|
| `db-service` | PostgreSQL — stores hot-path price ticks and fuel metrics |
| `scraper-service` | Async Python worker — polls the availability endpoint on a schedule |
| `fuel-worker` | Python worker — fetches Brent crude price + USD/VND rate, estimates Jet A1 cost, writes monthly CSV snapshots |
| `dashboard-service` | Streamlit app — read-only connection to PostgreSQL, displays real-time charts |

Cold-storage data lakes:
- `raw_data/YYYY/MM/DD/vna_raw_YYYYMMDD.jsonl` — full request/response snapshots per scrape cycle
- `fuel_data/YYYY/MM/fuel_metrics_YYYYMM.csv` — monthly fuel metric snapshots

---

## Quick Start

### 1. Configure environment

Copy the example env file and fill in your values:

```powershell
Copy-Item .env.example .env
```

Key variables:

| Variable | Purpose |
|---|---|
| `VNA_API_URL` | Target availability endpoint URL |
| `VNA_API_METHOD` | `GET` or `POST` |
| `VNA_PARSER_MODE` | `best_price_calendar`, `fare_options`, `skyscanner_itineraries`, or `auto` |
| `VNA_HEADERS_TEMPLATE` | JSON object of request headers |
| `VNA_QUERY_TEMPLATE` | URL query params (GET) |
| `VNA_PAYLOAD_TEMPLATE` | Request body (POST) |
| `VNA_BEARER_TOKEN` | Bearer token if required |
| `VNA_COOKIE` | Session cookie if required |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | DB credentials |
| `DASHBOARD_DB_USER` / `DASHBOARD_DB_PASSWORD` | Read-only dashboard credentials |

The worker replaces `{origin}`, `{destination}`, and `{travel_date}` placeholders in the payload or query string on every cycle.

### 2. Configure routes

Edit [`scraper-worker/config/flights.json`](scraper-worker/config/flights.json):

```json
[
  {
    "origin": "HAN",
    "destination": "SGN",
    "days_ahead": [7, 14, 30]
  }
]
```

> **Note:** For session-bound endpoints (e.g. Skyscanner-style), the URL is tied to a specific search session. Keep only the route matching your captured request; refresh `VNA_COOKIE` and possibly `VNA_API_URL` when the session expires.

### 3. Start all services

```powershell
docker compose up --build -d
```

### 4. View logs

```powershell
docker compose logs -f scraper-service
docker compose logs -f fuel-worker
```

### 5. Open dashboard

```
http://localhost:8501
```

---

## Reverse Engineering the Availability API

The scraper does not use a pre-built API client — you capture your own request:

1. Open the airline booking page in incognito mode.
2. Enter a route (e.g. `HAN → SGN`), select a date, and click search.
3. Open **F12 → Network → Fetch/XHR**.
4. Find the request that returns JSON with itineraries, legs, segments, or fare options.
5. Copy the URL, method, headers, and body into your `.env`.

**Parser modes:**

| Mode | When to use |
|---|---|
| `best_price_calendar` | Calendar-style response with best price per date |
| `fare_options` | Fare family / brand response per flight |
| `skyscanner_itineraries` | Skyscanner-style itinerary availability |
| `auto` | Try detailed parser first, fallback to `best_price_calendar` |

---

## Database Schema

### `flight_price_ticks`

Stores one row per fare tick captured each scrape cycle.

| Column | Type | Description |
|---|---|---|
| `timestamp` | `TIMESTAMPTZ` | When the snapshot was taken |
| `flight_number` | `TEXT` | Flight identifier |
| `departure_time` | `TIMESTAMPTZ` | Scheduled departure |
| `fare_class` | `TEXT` | Fare class / brand name |
| `price` | `NUMERIC(12,2)` | Price in VND |

### `fuel_metrics`

Stores one row per fuel worker cycle, including data provenance fields.

| Column | Type | Description |
|---|---|---|
| `timestamp` | `TIMESTAMPTZ` | When the snapshot was taken |
| `brent_price_usd` | `NUMERIC(12,4)` | Brent crude price (USD/barrel) |
| `exchange_rate` | `NUMERIC(12,2)` | USD/VND exchange rate |
| `jet_a1_est_vnd` | `NUMERIC(14,2)` | Estimated Jet A1 price (VND/liter) |
| `han_sgn_fuel_cost` | `NUMERIC(18,2)` | Estimated fuel cost for HAN–SGN (VND) |
| `brent_source` | `TEXT` | Source used for Brent price |
| `exchange_rate_source` | `TEXT` | Source used for exchange rate |
| `brent_price_timestamp` | `TIMESTAMPTZ` | Timestamp of the Brent data point |
| `exchange_rate_timestamp` | `TIMESTAMPTZ` | Timestamp of the exchange rate data point |
| `is_fallback` | `BOOLEAN` | `true` if a fallback source was used |
| `source_note` | `TEXT` | Free-text note about data quality |

---

## Jet A1 Fuel Cost Formula

$$
P_{\text{JetA1}} = \left( \frac{P_{\text{Brent}} \times k_{\text{proxy}} \times R_{\text{USD/VND}}}{158.987} \right) + T_{\text{import}} + T_{\text{env}} + P_{\text{premium}}
$$

$$
C_{\text{HAN-SGN}} = P_{\text{JetA1}} \times V_{\text{liters}}
$$

Default constants from [`fuel-worker/config/pricing.json`](fuel-worker/config/pricing.json):

| Parameter | Value |
|---|---|
| `mops_proxy_multiplier` | 1.0 |
| `barrel_to_liters` | 158.987 L |
| `import_tax_vnd_per_liter` | 0 VND |
| `environment_tax_vnd_per_liter` | 1,000 VND |
| `premium_vnd_per_liter` | 1,800 VND |
| `han_sgn_estimated_liters` | 9,800 L |

- **Brent price** — fetched via `yfinance` symbol `BZ=F`; falls back to Stooq if blocked.
- **USD/VND rate** — fetched from the Vietcombank XML/JSON endpoint (`VCB_EXCHANGE_URL`).

---

## Fuel Worker Schedule

| Variable | Description |
|---|---|
| `FUEL_SCHEDULE_MODE` | `daily`, `hourly`, or `interval` |
| `FUEL_DAILY_HOUR` | Hour to run in daily mode (e.g. `8` = 08:00) |
| `FUEL_TIMEZONE` | Timezone for daily scheduling |
| `FUEL_HOURLY_INTERVAL` | Run every N hours in hourly mode |
| `FUEL_INTERVAL_MINUTES` | Run every N minutes in interval mode |
| `FUEL_RUN_ON_STARTUP` | `true` to capture one sample immediately on container start |

---

## Cold Storage & DVC

### Flight raw data

```powershell
dvc init
dvc add raw_data
git add raw_data.dvc .gitignore .dvc/
dvc remote add -d storage <s3-or-minio-url>
dvc push
```

### Fuel metrics

```powershell
dvc add fuel_data
git add fuel_data.dvc .gitignore
dvc remote add -d storage s3://<bucket>/<path>
dvc push
```

To backtest or replay historical data, run `dvc pull` for the specific version you need.

---

## Operational Notes

- Both workers are wrapped in `try/except` — a failed cycle is logged and the worker continues on the next tick.
- If the API endpoint is session-bound, refresh `VNA_COOKIE` (and possibly `VNA_API_URL`) when the session expires.
- The `is_fallback` and `source_note` columns in `fuel_metrics` let you distinguish live data from fallback values in the dashboard.
- Always comply with the terms of service of any data source you use.

---

## Project Structure

```
flight-price-tracker/
├── docker-compose.yml
├── db/
│   ├── 01-schema.sql                    # Table definitions
│   ├── 02-grants.sh                     # Read-only dashboard user grants
│   └── 03-fuel-metrics-provenance.sql   # Migration for provenance columns
├── scraper-worker/
│   ├── app/                             # Scraper worker source
│   └── config/flights.json              # Route configuration
├── fuel-worker/
│   ├── app/                             # Fuel worker source
│   └── config/pricing.json              # Jet A1 cost parameters
├── dashboard-service/
│   └── app.py                           # Streamlit dashboard
├── raw_data/                            # Cold storage — JSONL snapshots by date
└── fuel_data/                           # Cold storage — CSV snapshots by month
```

---

## Tech Stack

| Component | Technology | Reason |
|---|---|---|
| HTTP client | `httpx` | Async, HTTP/2, easy header/cookie control |
| Database | PostgreSQL 16 | Stable, fast queries, time-series friendly |
| Dashboard | Streamlit + Plotly | Fast to build, interactive charts, read-only |
| Containerization | Docker Compose | One-command startup, isolated services |
| Data versioning | DVC | Version cold-storage snapshots alongside code |
| Fuel pricing | `yfinance` + Vietcombank | Real Brent futures + official VND rate |
