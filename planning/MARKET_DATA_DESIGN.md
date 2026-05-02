# Market Data Backend — Detailed Design

Implementation-ready design for the FinAlly market data subsystem. Covers the
unified `MarketDataSource` interface, the in-memory `PriceCache`, the GBM
simulator (default), the Massive (Polygon.io) REST poller, the SSE streaming
endpoint, and FastAPI lifecycle integration.

All code in this document lives under `backend/app/market/`.

---

## Table of Contents

1. [Goals & Architecture](#1-goals--architecture)
2. [File Layout](#2-file-layout)
3. [Data Model — `models.py`](#3-data-model--modelspy)
4. [Price Cache — `cache.py`](#4-price-cache--cachepy)
5. [Abstract Interface — `interface.py`](#5-abstract-interface--interfacepy)
6. [Seed Data & Ticker Parameters — `seed_prices.py`](#6-seed-data--ticker-parameters--seed_pricespy)
7. [GBM Simulator — `simulator.py`](#7-gbm-simulator--simulatorpy)
8. [Massive API Client — `massive_client.py`](#8-massive-api-client--massive_clientpy)
9. [Factory — `factory.py`](#9-factory--factorypy)
10. [SSE Streaming Endpoint — `stream.py`](#10-sse-streaming-endpoint--streampy)
11. [Package Exports — `__init__.py`](#11-package-exports--initpy)
12. [FastAPI Lifecycle Integration](#12-fastapi-lifecycle-integration)
13. [Watchlist Coordination](#13-watchlist-coordination)
14. [Testing Strategy](#14-testing-strategy)
15. [Error Handling & Edge Cases](#15-error-handling--edge-cases)
16. [Configuration Summary](#16-configuration-summary)
17. [Dependencies (`pyproject.toml`)](#17-dependencies-pyprojecttoml)

---

## 1. Goals & Architecture

### Goals

- **One interface, two backends.** Downstream code (SSE, portfolio,
  trades) must not care whether prices come from the simulator or a real API.
- **Push, don't pull.** Data sources write to a shared cache on their own
  schedule. Consumers read from the cache at their own cadence. Timing is
  decoupled.
- **Zero-config simulator default.** With no API key, the simulator runs and
  the app feels alive immediately. Real market data is opt-in via an env var.
- **Resilient.** A bad API call, a malformed snapshot, or a single bad tick
  must not kill the data feed. The SSE stream keeps streaming whatever the
  cache last saw.
- **Cheap to extend.** Adding a third data source (e.g. Alpaca, IEX) should
  be a new file implementing the same ABC — nothing else changes.

### Architecture diagram

```
                    ┌────────────────────────────────────────────┐
                    │      MarketDataSource  (ABC)               │
                    │      start / stop / add / remove           │
                    └──────┬──────────────────────────────┬──────┘
                           │                              │
                ┌──────────▼─────────┐         ┌──────────▼──────────┐
                │ SimulatorDataSource│         │ MassiveDataSource    │
                │ (GBMSimulator +    │         │ (Polygon REST poller │
                │  asyncio loop)     │         │  in to_thread)       │
                └──────────┬─────────┘         └──────────┬──────────┘
                           │   writes                     │  writes
                           ▼                              ▼
                    ┌─────────────────────────────────────────┐
                    │           PriceCache                    │
                    │  (thread-safe; version counter)         │
                    └──────────────────┬──────────────────────┘
                                       │ reads
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
     ┌────────────────┐       ┌────────────────┐       ┌────────────────┐
     │  SSE endpoint  │       │ Trade execution│       │ Portfolio value│
     │ /api/stream/..  │       │   handler      │       │   calculator   │
     └────────────────┘       └────────────────┘       └────────────────┘
```

The cache is the only point of contact between producers and consumers. This
is what makes the backend swappable.

---

## 2. File Layout

```
backend/
  app/
    market/
      __init__.py            # Re-exports the public API
      models.py              # PriceUpdate dataclass
      cache.py               # PriceCache (thread-safe in-memory store)
      interface.py           # MarketDataSource ABC
      seed_prices.py         # SEED_PRICES, TICKER_PARAMS, correlation groups
      simulator.py           # GBMSimulator + SimulatorDataSource
      massive_client.py      # MassiveDataSource
      factory.py             # create_market_data_source()
      stream.py              # SSE endpoint (FastAPI router factory)
  tests/
    market/
      test_models.py
      test_cache.py
      test_simulator.py
      test_simulator_source.py
      test_massive.py
      test_factory.py
```

Single responsibility per file. The `__init__.py` re-exports the public API so
the rest of the backend imports from `app.market`, not from submodules.

---

## 3. Data Model — `models.py`

`PriceUpdate` is the only type that leaves the market data layer. Every
downstream component works with it.

```python
# backend/app/market/models.py
from __future__ import annotations

import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of one ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round(
            (self.price - self.previous_price) / self.previous_price * 100, 4
        )

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        if self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for SSE / JSON API responses."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

### Why these choices

- **`frozen=True`** — immutable value object; safe to share across tasks
  without copies.
- **`slots=True`** — these are created by the thousand per minute; slots cuts
  per-instance memory.
- **Computed properties** — `change`, `change_percent`, `direction` are
  derived from `price`/`previous_price`, so they can never disagree with each
  other. No risk of a stale `direction` field.
- **`to_dict()`** — single serialization point for SSE and any future REST
  responses. If we change the wire format, we change it here.

### Example

```python
>>> u = PriceUpdate(ticker="AAPL", price=190.50, previous_price=190.00)
>>> u.change
0.5
>>> u.direction
'up'
>>> u.to_dict()
{'ticker': 'AAPL', 'price': 190.5, 'previous_price': 190.0,
 'timestamp': 1714680000.123, 'change': 0.5,
 'change_percent': 0.2632, 'direction': 'up'}
```

---

## 4. Price Cache — `cache.py`

The hub between producers and consumers. Producers (simulator / Massive
poller) write; consumers (SSE generator, trade handler, valuation) read.

```python
# backend/app/market/cache.py
from __future__ import annotations

import time
from threading import Lock

from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of latest price per ticker.

    Writers (one at a time): SimulatorDataSource or MassiveDataSource.
    Readers (many): SSE stream, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Bumped on every successful update

    def update(
        self,
        ticker: str,
        price: float,
        timestamp: float | None = None,
    ) -> PriceUpdate:
        """Record a new price. Returns the resulting PriceUpdate.

        previous_price is automatically taken from the prior cached value
        (or set equal to price on first update, so direction='flat').
        """
        with self._lock:
            ts = timestamp if timestamp is not None else time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        u = self.get(ticker)
        return u.price if u else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Shallow copy snapshot of all current prices."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

### Why a version counter

The SSE generator polls the cache every 500ms. Without a version counter, it
would serialize and emit every tick even when nothing changed (e.g. Massive
only updates every 15s, so 29 of every 30 ticks would be redundant). The
version counter lets the generator skip work cheaply:

```python
last_version = -1
while True:
    if cache.version != last_version:
        last_version = cache.version
        yield format_sse(cache.get_all())
    await asyncio.sleep(0.5)
```

### Why `threading.Lock`, not `asyncio.Lock`

- The Massive client is synchronous; it runs inside `asyncio.to_thread()`,
  i.e. an OS thread. `asyncio.Lock` does not protect against another thread.
- `threading.Lock` works correctly from both real threads *and* the event
  loop (acquire is non-blocking when uncontended).
- Critical sections are tiny (a dict get/set), so contention is negligible
  even at hundreds of updates per second.

---

## 5. Abstract Interface — `interface.py`

```python
# backend/app/market/interface.py
from __future__ import annotations

from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data producers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never asks the source for a price — it reads
    from the cache.

    Lifecycle:
        cache = PriceCache()
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # ... shutdown ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Called exactly once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources. Idempotent."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

### Why push, not pull

The simulator ticks at 500ms; Massive polls at 15s; SSE pushes at 500ms.
These three cadences are independent. If consumers had to pull, the SSE
loop would need to know the source's interval. With push-to-cache, every
component runs on its own clock and the cache absorbs the timing mismatch.

---

## 6. Seed Data & Ticker Parameters — `seed_prices.py`

Constants only — no logic, stdlib only. Shared by the simulator (initial
prices, GBM params) and potentially the Massive client (fallback during the
first poll gap).

```python
# backend/app/market/seed_prices.py
"""Seed prices and per-ticker parameters for the GBM simulator."""

# Realistic starting prices for the default watchlist
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM": 195.00,
    "V": 280.00,
    "NFLX": 600.00,
}

# Per-ticker GBM parameters
#   sigma: annualized volatility (higher = more movement)
#   mu:    annualized drift / expected return
TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # high vol
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # high vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

# Defaults for tickers added at runtime that aren't in TICKER_PARAMS
DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

# Sector groups used to build the correlation matrix
CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

# Pairwise correlation coefficients
INTRA_TECH_CORR    = 0.6   # tech ↔ tech (excl. TSLA)
INTRA_FINANCE_CORR = 0.5   # finance ↔ finance
CROSS_GROUP_CORR   = 0.3   # everything else (cross-sector / unknown / TSLA)
TSLA_CORR          = 0.3   # TSLA does its own thing
```

> Note: an earlier draft also defined `DEFAULT_CORR` as a duplicate of
> `CROSS_GROUP_CORR`. The review removed it — keep one name only.

---

## 7. GBM Simulator — `simulator.py`

Two classes, one file:

- `GBMSimulator` — pure math; stateful; `step()` advances all prices once.
- `SimulatorDataSource` — implements `MarketDataSource`; runs a 500ms
  asyncio loop that calls `step()` and writes to the cache.

### 7.1 The math

At each step:

```
S(t+dt) = S(t) · exp((μ − σ²/2)·dt + σ·√dt · Z)
```

with `dt` as a fraction of a trading year:

```
TRADING_SECONDS_PER_YEAR = 252 days × 6.5 h × 3600 s = 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  ≈ 8.48 × 10⁻⁸
```

Correlated random draws come from a Cholesky decomposition of a sector-based
correlation matrix:

```
Z_correlated = L · Z_independent     where  L = cholesky(C)
```

A small per-tick chance (~0.001) of a 2–5% shock event adds visible drama.

### 7.2 `GBMSimulator`

```python
# backend/app/market/simulator.py  (top half)
from __future__ import annotations

import asyncio
import logging
import math
import random

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
from .seed_prices import (
    CORRELATION_GROUPS,
    CROSS_GROUP_CORR,
    DEFAULT_PARAMS,
    INTRA_FINANCE_CORR,
    INTRA_TECH_CORR,
    SEED_PRICES,
    TICKER_PARAMS,
    TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Geometric Brownian Motion simulator with sector-correlated moves."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for t in tickers:
            self._add_internal(t)
        self._rebuild_cholesky()

    # ----- public API -----

    def step(self) -> dict[str, float]:
        """Advance all tickers one step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_indep = np.random.standard_normal(n)
        z = self._cholesky @ z_indep if self._cholesky is not None else z_indep

        out: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            p = self._params[ticker]
            mu, sigma = p["mu"], p["sigma"]
            drift = (mu - 0.5 * sigma ** 2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random shock event
            if random.random() < self._event_prob:
                magnitude = random.uniform(0.02, 0.05)
                sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + magnitude * sign
                logger.debug(
                    "Random event on %s: %.1f%% %s",
                    ticker, magnitude * 100, "up" if sign > 0 else "down",
                )

            out[ticker] = round(self._prices[ticker], 2)
        return out

    def add_ticker(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._add_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        # Public accessor — avoid touching `_tickers` from outside.
        return list(self._tickers)

    # ----- internals -----

    def _add_internal(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(
            ticker, random.uniform(50.0, 300.0)
        )
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_corr(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_corr(t1: str, t2: str) -> float:
        tech = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

### 7.3 `SimulatorDataSource`

```python
# backend/app/market/simulator.py  (bottom half, same file)

class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator."""

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(
            tickers=tickers, event_probability=self._event_prob
        )
        # Seed the cache with starting prices so SSE has data immediately.
        for t in tickers:
            price = self._sim.get_price(t)
            if price is not None:
                self._cache.update(ticker=t, price=price)
        self._task = asyncio.create_task(self._loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if not self._sim:
            return
        self._sim.add_ticker(ticker)
        # Seed the cache immediately so callers can read a price right away.
        price = self._sim.get_price(ticker)
        if price is not None:
            self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _loop(self) -> None:
        while True:
            try:
                if self._sim:
                    for t, p in self._sim.step().items():
                        self._cache.update(ticker=t, price=p)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

### Behaviors worth knowing

- **Immediate seeding** — the cache is populated *before* the loop starts, so
  the SSE endpoint never serves a blank page.
- **Per-step exception isolation** — a single bad tick does not kill the
  feed. The loop logs and keeps going.
- **Graceful cancellation** — `stop()` cancels and awaits the task, swallowing
  `CancelledError`. Safe to call twice.

---

## 8. Massive API Client — `massive_client.py`

Polls the Massive (formerly Polygon.io) snapshot endpoint on a configurable
interval. The Massive REST client is synchronous, so the network call runs in
`asyncio.to_thread()` to keep the event loop responsive.

```python
# backend/app/market/massive_client.py
from __future__ import annotations

import asyncio
import logging
from typing import Any

from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """REST poller for Massive (Polygon.io) snapshot endpoint.

    Calls GET /v2/snapshot/locale/us/markets/stocks/tickers for the current
    ticker set in a single API call, then updates the PriceCache.

    Polling cadence:
        Free tier (5 req/min):   poll_interval=15.0  (default)
        Paid tiers:              poll_interval=2-5
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = [t.upper().strip() for t in tickers]

        # First poll happens immediately so the cache has data before the SSE
        # stream sends its first event.
        await self._poll_once()

        self._task = asyncio.create_task(
            self._poll_loop(), name="massive-poller"
        )
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval",
            len(tickers), self._interval,
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added %s (visible after next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # ----- internals -----

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
        except Exception as e:
            # 401 (bad key), 429 (rate limit), network errors, etc.
            # Don't re-raise — try again next interval.
            logger.error("Massive poll failed: %s", e)
            return

        processed = 0
        for snap in snapshots:
            try:
                price = snap.last_trade.price
                ts = snap.last_trade.timestamp / 1000.0  # ms → s
                self._cache.update(
                    ticker=snap.ticker, price=price, timestamp=ts
                )
                processed += 1
            except (AttributeError, TypeError) as e:
                logger.warning(
                    "Skipping snapshot for %s: %s",
                    getattr(snap, "ticker", "???"), e,
                )
        logger.debug(
            "Massive poll: %d/%d tickers updated",
            processed, len(self._tickers),
        )

    def _fetch_snapshots(self) -> list[Any]:
        """Synchronous call — runs in a worker thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Error handling philosophy

| Error               | Behavior                                                     |
|---------------------|--------------------------------------------------------------|
| 401 Unauthorized    | Log error. Keep polling — user may fix `.env` & restart.     |
| 429 Rate Limited    | Log error. Next poll retries after `poll_interval`.          |
| Network timeout     | Log error. Retries on the next cycle.                        |
| Malformed snapshot  | Skip that one ticker; keep processing the rest.              |
| All tickers fail    | Cache retains last-known prices; SSE keeps streaming stale.  |

### Why `asyncio.to_thread`

The `massive` Python client is synchronous (uses `requests` under the hood).
Calling it directly on the event loop would block all other async work — SSE
generators, FastAPI route handlers, the simulator — for the duration of the
HTTP call. `asyncio.to_thread()` runs it on the default executor and returns
control to the loop while the request is in flight.

### Massive snapshot — sample response

```json
{
  "ticker": "AAPL",
  "last_trade": {
    "price": 190.42,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1714680000000
  },
  "day": {
    "open": 189.10,
    "high": 191.20,
    "low": 188.95,
    "close": 190.42,
    "volume": 32150000,
    "previous_close": 188.85,
    "change": 1.57,
    "change_percent": 0.83
  }
}
```

We extract `last_trade.price` and `last_trade.timestamp` for each ticker.
Daily OHLC is currently unused but available if a chart needs it later.

---

## 9. Factory — `factory.py`

```python
# backend/app/market/factory.py
from __future__ import annotations

import logging
import os

from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Pick the right data source based on environment.

    MASSIVE_API_KEY set & non-empty -> MassiveDataSource (real data)
    Otherwise                       -> SimulatorDataSource (GBM)

    Returns an unstarted source. Caller must `await source.start(tickers)`.
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    logger.info("Market data source: GBM Simulator")
    return SimulatorDataSource(price_cache=price_cache)
```

> Note: the review removed the lazy `from .massive_client import ...` and
> `from .simulator import ...` inside the function. `massive` is a hard
> dependency in `pyproject.toml`; lazy imports added complexity for no real
> benefit. Keep all imports at the top.

---

## 10. SSE Streaming Endpoint — `stream.py`

```python
# backend/app/market/stream.py
from __future__ import annotations

import asyncio
import json
import logging
from typing import AsyncGenerator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from .cache import PriceCache

logger = logging.getLogger(__name__)


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Build a FastAPI router for the SSE price stream.

    Factory pattern so we can inject the cache without globals.
    """
    router = APIRouter(prefix="/api/stream", tags=["streaming"])

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # nginx: don't buffer SSE
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """SSE generator: emit all prices whenever the cache version changes."""
    yield "retry: 1000\n\n"  # browser auto-reconnect after 1s

    last_version = -1
    client = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client)
                break

            v = price_cache.version
            if v != last_version:
                last_version = v
                prices = price_cache.get_all()
                if prices:
                    payload = json.dumps(
                        {t: u.to_dict() for t, u in prices.items()}
                    )
                    yield f"data: {payload}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled: %s", client)
```

### Wire format

```
retry: 1000

data: {"AAPL":{"ticker":"AAPL","price":190.50,"previous_price":190.42,"timestamp":1714680000.5,"change":0.08,"change_percent":0.042,"direction":"up"},"GOOGL":{...}}

```

### Browser side

```javascript
const es = new EventSource("/api/stream/prices");
es.onmessage = (event) => {
  const prices = JSON.parse(event.data);
  // prices is { AAPL: {ticker, price, previous_price, ...}, GOOGL: {...} }
};
es.onerror = () => {
  // EventSource auto-reconnects per the `retry:` directive.
  setStatus("reconnecting");
};
```

### Why poll-and-push (not event-driven)

The SSE generator polls the cache on a fixed interval rather than
subscribing to a producer signal. Reasons:

1. **Even spacing.** The frontend builds sparkline charts from the SSE feed;
   regular intervals look smoother than bursts.
2. **Source-agnostic.** The generator does not need to know whether the cache
   is being filled at 500ms or 15s. It just emits when the version changes.
3. **Backpressure-safe.** A slow client cannot stall a producer; it just
   misses an interim version.

---

## 11. Package Exports — `__init__.py`

```python
# backend/app/market/__init__.py
"""Market data subsystem for FinAlly.

Public API:
    PriceUpdate                 - immutable price snapshot
    PriceCache                  - thread-safe price store
    MarketDataSource            - abstract producer interface
    create_market_data_source   - factory (simulator vs Massive)
    create_stream_router        - SSE FastAPI router factory
"""
from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceUpdate",
    "PriceCache",
    "MarketDataSource",
    "create_market_data_source",
    "create_stream_router",
]
```

The rest of the backend imports from `app.market`, never from
`app.market.simulator` or other submodules.

---

## 12. FastAPI Lifecycle Integration

```python
# backend/app/main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI

from app.market import (
    PriceCache,
    MarketDataSource,
    create_market_data_source,
    create_stream_router,
)
# ... import db helpers, other routers ...


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ----- startup -----
    cache = PriceCache()
    app.state.price_cache = cache

    source = create_market_data_source(cache)
    app.state.market_source = source

    # All currently-watched tickers (initial seed = the 10 defaults)
    initial_tickers = await load_watchlist_tickers()
    await source.start(initial_tickers)

    app.include_router(create_stream_router(cache))

    yield

    # ----- shutdown -----
    await source.stop()


app = FastAPI(title="FinAlly", lifespan=lifespan)


# Dependencies for route handlers
def get_price_cache() -> PriceCache:
    return app.state.price_cache


def get_market_source() -> MarketDataSource:
    return app.state.market_source
```

### Using the cache from other routes

```python
from fastapi import Depends, HTTPException

from app.market import PriceCache, MarketDataSource


@router.post("/portfolio/trade")
async def execute_trade(
    trade: TradeRequest,
    cache: PriceCache = Depends(get_price_cache),
):
    price = cache.get_price(trade.ticker)
    if price is None:
        raise HTTPException(
            400,
            f"Price not yet available for {trade.ticker}; try again in a moment.",
        )
    # ... validate cash/position, persist trade at `price` ...


@router.post("/watchlist")
async def add_to_watchlist(
    payload: WatchlistAdd,
    source: MarketDataSource = Depends(get_market_source),
):
    await db.insert_watchlist(payload.ticker)
    await source.add_ticker(payload.ticker)
    return {"status": "ok"}
```

---

## 13. Watchlist Coordination

The watchlist (in SQLite) and the data source's active ticker set must stay
in sync.

### Add flow

```
POST /api/watchlist  {ticker: "PYPL"}
  ├─ INSERT INTO watchlist ...
  └─ await source.add_ticker("PYPL")
        ├─ Simulator: add to GBMSimulator, rebuild Cholesky, seed cache
        └─ Massive:   append to ticker list (appears on next poll)
```

### Remove flow

```
DELETE /api/watchlist/PYPL
  ├─ DELETE FROM watchlist WHERE ticker = ?
  └─ if no open position:
        await source.remove_ticker("PYPL")
            ├─ Simulator: remove from GBMSimulator, rebuild Cholesky, drop from cache
            └─ Massive:   remove from ticker list, drop from cache
```

### Edge case: removing a watched ticker the user still owns

If the user removes `PYPL` from the watchlist but still holds shares,
portfolio valuation needs that price. Don't stop tracking it:

```python
@router.delete("/watchlist/{ticker}")
async def remove_from_watchlist(
    ticker: str,
    source: MarketDataSource = Depends(get_market_source),
):
    await db.delete_watchlist_entry(ticker)
    pos = await db.get_position(ticker)
    if pos is None or pos.quantity == 0:
        await source.remove_ticker(ticker)
    return {"status": "ok"}
```

---

## 14. Testing Strategy

Six test modules under `backend/tests/market/`. Target: ≥80% coverage. The
real suite that ships with the implementation is 73 tests at 84% overall
coverage — these examples are representative.

### 14.1 `test_models.py`

```python
from app.market.models import PriceUpdate

def test_direction_up():
    u = PriceUpdate(ticker="AAPL", price=190.5, previous_price=190.0)
    assert u.direction == "up"
    assert u.change == 0.5

def test_direction_down():
    u = PriceUpdate(ticker="AAPL", price=189.0, previous_price=190.0)
    assert u.direction == "down"

def test_direction_flat():
    u = PriceUpdate(ticker="AAPL", price=190.0, previous_price=190.0)
    assert u.direction == "flat"
    assert u.change == 0.0

def test_change_percent_zero_previous():
    u = PriceUpdate(ticker="AAPL", price=10.0, previous_price=0.0)
    assert u.change_percent == 0.0  # avoid ZeroDivisionError

def test_to_dict_round_trip():
    u = PriceUpdate(ticker="AAPL", price=190.5, previous_price=190.0)
    d = u.to_dict()
    assert d["ticker"] == "AAPL"
    assert d["direction"] == "up"
    assert "timestamp" in d

def test_frozen():
    u = PriceUpdate(ticker="AAPL", price=190.5, previous_price=190.0)
    with pytest.raises(Exception):
        u.price = 200.0  # frozen dataclass
```

### 14.2 `test_cache.py`

```python
from app.market.cache import PriceCache

def test_first_update_is_flat():
    c = PriceCache()
    u = c.update("AAPL", 190.5)
    assert u.direction == "flat"
    assert u.previous_price == 190.5

def test_subsequent_update_uses_prior_price():
    c = PriceCache()
    c.update("AAPL", 190.0)
    u = c.update("AAPL", 191.0)
    assert u.direction == "up"
    assert u.previous_price == 190.0

def test_version_increments():
    c = PriceCache()
    v = c.version
    c.update("AAPL", 1.0)
    c.update("AAPL", 2.0)
    assert c.version == v + 2

def test_remove():
    c = PriceCache()
    c.update("AAPL", 190.0)
    c.remove("AAPL")
    assert c.get("AAPL") is None

def test_remove_unknown_is_noop():
    PriceCache().remove("NOPE")  # must not raise
```

### 14.3 `test_simulator.py`

```python
from app.market.simulator import GBMSimulator
from app.market.seed_prices import SEED_PRICES

def test_step_returns_all_tickers():
    sim = GBMSimulator(["AAPL", "GOOGL"])
    assert set(sim.step()) == {"AAPL", "GOOGL"}

def test_prices_stay_positive():
    """GBM is multiplicative (exp), so prices are bounded below by zero."""
    sim = GBMSimulator(["TSLA"])  # high volatility
    for _ in range(10_000):
        assert sim.step()["TSLA"] > 0

def test_initial_price_is_seed():
    sim = GBMSimulator(["AAPL"])
    assert sim.get_price("AAPL") == SEED_PRICES["AAPL"]

def test_unknown_ticker_random_seed():
    sim = GBMSimulator(["ZZZZ"])
    p = sim.get_price("ZZZZ")
    assert 50.0 <= p <= 300.0

def test_add_remove():
    sim = GBMSimulator(["AAPL"])
    sim.add_ticker("TSLA")
    assert "TSLA" in sim.get_tickers()
    sim.remove_ticker("TSLA")
    assert "TSLA" not in sim.get_tickers()

def test_cholesky_rebuilds():
    sim = GBMSimulator(["AAPL"])
    assert sim._cholesky is None  # 1x1: skip
    sim.add_ticker("GOOGL")
    assert sim._cholesky is not None  # 2x2 now
```

### 14.4 `test_simulator_source.py` — async integration

```python
import asyncio
import pytest
from app.market.cache import PriceCache
from app.market.simulator import SimulatorDataSource


@pytest.mark.asyncio
async def test_start_seeds_cache_immediately():
    cache = PriceCache()
    src = SimulatorDataSource(cache, update_interval=10.0)  # slow loop
    await src.start(["AAPL", "GOOGL"])
    # Cache must be populated before the loop ticks
    assert cache.get("AAPL") is not None
    assert cache.get("GOOGL") is not None
    await src.stop()


@pytest.mark.asyncio
async def test_loop_updates_cache_over_time():
    cache = PriceCache()
    src = SimulatorDataSource(cache, update_interval=0.05)
    await src.start(["AAPL"])
    v0 = cache.version
    await asyncio.sleep(0.25)
    assert cache.version > v0
    await src.stop()


@pytest.mark.asyncio
async def test_double_stop_safe():
    cache = PriceCache()
    src = SimulatorDataSource(cache)
    await src.start(["AAPL"])
    await src.stop()
    await src.stop()  # must not raise
```

### 14.5 `test_massive.py` — mocked

```python
from unittest.mock import MagicMock, patch
import pytest
from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource


def _snap(ticker, price, ts_ms=1714680000000):
    s = MagicMock()
    s.ticker = ticker
    s.last_trade.price = price
    s.last_trade.timestamp = ts_ms
    return s


@pytest.mark.asyncio
async def test_poll_updates_cache():
    cache = PriceCache()
    src = MassiveDataSource("k", cache, poll_interval=60.0)
    src._tickers = ["AAPL", "GOOGL"]
    src._client = MagicMock()  # poll_once requires _client truthy

    snaps = [_snap("AAPL", 190.5), _snap("GOOGL", 175.25)]
    with patch.object(src, "_fetch_snapshots", return_value=snaps):
        await src._poll_once()

    assert cache.get_price("AAPL") == 190.50
    assert cache.get_price("GOOGL") == 175.25


@pytest.mark.asyncio
async def test_malformed_snapshot_skipped():
    cache = PriceCache()
    src = MassiveDataSource("k", cache, poll_interval=60.0)
    src._tickers = ["AAPL", "BAD"]
    src._client = MagicMock()

    good = _snap("AAPL", 190.5)
    bad = MagicMock()
    bad.ticker = "BAD"
    bad.last_trade = None  # AttributeError on .price

    with patch.object(src, "_fetch_snapshots", return_value=[good, bad]):
        await src._poll_once()

    assert cache.get_price("AAPL") == 190.50
    assert cache.get_price("BAD") is None


@pytest.mark.asyncio
async def test_api_failure_swallowed():
    cache = PriceCache()
    src = MassiveDataSource("k", cache, poll_interval=60.0)
    src._tickers = ["AAPL"]
    src._client = MagicMock()
    with patch.object(src, "_fetch_snapshots", side_effect=Exception("boom")):
        await src._poll_once()  # must not raise
    assert cache.get_price("AAPL") is None
```

### 14.6 `test_factory.py`

```python
import os
from unittest.mock import patch
from app.market.cache import PriceCache
from app.market.factory import create_market_data_source
from app.market.simulator import SimulatorDataSource
from app.market.massive_client import MassiveDataSource


def test_no_key_returns_simulator():
    with patch.dict(os.environ, {"MASSIVE_API_KEY": ""}, clear=False):
        src = create_market_data_source(PriceCache())
    assert isinstance(src, SimulatorDataSource)


def test_empty_string_key_returns_simulator():
    with patch.dict(os.environ, {"MASSIVE_API_KEY": "   "}, clear=False):
        src = create_market_data_source(PriceCache())
    assert isinstance(src, SimulatorDataSource)


def test_real_key_returns_massive():
    with patch.dict(os.environ, {"MASSIVE_API_KEY": "abc"}, clear=False):
        src = create_market_data_source(PriceCache())
    assert isinstance(src, MassiveDataSource)
```

---

## 15. Error Handling & Edge Cases

### 15.1 Empty watchlist on startup

If the database has no watchlist entries, `start([])` is a valid call. Both
sources handle it: simulator produces no prices; Massive skips the API call
entirely. SSE emits no events until something is added.

### 15.2 Trade attempt before first price arrives

With Massive, the cache may briefly be empty for a newly-added ticker (next
poll up to `poll_interval` away). Trade endpoint must check:

```python
price = cache.get_price(trade.ticker)
if price is None:
    raise HTTPException(
        400,
        f"Price not yet available for {trade.ticker}. Try again in a few seconds.",
    )
```

The simulator avoids this by seeding the cache inside `add_ticker()`.

### 15.3 Invalid Massive API key

First poll fails with 401. The poller logs and keeps trying. The frontend
sees a healthy SSE connection but no price data. The user fixes `.env` and
restarts the container.

### 15.4 Numerical stability of GBM

Prices are always positive (multiplicative GBM, `exp()` is positive).
Per-tick moves are sub-cent (`dt ≈ 8.5e-8`), well above float64 epsilon.
`round(price, 2)` on output keeps display values tidy.

### 15.5 Cholesky on a singleton ticker

`np.linalg.cholesky` requires `n >= 1`, but a 1×1 matrix yields a no-op
multiplier. We skip the decomposition entirely when `n <= 1` and pass the
independent draw straight through.

### 15.6 Lock contention

`PriceCache._lock` is held only for a single dict `get` + `set` on write,
or a single `dict()` copy on read. With 10–50 tickers and a few SSE clients,
contention is unmeasurable. If we ever scaled to hundreds of concurrent SSE
connections, a `RWLock` would be the optimization — but not now.

### 15.7 Massive timestamp units

Massive returns `last_trade.timestamp` in **milliseconds**. We divide by
1000 before storing. Forgetting this gives timestamps in the year ~56000.

---

## 16. Configuration Summary

| Parameter           | Where                          | Default       | Description                                         |
|---------------------|--------------------------------|---------------|-----------------------------------------------------|
| `MASSIVE_API_KEY`   | env var                        | `""` (empty)  | If set, use Massive; otherwise simulator.           |
| `update_interval`   | `SimulatorDataSource(...)`     | `0.5` s       | Simulator tick rate.                                |
| `poll_interval`     | `MassiveDataSource(...)`       | `15.0` s      | Massive REST poll rate.                             |
| `event_probability` | `GBMSimulator(...)`            | `0.001`       | Per-tick chance of a 2–5% shock per ticker.         |
| `dt`                | `GBMSimulator(...)`            | `~8.5e-8`     | GBM time step (fraction of a trading year).        |
| SSE push interval   | `_generate_events(...)`        | `0.5` s       | How often SSE checks for cache changes.             |
| SSE retry directive | `_generate_events()` first line| `1000` ms     | Browser EventSource auto-reconnect delay.          |

---

## 17. Dependencies (`pyproject.toml`)

```toml
[project]
name = "finally-backend"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "numpy>=2.0",
    "massive>=1.0",       # Polygon.io REST client
    # ... db, llm, etc. ...
]

[tool.hatch.build.targets.wheel]
packages = ["app"]        # required so `uv build` finds the app/ package

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

> The `[tool.hatch.build.targets.wheel] packages = ["app"]` block was added
> as part of the review — without it, the wheel build silently produces an
> empty package. Don't omit it.

---

## End-to-end usage example

```python
# startup
from app.market import PriceCache, create_market_data_source

cache = PriceCache()
source = create_market_data_source(cache)        # picks sim vs massive
await source.start(["AAPL", "GOOGL", "MSFT"])    # seeds cache, starts loop

# any time
price = cache.get_price("AAPL")                  # float | None
update = cache.get("AAPL")                       # PriceUpdate | None
all_prices = cache.get_all()                     # dict[str, PriceUpdate]

# watchlist mutation
await source.add_ticker("TSLA")
await source.remove_ticker("GOOGL")

# shutdown
await source.stop()
```

That's the entire surface area downstream code touches. The simulator vs
Massive choice, the polling, the cache locking, the Cholesky math, the SSE
formatting — all of it is sealed behind these calls.
