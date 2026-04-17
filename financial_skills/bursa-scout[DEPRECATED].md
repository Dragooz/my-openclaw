---
name: bursa-scout
description: Scout top 10 Bursa Malaysia small-cap stocks. Runs full pipeline: regime check → screen → deep dive → score → ranked summary with plain-English verdict. Trigger with "scout top 10 stocks", "run bursa scout", or on schedule.
version: 2.1.0
---

# Bursa Scout

Scout the top 10 Bursa Malaysia small-cap stocks. One command gives you deep analysis + plain-English verdict for each.

## Trigger phrases

- "scout top 10 stocks" / "run bursa scout" / "bursa scout now"
- "screen bursa [theme/sector]"
- "scout [stock code]" — single stock deep dive
- Scheduled: Friday position review / Monday watchlist refresh

---

## Data Sources

Use in priority order — API first, web fetch only if API can't provide it.

### APIs (free, prefer these)

| Source                   | What it gives                                                   | How                                                          |
| ------------------------ | --------------------------------------------------------------- | ------------------------------------------------------------ |
| `yfinance` `.KL` tickers | Price history, market cap, PE, EPS, revenue, FCF, balance sheet | `pip install yfinance` · `yf.Ticker("1818.KL")`              |
| BNM OpenAPI              | MYR/USD rate, OPR interest rate                                 | `https://apikijangportal.bnm.gov.my/openapi` — no key needed |

**yfinance usage pattern:**

```python
import yfinance as yf
t = yf.Ticker("CODE.KL")
info = t.info          # market cap, PE, sector, volume
hist = t.history(period="1y")   # OHLCV for technicals
fin  = t.income_stmt   # revenue, earnings (may be empty for micro-caps)
cf   = t.cashflow      # FCF check
```

Note: financial statements may be empty for very small caps — flag as "data unavailable" and rely on web fetch fallback only for those.

### Web Fetch (fallback only — use sparingly)

| Site                  | When to use                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| `klse.i3investor.com` | Shareholding changes, director transactions, announcements — no free API covers this |
| `theedgemalaysia.com` | News, catalysts, sector narrative                                                    |
| `klsescreener.com`    | Screener results if yfinance can't filter by PE/sector                               |

Rule: **fetch only what the API couldn't provide.** Do not re-fetch data already obtained from yfinance.

---

## Pipeline

### Step 1 — Regime Gate

Classify market as **Offensive / Cautious / Defensive** before scouting.

**API calls:**

- `yf.Ticker("^KLSE").history(period="1y")` → KLCI price vs 200MA
- BNM OpenAPI `/interest-rate` → current OPR
- BNM OpenAPI `/exchange-rate` → MYR/USD

**Web fetch (only if needed):**

- theedgemalaysia.com for recent macro/sentiment headlines

**Signals to check:**

- KLCI trend (above/below 200MA)
- Sector rotation (where is smart money flowing?)
- Macro risk: OPR direction, USD/MYR, commodity prices
- Sentiment: institutional activity

**Regime rules:**

- Offensive: KLCI uptrend, low macro risk → normal scoring (≥7 to qualify)
- Cautious: mixed signals → raise bar to ≥8, prefer defensives
- Defensive: downtrend or high macro risk → ≥9 only, catalyst must be <2 weeks

Report regime at top of output before proceeding.

---

### Step 2 — Screen

**API first:** Pull a broad list of `.KL` tickers via yfinance and filter by `info` fields:

- `marketCap`: RM50M–500M
- `averageVolume`: >200K
- `trailingPE`: below sector average (compare within same `sector` field)
- `recommendationKey`: "none" or absent = underfollowed (≤2 analysts)

**Web fetch fallback:** If yfinance can't filter sufficiently, use klsescreener.com to get the initial candidate list, then pull fundamentals per-ticker via yfinance.

**Filters (all must pass):**

- Market cap: RM50M–500M (small-cap focus)
- Volume: >200K daily average
- PE: below sector average
- Revenue/earnings growth: 2+ consecutive quarters
- Analyst coverage: ≤2 (underfollowed = edge)
- No ongoing legal/regulatory issues

