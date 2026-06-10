# CLAUDE.md — Trading Bot Routine

## What this repo is

An autonomous trading bot. It operates in two modes:

- **Paper trading** — uses Alpaca for everything (account, market data, orders). Governed by `STRATEGY.md`.
- **Live trading** — uses **Robinhood** for account, positions, and orders; uses **Alpaca only for market data** (quotes, bars, option chains). Trading logic and thresholds are provided in the routine-level prompt — there is no strategy file for live mode.

This repo is cloned fresh by a Claude Code Routine on Anthropic's cloud infrastructure 3× per weekday (11 AM, 1 PM, 3 PM ET) and each run is completely stateless.

## First thing to do on every run

1. **Determine trading mode** — check whether the Robinhood MCP server is connected. If it is, this is a **live trading run**. If it is not, this is a **paper trading run** using Alpaca.
2. Verify the `alpaca` MCP server is connected (required in both modes). If not connected, log error and exit.
3. For paper mode, read `STRATEGY.md` in full before taking any action. For live mode, trading logic is provided in the routine-level prompt.

---

## Paper trading mode (Robinhood MCP not connected)

### Strategy file

Read `STRATEGY.md` in full. It is the source of truth for all paper trading logic. Do not improvise.

### Pre-flight checks

1. Verify the `alpaca` MCP server is connected.
2. Confirm the Alpaca account is in **paper trading mode**. If detected as live, STOP immediately and log a SAFETY ABORT.

### Tool usage

- Use the `alpaca` MCP server tools for all account, market data, and order operations.
- Required env vars: `ALPACA_API_KEY`, `ALPACA_SECRET_KEY`, `ALPACA_PAPER_TRADE=true` — supplied via Routine secrets, never hardcoded.
- Every order MUST include a unique `client_order_id` of the form `wheel-YYYYMMDD-HHMM-<symbol>` for idempotency.

### Safety rails (paper)

- **PAPER TRADING ONLY.** If `ALPACA_PAPER_TRADE` is not `true` or the account is detected as live, STOP immediately.
- **Never close a position at a loss.** No exceptions.
- **Never use leverage.** Never short stock. Never buy options to open.
- **Never place market orders.** Use limit orders only.
- **Max 3 open option positions** and **max 1 new sell order per calendar day.**
- **Only trade symbols on the user's Alpaca watchlist.** If watchlist is empty, exit cleanly.
- **Only trade during regular US market hours** (9:30 AM – 4:00 PM ET, weekdays).

### How to reason about a paper run

1. Pre-flight checks (market open, paper mode, account info, watchlist, positions, orders)
2. **Manage existing positions first** — close any that meet profit-taking rules in STRATEGY.md
3. **Evaluate new opportunities** — CSPs and CCs per STRATEGY.md conditions
4. Place at most ONE new sell-to-open order per run (and per day)
5. Print the structured end-of-run summary specified in STRATEGY.md

---

## Live trading mode (Robinhood MCP connected)

Trading logic, thresholds, and rules for live runs are provided in the routine-level prompt at invocation time. Follow those instructions exactly — do not improvise.

### Pre-flight checks

1. Confirm Robinhood account details look correct (non-zero buying power, expected account type).
2. Fetch the watchlist from Robinhood. If empty, log and exit.

### Tool usage

- **Robinhood MCP** — all account queries, position reads, order placement, and order management.
- **Alpaca MCP** — market data **only**: stock quotes, bars, option chains, option snapshots. Do NOT place any orders through Alpaca in live mode.
- Required env vars: `ALPACA_API_KEY`, `ALPACA_SECRET_KEY` — supplied via Routine secrets, never hardcoded. The Robinhood MCP connection is configured at the Routine connector level and requires no env vars.

### Safety rails (live)

- **REAL MONEY.** Default to inaction when uncertain. A skipped run costs nothing; a bad trade costs real money.
- **Never close a position at a loss.** No exceptions.
- **Never use leverage.** Never short stock. Never buy options to open.
- **Never place market orders.** Use limit orders only.
- **Only trade symbols on the user's Robinhood watchlist.** If watchlist is empty, exit cleanly.
- Do NOT use Alpaca for any order placement or account mutations in live mode.

### How to reason about a live run

1. Pre-flight checks (market open, Robinhood account info, watchlist, positions, orders)
2. Fetch all market data needed (quotes, option chains) via Alpaca MCP
3. **Manage existing positions first** — close any that meet profit-taking rules per the routine-level prompt
4. **Evaluate new opportunities** — per the routine-level prompt
5. Print a structured end-of-run summary

---

## When in doubt

If instructions are ambiguous, if market data looks wrong, or if anything unexpected shows up (negative buying power, unknown positions, stale quotes) — **do nothing and log the concern**. A skipped run is always safer than an incorrect trade.

## Do NOT

- Do not modify this repo's files during a run.
- Do not push branches or open PRs — this routine is read-only on the repo.
- Do not use web search or external APIs beyond the MCP servers listed above.
- Do not retry failed orders more than once (5-second backoff). Failed twice = skip.
- Do not be "creative" with the strategy. For paper mode, follow `STRATEGY.md` exactly. For live mode, follow the routine-level prompt exactly.
- Do not use Alpaca for order placement or account mutations during live trading.
