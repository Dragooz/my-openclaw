---
name: bursa-scout
description: Scout top 10 Bursa Malaysia small-cap stocks via a staged pipeline. Activated when user says "scout stage 1", "run bursa scout", or "scout [CODE]". The agent executes each stage directly using yfinance, web_fetch, and reasoning — no external binary needed.
---

## ⚠️ Agent Execution Model

You (the LLM agent) ARE the pipeline executor. There is no `bursa-scout` binary or CLI tool.
When the user says "run bursa scout" or "scout stage N", you directly:

1. Run Python via the `exec` tool: `pip install yfinance -q && python3 -c "..."`
2. Fetch web sources via the `web_fetch` tool
3. Reason and format output yourself
4. Send Telegram via the available channel tool (or print if unavailable)

Never attempt to exec a binary called `bursa-scout`. Never look for a cron job.
Parse "scout stage 1" as a trigger phrase — not a shell command.

# Bursa Scout — Staged Pipeline

> **Why stages?** Local LLMs (gemma4:e4b) time out on long pipelines. Each stage is capped at ~90 seconds of LLM work. Run them in sequence. Each stage sends a Telegram update and hands off a compact state blob to the next stage.

---

## How to Run

```
Stage 1: "scout stage 1"              → Regime check + screen candidates
Stage 2: "scout stage 2"              → Deep dive batch A (stocks 1–5)
Stage 3: "scout stage 3"              → Deep dive batch B (stocks 6–10)
Stage 4: "scout stage 4"              → Score + rank all candidates
Stage 5: "scout stage 5"              → Final Telegram report
```

Each stage reads the **state blob** printed at the end of the previous stage. Paste it at the start of the next prompt like:

```
scout stage 2
STATE: {"regime":"Offensive","opr":3.0,"usdmyr":4.72,"candidates":["1818.KL","0023.KL",...]}
```

---

## Data Sources

### APIs (prefer first)

| Source                   | What it gives                                           | How                                                   |
| ------------------------ | ------------------------------------------------------- | ----------------------------------------------------- |
| `yfinance` `.KL` tickers | Price, market cap, PE, EPS, revenue, FCF, balance sheet | `pip install yfinance` · `yf.Ticker("1818.KL")`       |
| BNM OpenAPI              | MYR/USD rate, OPR                                       | `https://apikijangportal.bnm.gov.my/openapi` — no key |

```python
import yfinance as yf
t = yf.Ticker("CODE.KL")
info = t.info
hist = t.history(period="1y")
fin  = t.income_stmt
cf   = t.cashflow
```

### Web Fetch (fallback only)

| Site                  | When                                                   |
| --------------------- | ------------------------------------------------------ |
| `klse.i3investor.com` | Director transactions, substantial shareholder filings |
| `theedgemalaysia.com` | News, catalysts, macro headlines                       |
| `klsescreener.com`    | If yfinance can't filter by PE/sector                  |

Rule: **fetch only what yfinance couldn't provide.**

---

## Stage 1 — Regime + Screen

**Scope:** Market regime classification + candidate shortlist. No deep analysis yet.
**Time budget:** ~60–80s.

### What to do

**1a. Regime (via API):**

- `yf.Ticker("^KLSE").history(period="1y")` → KLCI vs 200MA
- BNM OpenAPI `/interest-rate` → OPR
- BNM OpenAPI `/exchange-rate` → MYR/USD
- Web fetch theedgemalaysia only if KLCI signal is ambiguous

Classify: **Offensive / Cautious / Defensive**

- Offensive: KLCI above 200MA, stable OPR, MYR not in freefall
- Cautious: mixed (e.g. KLCI above MA but OPR hiking)
- Defensive: KLCI below 200MA or MYR > 4.80 or acute macro risk

**1b. Screen (via API + fallback):**
Filter `.KL` tickers:

- Market cap: RM50M–500M
- Volume: >200K daily avg
- PE: below sector average
- Revenue/earnings: 2+ consecutive quarters growth
- Analyst coverage: ≤2 (underfollowed)
- No active legal/regulatory issues

Target: **10–15 candidates** max. If <5 pass, note it and widen PE filter to 120% of sector avg.

