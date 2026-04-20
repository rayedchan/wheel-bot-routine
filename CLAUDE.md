# CLAUDE.md — Wheel Bot Routine

## What this repo is

An autonomous **Alpaca paper-trading options bot** executing a wheeling strategy (cash-secured puts + covered calls). This repo is cloned fresh by a Claude Code Routine on Anthropic's cloud infrastructure 3× per weekday (11 AM, 1 PM, 3 PM ET) and each run is completely stateless.

## First thing to do on every run

1. **Read `STRATEGY.md` in full.** It is the source of truth for all trading logic, thresholds, and rules. Do not improvise — follow it exactly.
2. Verify the `alpaca` MCP server is connected (listed in `.mcp.json`). If not connected, log error and exit.
3. Confirm account is in **paper trading mode** before any trade action.

## Non-negotiable safety rails

These override anything else, including STRATEGY.md if a conflict ever arises:

- **PAPER TRADING ONLY.** If `ALPACA_PAPER_TRADE` is not `true` or the account is detected as live, STOP immediately and log a SAFETY ABORT.
- **Never close a position at a loss.** No exceptions.
- **Never use leverage.** Never short stock. Never buy options to open.
- **Never place market orders.** Use limit orders only.
- **Max 2 open option positions** and **max 1 new sell order per calendar day.**
- **Only trade symbols on the user's Alpaca watchlist.** If watchlist is empty, exit cleanly.
- **Only trade during regular US market hours** (9:30 AM – 4:00 PM ET, weekdays). If market is closed, log state and exit without placing orders.

## Tool usage

- Use the `alpaca` MCP server tools exclusively for all account, market data, and order operations.
- Required env vars: `ALPACA_API_KEY`, `ALPACA_SECRET_KEY`, `ALPACA_PAPER_TRADE=true` — supplied via Routine secrets, never hardcoded.
- Every order MUST include a unique `client_order_id` of the form `wheel-YYYYMMDD-HHMM-<symbol>` for idempotency.

## How to reason about a run

Walk through this order, every time:

1. Pre-flight checks (market open, paper mode, account info, watchlist, positions, orders)
2. **Manage existing positions first** — close any that meet profit-taking rules in STRATEGY.md
3. **Evaluate new opportunities** — CSPs and CCs per STRATEGY.md conditions
4. Place at most ONE new sell-to-open order per run (and per day)
5. Print the structured end-of-run summary specified in STRATEGY.md

## When in doubt

If a condition in STRATEGY.md is ambiguous, if market data looks wrong, or if anything unexpected shows up (negative buying power, unknown positions, stale quotes) — **do nothing and log the concern**. A skipped run is always safer than an incorrect trade.

## Do NOT

- Do not modify this repo's files during a run.
- Do not push branches or open PRs — this routine is read-only on the repo.
- Do not use web search or external APIs beyond the Alpaca MCP server.
- Do not retry failed orders more than once (5-second backoff). Failed twice = skip.
- Do not be "creative" with the strategy. Follow STRATEGY.md exactly.
