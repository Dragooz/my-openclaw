---
name: bursa-scout-data-integrity
description: Data integrity patch for bursa-scout. Prevents entry/stop/target mismatch by anchoring all price-derived fields to live fetched currentPrice. Apply alongside bursa-scout.md.
---

## Problem This Solves

Stage 2/3 can hallucinate entry zones that don't match actual market price (e.g. entry `0.45` when stock trades at `0.215`). Bad data freezes into state blob and propagates silently through Stage 4 → Stage 5 final report.

---

## Rule 1 — Fetch currentPrice FIRST in Stage 2 and 3

Before running any lens analysis on a stock, fetch live price:

```python
import yfinance as yf
t = yf.Ticker("CODE.KL")
info = t.info
price = info.get("currentPrice") or info.get("regularMarketPrice") or info.get("previousClose")
```

If all three return `None` or `0`:
- Fallback: web fetch `klsescreener.com/stock/CODE` for last price
- If still unavailable: set `"price": "no data"` in state blob, skip entry/stop/target fields, flag inline

**Never estimate or assume a price.**

---

## Rule 2 — Anchor entry/stop/target to currentPrice

Entry zone = technical support/resistance zone near current price.

```
entry_low  ≥ currentPrice * 0.85   (max 15% below current)
entry_high ≤ currentPrice * 1.10   (max 10% above current)
stop       <  entry_low            (always below entry)
target     >  currentPrice * 1.10  (must be above current price)
```

If computed entry range violates these bounds → re-check technicals. Do not force an entry that is far from current price.

---

## Rule 3 — Add `price` field to state blob

Every scored object in the state blob MUST include actual fetched price:

```json
{
  "code": "0023.KL",
  "price": 0.215,
  "entry": "0.20-0.22",
  "stop": "0.18",
  "target": "0.30",
  ...
}
```

This creates an audit trail. If `entry` and `price` diverge significantly in later stages, it's visible.

---

## Rule 4 — Stage 5 stale-data check

Before formatting each stock card in Stage 5, verify:

```
entry_low (parsed from entry field) <= price * 1.20
```

If check fails → prepend warning to that stock's card:

```
⚠️ Price data may be stale. Entry RM{entry} vs fetched price RM{price}. Verify before acting.
```

Do not suppress the stock — just warn.

---

## Rule 5 — Show currentPrice in Stage 5 report

Add `Current: RM{price}` to each stock card so mismatches are immediately visible:

```
Cap: RM{X}M | PE: {X}x (sector: {X}x) | Current: RM{price}
```

---

## Summary Checklist (Stage 2 + 3)

Before writing any stock into state blob:
- [ ] `currentPrice` fetched from yfinance (not assumed)
- [ ] `entry_low` within 15% below current price
- [ ] `stop` below `entry_low`
- [ ] `target` above current price
- [ ] `price` field written to state blob
