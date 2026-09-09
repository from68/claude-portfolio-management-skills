---
name: portfolio-management
description: Comprehensive portfolio analysis that reads live holdings from the Interactive Brokers (IBKR) MCP connector — or from a Portfolio Performance XML file via the pp CLI — enriches them with FMP fundamentals, then analyzes asset allocation, risk metrics, individual positions, diversification, and generates rebalancing recommendations. Use when user requests portfolio review, position analysis, risk assessment, performance evaluation, or rebalancing suggestions for their brokerage account or portfolio file.
---

# Portfolio Manager

## Overview

Analyze and manage investment portfolios by reading current holdings — from the **Interactive Brokers (IBKR) MCP connector** where available, otherwise from a Portfolio Performance XML file via the `pp` CLI — then performing comprehensive analysis covering asset allocation, diversification, risk metrics, individual position evaluation, and rebalancing recommendations. Position-level fundamentals and prices come from the **FMP MCP connector**. Generate detailed portfolio reports with actionable insights.

Two reference skills support this one — invoke them rather than guessing at syntax:

- **`market-data`** — endpoint names, parameters, and field mappings for the IBKR and FMP connectors, plus the caveats that matter when reporting IBKR allocation and performance figures.
- **`pp-usage`** — `pp` CLI command syntax, for the Portfolio Performance fallback path.

**This skill never places, modifies, or cancels an order.** Its output is a recommendation the user executes themselves.

## When to Use

Invoke this skill when the user requests:

- "Analyze my portfolio"
- "Review my current positions"
- "What's my asset allocation?"
- "Check my portfolio risk"
- "Should I rebalance my portfolio?"
- "Evaluate my holdings"
- "Portfolio performance review"
- "What stocks should I buy or sell?"
- Any request involving portfolio-level analysis or management

## Prerequisites

Portfolio data comes from **one** of two sources. Determine which at the start of every
run — do not assume.

### Source A (preferred): Interactive Brokers MCP connector

If the IBKR connector is available, use it. It provides live, authoritative data:

- `get_account_positions()` — every open position with quantity, average cost, market value, and unrealised P&L
- `get_account_balances()` — cash and market value per currency
- `get_account_summary()` — net liquidation value, buying power, margin requirements
- `get_pa_allocation(type="ALL")` — NAV broken down by asset class, sector, region, country, and instrument type
- `get_pa_performance_all_periods()` — return series for 1D, 7D, MTD, 1M, YTD, 1Y

Invoke the `market-data` skill for the reporting caveats — allocation denominators differ
per dimension, `cps` values are fractions not percentages, and `portfolio_measure` tells
you whether returns are time-weighted or money-weighted.

### Source B (fallback): Portfolio Performance XML via the `pp` CLI

Use when IBKR is not connected, or when the user explicitly points at a `.xml` file.

**Install `pp` at the start of every run that uses this path.** Never assume the binary is
present or current:

```bash
go install github.com/from68/pp-cli/cmd/pp@latest
```

The command is idempotent and fast once cached, though the first run may pull a matching Go
toolchain. It installs into `$(go env GOPATH)/bin` (usually `~/go/bin`); if `pp` is still not
on `PATH` afterwards, call it by that full path. Run `pp --version` to confirm the binary is
reachable — it prints `pp version dev`, since the build is not version-stamped, so treat it
as a smoke test, not a version check. `pp` needs the Go toolchain — if `go` is missing, say
so and ask the user to install it rather than installing a toolchain yourself.

**Key `pp` commands used:**

- `pp -f <file> -o json portfolios list` — list all portfolios
- `pp -f <file> -o json portfolios holdings "<name>"` — current positions with shares, price, value
- `pp -f <file> -o json accounts list` — cash balances per account
- `pp -f <file> -o json portfolios transactions "<name>"` — transaction history
- `pp -f <file> -o json securities list` — all securities with latest prices

Ask the user for the path to their Portfolio Performance XML file if it is not already
known. **Invoke the `pp-usage` skill** to look up exact command syntax whenever needed.

