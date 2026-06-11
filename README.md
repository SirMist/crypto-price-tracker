[Crypto Price Tracker](https://apify.com/khadinakbar/crypto-price-tracker?fpr=data)

# 🪙 Crypto Price Tracker — Multi-Source Watchlist & Market API

Real-time cryptocurrency price tracker that fetches **prices, market cap, 24h/7d/30d changes, ATH/ATL, supply, and dominance** for a watchlist of specific coins (BTC, ETH, SOL...) **or** the top N coins by market cap. Multi-source: **CoinGecko** primary, with optional **Binance** fallback for fresher tick data and resilience to rate limits.

Built MCP-first for AI trading agents, portfolio bots, and crypto research. Cheap targeted runs (5-coin watchlist ≈ $0.015) or full market sweeps (top 100 ≈ $0.30).

## What you get

| Field | Type | Notes |
| --- | --- | --- |
| `symbol` | string | Uppercase ticker (BTC, ETH, …) |
| `name` | string | "Bitcoin", "Ethereum" |
| `coingeckoId` | string | Stable slug (e.g. `bitcoin`) |
| `quoteCurrency` | string | `usd`, `eur`, `gbp`, `jpy`, `cny`, `btc`, or `eth` |
| `price` | number | Current price in `quoteCurrency` |
| `marketCap` | number | Market capitalization in `quoteCurrency` |
| `volume24h` | number | 24h trading volume in `quoteCurrency` |
| `change1h` / `change24h` / `change7d` / `change30d` | number | Percentage changes |
| `ath` / `athDate` / `athChangePct` | number/string | All-time high data |
| `atl` / `atlDate` | number/string | All-time low data |
| `circulatingSupply` / `totalSupply` / `maxSupply` | number | Supply metrics |
| `marketCapRank` | number | 1 = highest |
| `dominancePct` | number | Share of total crypto market cap (market mode) |
| `binancePriceUsd` / `binanceVolume24hUsd` | number | Binance USDT fallback ticker (always USD) |
| `priceSource` | string | `coingecko` or `binance` |
| `lastUpdated` | string | ISO 8601 |
| `coingeckoUrl` | string | Public CoinGecko page |
| `scrapedAt` | string | ISO 8601 |

## Why use this

Most crypto scrapers on Apify dump the entire CoinGecko/CoinMarketCap database on every run — paying for thousands of coins you don't care about. This actor gives you a **watchlist mode** so you only pay for the 5–20 coins you actually track, and a **market mode** for the top N when you need a broader sweep.

- **Watchlist mode** — track BTC, ETH, SOL, your portfolio. 5 coins = $0.015.
- **Market mode** — top 100 by market cap = $0.30. Top 1000 = $3.00.
- **Multi-source resilience** — CoinGecko gets rate-limited at 30 req/min; Binance fallback keeps fresh prices flowing.
- **AI-agent friendly** — flat schema, stable keys, optional `concise` response format for LLM context.
- **No API keys** — uses free public endpoints with smart rate-limit handling.

## Pricing — Pay Per Event

| Event | Price |
| --- | --- |
| Actor Start (per GB-RAM) | $0.00005 |
| Coin Tracked | **$0.003** per coin |

Plus standard Apify platform usage (proxy, compute, storage) — billed against your Apify plan. The actor uses 512 MB and pure HTTP (no browser), so a typical 100-coin run uses < 0.0002 compute units.

### Cost examples

- 5-coin watchlist (BTC, ETH, SOL, XRP, DOGE): **$0.015** + ~$0.00005 start
- 25-coin watchlist: **$0.075** + start
- Top 100 market sweep: **$0.30** + start
- Top 1000 market sweep: **$3.00** + start

## How to use it

### Inputs

| Field | Required | Default | Notes |
| --- | --- | --- | --- |
| `mode` | yes | `watchlist` | `watchlist` or `market` |
| `symbols` | conditional | `["BTC","ETH","SOL","XRP","DOGE"]` | Required when `mode=watchlist`. Up to 100 |
| `topN` | no | 100 | Used in `market` mode. 10–1000 |
| `vsCurrency` | no | `usd` | usd, eur, gbp, jpy, cny, btc, eth |
| `useBinanceFallback` | no | `true` | Adds Binance ticker data |
| `responseFormat` | no | `detailed` | `detailed` (all fields) or `concise` (LLM-friendly) |

### Watchlist example (JavaScript)

```
const { ApifyClient } = require('apify-client');
const client = new ApifyClient({ token: process.env.APIFY_TOKEN });

const run = await client.actor('khadinakbar/crypto-price-tracker').call({
    mode: 'watchlist',
    symbols: ['BTC', 'ETH', 'SOL', 'XRP', 'DOGE'],
    vsCurrency: 'usd',
});

const { items } = await client.dataset(run.defaultDatasetId).listItems();
items.forEach((c) => console.log(`${c.symbol}: $${c.price} (${c.change24h?.toFixed(2)}%)`));
```

### Market sweep example (Python)

```
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("khadinakbar/crypto-price-tracker").call(run_input={
    "mode": "market",
    "topN": 250,
    "vsCurrency": "usd",
    "responseFormat": "detailed",
})

for coin in client.dataset(run["defaultDatasetId"]).iterate_items():
    print(f"#{coin['marketCapRank']} {coin['symbol']}: ${coin['price']} ({coin['change24h']:.2f}%)")
```

### MCP / AI-Agent usage

This actor is registered with Apify MCP as `apify--crypto-price-tracker`. From any Claude Desktop / Cursor / GPT setup with Apify MCP enabled:

```
"Get me current prices and 24h changes for BTC, ETH, and SOL"
"What are the top 50 coins by market cap right now?"
"How far is each of these coins from its all-time high: BTC, ETH, SOL, AVAX, MATIC?"
```

The agent will call this actor, get a flat JSON response, and reason directly over price + change + ATH data.

## Two modes explained

### Watchlist mode

Provide a list of tickers. The actor:

1. Loads CoinGecko's full coin list (cached 24h in KV store).
2. Resolves each symbol → CoinGecko ID using a curated preference map for top-50 coins (avoids ambiguity for tickers like `UNI` which 8+ coins claim).
3. Fetches `/coins/markets` with the resolved IDs in one call (up to 250).
4. Optionally enriches with Binance USDT-pair tickers.
5. Pushes one record per coin to the dataset.

### Market mode

Sweeps the top N coins by market cap:

1. Pages through `/coins/markets` (250 per page).
2. Computes `dominancePct` from `/global` for each coin.
3. Same enrichment + output as watchlist mode.

## Reliability

- **Retry with backoff** — 4 retries with exponential backoff (1.5s → 24s) on 5xx errors.
- **Rate-limit aware** — 429 triggers `Retry-After`-aware backoff, 30s minimum.
- **Multi-source fallback** — when CoinGecko's price is null (rare but happens during cache misses), Binance USDT ticker fills in.
- **Coin-list cache** — the 17K-coin list is cached 24h in the actor's KV store, so repeat runs skip the heavy enumeration.
- **Per-page failure tolerance** — market mode tolerates a single page failure and continues with what it has.
- **PPE limit aware** — gracefully exits when the user-set per-run PPE cap is reached.

## FAQ

**Q: How fresh is the data?**
CoinGecko refreshes every 1–5 minutes per coin. Binance fallback is sub-second. The `lastUpdated` field tells you exactly when each coin was last refreshed upstream.

**Q: Why am I getting fewer coins than I requested?**
Some tickers (like obscure altcoins) may not exist on CoinGecko, or there are multiple coins with the same ticker. Check the `summary_run` key in the run's KV store for the `unresolvedSymbols` list.

**Q: Can I track tokens not on CoinGecko?**
This actor uses CoinGecko as its source of truth for IDs. For DEX tokens not on CoinGecko, see the DexScreener actor instead.

**Q: How do I avoid duplicates across runs?**
Each run produces its own dataset by default. Use the same `defaultDatasetId` on subsequent runs (Apify SDK option) to append, or filter by `scrapedAt`.

**Q: Is this real-time?**
This is on-demand polling, not WebSocket streaming. For real-time tick data, run this actor on a schedule (every 1–5 min) using Apify Schedules.

## Legal

This actor uses CoinGecko's free public API and Binance's free public ticker API. Both APIs explicitly allow non-commercial polling at reasonable rates. This actor respects rate limits and adds smart back-off on 429 responses.

The data returned is the property of CoinGecko / Binance respectively. Cryptocurrency prices change constantly — data may be 1–5 minutes stale. **Not financial advice.** Trading cryptocurrencies involves substantial risk; do your own research.

## Support

- 📧 Issues: open one on the actor's [Issues tab](https://apify.com/khadinakbar/crypto-price-tracker/issues)
- 🐦 X: [@khadinakbar](https://x.com/khadinakbar)
- 💬 Other actors: [browse my store](https://apify.com/khadinakbar)

Related actors:

- [google-finance-stock-news-scraper](https://apify.com/khadinakbar/google-finance-stock-news-scraper) — same idea, but for stocks
- [google-news-scraper](https://apify.com/khadinakbar/google-news-scraper) — pair with crypto news monitoring
- [ai-search-brand-monitor](https://apify.com/khadinakbar/ai-search-brand-monitor) — track how AI search engines describe crypto projects