# RESULTS — btc-race-24h-20261117-0603-ono10

Entry for the make.fun competition `btc-race-24h-20261117-0603`
("Race a BTC trading bot live for 24 hours"), built from scratch for account
@ono10. Bot: `main.ts` (Node 22, `agent-fun/bars-replay/v1`).

## Strategy summary

A slow, patient swing around a long anchor. On 1-minute bars a full round trip
costs ~30 bps (4 bps fee + 2 bps modelled slippage + ~11 bps of price impact
per $1,000 leg) while the typical 1-minute move is only ~6 bps, so fast
systems (EMA crosses, RSI, Bollinger, Donchian) all lost money on the dev
window in local replay — they pay the round trip to chase noise.

- **Anchor:** 240-bar (4-hour) simple moving average of closes.
- **Buy** the whole cash balance (capped at the harness weight limit) when the
  close dips 0.4% below the anchor.
- **Sell** the whole position when the close rallies 0.6% above the anchor.
- **Crash guard:** 3% trailing stop from the highest close since entry.
- **Re-entry:** after a stop-out, buy again only once the close has recovered
  1% above the stop-out price.
- Long only, at most one position, at most one order per batch, max slippage
  1,000 bps (binds only on a gap). All decisions are exact bigint comparisons
  over closes seen so far — no clock, no environment, no randomness.

## Dev replay numbers (kit `grade-local`, `--driver process`)

Dev window: 1,440 one-minute CBBTC bars, 2026-11-16 06:04 → 2026-11-17 06:03 UTC.
Graded twice by the kit (two executions); both runs produced the identical
action log, so the bot is deterministic.

| Metric | Value |
|---|---|
| Net PnL | **+$12.339954** |
| Return | +1.234% |
| Trades (fills) | 6 (3 round trips), 0 rejected |
| Max drawdown | 1.086% |
| Fees paid | $2.421456 |
| Win rate | 66.7% (2 of 3 round trips) |
| Sharpe (per-bar marks) | 0.021 |
| Exposure | 29.9% of the window |
| Net PnL at 2x fees | +$9.921040 (still positive) |
| `result_sha256` | `5a0780997978bd09dfe9ba0d8f8a963afa60978a06cf353410227822d20372a7` |
| `action_log_sha256` | `4c0deb2d566d4d714fc8477810f0dd599574f4c035f98ad5a1165d6ad3f0f3f3` |

## Baseline / floor comparison (same kit, same command)

| Bot | Net PnL | Trades | Max drawdown |
|---|---|---|---|
| **This entry (swing)** | **+$12.34** | **6** | **1.09%** |
| momentum (house) | +$2.45 | 1 | 2.12% |
| buy-and-hold (house) | +$1.07 | 1 | 2.12% |
| hold-cash (house) | $0.00 | 0 | 0.00% |
| rsi-bollinger | −$24.25 | 11 | 2.79% |
| adx-momentum | −$32.66 | 26 | 3.60% |
| ema-cross (house) | −$106.21 | 68 | 10.96% |

The entry beats the best house baseline (momentum, +$2.45) by **+$9.89**,
about 5x, and beats buy-and-hold by +$11.27.

## Gate verdict: PASS

- **Dev gate:** net PnL +$12.34 > $0 — passes with margin (the gate needs any
  positive PnL; buy-and-hold passes by only +$1.07).
- **Race guardrails:** 6 trades (≥1 required) and 1.09% max drawdown (≤25%
  required) — both met with wide margin.
- **Determinism:** two grade-local executions gave byte-identical action logs.

## Robustness checks (local replay, same fee/impact model)

- The parameter family is positive throughout, not one tuned point:
  anchor 180–300 bars, dip 0.3–0.6%, rally 0.5–0.9% → +$4.26 to +$14.29.
- First 12h of the dev window alone: +$13.11; second 12h: −$0.73 (flat).
- Rejected on the numbers, after independent relay review: ATR-scaled bands
  (−$9 to −$31, 14–27 trades, whipsaw) and a trend-chase add-on
  (−$1.07 to −$5.07). Wider fixed bands (0.8%/0.8%) still made +$10.82.
- Relay second opinions (Groq, Xkiro, Xingchi/Codex) found no lookahead or
  fill-latency flaw; Apinex, Requesty and CrowLLM were unavailable
  (billing/channel errors) and Bluesminds' models were end-of-life — no
  review was obtained from those four.

## Caveats

- The dev window is a single 24-hour slice; the race window (starting
  2026-11-17 19:18 WIB) is unseen and may behave differently. A win is never
  guaranteed — only placed entries are paid, and places go by race-window net
  PnL. In a persistent decline the bot buys dips and relies on the 3%
  trailing stop; in a fast rally it may sit in cash between swings.
- Local grade used Node v24.22.1; the server harness runs Node v26.13.1
  (grade-local notes this as its only mismatch). The bot uses only erasable
  TypeScript and the kit's SDK shim, so the version difference should not
  change behaviour.
