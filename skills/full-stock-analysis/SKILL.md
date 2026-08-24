---
name: full-stock-analysis
description: Comprehensive US stock analysis including fundamental analysis (financial metrics, business quality, valuation), technical analysis (indicators, chart patterns, support/resistance), stock comparisons, and investment report generation. Use when user requests analysis of US stock tickers (e.g., "analyze AAPL", "compare TSLA vs NVDA", "give me a report on Microsoft"), evaluation of financial metrics, technical chart analysis, or investment recommendations for American stocks.
---

# US Stock Analysis

## Overview

Perform comprehensive analysis of US stocks covering fundamental analysis (financials, business quality, valuation), technical analysis (indicators, trends, patterns), peer comparisons, and generate detailed investment reports. Fetch market data and fundamentals from the FMP MCP connector and apply structured analytical frameworks.

## Data Sources

**All numbers come from the FMP MCP connector.** Invoke the `market-data` skill for exact
endpoint names, parameters, and the metric-to-field mapping. WebSearch is a supplement for
narrative context only — management commentary, competitive dynamics, breaking events — and
is never a substitute for a figure FMP can provide. Where a web source is used, cite it.

**Standard opening calls** (make these for every analysis, before any interpretation):

```
company(endpoint="profile-symbol", symbol=<TICKER>)                                  # sector, industry, description, beta
quote(endpoint="quote", symbol=<TICKER>)                                             # price, 52w range, volume, mkt cap, P/E
statements(endpoint="income-statement",        symbol=<TICKER>, period="annual", limit=5)
statements(endpoint="balance-sheet-statement", symbol=<TICKER>, period="annual", limit=5)
statements(endpoint="cashflow-statement",      symbol=<TICKER>, period="annual", limit=5)
statements(endpoint="metrics-ratios",          symbol=<TICKER>, period="annual", limit=5)
```

**Add per analysis type:**

| Need | Call |
| --- | --- |
| Analyst view | `analyst(endpoint="ratings-snapshot" \| "price-target-consensus" \| "grades-summary", symbol=…)` |
| Forward estimates | `analyst(endpoint="financial-estimates", symbol=…, period="annual")` |
| Peer set for comparison | `company(endpoint="peers", symbol=…)` |
| Quality/distress screens | `statements(endpoint="financial-scores", symbol=…)` → Piotroski, Altman Z |
| Growth rates | `statements(endpoint="income-statement-growth", symbol=…, period="annual", limit=5)` |
| Recent developments | `news(endpoint="search-stock-news", symbols=[…], limit=20)`, `news(endpoint="search-press-releases", symbols=[…])` |
| Earnings history & surprises | `calendar(endpoint="earnings-company", symbol=…, limit=8)` |
| Dividends | `calendar(endpoint="dividends-company", symbol=…)` |
| Price history | `chart(endpoint="historical-price-eod-light", symbol=…, from_date=…, to_date=…)` |
| Moving averages, RSI, ADX | `technicalIndicators(endpoint=…, symbol=…, periodLength=…, timeframe="1day")` |
| Sector valuation context | `marketPerformance(endpoint="sector-PE-snapshot", date=…)` |
| Filings for detail | `secFilings(...)`, `earningsTranscript(...)` |

**IBKR connector (optional, when connected):** use `search_contracts` → `get_company_themes`
to place the company in its thematic peer group, and `get_price_snapshot` for a live
bid/ask when timing an entry. If the user holds the stock,
`get_account_positions()` gives their cost basis and unrealised P&L — fold that into the
recommendation rather than analysing in a vacuum.

**Data hygiene:**

- Ratios from FMP are **fractions** (`0.147` = 14.7%). Convert before display.
- `metrics-ratios` is as of the fiscal period end; `quote` is live. Never put both in one
  column without labelling which is which.
- Always state the fiscal years covered and the quote timestamp in the report.
- `technicalIndicators` needs a paid FMP plan. If it errors, derive moving averages and
  volatility from `chart` closes and say the indicator was computed locally.
- MACD and Bollinger Bands are not FMP endpoints — derive them from `chart` data or omit
  them and say so. Do not report an indicator you did not actually compute.
- If a call returns empty, retry once with adjusted parameters, then mark the field `N/A`
  and note the gap. Never interpolate.

## Analysis Types

This skill supports four types of analysis. Determine which type(s) the user needs:

