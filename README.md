# 🏦 Mini Hedge Fund

A systematic, momentum-based paper trading system built on a 4-agent architecture. Runs autonomously on a VPS, sends daily briefings via Telegram, and executes trades through Alpaca's paper trading API — all within strict risk guardrails.

**Status:** 30-day validation period (confirm mode — no auto-execution)  
**Capital:** $2,000 max deployed from $100k Alpaca paper account  
**Universe:** 12 uncorrelated ETFs spanning equities, bonds, commodities, and real estate  
**Strategy:** Dual momentum (Jegadeesh-Titman 1993 / Antonacci GEM)

---

## Why This Exists

Most retail algo-trading projects start with indicators (RSI, MACD, Bollinger Bands) applied to a handful of correlated tech stocks. The academic evidence for these approaches is weak. This project takes the opposite path: start with the most empirically validated strategy in quantitative finance — momentum — and apply it to a diversified, uncorrelated ETF universe where the strategy actually has something to exploit.

The system is intentionally conservative. Small capital, strict daily limits, mandatory human confirmation, and a 30-day validation period before any consideration of autonomous execution.

---

## Architecture

Four specialized agents, each with a single responsibility, communicating through shared JSON files on disk:

```
┌─────────────────────────────────────────────────┐
│                    MD Agent                      │
│         (orchestrator + Telegram + execution)    │
│                                                  │
│   7am PT daily:  sync → analyst → allocator      │
│   sends Telegram briefing, waits for approval    │
│   executes via Alpaca API on confirmation        │
└──────────┬──────────┬──────────┬────────────────┘
           │          │          │
     ┌─────▼──┐  ┌────▼───┐  ┌──▼──────────┐
     │Analyst │  │Allocator│  │  Tax/Risk   │
     │        │  │         │  │             │
     │signals │  │proposals│  │wash sales   │
     │.json   │  │.json    │  │holding pds  │
     └────────┘  └────┬────┘  └─────────────┘
                      │
                ┌─────▼──────┐
                │ Guardrails │
                │            │
                │ validates  │
                │ every trade│
                └────────────┘
```

### Agent Responsibilities

**Analyst** — Scores the 12-ETF watchlist using dual momentum. Outputs `signals.json` with score (-5 to +5), confidence, and signal (BUY/HOLD/SELL) for each ticker. Runs in ~10 seconds via yfinance.

**Allocator** — Reads signals and portfolio state, generates trade proposals sized to fit within all constraints. Equal-weights the top 3-4 BUY signals. Calls Tax/Risk to check the wash sale blocklist before proposing. Passes proposals through Guardrails for validation.

**Tax/Risk** — Maintains a 30-day wash sale window from `trade_log.json`. Tracks holding periods (short-term vs long-term capital gains). Flags portfolio concentration risk. The Allocator consults this *before* proposing trades.

**Guardrails** — Pure validation layer with no opinions on direction. Every proposed trade must pass six hard checks before it can execute. Validates sequentially so earlier trades in a batch reduce the budget for later ones.

**MD (Managing Director)** — Orchestrates the daily pipeline, formats and sends Telegram briefings, handles trade execution through Alpaca's API, and provides a kill switch. The only agent that touches external systems.

---

## Strategy: Dual Momentum

The scoring methodology combines two well-documented momentum signals:

### Absolute Momentum (Trend Filter)
Is the ETF above its 40-week moving average (~200 trading days)?

- Above by >2%: **+2 points**
- Above by 0-2%: **+1 point**
- Below by 0-2%: **-1 point**
- Below by >2%: **-2 points**

This is the trend filter. If an asset is below its long-term average, we don't buy it regardless of relative performance. This rule alone would have kept you out of the 2008 crash and the early 2020 drawdown.

### Relative Momentum (Cross-Sectional Rank)
Rank all 12 ETFs by their 52-week return minus the most recent 4-week return (the standard Jegadeesh-Titman formulation — stripping recent return removes the short-term reversal effect).