### Telegram output (send after stage completes)

```
📊 BURSA SCOUT — {DATE} | Stage 1/5 ✅
Regime: {Offensive/Cautious/Defensive}
OPR: {X}% | MYR/USD: {X}
KLCI: {above/below} 200MA

Candidates shortlisted: {N}
{CODE} · {CODE} · {CODE} · ...

⏭ Run: scout stage 2
STATE: {compact JSON blob below}
```

### State blob to print (paste into next prompt)

```json
{
    "date": "YYYY-MM-DD",
    "regime": "Offensive",
    "min_score": 7,
    "opr": 3.0,
    "usdmyr": 4.72,
    "klci_trend": "above_200ma",
    "candidates": ["CODE1.KL", "CODE2.KL", "...up to 15"]
}
```

---

## Stage 2 — Deep Dive Batch A (Candidates 1–5)

**Scope:** Full 7-lens analysis on first 5 candidates only.
**Time budget:** ~80–90s.
**Input:** STATE blob from Stage 1.

### What to do

For each of candidates[0..4], run all 7 lenses:

**1. Fundamentals** (yfinance)

- Revenue trend 3Y, margin direction, FCF vs reported profit, debt/equity
- Hard fail: revenue declining + margins shrinking + negative FCF simultaneously

**2. Moat** (reasoning from fundamentals + web fetch if needed)

- What protects the business? Score higher if durable and hard to replicate.

**3. Catalyst** (web fetch: i3investor, theedge)

- Specific 2–8 week trigger. Hard fail: none identifiable.

**4. Smart Money** (web fetch: i3investor substantial shareholder changes)

- Institutional buying, director buy > sell, unusual volume spike.

**5. Red Flags** (web fetch: i3investor announcements)

- RPT, frequent placements, auditor change, late filings.
- Hard fail: active Bursa/SC investigation.

**6. Technicals** (yfinance hist)

- 20/50/200MA, RSI (compute from close), volume confirmation, entry zone, stop loss.

**7. Peers** (yfinance, same sector)

- PE/PB vs 2–3 sector peers. Genuinely cheap or cheap for a reason?

For hard-fail stocks: log reason and skip — do not include in output.

### Telegram output

```
📊 BURSA SCOUT — Stage 2/5 ✅ (Batch A)
Analysed: {CODE}, {CODE}, {CODE}, {CODE}, {CODE}
Hard fails: {CODE} ({reason}) / None

Quick view:
{CODE} — {score}/13 — {one-line verdict}
{CODE} — {score}/13 — {one-line verdict}
...

⏭ Run: scout stage 3
STATE: {updated blob}
```

### State blob to print

```json
{
    "date": "YYYY-MM-DD",
    "regime": "Offensive",
    "min_score": 7,
    "opr": 3.0,
    "usdmyr": 4.72,
    "klci_trend": "above_200ma",
    "candidates": ["...remaining for stage 3"],
    "scored": [
        {
            "code": "CODE1.KL",
            "name": "Company Name",
            "score": 10,
            "hard_fail": false,
            "cap_rm": 120,
            "pe": 8.2,
            "sector_pe": 12.0,
            "fcf": "positive",
            "moat": "one-line",
            "catalyst": "contract win Q3, ~4wks",
            "smart_money": "buying",
            "flags": "none",
            "ma50": "above",
            "rsi": 54,
            "entry": "0.85-0.90",
            "stop": "0.78",
            "target": "1.10",
            "upside_pct": 25,
            "rr": "2.5:1",
            "verdict": "two sentence plain English",
            "play": "Buy at 0.88"
        }
    ]
}
```

---

## Stage 3 — Deep Dive Batch B (Candidates 6–10+)

**Scope:** Same 7-lens analysis on remaining candidates.
**Time budget:** ~80–90s.
**Input:** STATE blob from Stage 2.

### What to do

