# Kalshi Copybot (Go)

A minimal copy-trading bot for Kalshi written in Go. It polls fills from one or more leader accounts and mirrors their trades on your account with configurable sizing, slippage and filters.

Status: scaffold with working structure and clear TODOs to wire real Kalshi REST auth and endpoints. Designed to mirror the Python client semantics in this repo.

## Features
- Polls leader fills since last sync; mirrors new fills once.
- Scalable sizing via `COPY_FRACTION`, `MIN_COST_USD`, `MAX_COST_USD`.
- Price fences: max slippage in cents; optional post-only (maker) or IOC (taker).
- Idempotent copy: stores mirrored leader fill IDs in a local JSON store.
- Graceful loop with interval and jitter.

## Configure
Create a `.env` alongside the binary or set env vars:

```
# Kalshi creds
KALSHI_API_KEY_ID=
KALSHI_PRIVATE_KEY_PATH=
KALSHI_USE_DEMO=false
KALSHI_BASE_URL=https://api.elections.kalshi.com/trade-api/v2

# Mode
DRY_RUN=true                  # Enable to simulate orders without placing them

# Leaders (comma separated account IDs or usernames)
LEADER_IDS=leader_a,leader_b

# Copy sizing
COPY_FRACTION=0.25          # 25% of leader size
MIN_COST_USD=1.00
MAX_COST_USD=20.00

# Execution
TIME_IN_FORCE=ioc           # ioc | gtc
MAX_SLIPPAGE_CENTS=2        # allowed price move vs leader fill
POST_ONLY=false
REDUCE_ONLY_SELLS=true      # prevent unintended shorts; sells only if we have holdings
MIN_EDGE_CENTS=0            # require (leader - exec) >= MIN_EDGE_CENTS (buys); (exec - leader) for sells
TAKER_FEE_CENTS=0           # subtract fee from edge when checking
MAX_EXPOSURE_PER_MARKET_USD=0   # cap total exposure per market; 0=off
EXPOSURE_PRICE_BASIS=ask        # ask|mid|bid for exposure calculation

# Loop
POLL_INTERVAL_SEC=15
POLL_JITTER_PCT=0.2          # +/- 20% jitter on loop sleep
SUMMARY_INTERVAL_SEC=60       # seconds between summary logs

# Store
STATE_PATH=copybot_state.json
LOG_LEVEL=INFO
TRADE_LOG_FILE=trades.jsonl   # JSONL trade/action log
POSITIONS_CACHE_SEC=10        # cache horizon for positions if integrated later
METRICS_ADDR=:9090            # Prometheus /metrics and /healthz
METRICS_ENABLE_LABELED=true   # enable per-leader labeled metrics
METRICS_ENABLE_PPROF=false    # expose /debug/pprof endpoints
MAX_ORDERS_PER_TICK=0         # 0 = unlimited; cap orders mirrored per loop
READY_AFTER_TICKS=1           # readiness gate: require N ticks
READY_REQUIRE_MIRRORED=false  # if true, require at least one mirrored fill
READY_INITIAL_DELAY_MS=0      # optional initial delay before reporting ready
READY_WAIT_FOR_METRICS=false  # wait until metrics server is reachable before ready
METRICS_LEADER_ALLOWLIST=     # comma-separated allowlist for labeled per-leader metrics

# Optional: External leader-fills HTTP feed
# URL can contain placeholders {leader}, {since}, {limit} or accept them as query params
# Example: https://my-feed.example.com/fills?leader={leader}&since={since}&limit={limit}
LEADER_FILLS_URL=
# Set as full Authorization header value if required, e.g., "Bearer abc123"
LEADER_FILLS_AUTH=
# Additional headers, either JSON or pairs:
#  - JSON: {"X-API-Key":"abc","X-Tenant":"xyz"}
#  - Pairs: X-API-Key=abc;X-Tenant=xyz
LEADER_FILLS_HEADERS=
```

## Build & Run

```
cd Kalshi/Copybot_GO
go mod tidy
go build -o kalshi-copybot ./cmd/copybot
./kalshi-copybot
```

### Helper scripts
From `Kalshi/Copybot_GO/scripts`:

```
cd Kalshi/Copybot_GO/scripts
# Windows
./run.ps1 -Build

# macOS/Linux
chmod +x run.sh
./run.sh -b
# logs will be written to Kalshi/Copybot_GO/logs/run_*.log
```

## Notes
- This scaffold includes a minimal Kalshi client with TODOs for RSA-PSS auth (mirror `Kalshi/shared/kalshi_client.py`).
- API shapes follow the Python client in this repo: `get_fills`, `get_market`, `create_order`.
- For production: add retries/backoff, per-endpoint rate limits, and better error telemetry.

### Dry-run mode
- When `DRY_RUN=true`, no credentials are required. The bot logs intended orders and marks leader fills as mirrored with `DRYRUN` so they are not re-sent.
- Set `DRY_RUN=false` to enable live order placement (requires `KALSHI_API_KEY_ID` and `KALSHI_PRIVATE_KEY_PATH`).

