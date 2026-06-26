# Genesis + Exodus — Autonomous Robinhood Trading Scanner (Template)

**READ THIS FIRST. This is a structural template for a fully-autonomous, live trading bot.**

## What this is

A risk-first, long-only, cash-aware market scanner built as a Claude Code skill (`SKILL.md`) plus two scheduled tasks. It reads a connected Robinhood account, classifies the market regime, manages protective stops and profit-taking, and — only when strict rules pass — places a single momentum/rebound buy. The default decision is **NO TRADE**.

This repo contains **structure only**. No API keys, account numbers, balances, or trade history are included. Every `<PLACEHOLDER>` must be replaced with your own values.

## Serious warnings — understand these before you touch it

**Configured LIVE / FULL-AUTO.** The reference config sets `LIVE_TRADING = true`, `AUTO_BUY = true`, and `REQUIRE_HUMAN_CONFIRMATION = false`. It places real buy and sell orders with no human in the loop.

**Autonomous live trading can lose money fast.** A bug, a data error, a market gap, or a bad rule can drain capital with no one watching. Monitored stops only fire on scan runs, so price gaps between scans are unprotected.

**Run it in PAPER mode first.** Set `LIVE_TRADING = false` and `PAPER_MODE = true`, and watch it for a long time before risking a single dollar.

**Know the kill switch before you arm it.** `python3 scripts/ops.py halt` stops new buys and `ops.py live off` disables live trading. Editing this prose alone does NOT stop a scheduled run — the control file does.

**This is not financial advice.** It is a framework. You are fully responsible for anything it does with your money.

**Never commit secrets.** The FMP API key lives in `state/fmp.env` and must be gitignored. Never hard-code your account number — it is resolved at runtime.

## What is in here

`genesis-scanner-build-guide.md` — the full build guide: folder structure, the two deterministic safety scripts (`ops.py`, `fmp.py`), the `SKILL.md` brain, the two scheduled tasks, and the setup checklist.

## Use

Shared as a structural template. Replace all `<PLACEHOLDER>` values, start in paper mode, and understand that autonomous live trading carries real financial risk. Use at your own risk.