### If both are available

Prefer IBKR for holdings, cash, and performance — it is live and needs no manual price
updates. Use the XML file only for history that predates the IBKR account, and say in the
report which source each figure came from. If the two disagree on a position, report the
discrepancy rather than silently picking one.

### If neither is available

Ask the user to paste their holdings as a table (symbol, shares, cost basis, optionally
account). Run the full analysis on that input and mark the report as manual-input, since
prices and weights are only as fresh as what they pasted.

## Workflow

### Step 1: Fetch Portfolio Data

Pick the source per the Prerequisites section, then gather holdings, cash, and (optionally)
history.

#### Source A — Interactive Brokers connector

**1.1 Positions:**

```
get_account_positions()
```

Returns per position: contract id, symbol, description, quantity, average cost, market
price, market value, unrealised and realised P&L, currency, asset class.

**1.2 Cash and account size:**

```
get_account_balances()      # per-currency cash and market value
get_account_summary()       # net liquidation value, buying power, margin
```

Use net liquidation value from `get_account_summary()` as the denominator for position
weights — it already includes cash and accrued items.

**1.3 Allocation breakdown:**

```
get_pa_allocation(type="ALL")
```

This gives IBKR's own asset-class, sector, region, country and instrument-type splits in
one call. Prefer it over computing sector weights yourself. Read the caveats in
`market-data` before quoting these percentages: weights sum to 1.0 within each bucket,
denominators differ per dimension, and `short_positions` may be absent.

**1.4 Performance and history (optional):**

```
get_pa_performance_all_periods()    # 1D/7D/MTD/1M/YTD/1Y return series
get_account_trades()                # execution history for cost-basis and turnover analysis
get_account_orders()                # working orders that will change the picture
```

`cps` values are cumulative **fractions** (`-0.106` = −10.6%). Report the method named in
`portfolio_measure` (TWR or MWR).

**Data validation:**

- Check `get_account_orders()` for unfilled orders — a large working order materially
  changes any rebalancing advice. Flag them.
- Note the account base currency; multi-currency accounts need FX-consistent weights.
- Watch for short and derivative positions — they break naive weight arithmetic.

#### Source B — Portfolio Performance XML via `pp`

Invoke the `pp-usage` skill for exact command syntax if needed.

**1.0 Install/update the `pp` CLI — mandatory, every run:**

```bash
go install github.com/from68/pp-cli/cmd/pp@latest && pp --version
```

Run this before any other `pp` command, every single run. Do not skip it because `pp` worked
in a previous session — the commands and JSON fields below track the current CLI. `pp --version`
prints `pp version dev` even on a fresh install; it only confirms the binary is on `PATH`.

**1.1 List Portfolios:**

```bash
pp -f <file.xml> -o json portfolios list
```

Identify the portfolio name(s) to use in subsequent commands.

**1.2 Get Current Holdings:**

```bash
pp -f <file.xml> -o json portfolios holdings "<Portfolio Name>"
```

Returns per position: Security name, ISIN, Shares, Latest Price, Value, Currency. Compute each position's percentage weight by dividing its value by the total portfolio value.

**1.3 Get Cash Balances:**

```bash
pp -f <file.xml> -o json accounts list
```

Returns each account's name and computed balance. Sum balances to determine total cash.

**1.4 Get Transaction History (Optional):**

```bash
pp -f <file.xml> -o json portfolios transactions "<Portfolio Name>"
pp -f <file.xml> -o json portfolios transactions "<Portfolio Name>" --from 2024-01-01
```

Use for performance analysis, drawdown estimation, and cost-basis reconstruction.

**Data Validation:**

- Verify all positions have valid security names/ISINs
- Confirm holdings values plus cash balances approximate total portfolio value
- Check for zero or negative positions (use `--include-zero` flag if needed)
- Handle fractional shares (pp reports 8 decimal places)
- Prices in a Portfolio Performance file are only as fresh as the last sync — check the
  latest price date via `pp securities list` and refresh stale ones from FMP `quote`