### Slippage protection
- The bot fetches current quotes for the ticker (best bid/ask for yes/no) and compares against the leader’s fill price.
- If the current executable price moved beyond `MAX_SLIPPAGE_CENTS`, the fill is skipped and recorded as `SKIPPED_SLIPPAGE`.
- If quote fields are not present in the market payload, the check is skipped gracefully.

### Maker-only (post-only) price fences
- When `POST_ONLY=true`, the bot adjusts the order price to avoid crossing the spread:
  - Buy: price is reduced to below best ask (ask-1) if needed.
  - Sell: price is increased to above best bid (bid+1) if needed.
- If the post-only adjustment moves price beyond `MAX_SLIPPAGE_CENTS` from the leader’s fill, the order is skipped.


## Leader Source (Postgres)
By default, the bot can read leader fills from your existing dashboard database (e.g., kalshi_lmsr_trades). Configure:

- POSTGRES_DSN=postgres://user:pass@host:5432/dbname?sslmode=disable
- LEADER_SOURCE_TABLE=kalshi_lmsr_trades
- LEADER_SOURCE_TAG_COL=strategy
- LEADER_IDS=leader_a,leader_b   # values from the tag column

The bot will select rows ordered by created_at and build synthetic fills (ticker, side, action, count, price_cents).

Example: Tag a leader strategy in your sync process or via SQL:

```
-- mark specific trades as leader-driven (example)
UPDATE kalshi_lmsr_trades SET strategy='leader_anna'
WHERE ticker LIKE 'KXINXU-%' AND created_at::date = current_date;

-- index to accelerate polling
CREATE INDEX IF NOT EXISTS idx_kalshi_lmsr_trades_tag_created
ON kalshi_lmsr_trades(strategy, created_at);
```

If you have a dedicated leader-fills feed, implement GetLeaderFills in internal/api/kalshi.go accordingly.
Alternatively, provide `LEADER_FILLS_URL` and optional `LEADER_FILLS_AUTH` to ingest fills from an HTTP endpoint returning an array of:

```
[
  {"fill_id":"id","ticker":"KX-...","side":"yes","action":"buy","count":5,"price_cents":42,"ts":1710000000}
]
```

### HTTP reliability
- Configure request retries and backoff:

```
HTTP_RETRY=3
HTTP_BACKOFF_MS=300
```

### Action log and metrics
- Every mirrored/attempted action is appended to `TRADE_LOG_FILE` as JSONL with fields:
  `ts,event,leader,fill_id,ticker,side,action,count,scaled_count,price_cents,status`.
- Periodic summary (every ~60s) to stdout: `ticks, placed, filled, skipped_slippage, dry_run, errors`.
- Prometheus endpoint on `METRICS_ADDR`:
  - `/metrics` for scraping (counters: ticks, orders_placed, orders_filled, skipped_slippage, dryrun, errors; gauge: up)
  - `/healthz` returns 200 OK when running
  - `/readyz` returns 200 OK after first successful tick
  - `/debug/pprof/*` if `METRICS_ENABLE_PPROF=true`

Graceful shutdown: Ctrl+C or SIGTERM stops the loop cleanly; readiness goes false and `copybot_up` drops to 0.

Metrics details
- Histograms:
  - `copybot_order_latency_seconds`: order request→response latency
  - `copybot_feed_latency_seconds`: time from leader fill ts to processing
  - `copybot_leader_feed_decode_seconds`: time to JSON-decode leader feed
- `copybot_leader_feed_http_latency_seconds`: leader HTTP feed request latency
- `copybot_unmirrored_queue`: pending fills not yet mirrored (snapshot per cycle)
- Labeled metrics can be disabled (`METRICS_ENABLE_LABELED=false`) or restricted via `METRICS_LEADER_ALLOWLIST`.
# Liquidity filters
LIQUIDITY_REQUIRE_QUOTES=true     # skip if we cannot fetch best bid/ask quotes
LIQUIDITY_MAX_SPREAD_CENTS=10     # skip if (ask - bid) exceeds threshold for the traded side
LIQUIDITY_MIN_BID_CENTS=1         # skip if bid is too low (near 0)
LIQUIDITY_MAX_ASK_CENTS=99        # skip if ask is too high (near 99)
LIQUIDITY_MIN_BID_SIZE=0          # skip if best bid size < N (0=off)
LIQUIDITY_MIN_ASK_SIZE=0          # skip if best ask size < N (0=off)
QUOTE_FRESHNESS_SEC=0             # require quotes newer than N seconds (0=off)
QUOTE_REQUIRE_FRESH=false         # when freshness is set, require timestamped quotes; skip if ts missing






#Kalshi Sports Bot (Go) — User Guide

Overview
- A lightweight Go bot that trades Kalshi sports markets using fair probabilities from bookmaker odds and ML predictions produced by the Python bot. It supports dry-run, live trading, basic risk controls, and Prometheus metrics.

What’s Included (zip)
- kalshi-sports.exe — Windows x64 binary
- .env.example — configuration template
- README.md — quick intro
- USER_GUIDE.md — this guide
- mappings.example.yml, kalshi_tickers.example.yml — mapping templates
- sportsbot.db — SQLite database (predictions/signals/bets); pre-populated via importer

