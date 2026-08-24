---
name: market-data
description: Reference guide for the two market-data connectors this plugin depends on — the FMP (Financial Modeling Prep) MCP connector for prices, fundamentals, news and screening, and the Interactive Brokers (IBKR) MCP connector for live brokerage positions, allocation and performance. Use when constructing a connector call, when unsure which tool or endpoint provides a metric, or when a connector is unavailable.
metadata:
  version: "1.0"
---

# Market Data Connectors

This plugin gets **all** market and portfolio data from two MCP connectors. Do not
substitute scraped web pages, `yfinance`, or any other library for these — WebSearch is
a supplement for narrative context only (news colour, management commentary), never a
source of numbers that a connector can provide.

| Need | Connector |
| ---- | --------- |
| Quotes, fundamentals, ratios, estimates, news, screening | **FMP** |
| Your actual holdings, cash, allocation, performance, watchlists | **IBKR** |

Both are configured by the user in Claude (Connectors / MCP settings), not by this
plugin. This repository intentionally ships no `.mcp.json`.

---

## FMP connector

Each FMP tool is a family: you pass an `endpoint` plus that endpoint's parameters.
Endpoint names are exact strings — do not invent them. `period` is `"annual"` or
`"quarter"`; `limit` caps the number of rows returned (rows come back newest-first).

### `quote` — live prices

```
quote(endpoint="quote", symbol="AAPL")
quote(endpoint="batch-quote", symbols=["AAPL","MSFT"])
quote(endpoint="quote-short", symbol="AAPL")
```

`quote` returns price, change, day range, 52-week range, volume, average volume,
market cap, trailing P/E, EPS, earnings announcement date, and previous close.

### `statements` — financials, ratios, metrics

```
statements(endpoint="income-statement",        symbol="AAPL", period="annual", limit=5)
statements(endpoint="balance-sheet-statement", symbol="AAPL", period="annual", limit=5)
statements(endpoint="cashflow-statement",      symbol="AAPL", period="annual", limit=5)
statements(endpoint="metrics-ratios",          symbol="AAPL", period="annual", limit=5)
statements(endpoint="key-metrics",             symbol="AAPL", period="annual", limit=5)
statements(endpoint="financial-scores",        symbol="AAPL")
statements(endpoint="income-statement-growth", symbol="AAPL", period="annual", limit=5)
```

TTM variants: `income-statements-ttm`, `balance-sheet-statements-ttm`,
`cashflow-statements-ttm`, `key-metrics-ttm`, `metrics-ratios-ttm`.

**Where the common metrics live:**

| Metric | Endpoint | Field |
| --- | --- | --- |
| Revenue | `income-statement` | `revenue` |
| Revenue growth YoY | `income-statement-growth` | `growthRevenue` |
| Net income, EPS | `income-statement` | `netIncome`, `epsDiluted` |
| Gross / operating / net margin | `metrics-ratios` | `grossProfitMargin`, `operatingProfitMargin`, `netProfitMargin` |
| ROE | `metrics-ratios` | `returnOnEquity` |
| ROIC | `key-metrics` | `returnOnInvestedCapital` |
| Current / quick ratio | `metrics-ratios` | `currentRatio`, `quickRatio` |
| Debt-to-equity, interest coverage | `metrics-ratios` | `debtToEquityRatio`, `interestCoverageRatio` |
| P/E, P/B, P/S, EV/EBITDA | `metrics-ratios` | `priceToEarningsRatio`, `priceToBookRatio`, `priceToSalesRatio`, `enterpriseValueMultiple` |
| Free cash flow | `cashflow-statement` | `freeCashFlow` |
| Cash & equivalents, inventory | `balance-sheet-statement` | `cashAndCashEquivalents`, `inventory` |
| Piotroski / Altman | `financial-scores` | `piotroskiScore`, `altmanZScore` |

**Ratios are period-end, quotes are live.** A P/E from `metrics-ratios` reflects the
price on the statement date. For a current P/E use `quote` → `pe`. Never mix the two in
one table without saying which is which.

### `company` — profile and peers

```
company(endpoint="profile-symbol", symbol="AAPL")   # sector, industry, description, beta, country
company(endpoint="peers",          symbol="AAPL")   # peer tickers for comparison
company(endpoint="market-cap",     symbol="AAPL")
```

### `analyst` — ratings and estimates

```
analyst(endpoint="ratings-snapshot",        symbol="AAPL")
analyst(endpoint="price-target-consensus",  symbol="AAPL")
analyst(endpoint="grades-summary",          symbol="AAPL")
analyst(endpoint="financial-estimates",     symbol="AAPL", period="annual")
```

Forward EPS growth for a PEG calculation comes from `financial-estimates`.

### `chart` and `technicalIndicators` — technicals

```
chart(endpoint="historical-price-eod-light", symbol="AAPL", from_date="2025-01-01", to_date="2026-01-01")
chart(endpoint="historical-price-eod-full",  symbol="AAPL", from_date="2025-06-01")

technicalIndicators(endpoint="simple-moving-average",     symbol="AAPL", periodLength=200, timeframe="1day")
technicalIndicators(endpoint="relative-strength-index",   symbol="AAPL", periodLength=14,  timeframe="1day")
technicalIndicators(endpoint="average-directional-index", symbol="AAPL", periodLength=14,  timeframe="1day")
technicalIndicators(endpoint="standard-deviation",        symbol="AAPL", periodLength=20,  timeframe="1day")
```

`timeframe` is one of `1min`, `5min`, `15min`, `30min`, `1hour`, `4hour`, `1day`.