### Step 2: Enrich Position Data

Holdings data tells you *what* is held, not whether it is worth holding. Enrich every
position from the **FMP connector**. Resolve each holding to a ticker first — IBKR gives
you the symbol directly; for `pp` holdings use
`search(endpoint="search-ISIN", isin=<ISIN>)` to map ISIN to ticker.

Invoke the `market-data` skill for endpoint details.

**2.1 Current Market Data:**

```
quote(endpoint="batch-quote", symbols=[<all tickers>])
```

One call covers price, day change, 52-week range, volume vs average volume, market cap,
and trailing P/E for the whole portfolio. For an intraday IBKR fill price, use
`get_price_snapshot(contract_id=…)` instead.

**2.2 Fundamental Data:**

```
company(endpoint="profile-symbol", symbol=…)        # sector, industry, beta, country
statements(endpoint="metrics-ratios", symbol=…, period="annual", limit=5)
statements(endpoint="financial-scores", symbol=…)   # Piotroski, Altman Z
analyst(endpoint="price-target-consensus", symbol=…)
analyst(endpoint="ratings-snapshot", symbol=…)
calendar(endpoint="dividends-company", symbol=…)    # for income analysis
news(endpoint="search-stock-news", symbols=[…], limit=10)
```

Scale the depth to the portfolio: run the full set on the largest positions and anything
flagged as a concern, and the profile plus ratios on the tail. Say in the report which
positions got the deep treatment.

**2.3 Technical Analysis:**

```
chart(endpoint="historical-price-eod-light", symbol=…, from_date=…)
technicalIndicators(endpoint="simple-moving-average", symbol=…, periodLength=50|200, timeframe="1day")
technicalIndicators(endpoint="relative-strength-index", symbol=…, periodLength=14, timeframe="1day")
```

Use for trend, relative strength, support/resistance, and momentum. `technicalIndicators`
requires a paid FMP plan — if unavailable, derive moving averages from `chart` closes and
say the values were computed locally.

**2.4 Thematic overlap (optional, IBKR):**

```
search_contracts(...) -> get_company_themes(contract_id=…, max_themes=5)
```

Useful for spotting hidden concentration: several positions sharing one theme is a
correlation risk that a sector breakdown alone will miss.

### Step 3: Portfolio-Level Analysis

Perform comprehensive portfolio analysis using frameworks from reference files:

#### 3.1 Asset Allocation Analysis

**Read references/asset-allocation.md** for allocation frameworks

Analyze current allocation across multiple dimensions:

**By Asset Class:**

- Equities vs Fixed Income vs Cash vs Alternatives
- Compare to target allocation for user's risk profile
- Assess if allocation matches investment goals

**By Sector:**

- Technology, Healthcare, Financials, Consumer, etc.
- Identify sector concentration risks
- Compare to benchmark sector weights (e.g., S&P 500)

**By Market Cap:**

- Large-cap vs Mid-cap vs Small-cap distribution
- Concentration in mega-caps
- Market cap diversification score

**By Geography:**

- US vs International vs Emerging Markets
- Domestic concentration risk assessment

**Output Format:**

```markdown
## Asset Allocation

### Current Allocation vs Target

| Asset Class | Current | Target | Variance |
| ----------- | ------- | ------ | -------- |
| US Equities | XX.X%   | YY.Y%  | +/- Z.Z% |
| ...         |

### Sector Breakdown

[Pie chart description or table with sector percentages]

### Top 10 Holdings

| Rank | Symbol | % of Portfolio | Sector     |
| ---- | ------ | -------------- | ---------- |
| 1    | AAPL   | X.X%           | Technology |
| ...  |
```

#### 3.2 Diversification Analysis

**Read references/diversification-principles.md** for diversification theory

Evaluate portfolio diversification quality:

**Position Concentration:**

- Identify top holdings and their aggregate weight
- Flag if any single position exceeds 10-15% of portfolio
- Calculate Herfindahl-Hirschman Index (HHI) for concentration measurement