1. **Basic Stock Info** - Quick overview with key metrics
2. **Fundamental Analysis** - Deep dive into business, financials, valuation
3. **Technical Analysis** - Chart patterns, indicators, trend analysis
4. **Comprehensive Report** - Complete analysis combining all approaches

## Analysis Workflows

### 1. Basic Stock Information

**When to Use:** User asks for quick overview or basic info

**Steps:**

1. Search for current stock data (price, volume, market cap)
2. Gather key metrics (P/E, EPS, revenue growth, margins)
3. Get 52-week range and year-to-date performance
4. Find recent news or major developments
5. Present in concise summary format

**Output Format:**

- Company description (1-2 sentences)
- Current price and trading metrics
- Key valuation metrics (table)
- Recent performance
- Notable recent news (if any)

### 2. Fundamental Analysis

**When to Use:** User wants financial analysis, valuation assessment, or business evaluation

**Steps:**

1. **Gather comprehensive financial data:**
   - Revenue, earnings, cash flow (3-5 year trends)
   - Balance sheet metrics (debt, cash, working capital)
   - Profitability metrics (margins, ROE, ROIC)

2. **Read references/fundamental-analysis.md** for analytical framework

3. **Read references/financial-metrics.md** for metric definitions and calculations

4. **Analyze business quality:**
   - Competitive advantages
   - Management track record
   - Industry position

5. **Perform valuation analysis:**
   - Calculate key ratios (P/E, PEG, P/B, EV/EBITDA)
   - Compare to historical averages
   - Compare to peer group
   - Estimate fair value range

6. **Identify risks:**
   - Company-specific risks
   - Market/macro risks
   - Red flags from financial data

7. **Generate output** following references/report-template.md structure

**Critical Analyses:**

- Profitability trends (improving/declining margins)
- Cash flow quality (FCF vs earnings)
- Balance sheet strength (debt levels, liquidity)
- Growth sustainability
- Valuation vs peers and historical average

### 3. Technical Analysis

**When to Use:** User asks for technical analysis, chart patterns, or trading signals

**Steps:**

1. **Gather technical data:**
   - `chart(endpoint="historical-price-eod-light", symbol=…, from_date=…)` — at least 250
     sessions so a 200-day average is computable
   - `quote(endpoint="quote", symbol=…)` — current price, volume vs average volume, 52-week range
   - `technicalIndicators(endpoint="simple-moving-average", periodLength=20|50|200, timeframe="1day")`
   - `technicalIndicators(endpoint="relative-strength-index", periodLength=14, timeframe="1day")`
   - `technicalIndicators(endpoint="average-directional-index", periodLength=14, timeframe="1day")` — trend strength
   - MACD and Bollinger Bands: derive from the `chart` closes, or omit and say so

2. **Read references/technical-analysis.md** for indicator definitions and patterns

3. **Identify trend:**
   - Uptrend, downtrend, or sideways
   - Strength of trend

4. **Locate support and resistance levels:**
   - Recent highs and lows
   - Moving average levels
   - Round numbers

5. **Analyze indicators:**
   - RSI: Overbought (>70) or oversold (<30)
   - ADX: Trend strength (>25 trending, <20 rangebound)
   - MACD: Crossovers and divergences (if derived)
   - Volume: Confirmation or divergence
   - Bollinger Bands: Squeeze or expansion (if derived)

6. **Identify chart patterns:**
   - Reversal patterns (head and shoulders, double top/bottom)
   - Continuation patterns (flags, triangles)

7. **Generate technical outlook:**
   - Current trend assessment
   - Key levels to watch
   - Risk/reward analysis
   - Short and medium-term outlook

**Interpretation Guidelines:**

- Confirm signals with multiple indicators
- Consider volume for validation
- Note divergences between price and indicators
- Always identify risk levels (stop-loss)

### 4. Comprehensive Investment Report

**When to Use:** User asks for detailed report, investment recommendation, or complete analysis

**Steps:**

1. **Perform data gathering** (as in Basic Info)

2. **Execute fundamental analysis** (follow workflow above)

3. **Execute technical analysis** (follow workflow above)

4. **Read references/report-template.md** for complete report structure

5. **Synthesize findings:**
   - Integrate fundamental and technical insights
   - Develop bull and bear cases
   - Assess risk/reward