**`technicalIndicators` requires a paid FMP plan** (Starter and above). On a free key it
returns an error — fall back to deriving moving averages and ranges from `chart` data
yourself, and say in the report that the indicator was computed locally.

**MACD and Bollinger Bands are not offered by this connector.** Derive them from `chart`
EOD closes (MACD = EMA12 − EMA26, signal = EMA9 of MACD; Bollinger = 20-day SMA ± 2σ,
using `standard-deviation` or a local calculation) or omit them and say so.

### `news`, `calendar`, `marketPerformance`, `search` — discovery

```
news(endpoint="search-stock-news",    symbols=["AAPL"], limit=20)
news(endpoint="search-press-releases", symbols=["AAPL"], limit=10)
news(endpoint="general-news",          limit=50)

calendar(endpoint="earnings-calendar", from_date="2026-08-24", to_date="2026-08-31")
calendar(endpoint="earnings-company",  symbol="AAPL", limit=8)   # surprise history
calendar(endpoint="dividends-company", symbol="AAPL")

marketPerformance(endpoint="biggest-gainers")
marketPerformance(endpoint="most-active")
marketPerformance(endpoint="sector-performance-snapshot", date="2026-08-24")
marketPerformance(endpoint="sector-PE-snapshot",          date="2026-08-24")

search(endpoint="search-symbol", query="Apple")
search(endpoint="search-name",   query="Apple Inc")
search(endpoint="search-ISIN",   isin="US0378331005")
search(endpoint="search-company-screener",
       sector="Technology", marketCapMoreThan=2000000000,
       betaLowerThan=1.5, isActivelyTrading=true, limit=50)
```

Other available FMP tools, used less often: `secFilings`, `earningsTranscript`,
`insiderTrades`, `institutionalOwnership`/`form13F`, `senate`, `etfAndMutualFunds`,
`economics`, `discountedCashFlow`, `indexes`, `forex`, `crypto`, `commodity`, `ESG`,
`tipranks`, `marketHours`, `commitmentOfTraders`.

---

## IBKR connector

IBKR tools take plain arguments, not an `endpoint`. They read the **live brokerage
account** — treat every number as real money.

### Portfolio state

```
get_account_positions()                  # every open position: qty, avg cost, mkt value, unrealised P&L
get_account_balances()                   # cash and market value per currency
get_account_summary()                    # net liquidation value, buying power, margin, available funds
get_pa_allocation(type="ALL")            # NAV split by asset class, sector, region, country, instrument
get_pa_performance_all_periods()         # 1D / 7D / MTD / 1M / YTD / 1Y return series
get_account_orders()                     # working and recent orders
get_account_trades()                     # execution history
```

Notes that matter when reporting:

- `get_pa_allocation` weights sum to 1.0 **within each bucket**, and the long-side
  denominator differs per dimension (netted of shorts for `ASSET_CLASS` and
  `FINANCIAL_INSTRUMENT`, gross for `SECTOR`/`REGION`/`COUNTRY`). Do not compare
  percentages across dimensions as if they shared a base.
- `short_positions` is absent when there are no shorts — null-check it.
- Item `id` values are opaque; display `name`.
- `get_pa_performance_all_periods` returns `cps` as **fractions** (`-0.106` = −10.6%),
  and `portfolio_measure` tells you whether it is time-weighted (TWR) or money-weighted
  (MWR). State which one you used.
- Prefer `type="ALL"` over five separate `get_pa_allocation` calls.

### Instruments, themes, market data

```
search_contracts(...)                    # resolve a ticker/name to a contract_id — needed by most tools below
get_price_snapshot(contract_id=265598, market_data_names=["last","bid_ask","change","volume"])
get_price_history(...)                   # historical OHLCV bars
get_company_themes(contract_id=265598, max_themes=5, max_companies=5)
get_company_connections(...)             # products, countries, competitors, evidence text
search_investment_topics(query="battery", max=5)   # → {key, name}; feed key to get_theme_details
get_theme_details(...)                   # the company list behind a theme
```

`search_investment_topics` is keyword-based and narrow: use **short, singular** nouns
(`"battery"`, not `"electric vehicle batteries"`; `"robot"`, not `"robotics companies"`).
If it returns nothing, retry with a shorter or singular form before concluding no match
exists. Never construct or modify a theme `key` — only pass keys the tool returned.

### Watchlists and alerts

```
get_watchlists() / get_watchlist(...) / create_watchlist(...) / edit_watchlist(...)
get_alerts() / create_alert(...) / update_alert(...) / set_alert_status(...)
```

Use watchlists to persist a shortlist of candidates back to the user's IBKR account.

### Order tools — never call unprompted

`create_order_instruction`, `delete_order_instruction` and `get_order_instructions`
stage real orders. **No skill or agent in this plugin may place, modify, or cancel an
order.** Analysis output is a recommendation; the user executes it themselves. Only call
an order tool if the user gives an explicit, unambiguous instruction to do so in that
same turn, and confirm the full order details back to them first.

---

## When a connector is missing

Never silently degrade — say which connector is unavailable and what that costs.

**FMP unavailable:** stop. Fundamental and technical analysis without it would be
guesswork. Tell the user to enable the FMP connector.

**IBKR unavailable:** portfolio skills fall back to a Portfolio Performance XML file via
the `pp` CLI — invoke the `pp-usage` skill. If the user has neither, ask them to paste
holdings as a table (symbol, shares, cost basis) and mark the report as manual-input.

**A single call fails or returns empty:** retry once with adjusted parameters (shorter
query, wider date range, different endpoint). If it still fails, record the field as
`N/A` in the output and note the gap in the report. Never interpolate or invent a number.