**Sector Concentration:**

- Identify dominant sectors
- Flag if any sector exceeds 30-40% of portfolio
- Compare to benchmark sector diversity

**Correlation Analysis:**

- Estimate correlation between major positions
- Identify highly correlated holdings (potential redundancy)
- Assess true diversification benefit

**Number of Positions:**

- Optimal range: 15-30 stocks for individual portfolios
- Flag if under-diversified (<10 stocks) or over-diversified (>50 stocks)

**Output:**

```markdown
## Diversification Assessment

**Concentration Risk:** [Low / Medium / High]

- Top 5 holdings represent XX% of portfolio
- Largest single position: [SYMBOL] at XX%

**Sector Diversification:** [Excellent / Good / Fair / Poor]

- Dominant sector: [Sector Name] at XX%
- [Assessment of balance across sectors]

**Position Count:** [Optimal / Under-diversified / Over-diversified]

- Total positions: XX stocks
- [Recommendation]

**Correlation Concerns:**

- [List any highly correlated position pairs]
- [Diversification improvement suggestions]
```

#### 3.3 Risk Analysis

**Read references/portfolio-risk-metrics.md** for risk measurement frameworks

Calculate and interpret key risk metrics:

**Volatility Measures:**

- Estimated portfolio beta (weighted average of position betas)
- Individual position volatilities
- Portfolio standard deviation (if historical data available)

**Downside Risk:**

- Maximum drawdown (from portfolio history)
- Current drawdown from peak
- Positions with significant unrealized losses

**Risk Concentration:**

- Percentage in high-volatility stocks (beta > 1.5)
- Percentage in speculative/unprofitable companies
- Leverage usage (if applicable)

**Tail Risk:**

- Exposure to potential black swan events
- Single-stock concentration risk
- Sector-specific event risk

**Output:**

```markdown
## Risk Assessment

**Overall Risk Profile:** [Conservative / Moderate / Aggressive]

**Portfolio Beta:** X.XX (vs market at 1.00)

- Interpretation: Portfolio is [more/less] volatile than market

**Maximum Drawdown:** -XX.X% (from $XXX,XXX to $XXX,XXX)

- Current drawdown from peak: -XX.X%

**High-Risk Positions:**
| Symbol | % of Portfolio | Beta | Risk Factor |
|--------|----------------|------|-------------|
| [TICKER] | XX% | X.XX | [High volatility / Recent loss / etc] |

**Risk Concentrations:**

- XX% in single sector ([Sector])
- XX% in stocks with beta > 1.5
- [Other concentration risks]

**Risk Score:** XX/100 ([Low/Medium/High] risk)
```

#### 3.4 Performance Analysis

Evaluate portfolio performance using available data:

**Absolute Returns:**

- Overall portfolio unrealized P&L ($ and %)
- Best performing positions (top 5 by % gain)
- Worst performing positions (bottom 5 by % loss)

**Time-Weighted Returns (if history available):**

- YTD return
- 1-year, 3-year, 5-year annualized returns
- Compare to benchmark (S&P 500, relevant index)

**Position-Level Performance:**

- Winners vs Losers ratio
- Average gain on winning positions
- Average loss on losing positions
- Positions near 52-week highs/lows

**Output:**

```markdown
## Performance Review

**Total Portfolio Value:** $XXX,XXX
**Total Unrealized P&L:** $XX,XXX (+XX.X%)
**Cash Balance:** $XX,XXX (XX% of portfolio)

**Best Performers:**
| Symbol | Gain | Position Value |
|--------|------|----------------|
| [TICKER] | +XX.X% | $XX,XXX |
| ... |

**Worst Performers:**
| Symbol | Loss | Position Value |
|--------|------|----------------|
| [TICKER] | -XX.X% | $XX,XXX |
| ... |

**Performance vs Benchmark (if available):**

- Portfolio return: +X.X%
- S&P 500 return: +Y.Y%
- Alpha: +/- Z.Z%
```

