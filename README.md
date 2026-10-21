# Finance Skills

> Stock analysis and portfolio management skills for retail investors, built as a Claude Code plugin.

A set of skills and agents for screening stocks, analysing them in depth, and reviewing a
portfolio. All market data comes from the **Financial Modeling Prep (FMP)** MCP connector;
live holdings come from the **Interactive Brokers (IBKR)** MCP connector, with a
Portfolio Performance file as an alternative.

This plugin produces **analysis only**. It never places, modifies, or cancels an order, and
nothing it outputs is financial advice.

## Prerequisites

### Required: FMP connector

Every analysis skill depends on it — prices, fundamentals, ratios, estimates, news, and
screening. Add the **FMP** connector in Claude's connector settings (Settings →
Connectors) and authenticate with your Financial Modeling Prep account.

A free FMP key covers most of the plugin. The `technicalIndicators` endpoints (SMA, EMA,
RSI, ADX, standard deviation) need a paid plan — without one, the skills derive moving
averages from raw price history instead and say that they did.

### Optional: Interactive Brokers connector

Needed only for portfolio features. Add the **Interactive Brokers** connector in Claude's
connector settings and authenticate with your IBKR account. It supplies live positions,
cash balances, allocation breakdowns, performance series, watchlists, and thematic
company search.

Without it, portfolio analysis falls back to a Portfolio Performance XML file read through
the [`pp` CLI](https://github.com/from68/pp-cli) — see the `pp-usage` skill — or to holdings
you paste in manually. `portfolio-management` installs/updates `pp` itself at the start of
every run that uses this path, so the only requirement is a Go toolchain on `PATH`:

```bash
go install github.com/from68/pp-cli/cmd/pp@latest
```

### Optional: last30days

If the [last30days](https://github.com/mvanhorn/last30days-skill) skill (or an MCP server
exposing it) is installed, the `stock-explorer` agent uses it to find stocks that are being
discussed on Reddit, X, YouTube and elsewhere. Candidates it surfaces are always confirmed
through FMP; it is never a source of numbers. Without it, the agent skips that step.

**No `.mcp.json` ships with this plugin.** Connectors are configured per user in Claude,
so the repository deliberately contains no server definitions, URLs, or API keys.

## Installation

Add the marketplace, then install the plugin:

```
/plugin marketplace add <this-repo-url>
/plugin install finance-skills@finance-skills
```

Or, from a local clone:

```
/plugin marketplace add /path/to/this/repo
/plugin install finance-skills@finance-skills
```

## What's included

### Skills

| Skill | Purpose |
| --- | --- |
| `quick-stock-analysis` | 5-gate mechanical screen (revenue growth, P/E, PEG, ROE, quick ratio) → score out of 5 plus verdict, written to `./Stock Screener/<TICKER>.md` |
| `full-stock-analysis` | Fundamentals, valuation, technicals, peer comparison and a full investment report, written to `./Shortlist/<TICKER>.md` |
| `portfolio-management` | Allocation, diversification, risk, position-by-position evaluation and a rebalancing plan |
| `market-data` | Reference for the FMP and IBKR connectors — endpoints, parameters, field mappings, fallbacks |
| `pp-usage` | Reference for the `pp` CLI, for the Portfolio Performance fallback path |

### Agents

| Agent | Purpose |
| --- | --- |
| `stock-explorer` | Surfaces candidate companies from market movers, earnings surprises, news, screeners, IBKR themes and — when installed — community discussion via last30days |
| `stock-analyzer` | Analyses a named ticker in depth using the screening and analysis skills |

## Usage

### Stock analysis

```
"Quick check on NVDA"
"Analyze AAPL"
"Compare TSLA vs RIVN"
"Is Microsoft overvalued?"
```

`quick-stock-analysis` triages, `full-stock-analysis` goes deep. All five gates are always
evaluated, so the screen reports a score (`4/5`, `2/5`, …) rather than stopping at the first
miss — a stock that fails one gate can still be worth a look. A 5/5 INVEST is a prompt for
further research, not a buy signal.

### Finding ideas

```
"Find me interesting stocks right now"
"Who beat earnings this week?"
"What's happening in quantum computing?"
"Screen for large-cap healthcare with beta under 1"
```

The `stock-explorer` agent returns a ranked shortlist with a stated reason and source per
candidate. Feed the shortlist to `quick-stock-analysis` to triage it.

### Portfolio analysis

```
"Analyze my portfolio"
"What's my asset allocation?"
"Am I too concentrated in tech?"
"Should I rebalance?"
```

With the IBKR connector, holdings are read live. Otherwise point the skill at a Portfolio
Performance XML file, or paste your holdings as a table.

## Output files

Analysis skills write markdown into the current working directory:

- `./Stock Screener/<TICKER>.md` — quick screen results
- `./Shortlist/<TICKER>.md` — full analysis reports

Both are frontmatter-tagged, so they work well inside a notes vault. Both directories are
gitignored here.

## Disclaimer

This plugin is for informational and educational purposes only. It does not constitute
financial, investment, or tax advice. Data accuracy depends on third-party providers and is
not guaranteed. Investment decisions should reflect your own circumstances, risk tolerance,
and goals — consult a qualified financial advisor. Past performance does not guarantee
future results.

## License

MIT
