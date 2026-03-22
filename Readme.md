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