### Step 4: Individual Position Analysis

For key positions (top 10-15 by portfolio weight), perform detailed analysis:

**Read references/position-evaluation.md** for position analysis framework

For each significant position:

**4.1 Current Thesis Validation:**

- Why was this position initiated? (if known from user context)
- Has the investment thesis played out or broken?
- Recent company developments and news

**4.2 Valuation Assessment:**

- Current valuation metrics (P/E, P/B, etc.)
- Compare to historical valuation range
- Compare to sector peers
- Overvalued / Fair / Undervalued assessment

**4.3 Technical Health:**

- Price trend (uptrend, downtrend, sideways)
- Position relative to moving averages
- Support and resistance levels
- Momentum status

**4.4 Position Sizing:**

- Current weight in portfolio
- Is size appropriate given conviction and risk?
- Overweight or underweight vs optimal

**4.5 Action Recommendation:**

- **HOLD** - Position is well-sized and thesis intact
- **ADD** - Underweight given opportunity, thesis strengthening
- **TRIM** - Overweight or valuation stretched
- **SELL** - Thesis broken, better opportunities elsewhere

**Output per position:**

```markdown
### [SYMBOL] - [Company Name] (XX.X% of portfolio)

**Position Details:**

- Shares: XXX
- Avg Cost: $XX.XX
- Current Price: $XX.XX
- Market Value: $XX,XXX
- Unrealized P/L: $X,XXX (+XX.X%)

**Fundamental Snapshot:**

- Sector: [Sector]
- Market Cap: $XX.XB
- P/E: XX.X | Dividend Yield: X.X%
- Recent developments: [Key news or earnings]

**Technical Status:**

- Trend: [Uptrend / Downtrend / Sideways]
- Price vs 50-day MA: [Above/Below by XX%]
- Support: $XX.XX | Resistance: $XX.XX

**Position Assessment:**

- **Thesis Status:** [Intact / Weakening / Broken / Strengthening]
- **Valuation:** [Undervalued / Fair / Overvalued]
- **Position Sizing:** [Optimal / Overweight / Underweight]

**Recommendation:** [HOLD / ADD / TRIM / SELL]
**Rationale:** [1-2 sentence explanation]
```

### Step 5: Rebalancing Recommendations

**Read references/rebalancing-strategies.md** for rebalancing approaches

Generate specific rebalancing recommendations:

**5.1 Identify Rebalancing Triggers:**

- Positions that have drifted significantly from target weights
- Sector/asset class allocations requiring adjustment
- Overweight positions to trim (exceeded threshold)
- Underweight areas to add (below threshold)
- Tax considerations (capital gains implications)

**5.2 Develop Rebalancing Plan:**

**Positions to TRIM:**

- Overweight positions (>threshold deviation from target)
- Stocks that have run up significantly (valuation concerns)
- Concentrated positions exceeding 15-20% of portfolio
- Positions with broken thesis

**Positions to ADD:**

- Underweight sectors or asset classes
- High-conviction positions currently underweight
- New opportunities to improve diversification

**Cash Deployment:**

- If excess cash (>10% of portfolio), suggest deployment
- Prioritize based on opportunity and allocation gaps

**5.3 Prioritization:**
Rank rebalancing actions by priority:

1. **Immediate** - Risk reduction (trim concentrated positions)
2. **High Priority** - Major allocation drift (>10% from target)
3. **Medium Priority** - Moderate drift (5-10% from target)
4. **Low Priority** - Fine-tuning and opportunistic adjustments

**Output:**

