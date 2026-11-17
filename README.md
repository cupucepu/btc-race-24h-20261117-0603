# TIDE v2 — dip-buying mean-reversion bot for the BTC 24h race

[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/cupucepu/btc-race-24h-20261117-0603/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/cupucepu/btc-race-24h-20261117-0603/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

TIDE trades Coinbase Wrapped BTC (cbBTC) against USDC on 1-minute on-chain
bars, long only, with $1,000 of paper money. It races the 24 hours that start
15 minutes after the deadline of
[btc-race-24h-20261117-0603](https://make.fun/bounty/btc-race-24h-20261117-0603).

## Idea

The dev window is range-bound and mean-reverting: hourly moves of 50–100 bps
reverse instead of continuing (after a +80 bps hour, the next hour rose only
21% of the time). Trend-following on this data loses money — every kit
baseline that chases breakouts ends the dev window negative, because each
round trip costs ~30 bps in fees, slippage and pool impact while the moves
are ~1%. So TIDE does the opposite: it buys dips and sells back at the mean,
rarely.

## Rules

All exact bigint arithmetic on the bar closes; fully deterministic.

- **Mean** = 360-bar simple moving average of closes (6 hours of 1-min bars).
- **BUY** (full cash) when the close falls 85 bps or more below the mean.
- **SELL** (whole position) when the close recovers to 35 bps above the mean.
- **Kill-switch**: if the close falls 200 bps below the mean, the regime has
  broken down, not dipped — sell if holding and stop buying dips until the
  close recovers to within 100 bps below the mean.
- **No fresh dip entries** in the last 30 bars of the window (a fresh entry
  there would pay a full round trip for minutes of exposure).
- **Fallback**: if no trade has happened by bar 1080 and the close is within
  100 bps below the mean, buy once and hold to the end. This guarantees the
  entry meets the race guardrail (at least 1 trade) even on a steady up-trend
  day where no dip ever comes, and it catches late upside. The fallback never
  buys a spike top: while the close sits more than 50 bps above the mean it
  keeps waiting for a pullback, but by bar 1200 it buys regardless, so the
  guardrail trade is delayed, never skipped.

Tuned from v1 (dev +$8.96) by a full parameter sweep on the dev window plus
an 8-window stress test; the DIP 90 / EXIT 30 config scored higher on dev
(+$13.57) but lost −$5.71 on the −3% drift day, so it was rejected — this
config wins on the stress average, not just on dev.

One asset, long only, at most one position, at most one order per reply.
No network, no wall clock, no randomness, no files outside the repo.

## Local grade (dev window, official harness)

```
node cli/makefun.mjs grade-local --kit <kit> --bot . --driver process
```

| | TIDE v2 | buy-and-hold | momentum | ema-cross | rsi-bollinger | adx-momentum |
|---|---|---|---|---|---|---|
| Net PnL | **+$11.39** | +$1.07 | +$2.45 | −$106.21 | −$24.25 | −$32.66 |
| Trades | 4 | 1 | 1 | 68 | 11 | 26 |
| Max drawdown | 0.72% | 2.12% | 2.12% | 10.96% | 2.79% | 3.60% |

Dev gate (net PnL > $0) passed with wide margin; determinism check
(two executions, byte-identical action logs) reproduced. Sharpe 0.0238;
net PnL at 2× fees still +$9.78.

Robustness was checked on 8 windows (dev halves, reversed dev, synthetic
±1.5% and ±3% drift days): average net PnL +$5.90, worst +$0.89 (every
window profitable), max drawdown never above 1.17% — the strategy holds up
across regimes instead of being tuned to one day.

## Files

- `main.ts` — the bot (Node 22, standard library only)
- `vendor/makefun-bars.ts` — the kit's bars-replay SDK shim, byte-identical
  copy under `vendor/` as the kit instructs

## License

MIT. See [LICENSE](LICENSE).
