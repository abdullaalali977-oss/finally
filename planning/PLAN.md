# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)
- Up / Profit Green: `#26a69a` (price upticks, profitable positions, heatmap green tiles)
- Down / Loss Red: `#ef5350` (price downticks, losing positions, heatmap red tiles)

Greens and reds are used consistently across the price flash animation, the heatmap, the positions table, and the P&L chart so a single glance reads the same way everywhere.

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving         │
│                      (Next.js export)            │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim        │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file at `db/finally.db`, volume-mounted for persistence
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |

---

## 4. Directory Structure

The tree below is **illustrative**. The top-level layout is fixed (`frontend/`, `backend/`, `planning/`, `test/`, `db/`, `Dockerfile`, `docker-compose.yml`, `.env`). Internal structure within `frontend/` and `backend/` is at agent discretion — for example, `backend/app/market/` already exists for the completed market data subsystem.

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python). Internal layout is up to the backend agent.
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # Volume mount target (SQLite file lives here at runtime)
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── docker-compose.yml        # Single source of truth for running the app (volume + env-file + port)
├── .env                      # Environment variables (gitignored, .env.example committed)
└── .gitignore
```

There are intentionally no per-OS start scripts — `docker compose up -d` / `docker compose down` works identically on macOS, Linux, and Windows (PowerShell or bash). See §11.

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents. Database schema definitions and seed logic live inside the backend (e.g. `backend/app/db/`); the backend lazily initializes the database on first request — creating tables and seeding default data if the SQLite file doesn't exist or is empty.
- **`db/`** at the top level is the runtime volume mount point. The SQLite file (`db/finally.db`) is created here by the backend and persists across container restarts via Docker volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive API key (Polygon.io client) for real market data.
# If not set, the built-in market simulator is used (recommended for most users).
MASSIVE_API_KEY=

# Optional: set LLM_MOCK=true to return deterministic mock LLM responses (for E2E tests).
# Absent = false; do not include in .env.example.
```

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive (Polygon.io REST) for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (for E2E tests)
- The backend reads `.env` from the project root (mounted into the container or read via docker `--env-file`)

Throughout this doc, "Massive" refers to the `massive` Python package, which is a thin client for Polygon.io. The two names are used interchangeably; "Massive" is preferred in code, "Polygon.io" when describing the upstream data provider.

---

## 6. Market Data

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.)
- Runs as an in-process background task — no external dependencies

### Massive (Optional)

- REST polling (not WebSocket) — simpler, works on all tiers
- Polls for the union of all currently-streamed tickers on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses the REST response into the same `PriceUpdate` shape as the simulator

### Streaming Set

The backend streams prices for **`watchlist ∪ tickers held in positions`**. Held positions must always have a live price so portfolio valuation stays correct, even after the user removes a ticker from their watchlist. When a position is fully sold and is no longer in the watchlist, the data source stops streaming it. When a ticker is added to the watchlist or to positions (via a trade), the data source begins streaming it.

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, timestamp, and a monotonic version counter per ticker
- SSE streams read from this cache and push updates to connected clients
- The cache is the single point of truth — no direct coupling between producers and consumers

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API
- The server pushes a price event **only when a ticker's cache version advances** (i.e. on price change). This is correct for both the 500ms simulator and the 15s Massive poller — clients see updates as soon as they happen and never receive redundant events.
- Each SSE event contains ticker, price, previous price, timestamp, and change direction
- Initial connection: the server emits one event per ticker currently in the cache so the frontend has a starting state before any tick fires
- Client handles reconnection automatically (`EventSource` has built-in retry); on reconnect the same initial-state burst replays

### Ticker Validation

Adding a ticker (manually or via the AI) goes through the active data source:

- **Simulator**: any uppercase ticker symbol is accepted. If the ticker is not in the seed-prices table, a default starting price (e.g. $100) and default GBM parameters are used. The ticker simply joins the simulation.
- **Massive**: the ticker is probed against the Polygon.io API. If the API returns no data for that symbol, the add is rejected with `400 {"error": "unknown_ticker"}`.

The same validation applies whether the source of the add is the user's manual input or an LLM-generated `watchlist_changes` action.

---

## 7. Database

### SQLite with Lazy Initialization

The backend checks for the SQLite database on startup (or first request). If the file doesn't exist or tables are missing, it creates the schema and seeds default data. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