```markdown
## Rebalancing Recommendations

### Summary

- **Rebalancing Needed:** [Yes / No / Optional]
- **Primary Reason:** [Concentration risk / Sector drift / Cash deployment / etc]
- **Estimated Trades:** X sell orders, Y buy orders

### Recommended Actions

#### HIGH PRIORITY: Risk Reduction

**TRIM [SYMBOL]** from XX% to YY% of portfolio

- **Shares to Sell:** XX shares (~$XX,XXX)
- **Rationale:** [Overweight / Valuation extended / etc]
- **Tax Impact:** $X,XXX capital gain (est)

#### MEDIUM PRIORITY: Asset Allocation

**ADD [Sector/Asset Class]** exposure

- **Target:** Increase from XX% to YY%
- **Suggested Stocks:** [SYMBOL1, SYMBOL2, SYMBOL3]
- **Amount to Invest:** ~$XX,XXX

#### CASH DEPLOYMENT

**Current Cash:** $XX,XXX (XX% of portfolio)

- **Recommendation:** [Deploy / Keep for opportunities / Reduce to X%]
- **Suggested Allocation:** [Distribution across sectors/stocks]

### Implementation Plan

1. [First action - highest priority]
2. [Second action]
3. [Third action]
   ...

**Timing Considerations:**

- [Tax year-end planning / Earnings season / Market conditions]
- [Suggested phasing if applicable]
```

### Step 6: Generate Portfolio Report

Create comprehensive markdown report saved to repository root:

**Filename:** `portfolio_analysis_YYYY-MM-DD.md`

**Report Structure:**

```markdown
# Portfolio Analysis Report

**Account:** [Account type if available]
**Report Date:** YYYY-MM-DD
**Portfolio Value:** $XXX,XXX
**Total P&L:** $XX,XXX (+XX.X%)

---

## Executive Summary

[3-5 bullet points summarizing key findings]

- Overall portfolio health assessment
- Major strengths
- Key risks or concerns
- Primary recommendations

---

## Holdings Overview

[Summary table of all positions]

---

## Asset Allocation

[Section from Step 3.1]

---

## Diversification Analysis

[Section from Step 3.2]

---

## Risk Assessment

[Section from Step 3.3]

---

## Performance Review

[Section from Step 3.4]

---

## Position Analysis

[Detailed analysis of top 10-15 positions from Step 4]

---

## Rebalancing Recommendations

[Section from Step 5]

---

## Action Items

**Immediate Actions:**

- [ ] [Action 1]
- [ ] [Action 2]

**Medium-Term Actions:**

- [ ] [Action 3]
- [ ] [Action 4]

**Monitoring Priorities:**

- [ ] [Watch list item 1]
- [ ] [Watch list item 2]

---

## Appendix: Full Holdings

[Complete table with all positions and metrics]
```

### Step 7: Interactive Follow-up

Be prepared to answer follow-up questions:

**Common Questions:**

**"Why should I sell [SYMBOL]?"**

- Explain specific concerns (valuation, thesis breakdown, concentration)
- Provide supporting data
- Offer alternative positions if applicable

**"What should I buy instead?"**

- Suggest specific stocks to improve allocation
- Explain how they address portfolio gaps
- Provide brief investment thesis

**"What's my biggest risk?"**

- Identify primary risk factor (concentration, sector exposure, volatility)
- Quantify the risk
- Suggest mitigation strategies

**"How does my portfolio compare to [benchmark]?"**

- Compare allocation, sector weights, risk metrics
- Highlight key differences
- Assess if differences are justified

**"Should I rebalance now or wait?"**

- Consider market conditions, tax implications, transaction costs
- Provide timing recommendation with rationale

**"Can you analyze [specific position] in more detail?"**

- Perform deep-dive analysis using us-stock-analysis skill if needed
- Integrate findings back into portfolio context

## Analysis Frameworks

### Target Allocation Templates

This skill includes reference allocation models for different investor profiles:

**Read references/target-allocations.md** for detailed models:

- **Conservative** (Capital preservation, income focus)
- **Moderate** (Balanced growth and income)
- **Growth** (Long-term capital appreciation)
- **Aggressive** (Maximum growth, high risk tolerance)

Each model includes:

- Asset class targets (Stocks/Bonds/Cash/Alternatives)
- Sector guidelines
- Market cap distribution
- Geographic allocation
- Position sizing rules