- Rank 1 (best momentum): **+3 points**
- Rank 12 (worst momentum): **-3 points**
- Linear interpolation between

### Composite Score
`score = absolute_score + rank_score` → range of **-5 to +5**

- **BUY:** score ≥ 3 (above trend + top momentum)
- **HOLD:** score 0 to 2.5
- **SELL:** score < 0

### Confidence
High when absolute and relative signals agree (both bullish or both bearish). Low when they conflict. The Allocator uses confidence for position sizing decisions.

### Academic Backing
- Jegadeesh & Titman (1993): Original momentum factor documentation
- Asness, Moskowitz, Pedersen (2013): "Value and Momentum Everywhere" — works across asset classes
- Antonacci (2012): Global Equitable Momentum (GEM) — the absolute + relative combination
- Faber (2007): 200-day SMA trend filter applied to asset allocation

---

## ETF Universe

12 ETFs chosen for maximum return driver diversity. Momentum only works when there's dispersion to exploit — 5 tech stocks all move together, but gold and emerging markets don't.

| # | Ticker | Exposure | Return Driver |
|---|--------|----------|---------------|
| 1 | SPY | US Large Cap | Domestic corporate earnings |
| 2 | IWM | US Small Cap | Domestic small-biz cycle |
| 3 | QQQ | US Tech | Tech/growth sector |
| 4 | IWD | US Value | Value factor tilt |
| 5 | EFA | Intl Developed | Non-US developed economies |
| 6 | EEM | Emerging Markets | EM growth + commodities |
| 7 | AGG | US Bonds | Interest rates / credit |
| 8 | TLT | Long-term Treasury | Duration / rate expectations |
| 9 | TIP | TIPS | Inflation expectations |
| 10 | GLD | Gold | Inflation hedge / risk-off |
| 11 | DBC | Broad Commodities | Global commodity demand |
| 12 | VNQ | Real Estate (REITs) | Real estate cycle |

The strategy holds the top 3-4 BUY signals at any time. In a risk-off regime, momentum naturally rotates into bonds and gold. In risk-on, it rotates into equities. The "go to cash" rule (nothing passes absolute momentum) provides crash protection.

---

## Risk Guardrails

Every trade passes through six hard checks. No exceptions, no overrides.

| Check | Limit | Rationale |
|-------|-------|-----------|
| Single trade size | ≤ $400 | No outsized single bets |
| Daily total spend | ≤ $500 | Pace deployment over multiple days |
| Total capital deployed | ≤ $2,000 | Hard cap on exposure |
| Position concentration | ≤ 20% of account | No single-name risk |
| Daily drawdown | ≤ $100 | Circuit breaker |
| Signal sanity | Cannot buy a SELL signal | Prevent analyst/allocator disagreement |

Trades are validated **sequentially** — if trade 1 uses $380 of the $500 daily budget, trade 2 only has $120 available. This prevents the Allocator from blowing through limits by batching.

Additionally: wash sale detection (30-day IRS window), holding period tracking (short-term vs long-term capital gains), and portfolio concentration warnings.

---

## Infrastructure

```
Hetzner VPS (Ubuntu 24.04)
├── OpenClaw 2026.2.19-2 (AI assistant framework)
│   └── Telegram bot: @justate_bot (paired, working)
├── Alpaca Paper Trading (account PA3710DUFUQP, $100k balance)
├── yfinance (market data, no API key needed)
└── Cron
    ├── Alpaca sync: every 5 min during market hours
    └── MD briefing: 7am PT weekdays
```

### File System

```
/root/.openclaw/workspace/
├── portfolio.json          # Live shared state (positions, limits, watchlist)
├── signals.json            # Analyst output (12 ETF scores)
├── proposals.json          # Allocator output (sized trade proposals)
├── trade_log.json          # Execution history (for wash sale tracking)
├── md_state.json           # MD state (pending proposals, halt status)
└── skills/
    ├── analyst.py           # Dual momentum scorer
    ├── allocator.py         # Trade proposal generator
    ├── guardrails.py        # Validation layer
    ├── tax_risk.py          # Wash sale + holding period checker
    ├── md.py                # Orchestrator + Telegram + execution
    └── alpaca_sync.py       # Portfolio sync with Alpaca API
```

