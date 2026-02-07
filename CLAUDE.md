# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Real-time terminal dashboard combining Binance order flow with Polymarket prediction market prices for crypto signals. Supports BTC, ETH, SOL, XRP across 15m/1h/4h/daily timeframes.

## Commands

```bash
pip install -r requirements.txt   # install deps (requests, websockets, rich)
python main.py                     # run — interactive menu selects coin + timeframe
```

No test suite, linter, or build step exists. The app runs as a single async process with no CLI flags — coin and timeframe are selected via interactive terminal menu at startup.

## Architecture

Four-module pipeline with shared mutable state:

```
main.py → user picks coin/tf → launches async tasks
                                    ↓
config.py ← constants (coins, URLs, indicator params, thresholds)
                                    ↓
feeds.py  → State object (central mutable store)
            ├── ob_poller()     — REST polls Binance orderbook every 2s
            ├── binance_feed()  — WS streams trades + klines
            ├── pm_feed()       — WS streams Polymarket up/down prices
            └── bootstrap()     — REST fetches historical klines on startup
                                    ↓
indicators.py → pure functions (no side effects, no state)
            obi, walls, depth_usd, cvd, vol_profile,
            rsi, macd, vwap, emas, heikin_ashi
                                    ↓
dashboard.py → reads State + calls indicators → Rich renderables
            render() returns composite layout (header + grid + signals)
            _score_trend() aggregates 11 indicators into BULLISH/BEARISH/NEUTRAL
```

### Key patterns

- **`feeds.State`** is the single shared object. All async tasks mutate it; the dashboard reads it. No locking — relies on GIL + asyncio single-thread model.
- **`asyncio.gather`** in `main.py` runs four concurrent coroutines: orderbook poller, Binance WS feed, Polymarket WS feed, and display loop.
- **Polymarket slug construction** (`feeds._build_slug`) is timeframe-dependent and uses ET (Eastern Time) calculated manually without `pytz`/`zoneinfo`. The 15m and 4h slugs use Unix timestamps; 1h and daily use human-readable date strings.
- **Indicator functions** are stateless — they take raw data (bids/asks/trades/klines lists) and return computed values. All thresholds live in `config.py`.
- **Trend scoring** (`dashboard._score_trend`) sums ±1 contributions from each indicator; ≥3 = BULLISH, ≤-3 = BEARISH.

### External dependencies

- **Binance**: REST (`api.binance.com/api/v3`) for orderbook + kline bootstrap; WS (`stream.binance.com`) for live trades + klines. No API key required.
- **Polymarket**: REST (`gamma-api.polymarket.com/events`) for market discovery; WS (`ws-subscriptions-clob.polymarket.com`) for live prices. No API key required.

## Module boundaries

| Module | Owns | Should NOT contain |
|--------|------|--------------------|
| `config.py` | All constants and thresholds | Logic, imports beyond stdlib |
| `feeds.py` | Network I/O, State class, slug building | Indicator math, rendering |
| `indicators.py` | Pure math on raw data | Network calls, state mutation |
| `dashboard.py` | Rich UI rendering, trend scoring | Network calls, data fetching |