All tables include a `user_id` column hardcoded to `"default"`. **The product is single-user; multi-user is explicitly out of scope.** The `user_id` column is preserved purely so a hypothetical future multi-user version wouldn't require a destructive migration — but no agent should write code that filters on, joins on, or treats `user_id` as variable. Treat it as a constant.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL NOT NULL (default: `10000.0`)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT NOT NULL
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT NOT NULL
- `quantity` REAL NOT NULL (fractional shares supported)
- `avg_cost` REAL NOT NULL
- `updated_at` TEXT NOT NULL (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log). Realized P&L is not stored; it is derivable from `trades` + cash balance if needed.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT NOT NULL
- `side` TEXT NOT NULL (`"buy"` or `"sell"`)
- `quantity` REAL NOT NULL (fractional shares supported)
- `price` REAL NOT NULL (the fill price used at execution time)
- `executed_at` TEXT NOT NULL (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded **every 60 seconds** by a background task and immediately after every trade execution. The on-trade snapshot is the meaningful one for visual P&L; the 60s tick captures idle drift.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL NOT NULL
- `recorded_at` TEXT NOT NULL (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT NOT NULL (`"user"` or `"assistant"`)
- `content` TEXT NOT NULL
- `actions` TEXT (JSON; null for user messages — see schema below)
- `created_at` TEXT NOT NULL (ISO timestamp)

The `chat_messages.actions` payload, when present, has the same shape as the chat API response envelope (see §8 → Chat) minus the `message` field:

```json
{
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10, "status": "ok", "fill_price": 192.34},
    {"ticker": "TSLA", "side": "buy", "quantity": 100, "status": "rejected", "reason": "insufficient_cash"}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add", "status": "ok"}
  ]
}
```

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade: `{ticker, quantity, side}` |
| GET | `/api/portfolio/history` | Portfolio value snapshots over time (for P&L chart) |

**Trade execution (`POST /api/portfolio/trade`).** The fill price is taken from `PriceCache.get_price(ticker)` at the moment the request is processed. Behavior:
- `200` on success — returns the executed trade including `fill_price`
- `409 {"error": "no_price"}` if the cache has no entry yet for that ticker (e.g. just-added ticker that hasn't ticked). The frontend should retry after the next SSE tick.
- `400 {"error": "insufficient_cash"}` for buys exceeding cash balance
- `400 {"error": "insufficient_shares"}` for sells exceeding held quantity
- `400 {"error": "unknown_ticker"}` for unknown ticker symbols

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers (symbols only — prices arrive via the SSE stream) |
| POST | `/api/watchlist` | Add a ticker: `{ticker}`. Validated against the active data source (see §6) |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker |

The `GET /api/watchlist` response intentionally does not include prices — the frontend joins the watchlist symbols against its local SSE-driven price state. This keeps the endpoint a trivial DB query and avoids a stale-price race between the cache snapshot and the next SSE event.

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a user message, receive a response envelope (assistant message + per-action results) |

**Request:** `{"message": "..."}`.

**Response envelope** (also persisted to `chat_messages.actions` minus `message`):

```json
{
  "message": "Bought 10 AAPL at $192.34. The TSLA buy was rejected — you only have $50 of cash left.",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10, "status": "ok", "fill_price": 192.34},
    {"ticker": "TSLA", "side": "buy", "quantity": 100, "status": "rejected", "reason": "insufficient_cash"}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add", "status": "ok"}
  ]
}
```

- `trades[].status` is `"ok"` or `"rejected"`. Rejected entries include a `reason` matching the trade-endpoint error codes above.
- `watchlist_changes[].status` is `"ok"` or `"rejected"` with a `reason` (`"unknown_ticker"`, `"already_present"`, `"not_found"`).
- The frontend renders rejected entries inline next to the assistant's message so the user sees what actually happened (see §10).

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | Health check (for Docker/deployment) |

`GET /api/health` returns `200` with:
```json
{"status": "ok", "data_source": "simulator", "price_cache_size": 12}
```
A `price_cache_size` of `0` after the first ~5 seconds of uptime indicates the background market-data task has died and the container should be considered unhealthy.

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

### How It Works

When the user sends a chat message, the backend:

1. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
2. Loads the **last 20 messages** of conversation history from the `chat_messages` table (sliding window — sufficient for short-horizon coherence without bloating the prompt)
3. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
4. Calls the LLM via LiteLLM → OpenRouter using the cerebras-inference skill, requesting JSON output
5. Parses the response into a Pydantic model. If parsing fails, retry up to 2 times with the parse error fed back as a follow-up user message ("Your previous response was not valid JSON: ..."); if it still fails, return a fallback response envelope with `message: "Sorry, I had trouble understanding that. Could you rephrase?"` and empty action arrays.
6. Auto-executes the trades and watchlist changes from the parsed response (see "Action execution" below)
7. Stores the user message and the assistant response (with the action-result envelope) in `chat_messages`
8. Returns the response envelope to the frontend (no token-by-token streaming — Cerebras inference is fast enough that a single loading indicator is sufficient)

### Structured Output Schema

The LLM is instructed to respond with JSON matching this schema:

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (required): The conversational text shown to the user
- `trades` (optional): Array of trades to auto-execute. Each trade goes through the same validation as manual trades (sufficient cash for buys, sufficient shares for sells)
- `watchlist_changes` (optional): Array of watchlist modifications

### Action Execution

Trades and watchlist changes specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

**Atomicity: sequential best-effort.** Each trade and each watchlist change is executed in array order, independently. A failure in one item does not roll back earlier successes or skip later items. Each item gets a per-item `status` (`"ok"` or `"rejected"` with a `reason`) in the response envelope (see §8 → Chat).

**Failed-item feedback.** Because the LLM has already finished generating by the time validation runs, the LLM cannot retroactively explain failures in its `message`. Instead, the **frontend renders rejected items inline** next to the assistant's message — e.g. a red pill below the message: "✕ TSLA buy 100 — insufficient cash". The next time the user sends a message, the rejected items appear in the conversation history fed back to the LLM, so it can self-correct on subsequent turns.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend returns deterministic mock responses instead of calling OpenRouter. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change %, and a sparkline mini-chart accumulated on the frontend from the SSE stream since page load. Sparklines fill in progressively — empty for the first few ticks. With the simulator (default) sparklines are usable within seconds; with Massive (15s polling) sparklines take minutes to fill, which is acceptable.
- **Main chart area** — larger chart for the currently selected ticker, with at minimum price over time. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight (over **equity only — cash is excluded**), colored by P&L using the green/red palette from §2. Cash is shown separately in the header rather than as a heatmap tile, so the visualization stays informative even when most of the portfolio is uninvested.
- **P&L chart** — line chart showing total portfolio value over time, using data from `portfolio_snapshots`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. Buttons use the purple submit color from §2.
- **AI chat panel** — docked **right-side** sidebar (collapsible). Message input, scrolling conversation history, loading indicator while waiting for LLM response. Each assistant message renders the response envelope: the `message` text on top, then any successful trades / watchlist changes as small green confirmation pills, and rejected items as red pills with the reason (e.g. "✕ TSLA buy 100 — insufficient cash"). See §9 for why rejections render in the frontend rather than in the LLM's text.
- **Header** — portfolio total value (updating live), cash balance, connection status indicator (green / yellow / red dot per §2)

### Technical Notes

- Use `EventSource` for SSE connection to `/api/stream/prices`
- **Charting library: Lightweight Charts** (TradingView's open-source canvas-based library). Chosen over Recharts because Recharts is SVG-based and degrades past a few hundred points, which a streaming price chart will hit quickly.
- Price flash effect: on receiving a new price, briefly apply a CSS class with the green or red background from §2, transitioning back to the default over ~500ms
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm install && npm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI serves the static frontend files and all API routes on port 8000.

### Running the App

A single `docker-compose.yml` at the project root is the canonical way to run FinAlly. It captures the volume mount, the env-file, and the port mapping in one place and works identically on macOS, Linux, and Windows (PowerShell or bash):

```bash
docker compose up -d --build   # build image, start container, mount db/, load .env
docker compose down            # stop and remove container; volume persists
```

The `db/` directory in the project root maps to `/app/db` in the container. The backend writes `finally.db` to this path. The volume is **not** removed by `docker compose down`, so portfolio state persists across restarts. To wipe state, delete the contents of `db/` or use `docker compose down -v` if the compose file declares a named volume.

There are intentionally **no per-OS start scripts**. Compose covers all platforms; a wrapper script would only add a layer of indirection.

### Cloud Deployment & Auth

> **The app has no authentication.** This is a deliberate simplification for a local, single-user demo. Anyone who can reach the URL can trade, modify the watchlist, and chat with the LLM (consuming OpenRouter credits).
>
> **Do not deploy this app to a public URL without first adding an auth shim.** The simplest possible shim is a shared password env var checked by a FastAPI dependency on every route. Implementing that shim is out of scope for the core build.
>
> Localhost / Docker-on-laptop is the only supported deployment.

A Terraform configuration for AWS App Runner (or similar) may be provided in a `deploy/` directory as a stretch goal, but only after the auth shim above exists.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: simulator generates valid prices, GBM math is correct, Massive API response parsing works, both implementations conform to the abstract interface
- Portfolio: trade execution logic, P&L calculations, edge cases (selling more than owned, buying with insufficient cash, selling at a loss)
- LLM: structured output parsing handles all valid schemas, graceful handling of malformed responses, trade validation within chat flow
- API routes: correct status codes, response shapes, error handling

**Frontend (React Testing Library or similar)**:
- Component rendering with mock data
- Price flash animation triggers correctly on price changes
- Watchlist CRUD operations
- Portfolio display calculations
- Chat message rendering and loading state

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` by default for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send a message, receive a response, trade execution appears inline; rejected trades render as red pills

SSE reconnection is **not** an E2E scenario — Playwright cannot cleanly kill the underlying TCP connection without CDP plumbing, and the value-to-cost ratio is poor. Reconnect behavior is covered at the unit level instead: a backend test that closes the SSE generator mid-stream, plus a frontend test that simulates `EventSource` `error` → `open` and asserts the connection-status indicator transitions yellow → green.

---

## 13. Resolution Log

A doc review pass against this spec was completed on 2026-05-01. All recommendations from that review have been folded into the relevant sections above. Summary of what changed and where:

| Decision | Lives in |
|---|---|
| Up/down green & red palette specified (`#26a69a` / `#ef5350`) | §2 |
| Directory tree marked illustrative; `scripts/` removed | §4 |
| `LLM_MOCK=false` removed from `.env.example`; Massive/Polygon naming unified | §5 |
| SSE cadence = version-based change detection, push-on-change | §6 |
| Streaming set = `watchlist ∪ positions` | §6 |
| Ticker validation rule per data source | §6 |
| `user_id` treated as a constant; multi-user explicitly out of scope | §7 |
| Unused `created_at` / `added_at` timestamps dropped | §7 |
| Snapshot interval changed from 30s to 60s | §7 |
| `chat_messages.actions` JSON schema defined | §7 |
| Trade fill price = `PriceCache.get_price` at request time; `409 no_price` if missing | §8 |
| `/api/watchlist` GET returns tickers only | §8 |
| Chat response envelope specified (`trades[]` + `watchlist_changes[]` with per-item `status`) | §8 |
| Health check response shape + "background task died" failure mode | §8 |
| Multi-trade execution: sequential best-effort with per-item status | §9 |
| Conversation history window: last 20 messages | §9 |
| Structured output via Pydantic + 2-retry parse loop (no provider-side `response_format`) | §9 |
| Failed-trade feedback rendered by the frontend, not by the LLM | §9, §10 |
| Heatmap weights are equity-only; cash shown in header | §10 |
| Sparkline cold-start behavior documented and accepted | §10 |
| Charting library: Lightweight Charts (canvas) | §10 |
| Chat panel: right-side sidebar | §10 |
| `docker compose up` is the only supported start path; per-OS scripts removed | §4, §11 |
| App is localhost-only; cloud deploy gated on adding an auth shim | §11 |
| SSE-disconnect E2E test dropped; replaced with backend + frontend unit tests | §12 |

### Items consciously deferred

- **`GET /api/prices/{ticker}/recent`** for warm-starting sparklines from server-side history. Not needed for the simulator (sparklines fill within seconds). Worth revisiting only if Massive becomes a primary deploy target.
- **Auth shim** for cloud deploys. A shared-password FastAPI dependency is the obvious shape but is out of scope for the core build (§11).
- **Realized P&L** as a stored or surfaced metric. Implicitly captured by `trades` + cash; revisit if the UI ever shows realized totals.