### Data Flow

```
yfinance → analyst.py → signals.json
                              ↓
portfolio.json + signals.json → allocator.py → proposals.json
                                     ↓
                              guardrails.py (validate)
                                     ↓
                              md.py → Telegram briefing
                                     ↓
                           human confirms → Alpaca API → trade_log.json
                                                              ↓
                                                     alpaca_sync.py → portfolio.json
```

---

## Telegram Interface

Daily briefing arrives at 7am PT with signals, proposals, and portfolio summary:

```
📊 Mini Hedge Fund — Daily Briefing
Wednesday, February 25 2026 — 07:00 AM PT

🔄 Portfolio synced with Alpaca
📈 Analyst complete: 4 BUY, 5 HOLD, 3 SELL
💰 Allocator complete: 4 approved buys, 0 sells

Signals:
  🟢 GLD: BUY (score: 5.0, mom: +72.6%)
  🟢 EEM: BUY (score: 4.5, mom: +40.1%)
  🟢 IWM: BUY (score: 3.9, mom: +29.9%)
  🟢 EFA: BUY (score: 3.4, mom: +26.4%)
  🟡 QQQ: HOLD (score: 2.8, mom: +24.5%)
  🟡 SPY: HOLD (score: 2.3, mom: +21.2%)
  ... and 6 more

Proposed Trades:
  🟢 BUY GLD $125.00
  🟢 BUY EEM $125.00
  🟢 BUY IWM $125.00
  🟢 BUY EFA $125.00

Total buy: $500.00

Reply approve to execute, reject to skip.

Portfolio: $0.00 deployed of $2,000 max
Positions: none
```

### Commands

| Command | Action |
|---------|--------|
| `briefing` | Run full pipeline and send briefing |
| `status` | Portfolio overview |
| `signals` | Latest analyst scores |
| `proposals` | Current trade proposals |
| `approve` | Execute all pending trades |
| `approve GLD` | Execute specific ticker |
| `reject` | Reject pending proposals |
| `halt trading` | Kill switch — stops everything |
| `resume trading` | Re-enable after halt |
| `run analyst` | Manually trigger analyst |
| `run allocator` | Manually trigger allocator |

---

## Deployment Timeline

The system deploys capital gradually due to the $500/day spending limit:

| Day | Action | Cumulative Deployed |
|-----|--------|-------------------|
| 1 | $125 × 4 ETFs | $500 |
| 2 | $125 × 4 ETFs | $1,000 |
| 3 | $125 × 4 ETFs | $1,500 |
| 4 | $125 × 4 ETFs | $2,000 (fully deployed) |

After full deployment, the system monitors daily and proposes rebalancing trades when momentum signals shift (e.g., an ETF drops from BUY to SELL → sell it, buy the new top-ranked BUY).

---

## Roadmap

- [x] Analyst agent (dual momentum scoring)
- [x] Guardrails engine (6 hard checks)
- [x] Tax/Risk agent (wash sale, holding periods)
- [x] Allocator agent (constraint-aware sizing)
- [x] MD agent (orchestrator, Telegram, execution)
- [x] Daily cron scheduling
- [ ] OpenClaw skill wrapper (approve/reject directly in Telegram)
- [ ] Performance tracking dashboard
- [ ] 30-day confirm mode validation
- [ ] Auto mode (after validation)
- [ ] Multi-strategy support (add mean reversion, carry)
- [ ] Expanded universe (sector ETFs, international bonds)

---

## Disclaimer

This is a paper trading system for educational and research purposes. It trades with simulated money on Alpaca's paper trading platform. Nothing here is investment advice. Past performance of momentum strategies does not guarantee future results. The academic research cited describes historical patterns that may not persist.

---

## License

MIT
