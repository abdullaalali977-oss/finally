# FinAlly — AI Trading Workstation

An AI-powered trading workstation that streams live market data, runs a simulated portfolio, and lets an LLM chat assistant analyze positions and execute trades via natural language.

Built entirely by coding agents as a capstone project for an agentic AI coding course.

## Status

In development. The **market data subsystem** is implemented (price simulator, in-memory cache, SSE streaming). Frontend, portfolio/trade engine, AI chat, and Docker packaging are not yet built. See `planning/PLAN.md` for the full spec and `planning/MARKET_DATA_SUMMARY.md` for what's done.

## Run the market data demo

```bash
cd backend
uv sync --extra dev
uv run market_data_demo.py
```

This launches a live terminal dashboard with simulated GBM prices for the default ten tickers (AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX).

## Planned architecture

A single Docker container on port 8000:

- **Frontend**: Next.js (static export) with TypeScript and Tailwind CSS
- **Backend**: FastAPI (Python/uv) serving REST + SSE
- **Database**: SQLite, lazy-initialized on first request
- **AI**: LiteLLM → OpenRouter (Cerebras inference) with structured outputs
- **Market data**: Built-in GBM simulator (default) or Massive/Polygon.io API (optional)

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes | OpenRouter API key for AI chat |
| `MASSIVE_API_KEY` | No | Massive (Polygon.io) key for real market data; omit to use simulator |
| `LLM_MOCK` | No | Set `true` for deterministic mock LLM responses (testing) |

## Project structure

```
finally/
├── backend/    # FastAPI uv project (market data subsystem implemented)
├── frontend/   # Next.js static export (planned)
├── planning/   # Spec, design docs, and agent contracts
├── test/       # Playwright E2E tests (planned)
└── db/         # SQLite volume mount (runtime)
```

## Deployment note

No authentication. Localhost / Docker-on-laptop only — do not expose to a public URL without first adding an auth shim.

## License

See [LICENSE](LICENSE).
