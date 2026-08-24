---
name: quick-stock-analysis
description: Perform a fast 30-second stock screen using a 5-step decision tree (revenue growth, P/E, PEG, ROE, quick ratio) against the FMP connector and write the result to a markdown file. Use when the user asks for a "quick analysis", "quick stock check", "30-second analysis", or wants a fast go/no-go verdict on a stock before doing deeper work.
metadata:
  version: 2.0.0
---

# Quick Stock Analysis

A fast, opinionated screen that answers one question: **should I bother doing deeper fundamental analysis on this stock?** It runs through a 5-gate decision tree. If any gate fails, the stock is rejected with a specific reason. Only stocks that pass all 5 gates earn an "Invest" verdict.

**The stock symbol MUST be provided by the user. If not, ask for it.**

This is a screening tool, not an investment decision. A passing verdict means "worth a closer look" — always follow up with the `full-stock-analysis` skill before committing capital.

## Data Access

**All data comes from the FMP MCP connector.** Do not use WebSearch, a scraped page, or a
local library for any gate value — if FMP cannot supply a field, the field is `UNKNOWN`.
Invoke the `market-data` skill if you need the exact endpoint or field name for anything
below.

Make these four calls before evaluating any gate:

```
1. statements(endpoint="income-statement",        symbol=<TICKER>, period="annual", limit=5)
2. statements(endpoint="metrics-ratios",          symbol=<TICKER>, period="annual", limit=5)
3. statements(endpoint="balance-sheet-statement", symbol=<TICKER>, period="annual", limit=5)
4. quote(endpoint="quote", symbol=<TICKER>)
```

Two optional calls, used only where noted:

```
5. company(endpoint="profile-symbol",       symbol=<TICKER>)   # industry + description for frontmatter
6. analyst(endpoint="financial-estimates",  symbol=<TICKER>, period="annual")   # forward EPS growth for PEG
```

Rows come back newest-first. If call 1, 2, or 4 fails or returns an empty array, report
the error to the user and stop — the screen is not meaningful without them.

**If the ticker is ambiguous or unrecognised,** resolve it first with
`search(endpoint="search-symbol", query=<user input>)` and confirm the match with the
user before screening.

## Deriving the Gate Values

| Gate | Source | Derivation |
| --- | --- | --- |
| 1. Revenue growth | call 1 | For each adjacent pair of annual `revenue` values: `(newer − older) / older`. Yields 4 YoY rates from 5 statements. |
| 2. P/E | call 4 | `pe` from the live quote. If null, `price / epsDiluted` (TTM) from calls 4 + 1. |
| 3. PEG | calls 4 + 1/6 | `P/E ÷ EPS growth rate in percent`. Prefer forward EPS growth from call 6; otherwise use the trailing CAGR of `epsDiluted` across the 5 annual statements. **Record which one you used.** |
| 4. ROE (5y avg) | call 2 | Arithmetic mean of `returnOnEquity` across the returned annual rows. Note the row count if fewer than 5. |
| 5. Quick ratio | call 2 | `quickRatio` from the most recent annual row. If null, compute from call 3: `(totalCurrentAssets − inventory) ÷ totalCurrentLiabilities`. |

FMP returns ratios as **fractions** (`0.147` = 14.7%). Convert before comparing against a
percentage threshold, and display them as percentages.

## Decision Tree

Walk through the gates in order. **Stop at the first failure** — do not evaluate later gates once one fails, but still record all gathered data in the result file.

### Gate 1 — Revenue Growth

- **Question:** Has revenue grown by at least 10% every year for the last 3–5 years?
- **Metric:** YoY revenue growth rate (annual)
- **Pass:** Every year ≥ 10%
- **Fail verdict:** `STOP — Low revenue growth`

### Gate 2 — P/E Ratio

- **Question:** Is the trailing P/E ratio below 25?
- **Metric:** Trailing twelve-month P/E
- **Pass:** P/E < 25 (and positive — negative P/E fails this gate)
- **Fail verdict:** `Likely overvalued`

### Gate 3 — PEG Ratio

- **Question:** Is the PEG ratio below 2?
- **Metric:** PEG = P/E ÷ EPS growth rate (%)
- **Pass:** 0 < PEG < 2
- **Fail verdict:** `Low profit growth`

### Gate 4 — Return on Equity

