---
name: quick-stock-analysis
description: Perform a fast 30-second stock screen using a 5-step decision tree (revenue growth, P/E, PEG, ROE, quick ratio) against the FMP connector, score it out of 5, and write the result to a markdown file. All five gates are always evaluated, so a stock that misses one still gets a score and a breakdown. Use when the user asks for a "quick analysis", "quick stock check", "30-second analysis", or wants a fast go/no-go verdict on a stock before doing deeper work.
metadata:
  version: 2.0.0
---

# Quick Stock Analysis

A fast, opinionated screen that answers one question: **should I bother doing deeper fundamental analysis on this stock?** It runs through 5 gates. **All five gates are always evaluated** — a failing gate never ends the screen — and the outcome is reported as a score out of five (e.g. `4/5`) alongside the per-gate breakdown, because a stock that misses one gate can still be interesting. Only a 5/5 earns an "Invest" verdict.

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

Evaluate **all five gates on every run**, in order. Never short-circuit on a failure: a
stock that fails gate 1 but clears the other four is a very different proposition from one
that fails everything, and only a complete scorecard shows the difference.

### Gate 1 — Revenue Growth

- **Question:** Has revenue grown by at least 10% every year for the last 3–5 years?
- **Metric:** YoY revenue growth rate (annual)
- **Pass:** Every year ≥ 10%
- **Fail reason:** `Low revenue growth`

### Gate 2 — P/E Ratio

- **Question:** Is the trailing P/E ratio below 25?
- **Metric:** Trailing twelve-month P/E
- **Pass:** P/E < 25 (and positive — negative P/E fails this gate)
- **Fail reason:** `Likely overvalued`

### Gate 3 — PEG Ratio

- **Question:** Is the PEG ratio below 2?
- **Metric:** PEG = P/E ÷ EPS growth rate (%)
- **Pass:** 0 < PEG < 2
- **Fail reason:** `Low profit growth`

### Gate 4 — Return on Equity

- **Question:** Has the company's ROE been over 5% on average for the last 5 years?
- **Metric:** 5-year average ROE
- **Pass:** Average ROE > 5%
- **Fail reason:** `Weak profitability`

### Gate 5 — Quick Ratio

- **Question:** Is the quick ratio above 1.5?
- **Metric:** Quick Ratio = (Current Assets − Inventory) ÷ Current Liabilities
- **Pass:** Quick Ratio > 1.5
- **Fail reason:** `Liquidity issues`

### Score and Final Verdict

Count the gates that PASS — `UNKNOWN` never counts as a pass — and report the total as
`<passed>/5`.

| Score | Verdict | Meaning |
| --- | --- | --- |
| 5/5 | **INVEST** | Worth deeper analysis. Run `full-stock-analysis` next. |
| 4/5 | **WATCH** | Misses one gate but may still be interesting. Name the failing gate and what would have to change. |
| 3/5 | **WATCH (weak)** | Two misses. Worth deeper work only if both are explainable — e.g. a high P/E on a genuine compounder. |
| 0–2/5 | **REJECT** | Too many misses to justify deeper work. |

If any gate is UNKNOWN, append `INCONCLUSIVE` to the verdict and say which gates could not
be evaluated. A 4/5 with one UNKNOWN is not the same as a 4/5 with one FAIL — never
present them as equivalent.

## Output

Write the result to `./Stock Screener/<TICKER>.md` in the current working directory. Use the template below. Fill every field — use `N/A` or `UNKNOWN` when data is genuinely missing. `invest` is `true` only at 5/5; `score`, `positive-gates` and `verdict` always carry the full picture, so a `4/5` stays discoverable even though `invest` is `false`.

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
score: <passed>/5
positive-gates: <number of gates passed, 0-5>
verdict: <INVEST | WATCH | WATCH (weak) | REJECT>
---

# Quick Stock Analysis: <TICKER> (<Company Name>)

**Score: <passed>/5 — <VERDICT>**

<Quick description of the Stock / Equity>

## Decision Tree Results

| #   | Gate           | Threshold    | Value   | Result            |
| --- | -------------- | ------------ | ------- | ----------------- |
| 1   | Revenue Growth | ≥ 10% yearly | <value> | PASS/FAIL/UNKNOWN |
| 2   | P/E Ratio      | < 25         | <value> | PASS/FAIL/UNKNOWN |
| 3   | PEG Ratio      | < 2          | <value> | PASS/FAIL/UNKNOWN |
| 4   | ROE (5y avg)   | > 5%         | <value> | PASS/FAIL/UNKNOWN |
| 5   | Quick Ratio    | > 1.5        | <value> | PASS/FAIL/UNKNOWN |

**Score: <passed>/5** — failing gates: <list, or "none">.

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

<2-4 sentences explaining the score. Name every failing gate and what would need to change
for each, and call out the strongest metric. For 5/5 or 4/5, recommend running the
`full-stock-analysis` skill next. For 3/5, say whether the two misses look explainable.
For 0-2/5, say what to watch for a future revisit.>

## Next Step

- 5/5 or 4/5 → run `full-stock-analysis` on <TICKER> for DCF and peer comparison.
- 3/5 → optional deeper look; decide on the merits of <the failing gates>.
- 0-2/5 → skip deeper analysis; revisit if <the failing metrics> improve.

---

_Data: FMP connector, fiscal years <first>–<last>. Quote as of <date>._
_Quick screen only — not a buy/sell recommendation. Not financial advice._
```

## Workflow

1. Confirm the ticker with the user if not provided.
2. **Fetch the data** — make the four required FMP calls above. This step is mandatory and must complete before any gate is evaluated.
3. Derive **every** gate value using the derivation table and evaluate all five gates. Do not stop early on a FAIL.
4. Count the passes and map the score to a verdict per the score table. Add `INCONCLUSIVE` if any gate is UNKNOWN.
5. Write the markdown file using the template above.
6. Print the score (`<passed>/5`), the verdict, the failing gates, and the file path in the chat response.

## Anti-Patterns

- **Don't skip gates** because "the company is famous." The whole point is a mechanical screen.
- **Don't short-circuit the decision tree.** All five gates get calculated on every run, even when gate 1 fails outright — the one-gate miss is often the interesting case.
- **Don't fudge missing data.** If revenue growth for one year is missing, mark UNKNOWN — do not interpolate.
- **Don't substitute a web search for a failed FMP call.** A scraped number is not comparable to the rest of the table. Mark it UNKNOWN.
- **Don't mix period-end and live figures silently.** The quote P/E is live; `metrics-ratios` P/E is as of the fiscal year end. Say which you used.
- **Don't treat INVEST as a buy signal.** It's a "worth deeper research" flag, nothing more.
- **Don't collapse the score into a bare pass/fail.** `4/5` and `1/5` are both "not INVEST" and are not remotely the same thing; always show the score and which gates failed.
- **Don't apply this to banks, insurers, REITs, or pre-profit growth stocks.** Quick ratio and P/E are misleading for financials; ROE and P/E are misleading for unprofitable growth. If the user requests one of these, note the limitation and suggest `full-stock-analysis` directly instead.