Use these as comparison benchmarks when user hasn't specified their allocation strategy.

### Risk Profile Assessment

If user's target allocation is unknown, assess appropriate risk profile based on:

- Age (if mentioned)
- Investment timeline (if mentioned)
- Current allocation (reveals preferences)
- Position types (conservative vs speculative stocks)

**Read references/risk-profile-questionnaire.md** for assessment framework

## Output Guidelines

**Tone and Style:**

- Objective and analytical
- Actionable recommendations with clear rationale
- Acknowledge uncertainty in market forecasts
- Balance optimism with risk awareness
- Quantify whenever possible

**Data Presentation:**

- Tables for comparisons and metrics
- Percentages for allocations and returns
- Dollar amounts for absolute values
- Consistent formatting throughout report

**Recommendation Clarity:**

- Explicit action verbs (TRIM, ADD, HOLD, SELL)
- Specific quantities (sell XX shares, add $X,XXX)
- Priority levels (Immediate, High, Medium, Low)
- Supporting rationale for each recommendation

**Visual Descriptions:**

- Describe allocation breakdowns as if creating pie charts
- Sector weights as bar chart equivalents
- Performance trends with directional indicators (↑ ↓ →)

## Reference Files

Load these references as needed during analysis:

**`pp-usage` skill**

- When: Need guidance on any `pp` CLI command syntax or flags
- Contains: Full command reference for querying Portfolio Performance XML files, including holdings, transactions, accounts, and securities

**references/asset-allocation.md**

- When: Analyzing portfolio allocation or creating rebalancing plan
- Contains: Asset allocation theory, optimal allocation by risk profile, sector allocation guidelines, rebalancing triggers

**references/diversification-principles.md**

- When: Assessing portfolio diversification quality
- Contains: Modern portfolio theory basics, correlation concepts, optimal position count, concentration risk thresholds, diversification metrics

**references/portfolio-risk-metrics.md**

- When: Calculating risk scores or interpreting volatility
- Contains: Beta calculation, standard deviation, Sharpe ratio, maximum drawdown, Value at Risk (VaR), risk-adjusted return metrics

**references/position-evaluation.md**

- When: Analyzing individual holdings for buy/hold/sell decisions
- Contains: Position analysis framework, thesis validation checklist, position sizing guidelines, sell discipline criteria

**references/rebalancing-strategies.md**

- When: Developing rebalancing recommendations
- Contains: Rebalancing methodologies (calendar-based, threshold-based, tactical), tax optimization strategies, transaction cost considerations, implementation timing

**references/target-allocations.md**

- When: Need benchmark allocations for comparison
- Contains: Model portfolios for conservative/moderate/growth/aggressive investors, sector target ranges, market cap distributions

**references/risk-profile-questionnaire.md**

- When: User hasn't specified risk tolerance or target allocation
- Contains: Risk assessment questions, scoring methodology, risk profile classification

## Error Handling

**If neither connector nor portfolio file is available:**

1. Say plainly which sources you checked and found missing
2. Point the user at the Connectors settings for IBKR, or ask for the `.xml` path
3. Offer the paste-your-holdings fallback so the analysis is not blocked entirely

**If the IBKR connector errors or returns empty positions:**

- Distinguish "not connected" from "connected but empty" — an empty
  `get_account_positions()` on a live connection means a genuinely empty account
- Re-authentication is the user's action, not yours; tell them what to do
- Fall back to the `pp` path if a portfolio file is available, and label the switch

**If the FMP connector is unavailable:**

- Holdings and weights still work from IBKR or `pp`, but position-level evaluation does not
- Produce the allocation and concentration analysis, and state clearly that fundamental
  and valuation sections were skipped for lack of data — do not fill them from WebSearch
  and present the result as equivalent

**If the Portfolio Performance XML file path is unknown:**

1. Ask the user for the path to their `.xml` file
2. Optionally run `find ~ -name "*.xml" -path "*portfolio*" 2>/dev/null` to help locate it