Prerequisites
- Windows x64
- Optional: The Odds API key (for bookmaker odds)
- For live: Kalshi API Key ID + RSA private key (PEM)

Quick Start (Windows)
1) Unzip kalshi_sports_win64.zip
2) Copy .env.example to .env and edit:
   - DRY_RUN=true to start safely
   - ODDS_API_KEY=<your-odds-api-key>
   - KALSHI_API_KEY_ID=<your-kalshi-key-id> (for live)
   - KALSHI_PRIVATE_KEY_PATH=<path-to-your-private-key.pem> (for live)
   - MAPPINGS_FILE=./mappings.yml (create from template)
3) Ensure mappings.yml has event→ticker entries (see “Mappings”)
4) Double-click kalshi-sports.exe or run from PowerShell
5) Metrics: http://localhost:9702/metrics (health: /healthz, readiness: /readyz)

Configuration (.env)
- DRY_RUN: true|false; safe mode that logs but does not place live orders
- LOG_LEVEL: INFO|DEBUG|…
- ODDS_API_KEY / ODDS_API_BASE: The Odds API key and base URL
- LEAGUES: comma-separated (e.g., NBA,NFL); first league drives odds ingest
- KALSHI_BASE: Kalshi API base URL
- KALSHI_API_KEY_ID, KALSHI_PRIVATE_KEY_PATH: required when DRY_RUN=false
- MIN_EDGE_BPS, MIN_CONF: edge/confidence thresholds for trading
- MAX_SLIPPAGE_BPS: max slippage vs Kalshi ask (bps)
- TIME_IN_FORCE: ioc|gtc; POST_ONLY: true|false
- BANKROLL_USD: baseline bankroll for metrics
- METRICS_TICKER_ALLOWLIST: comma-separated tickers to expose per-ticker exposure
- MAPPINGS_FILE: path to YAML mapping generated by Python tools
- LOOP_INTERVAL_SEC, JITTER_PCT: loop cadence and jitter

Predictions (from Python)
- Best accuracy comes from ML predictions generated by the Python bot.
- Three ways to populate the Go DB (sportsbot.db → predictions table):
  1) Python scheduled sync:
     - In Kalshi_Sports/.env set SYNC_PREDICTIONS_TO_GO=true and GO_SQLITE_PATH=../Kalshi_Sports_Go/sportsbot.db
     - Run the Python bot (loop or once). It writes latest predictions into the Go DB.
  2) Python one-shot tool:
     - python -m src.tools.write_predictions_to_go_sqlite --src ./src/sportsbot.db --dst ../Kalshi_Sports_Go/sportsbot.db
  3) Go importer:
     - In Kalshi_Sports_Go: go run ./cmd/import_predictions -src ../Kalshi_Sports/src/sportsbot.db -dst sportsbot.db

Mappings
- The bot needs a mapping from (league, home, away, date) → Kalshi ticker (and optional Polymarket market).
- Generate candidates in Python:
  - python -m src.map_events -o mappings.yml (odds-based suggestions)
  - python -m src.auto_map -o mappings.yml --league NBA (live sources)
  - Optional: fill from curated kalshi_tickers.yml via python -m src.match_kalshi
- Copy mappings.yml next to kalshi-sports.exe and set MAPPINGS_FILE=./mappings.yml

Running
- Dry-run (default): verifies data path, logs decisions, updates SQLite metrics
- Live: set DRY_RUN=false and Kalshi creds. Orders use IOC/POST_ONLY per .env. The bot chooses YES/NO by comparing fair probability vs Kalshi asks and trades when thresholds pass.
- Metrics endpoints:
  - /metrics: Prometheus
  - /healthz: 200 when process is up
  - /readyz: 200 after first successful Kalshi quotes fetch on a mapped ticker

Settlement
- Use the Go settle tool to settle bets and compute realized PnL:
  - go run ./cmd/settle -db sportsbot.db -odds_key <ODDS_API_KEY> -leagues NBA,NFL -days 3
- Assumes YES=home, NO=away; rely on consistent event keys in mappings and signals.

Troubleshooting
- Readiness stays 503:
  - Check mappings.yml covers current events; ensure ODDS and Kalshi are reachable.
  - Verify Kalshi creds when DRY_RUN=false.
- HTTP 429 from Odds API:
  - Reduce LOOP_INTERVAL_SEC; upgrade plan; verify Retry-After honored by upstream.
- No trades placed:
  - Inspect /metrics for edge distributions; adjust MIN_EDGE_BPS and MIN_CONF; verify predictions are synced.
- Exec errors placing orders:
  - Verify TIME_IN_FORCE and POST_ONLY compatibility; ensure price within [1,99].

Safety
- Start in DRY_RUN. Enable live only after verifying mappings and predictions.
- Keep private keys secure; PEM file path must be accessible to the bot.

Upgrading
- Replace kalshi-sports.exe and .env as needed. Database schema upgrades are applied automatically (non-destructive). Re-run import_predictions if you ship a new Python DB.