Repeat Stage 2 process for `candidates` array remaining in the state blob (candidates that haven't been scored yet).

### Telegram output

```
📊 BURSA SCOUT — Stage 3/5 ✅ (Batch B)
Analysed: {CODE}, {CODE}, ...
Hard fails: {CODE} ({reason}) / None

Quick view:
{CODE} — {score}/13 — {one-line verdict}
...

⏭ Run: scout stage 4
STATE: {updated blob with all scored candidates merged}
```

### State blob to print

Same structure as Stage 2 blob but `scored` array now contains **all** candidates (batches A + B merged). `candidates` array should now be empty `[]`.

---

## Stage 4 — Score + Rank

**Scope:** Apply scoring rules, sort, cut to top 10, classify watchlist.
**Time budget:** ~30–40s. No new API calls — pure reasoning on the state blob.
**Input:** STATE blob from Stage 3.

### Scoring

Score out of 13 (2 pts each × 6 lenses + 1 pt peers):

- Fundamentals: 0–2
- Moat: 0–2
- Catalyst: 0–2
- Smart Money: 0–2
- Red Flags: 0–2 (2 = clean, 0 = hard fail already excluded)
- Technicals: 0–2
- Peers: 0–1

Grade thresholds (apply regime-adjusted min):

- A: 11–13
- B: 8–10
- C: 7 (Offensive regime only)
- Reject: <7, or <8 in Cautious, or <9 in Defensive

Split output into:

- **Top 10** (highest scorers that pass regime threshold)
- **Watchlist** (scored 5–6, not ready — needs catalyst or price pullback)

### Telegram output

```
📊 BURSA SCOUT — Stage 4/5 ✅ Ranking done
Top pick: {CODE} ({X}/13)
Qualified: {N} stocks

Ranking:
1. {CODE} — {X}/13 — Grade {A/B}
2. {CODE} — {X}/13 — Grade {A/B}
...

Watchlist: {CODE}, {CODE}

⏭ Run: scout stage 5
STATE: {updated blob}
```

### State blob to print

```json
{
  "date": "YYYY-MM-DD",
  "regime": "Offensive",
  "opr": 3.0,
  "usdmyr": 4.72,
  "top10": ["CODE1.KL", "CODE2.KL", "...ranked order"],
  "watchlist": ["CODE11.KL", "CODE12.KL"],
  "scored": [ ... same scored objects, now with rank field added ... ]
}
```

---

## Stage 5 — Final Telegram Report

**Scope:** Format and send the full human-readable report. Pure formatting, no new API calls.
**Time budget:** ~40–60s.
**Input:** STATE blob from Stage 4.

### What to do

Format the complete Telegram-ready report from the state blob. No new data fetching.

### Full Telegram Output Format

**Header:**

```
📊 BURSA SCOUT — {DATE}
Regime: {Offensive/Cautious/Defensive}
OPR: {X}% | MYR/USD: {X}
Qualified: {N}/candidates screened
─────────────────────
```

**Per stock (repeat × 10):**

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

{SIGNAL} {plain-English verdict — 2 sentences max}
Play: {Buy at X / Wait for X / Avoid}
─────────────────────
```

Where `{SIGNAL}` = 🟢 strong buy · 🟡 watch · 🔴 avoid

**Footer:**

```
👀 Watchlist (scored 5–6, not ready):
· {CODE} — {one-line reason}
· {CODE} — {one-line reason}

Scout done ✅ Top pick: {CODE} ({X}/13).
```

---

## Scout Rules (All Stages)

- Never include a stock with a hard fail, even if score ≥7
- Flag missing data inline ("FCF: no data") — do not skip the stock
- Keep each stage output to ONE Telegram message (< 4096 chars). If it overflows, split into 2 messages and label "1/2", "2/2"
- Friday: compare open positions vs today's scouts
- Monday: refresh Stage 1 + 4 only (screen + rank, skip deep dives unless regime changed)

---

## Quick Reference — Trigger Phrases

| You say                 | What runs                                   |
| ----------------------- | ------------------------------------------- |
| `scout stage 1`         | Regime + screen                             |
| `scout stage 2` + STATE | Deep dive stocks 1–5                        |
| `scout stage 3` + STATE | Deep dive stocks 6–10                       |
| `scout stage 4` + STATE | Score + rank                                |
| `scout stage 5` + STATE | Final report                                |
| `scout [CODE]`          | Single stock: runs all 7 lenses in one shot |
| `scout resume` + STATE  | Re-run from last completed stage            |
