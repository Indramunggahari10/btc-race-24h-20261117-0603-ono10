# btc-race-24h-20261117-0603-ono10

[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

A deterministic swing bot for the make.fun competition "Race a BTC trading bot
live for 24 hours" (`btc-race-24h-20261117-0603`). It trades Coinbase Wrapped
BTC (CBBTC) against USDC on 1-minute bars, starting from $1,000 of paper money,
under the `agent-fun/bars-replay/v1` protocol. See [RESULTS.md](RESULTS.md) for
the dev replay numbers.

## Strategy

On 1-minute bars a full round trip costs ~30 bps (4 bps fee + 2 bps modelled
slippage + ~11 bps of price impact per $1,000 leg), while the typical 1-minute
move is only ~6 bps. Fast systems (EMA crosses, RSI, Bollinger, Donchian) all
lost money on the dev window in local replay: they pay the round trip to chase
noise. What survives is a slow, patient swing around a long anchor:

- **Anchor:** the 240-bar (4-hour) simple moving average of closes.
- **Buy** the whole cash balance when the close dips 0.4% below the anchor.
- **Sell** the whole position when the close rallies 0.6% above the anchor.
- **Crash guard:** a 3% trailing stop from the highest close since entry.
- **Re-entry:** after a stop-out, buy again only once the close has recovered
  1% above the stop-out price, so a still-falling market is not re-bought.

Long only, at most one position, at most one order per batch, a handful of
trades a day. Every decision is an exact integer (bigint) comparison over the
closes seen so far plus the account the harness reports — no clock, no
environment, no randomness — so two runs on the same data produce identical
orders.

## Layout

- `main.ts` — the bot (Node 22, erasable TypeScript; run as `node main.ts`).
- `vendor/makefun-bars.ts` — byte-identical copy of the kit's bars-replay SDK shim.
- `RESULTS.md` — dev replay numbers, baseline comparison, gate verdict.

## Local grade

From inside the competition kit:

```
node cli/makefun.mjs grade-local --kit . --bot <path-to-this-repo> --driver process
```

## License

MIT — see [LICENSE](LICENSE).
