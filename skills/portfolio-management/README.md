# Portfolio Management

Portfolio analysis skill: reads your current holdings, enriches them with fundamentals and
market data, and produces an allocation, risk, position-by-position and rebalancing report.

`SKILL.md` is the authoritative instruction set — this file is orientation for people
browsing the repository.

## Data sources

| Purpose | Source |
| --- | --- |
| Holdings, cash, allocation, performance | **Interactive Brokers MCP connector** (preferred) |
| Holdings, cash, transactions | **Portfolio Performance XML** via the `pp` CLI (fallback) |
| Prices, fundamentals, ratios, ratings, news | **FMP MCP connector** |

Both connectors are configured by the user in Claude's connector settings. This plugin
ships no MCP server definitions of its own. See the `market-data` skill for the endpoint
reference, and `pp-usage` for the `pp` CLI reference.

If no holdings source is available, the skill asks you to paste holdings as a table and
labels the report as manual-input.

## What it produces

- **Asset allocation** — by asset class, sector, market cap, and geography, against a target
- **Diversification** — concentration checks, correlation and thematic overlap, effective position count
- **Risk metrics** — portfolio beta, volatility, drawdown, and position-level risk contribution
- **Position evaluation** — buy / hold / trim / sell per holding, with the reasoning
- **Rebalancing plan** — specific trades to move from current to target, with tax and cost notes

## Usage

```
"Analyze my portfolio"
"What's my asset allocation?"
"Am I too concentrated in tech?"
"Should I rebalance?"
"What should I do with my NVDA position?"
```

If you have not stated a risk profile, the skill works one out with you using
`references/risk-profile-questionnaire.md` before recommending a target allocation.

## Reference files

| File | Contents |
| --- | --- |
| `references/asset-allocation.md` | Allocation frameworks and models |
| `references/diversification-principles.md` | Concentration and correlation analysis |
| `references/portfolio-risk-metrics.md` | Beta, volatility, drawdown, VaR, Sharpe |
| `references/position-evaluation.md` | Per-position buy/hold/sell framework and sell discipline |
| `references/rebalancing-strategies.md` | Calendar, threshold and tactical rebalancing; tax optimisation |
| `references/target-allocations.md` | Model portfolios by risk profile |
| `references/risk-profile-questionnaire.md` | Risk tolerance assessment and scoring |

## Limitations

- **Not financial advice.** Informational analysis only.
- **No order execution.** The skill never places, modifies, or cancels an order. You execute every recommendation yourself.
- **Data quality bounds analysis quality.** Portfolio Performance prices are only as fresh as your last sync; FMP ratios are as of the fiscal period end, not live.
- **Tax figures are estimates.** Consult a tax professional.