**If `pp` is missing or `go install` fails:**

1. Re-run `go install github.com/from68/pp-cli/cmd/pp@latest` and read the actual error
2. If `go` is not on `PATH`, tell the user `pp` needs the Go toolchain and ask them to
   install it — do not install a toolchain yourself
3. If the install succeeded but `pp` is not found, invoke it as `$(go env GOPATH)/bin/pp`
4. If `pp` still cannot be made to work, suggest the IBKR connector or the
   paste-your-holdings fallback; invoke the `pp-usage` skill for command reference

**If a command returns incomplete or unexpected data:**

- Proceed with available data
- Note limitations in the report
- For `pp`, use `pp -f <file> -o json validate` to check for file integrity issues

**If holdings data seems stale (latest prices are old):**

- Flag the issue — prices in Portfolio Performance are updated manually or via price feeds
- Note the last price date per security from `pp securities list` output
- Refresh with `quote(endpoint="batch-quote", symbols=[…])` from FMP and mark which prices
  were refreshed

**If a ticker cannot be resolved from an ISIN:**

- Try `search(endpoint="search-ISIN", isin=…)`, then `search(endpoint="search-name", query=…)`
- If still unresolved (common for non-US listings and some ETFs), keep the position in the
  weight arithmetic but mark its fundamentals `N/A` and list it as unenriched

**If user has no positions:**

- Acknowledge empty portfolio
- Offer portfolio construction guidance instead of analysis
- Suggest using the `quick-stock-analysis` or `full-stock-analysis` skills for stock ideas

## Advanced Features

### Tax-Loss Harvesting Opportunities

Identify positions with unrealized losses suitable for tax-loss harvesting:

- Positions with losses >5%
- Holding period considerations (avoid wash sale rule)
- Replacement security suggestions (similar but not substantially identical)

### Dividend Income Analysis

For portfolios with dividend-paying stocks:

- Estimate annual dividend income
- Dividend growth rate trajectory
- Dividend coverage and sustainability
- Yield on cost for long-term holdings

### Correlation Matrix

For portfolios with 5-20 positions:

- Estimate correlation between major positions
- Identify redundant positions (correlation >0.8)
- Suggest diversification improvements

### Scenario Analysis

Model portfolio behavior under different scenarios:

- **Bull Market** (+20% equity appreciation)
- **Bear Market** (-20% equity decline)
- **Sector Rotation** (Tech weakness, Value strength)
- **Rising Rates** (Impact on growth stocks and bonds)

## Example Queries

**Basic Portfolio Review:**

- "Analyze my portfolio"
- "Review my positions"
- "How's my portfolio doing?"

**Allocation Analysis:**

- "What's my asset allocation?"
- "Am I too concentrated in tech?"
- "Show me my sector breakdown"

**Risk Assessment:**

- "Is my portfolio too risky?"
- "What's my portfolio beta?"
- "What are my biggest risks?"

**Rebalancing:**

- "Should I rebalance?"
- "What should I buy or sell?"
- "How can I improve diversification?"

**Performance:**

- "What are my best and worst positions?"
- "How am I performing vs the market?"
- "Which stocks are winning and losing?"

**Position-Specific:**

- "Should I sell [SYMBOL]?"
- "Is [SYMBOL] overweight in my portfolio?"
- "What should I do with [SYMBOL]?"

## Limitations and Disclaimers

**Include in all reports:**

_This analysis is for informational purposes only and does not constitute financial advice. Investment decisions should be made based on individual circumstances, risk tolerance, and financial goals. Past performance does not guarantee future results. Consult with a qualified financial advisor before making investment decisions._

_Data accuracy depends on the connected data sources (Interactive Brokers, Financial Modeling Prep, or a user-supplied Portfolio Performance file) and on third-party market data. Verify critical information independently. Tax implications are estimates only; consult a tax professional for specific guidance._

_This skill produces analysis only. It does not place, modify, or cancel orders. Every recommendation must be reviewed and executed by the user._
