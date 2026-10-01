# Running on Robinhood Chain

This build targets Robinhood Chain (EVM, Arbitrum Orbit rollup, chain id 4663,
gas in ETH, blocks every ~100 ms) and tokens launched through Pons.

The engine, the renderer, the storage layer, the collector and the tests are
chain-agnostic: they consume an ordered list of trades and nothing else. Only
the feed and a few numbers change.

## What is configured, not rewritten

| Where | Key | Value |
|---|---|---|
| `src/savefly.config.json` | `network` | `robinhood` |
| `src/savefly.config.json` | `mint` / `ca` | the `0x...` token address |
| `src/savefly.config.json` | `quoteScale` | `22` |
| `ingest/config.json` | `network` | `robinhood` |
| `ingest/config.json` | `quoteScale` | `22` |
| `ingest/config.json` | `source` | `aggregator` |

GeckoTerminal indexes Robinhood Chain under the network id `robinhood`,
including Pons pools, so both the in-browser provider mode and the collector
work through the same endpoints they used before:

```
/networks/robinhood/tokens/<token>/pools
/networks/robinhood/pools/<pool>/trades
/networks/robinhood/pools/<pool>/ohlcv/day
```

## quoteScale: why it exists

The economy constants (`DUST_SOL`, `SIZE_REF` and everything derived from
them) were calibrated on pools quoted in SOL. Here pools are quoted in ETH,
which is roughly 22x more valuable, so the same trade arrives as a number 22x
smaller. Without correction almost every real trade would fall under the dust
threshold and the web would stay empty.

`quoteScale` multiplies every incoming amount before it reaches the
simulation:

> in the browser, in the provider feed parser
> in the collector, when a trade is turned into a log row

So the CSV log stores engine units, and the client reads them as is. Both
sides compute identical state, which is what the shared-state guarantee rests
on.

Two consequences worth knowing:

> `quoteScale` must be identical for the client and every collector. The
>   collector writes it into `meta.json` and refuses to append to a log that
>   was written with a different value.
> changing it on a live log is the same class of mistake as changing the
>   price unit: every size in the log would be on a different scale.

22 is a starting point, not a law. It is the ETH/SOL ratio at the time of the
port. If your token trades in much smaller or larger sizes than a typical
Solana memecoin, tune it with `test/economy.test.js` against your own pool
instead of guessing.

## What is Solana-only and is NOT used here

> `ingest/rpc-source.js` and `source: "rpc"` in the collector config. It reads
>   Solana slots through `logsSubscribe` and `getBlock`. It exists for tokens
>   above ~3400 trades/min, which the aggregator window cannot keep up with.
>   On Robinhood Chain the equivalent would be `eth_getLogs` on the pool, and
>   it is not written. Leave `source` at `aggregator`.
> `onchain/` (write.js, verify.js, loader.html). Those put the game into a
>   Solana account. The EVM equivalent means chunking the code across
>   contracts because of the 24 KB limit, which is a separate project. Nothing
>   in this repo claims the code is stored on chain.
> `tools/rpc-check.js`, which benchmarks a Solana RPC endpoint.

The measurement numbers in `docs/FEED.md`, `docs/ECONOMY.md` and
`docs/KNOWN-ISSUES.md` were taken on Solana pools. The mechanisms they
describe did not change; the specific numbers were not re-measured here.

## Launch order

Same rule as before, and it still cannot be fixed afterwards: the collector
must be running **before** the token starts trading, because genesis is the
first trade and the log is append-only. In `provider` mode (no collector) this
does not apply, since each viewer builds state from the last ~300 trades.

For a Pons launch, check first whether GeckoTerminal indexes the token while
it is still on the bonding curve. If it only appears after graduation, start
the collector then and say so plainly rather than claiming the fly was born
with the token.
