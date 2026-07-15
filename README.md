# SMC Inversion FVG Model [Institutional]

A production-grade TradingView indicator (Pine Script v6) that identifies high-probability
reversal trades using Smart Money Concepts. Inspired by the *concepts* behind inversion-FVG
reversal models — implemented entirely with original logic, no proprietary code.

**File:** [`SMC_IFVG_Model.pine`](SMC_IFVG_Model.pine)

## The model

```
Liquidity Sweep  →  Market Structure Confirmation  →  V-Shaped Recovery
      →  Fair Value Gap  →  Inversion FVG Retest  →  HTF Confirmation
      →  Liquidity Target  →  Trade Execution Plan (Entry/SL/BE/TP1/TP2)
```

Every signal is graded **A / A+ / A++** by a scoring engine, and each trade ships with an
automatic Entry, Stop Loss, Break-Even level, TP1 and TP2, plus a webhook-ready JSON alert.

---

## Architecture

The script is a single file (TradingView requirement) organized as 13 isolated modules that
communicate through a small set of well-defined state variables:

| # | Module | Produces | Consumed by |
|---|--------|----------|-------------|
| 1 | Market Structure Engine | `bias` / `effBias` (BOS, CHoCH, swing registry) | Setup engine, dashboard |
| 2 | Liquidity Engine | Pools (`bslArr`/`sslArr`), sweep snapshots | V-shape, DOL, stops |
| 3 | FVG Engine | Tracked gaps with lifecycle state | IFVG engine |
| 4 | IFVG Engine | Inversion zones + **retest trigger** | Setup engine (entry trigger) |
| 5 | V-Shape Recovery Engine | `vUpBar`/`vDnBar` displacement confirmation | Setup engine |
| 6 | HTF Engine | HTF FVG/IFVG zones + HTF bias ×3 | Scoring, dashboard |
| 7 | Premium/Discount Engine | `inPremium` / `inDiscount` | Scoring |
| 8 | Draw-On-Liquidity Engine | Nearest unswept opposite pool | Scoring, chart marker |
| 9 | SMT Divergence Engine | `smtBullValid` / `smtBearValid` | Scoring |
| 10 | Scoring + Setup Engine | Graded signals, opens trades | Trade mgmt, alerts |
| 11 | Trade Management Engine | SL/BE/TP tracking, win/loss stats | Dashboard, alerts |
| 12 | Dashboard Engine | Right-side table (bias, checklist, stats) | — |
| 13 | Alert Engine | 9 static alertconditions + JSON `alert()` calls | — |

### Setup state machine

1. **Sweep** — a pool (swing high/low, EQH/EQL, session extreme, PDH/PDL) is wicked and
   price closes back inside the range. Snapshot stored (`bar`, `level`, `excursion extreme`).
2. **V-shape** — within *Max Bars After Sweep*, one displacement candle
   (body ≥ `vDispAtr × ATR`) recovers ≥ `vRetrace`% of the sweep excursion while internal
   structure flips the same direction.
3. **IFVG retest** — an opposing FVG is closed through (inversion) and price returns to the
   flipped zone, which holds. This bar is the entry trigger.
4. **Gate** — market bias (`effBias`) must agree, and everything must occur inside the
   *Sweep → Entry Max Window*.
5. **Score → Trade** — base 4 points + confluence points; if the rating passes the minimum,
   a trade plan is drawn, stats update, and alerts fire.

### Scoring

| Component | Points |
|-----------|--------|
| Bias + Sweep + V-shape + IFVG retest (all required) | 4 (base) |
| HTF FVG / HTF IFVG supporting direction | +1 |
| Entry in discount (long) / premium (short) | +1 |
| Clear opposite DOL ≥ `dolMinRR` × risk away | +1 |
| Valid SMT divergence | +1 |

**4–5 → A · 6 → A+ · 7–8 → A++**

### Non-repainting policy