6. **Generate recommendation:**
   - Buy/Hold/Sell rating
   - Target price with timeframe
   - Conviction level
   - Entry strategy

7. **Create formatted report** following template structure

**Report Must Include:**

- Executive summary with recommendation
- Company overview
- Investment thesis (bull and bear cases)
- Fundamental analysis section
- Technical analysis section
- Valuation analysis
- Risk assessment
- Catalysts and timeline
- Conclusion

## Stock Comparison Analysis

**When to Use:** User asks to compare two or more stocks (e.g., "compare AAPL vs MSFT")

**Steps:**

1. **Gather data for all stocks:**
   - Run the standard opening calls for each ticker
   - Use `quote(endpoint="batch-quote", symbols=[…])` for a single-call price comparison
   - If the user names only one stock, get its peer set with `company(endpoint="peers", symbol=…)`
   - Ensure comparable fiscal periods — companies with different fiscal year ends are not
     directly comparable; say so if they differ

2. **Read references/fundamental-analysis.md** and references/financial-metrics.md

3. **Create side-by-side comparison:**
   - Business models comparison
   - Financial metrics table (all key ratios)
   - Valuation metrics table
   - Growth rates comparison
   - Profitability comparison
   - Balance sheet strength

4. **Identify relative strengths:**
   - Where each company excels
   - Quantified advantages

5. **Technical comparison:**
   - Relative strength
   - Momentum comparison
   - Which is in better technical position

6. **Generate recommendation:**
   - Which stock is more attractive and why
   - Consider both fundamental and technical factors
   - Portfolio allocation suggestion
   - Risk-adjusted return assessment

**Output Format:** Follow "Comparison Report Structure" in references/report-template.md

## Output

Write the result to `./Shortlist/<TICKER>.md` in the current working directory. Use the template below. Fill every field — use `N/A` or `UNKNOWN` when data is genuinely missing.

**General Principles:**

- Use tables for financial data and comparisons (easy to scan)
- Bold key metrics and findings
- Include data sources and dates
- Quantify whenever possible
- Present both bull and bear perspectives
- Be clear about assumptions and uncertainties

**Formatting:**

- **Headers** for clear section separation
- **Tables** for metrics, comparisons, historical data
- **Bullet points** for lists, factors, risks
- **Bold text** for key findings, important metrics
- **Percentages** for growth rates, returns, margins
- **Currency** formatted consistently ($B for billions, $M for millions)

**Tone:**

- Objective and balanced
- Acknowledge uncertainty
- Support claims with data
- Avoid hyperbole
- Present risks clearly

## Reference Files

Load these references as needed during analysis:

**`market-data` skill** (separate skill, not a file here)

- When: Constructing any FMP or IBKR connector call, or unsure which endpoint yields a metric
- Contains: Endpoint catalogue, parameter shapes, metric-to-field mapping, connector fallbacks

**references/technical-analysis.md**

- When: Performing technical analysis or interpreting indicators
- Contains: Indicator definitions, chart patterns, support/resistance concepts, analysis workflow

**references/fundamental-analysis.md**

- When: Performing fundamental analysis or business evaluation
- Contains: Business quality assessment, financial health analysis, valuation frameworks, risk assessment, red flags

**references/financial-metrics.md**

- When: Need definitions or calculation methods for financial ratios
- Contains: All key metrics with formulas (profitability, valuation, growth, liquidity, leverage, efficiency, cash flow)

**references/report-template.md**

- When: Creating comprehensive report or comparison
- Contains: Complete report structure, formatting guidelines, section templates, comparison format

## Example Queries

**Basic Info:**

- "What's the current price of AAPL?"
- "Give me key metrics for Tesla"
- "Quick overview of Microsoft stock"

**Fundamental:**

- "Analyze NVDA's financials"
- "Is Amazon overvalued?"
- "Evaluate Apple's business quality"
- "What's Google's debt situation?"

**Technical:**

- "Technical analysis of TSLA"
- "Is Netflix oversold?"
- "Show me support levels for AAPL"
- "What's the trend for AMD?"

**Comprehensive:**

- "Complete analysis of Microsoft"
- "Give me a full report on AAPL"
- "Should I invest in Tesla? Give me detailed analysis"

**Comparison:**

- "Compare AAPL vs MSFT"
- "Tesla vs Nvidia - which is better?"
- "Analyze Meta vs Google"