- **Question:** Has the company's ROE been over 5% on average for the last 5 years?
- **Metric:** 5-year average ROE
- **Pass:** Average ROE > 5%
- **Fail verdict:** `Weak profitability`

### Gate 5 — Quick Ratio

- **Question:** Is the quick ratio above 1.5?
- **Metric:** Quick Ratio = (Current Assets − Inventory) ÷ Current Liabilities
- **Pass:** Quick Ratio > 1.5
- **Fail verdict:** `Liquidity issues`

### Final Verdict

If all 5 gates pass → **INVEST (worth deeper analysis)**

## Output

Write the result to `./Stock Screener/<TICKER>.md` in the current working directory. Use the template below. Fill every field — use `N/A` or `UNKNOWN` when data is genuinely missing.

_If the Stock is pre-existing, update the Note!_.

```markdown
---
tags:
  - stock
  - quick-analysis
date: <YYYY-MM-DD>
current-price: <price>
industry: <industry>
invest: <true/false>
positive-gates: <number of successfull gates>
---

# Quick Stock Analysis: <TICKER> (<Company Name>)

<Quick description of the Stock / Equity>

## Decision Tree Results

| #   | Gate           | Threshold    | Value   | Result            |
| --- | -------------- | ------------ | ------- | ----------------- |
| 1   | Revenue Growth | ≥ 10% yearly | <value> | PASS/FAIL/UNKNOWN |
| 2   | P/E Ratio      | < 25         | <value> | PASS/FAIL/UNKNOWN |
| 3   | PEG Ratio      | < 2          | <value> | PASS/FAIL/UNKNOWN |
| 4   | ROE (5y avg)   | > 5%         | <value> | PASS/FAIL/UNKNOWN |
| 5   | Quick Ratio    | > 1.5        | <value> | PASS/FAIL/UNKNOWN |

## Supporting Data

### Revenue (last 5 years)

| Year | Revenue | YoY Growth |
| ---- | ------- | ---------- |
| ...  | ...     | ...        |

### Valuation

- Trailing P/E: <value>
- Forward P/E: <value>
- PEG: <value>
- EPS growth rate used for PEG: <value> (<forward estimate | trailing 5y CAGR>)

### Profitability

- ROE (current): <value>
- ROE (5y average): <value>
- Net margin: <value>

### Liquidity

- Current ratio: <value>
- Quick ratio: <value>
- Cash & equivalents: <value>

## Interpretation

<2-4 sentences explaining the result. If REJECT, explain which gate killed it and
what would need to change. If INVEST, call out the strongest metric and recommend
running the `full-stock-analysis` skill next.>

## Next Step

- If INVEST → run `full-stock-analysis` on <TICKER> for DCF and peer comparison.
- If REJECT → skip deeper analysis; revisit if <the failing metric> improves.

---

_Data: FMP connector, fiscal years <first>–<last>. Quote as of <date>._
_Quick screen only — not a buy/sell recommendation. Not financial advice._
```

## Workflow

1. Confirm the ticker with the user if not provided.
2. **Fetch the data** — make the four required FMP calls above. This step is mandatory and must complete before any gate is evaluated.
3. Derive each gate value using the derivation table, then evaluate the gates in order, stopping at the first FAIL.
4. Determine verdict: INVEST only if all 5 gates PASS. Any FAIL → REJECT. Any UNKNOWN on an otherwise-passing path → REJECT with `INCONCLUSIVE` note.
5. Write the markdown file using the template above.
6. Print the verdict and file path in the chat response.

## Anti-Patterns

- **Don't skip gates** because "the company is famous." The whole point is a mechanical screen.
- **Don't fudge missing data.** If revenue growth for one year is missing, mark UNKNOWN — do not interpolate.
- **Don't substitute a web search for a failed FMP call.** A scraped number is not comparable to the rest of the table. Mark it UNKNOWN.
- **Don't mix period-end and live figures silently.** The quote P/E is live; `metrics-ratios` P/E is as of the fiscal year end. Say which you used.
- **Don't treat INVEST as a buy signal.** It's a "worth deeper research" flag, nothing more.
- **Don't apply this to banks, insurers, REITs, or pre-profit growth stocks.** Quick ratio and P/E are misleading for financials; ROE and P/E are misleading for unprofitable growth. If the user requests one of these, note the limitation and suggest `full-stock-analysis` directly instead.