- All stateful logic runs under `barstate.isconfirmed` — nothing mutates intrabar.
- Every `request.security()` call uses the `[1]`-offset + `lookahead_on` idiom, which
  returns the **last fully closed** HTF bar: no lookahead, no future data, no repaint.
- Pivots confirm `length` bars after the extreme (inherent lag, never repaints).
- Entries are taken at the close of the confirmed trigger bar — exactly what an automation
  webhook receives.

---

## How to use

1. Open TradingView → Pine Editor → paste `SMC_IFVG_Model.pine` → **Add to chart**.
2. Trade in the direction of the dashboard **Market Bias** only (the engine already
   enforces this).
3. Wait for a triangle signal. The label shows rating + Entry/SL/BE/TP1/TP2.
4. Alerts: *Create Alert → Condition: SMC-IFVG* and pick a static condition
   (A/A+/A++ Setup, Long/Short Signal, TP/Stop Hit, Liquidity Sweep, IFVG Created), **or**
   choose *"Any alert() function call"* for the webhook JSON payload:

```json
{"symbol":"MNQ1!","direction":"long","rating":"A+","entry":"4052.68",
 "stop":"4039.83","tp1":"4073.12","tp2":"4092.11","timeframe":"5"}
```

TP/stop events post `{"symbol":…,"event":"tp_hit"|"stop_hit","timeframe":…}`.

## Recommended settings

| Setting | MNQ/NQ (2–5m) | ES/MES (2–5m) | Gold (5–15m) | Forex (15m–1H) | Crypto (15m–1H) |
|---|---|---|---|---|---|
| Swing Length | 8 | 10 | 8 | 10 | 8 |
| Displacement (×ATR) | 1.2 | 1.0 | 1.0 | 0.8 | 1.2 |
| Min Gap Size (×ATR) | 0.20 | 0.15 | 0.15 | 0.10 | 0.25 |
| Max Bars After Sweep | 4 | 5 | 5 | 6 | 5 |
| HTFs | 15/60/240 | 15/60/240 | 60/240/D | 60/240/D | 60/240/D |
| SMT symbol | `CME_MINI:ES1!` | `CME_MINI:NQ1!` | `TVC:DXY` (invert read) | correlated pair | `BINANCE:ETHUSDT` |
| Stop mode | Liquidity | Liquidity | Liquidity | Swing | ATR (1.8×) |
| Sessions TZ | America/New_York | America/New_York | America/New_York | exchange | UTC |

Futures tip: keep default session times (18:00–03:00 Asia, 03:00–08:30 London,
09:30–16:00 NY, New York time). Crypto: sessions still matter — liquidity concentrates at
the same clock times.

## Known limitations

- **Confirmation lag** — swing pivots confirm `swingLen` bars late; the model trades the
  *retest*, not the sweep low/high itself. This is the price of zero repainting.
- **Win-rate stats are indicative** — intrabar order-of-touch is unknowable on OHLC data;
  the engine assumes the stop is hit first when both stop and target print in one bar
  (conservative). Use TradingView's bar-magnifier strategy backtest for execution-grade
  numbers.
- **HTF bias is a proxy** (closed HTF close vs EMA-20/50), not a full HTF structure engine —
  Pine cannot run the pivot engine per-HTF without heavy `request.security` duplication.
- **SMT needs a liquid, correlated comparison symbol**; divergence on illiquid pairs is noise.
- One `Ifvg` zone fires one retest; later touches of the same zone are intentionally ignored.
- Session logic assumes the chart has extended/overnight data (futures: use continuous
  contracts with ETH).

## Future improvements

- Strategy-script port (`strategy.*`) for bar-magnifier backtesting and equity curves.
- Full HTF structure engine via chart-TF aggregation instead of the EMA proxy.
- Order-block + breaker-block confluence module; NWOG/NDOG gaps.
- Time-of-day filters (macro windows, silver-bullet hours) as scoring inputs.
- Partial-fill / scale-out management and trailing structural stops.
- Multi-zone concurrent trade tracking (currently one-at-a-time by default).