Produce a candidate list. If zero pass all filters, output: "No candidates passed screening today — {date}" and stop.

---

### Step 3 — Deep Dive (per candidate)

Run each candidate through all 7 lenses. Hard fails = reject immediately, no report.

**1. Fundamentals (Enron Rule)** — via yfinance

- `income_stmt`: revenue trend (3Y), margin direction
- `cashflow`: FCF vs reported profit (must match direction)
- `balance_sheet`: debt/equity, interest coverage
- Hard fail: revenue declining + margins shrinking + negative FCF simultaneously

**2. Moat** — reasoning from fundamentals + web fetch if needed

- What protects this business? (cost advantage, switching costs, niche dominance, licence/concession)
- Score higher if moat is durable and hard to replicate

**3. Catalyst (2–8 week horizon)** — web fetch: i3investor announcements, theedge news

- Specific near-term trigger: contract win, earnings release, product launch, sector tailwind
- Hard fail: no identifiable catalyst

**4. Smart Money** — web fetch: i3investor substantial shareholder changes

- Institutional/fund buying (shareholding changes, substantial shareholder filings)
- Director buying > selling is a positive signal
- Unusual volume spike (yfinance `hist` volume) without news = watch closely

**5. Red Flags** — web fetch: i3investor announcements + Bursa disclosures

- Related-party transactions, frequent private placements, auditor changes, late filings
- Promoter/insider selling heavily
- Hard fail: any active Bursa/SC investigation

**6. Technicals** — via yfinance `hist`

- Compute 20/50/200MA from OHLCV history
- Volume confirmation on breakout
- RSI (compute from close prices), support/resistance levels
- Entry zone, stop loss level

**7. Peers** — via yfinance (same `sector` group)

- Pull PE, PB for 2-3 peers in same sector
- Is candidate cheaper for a good reason, or genuinely undervalued?

---

### Step 4 — Score

Score out of 13 (2 pts each: Fundamentals, Moat, Catalyst, Smart Money, Red Flags, Technicals + 1 pt Peers).

- A: 11–13 · B: 8–10 · C: 7 (regime-dependent) · Below 7: reject

---

### Step 5 — Output (Telegram format)

Output is plain text optimised for Telegram — no markdown tables, no HTML.
Use _bold_ for labels, plain text for values, blank line between sections.

---

**Header (once, before stock list):**

```
📊 BURSA SCOUT — {DATE}
Regime: {Offensive/Cautious/Defensive}
OPR: {X}% | MYR/USD: {X}
Qualified: {N}/candidates screened
─────────────────────
```

---

**Per stock (repeat for each of top 10):**

```
#{RANK} {SIGNAL} {CODE} — {Company Name}
Score: {X}/13 (Grade {A/B/C})

Cap: RM{X}M | PE: {X}x (sector: {X}x)
Revenue: {+/-X}% QoQ | FCF: {positive/negative}
Moat: {one line}
Catalyst: {specific trigger — timeframe}
Smart $: {buying/neutral/selling}
Flags: {none / brief list}

Technicals: {above/below} 50MA, RSI {X}
Entry: RM{X}–{X} | Stop: RM{X} | Target: RM{X} ({X}% up)
R/R: {X}:1

{SIGNAL} {plain-English verdict — 2 sentences max. Why now, what's the risk.}
Play: {Buy at X / Wait for X / Avoid}
─────────────────────
```

Where `{SIGNAL}` = 🟢 strong buy · 🟡 watch · 🔴 avoid

---

**Footer (after all stocks):**

```
👀 Watchlist (scored 5–6, not ready yet):
· {CODE} — {one-line reason}
· {CODE} — {one-line reason}

Scout done. Top pick: {CODE} ({X}/13).
```

---

## Scout Rules

- Never report a stock with a hard fail, even if score ≥7
- Flag missing data inline (e.g. "FCF: no data") — do not skip the stock
- Friday: compare open positions vs today's scouts
- Monday: refresh watchlist only
