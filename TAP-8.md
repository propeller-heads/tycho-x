# Markout Analysis Tool
![Markout Analysis Header](./assets/markout-analysis-header.png)
# tldr;
A tool to judge the markout profitability of any maker or taker on-chain: DEX Pools, Market Makers or Traders. Per tx, or aggregated over a time-period.
# Implementations
NA - there are no implementations yet of this Tycho Extension.
# Motivation
- **Help LPs assess the true return of a DEX pool**: Most DEX pools report fee * volume as revenue – but this omits that spot prices can be stale. Markouts measure true fee earned. A dashboard showing markout across all pools and protocols lets LPs put their assets into the truly best pools – and this makes the market much more efficient.
- **Make it easy to protect against toxic flow**: It should be easy to check the markout of an address, so that you (as a solver or maker) can protect yourself more easily from toxic flow.
- **Know where to trade**: Mean markouts will also let users compare venues (aggregators, solver, frontends) by execution quality, and help those frontend with the best win more users.
- **Assess execution quality**: Anyone interested in the quality of a trade (e.g. done by an OTC desk, or custodian or broker) should be able to check it themselves. With markouts they can.
# Background
- **There is little transparency on Maker Profits**: There are only ad-hoc, and still very few, analysis of LP profit. The ones that exist are limited to single pools (usually ETH / USDC pools) and single protocols. They all rely on Dune data and combine trade tables with price tables (which updates prices once every 5 minutes).
- **There are no open source toxic flow filters**: There is, to my knowledge, no public project that allows you to qualify taker flow by its toxicity. So everyone who wants to protect themselves against toxic flow has to build their own monitoring.
# Goal
The goal of this project is to let anyone calculate the mean markout for any set of trades.
# Specification
## User Stories
- **Trader looking for the best price**: As a trader I want to know which router / auction has which execution quality.
	- **By trade type**: I might want to filter for trades similar to mine (size and possibly tokens)
- **Maker looking to protect against toxic flow**: As a market maker, or as a solver (acting for the market maker), I want to be able to assess whether to share quotes and settle trades for a specific taker – or whether to be careful with a taker, because they might be selecting quotes and taking adversarially.
- **LPs looking for best yield**: As an LP I want to know which pools have the best returns – and use markout as one of the metrics to judge this.
	- **Search by token**: I want to know the best pool for a certain token, or token pair.
	- **Display my LP positions**: I want to see the markout for all the pools in which I currently hold an LP position (pool markout not specific for my position, that's likely too complicated).
- **As an institutional client**: I want to assess the execution quality of my broker – enter their address and see their markout, and possibly compare it to my expectation or to the markout of other brokers.
## Definitions
- **Markout**: The percentage difference in price (in basis points (BPS=0.01%)) between the price of your trade and the market mid-price at a point in time after your trade (e.g. 5min).
- **Toxic Flow**: Trading activity that systematically captures value from liquidity providers through superior information or timing, resulting in negative expected value for the maker.
- **LP (Liquidity Provider)**: An entity that deposits assets into a DEX pool to facilitate trading and earn fees.
- **Maker**: The party providing liquidity in a trade (e.g., DEX pool, market maker).
- **Taker**: The party consuming liquidity in a trade (e.g., trader, arbitrageur).
- **Router/Solver**: An intermediary system that finds optimal trading paths and executes trades on behalf of users. 
## Requirements
### Essential Requirements
- **Track every Pool trade:** For every pool in every protocol in Tycho track every event where a pool swaps assets. Save this in permanent storage, for at least 3 months history.
- **Token Prices**: A dataset of prices for every token. Highly accurate on market mid-price (< 1 BIP) for that moment, and high resolution (at least one price every minute).
- **Pool, taker and router labels**: Labels for which pool this trade touched, who was the taker, who the router and or solver. 
- **Tool to calculate markout**: A user can specify a pool, a time period to analyse, and a markout period (e.g. 1h) and an address (taker, pool (maker) or router). The tool then retrieves the necessary data, calculates the markout for every trade at the markout period (e.g. 1h), and returns the full table of trades and markout, and the mean markout over all trades.
### Important Requirements
- **Calculate Pool Markouts Daily**: Calculate markouts for every pool above 50 ETH TVL once per day. For 0 seconds, 30 seconds, 5 min, 30 min, 2 h, 12 h, 24h and 72 h markout – averaging results over the last week.
- **Calculate Router Markouts Daily**: Calculate markouts for every hardcoded router / solver address once per day. For 0 seconds, 30 seconds, 5 min, 30 min, 2 h, 12 h, 24h and 72 h markout – averaging results over the last week.
- **Display Markouts UI**: Design and deploy a public UI that retrieves and displays the most recent 4 weeks of markout values for all pools and routers in one dashboard.
	- **Pool List**: List of pools with columns: Address, Protocol, Tokens, TVL, Number of trades, markout (for each markout period 0 seconds, 30 ...).
	- **DEX List**: Mean markout for all DEXs: DEX name, number of pools, number of trades, markout (for each markout period 0 seconds, 30 ...)
	- **Router List**: Mean markout for all Routers: Router name, number of trades, volume, markout (for each markout period 0 seconds, 30 ...)
- **Download saved markouts**: Add a tool to download the per-calculated markouts.
- **Ad-hoc Taker Markout**: Let users enter a taker address, and the tool, ad-hoc, calculates the markout for that taker address, and classifies it as likely toxic or not depending on its markout and number of transactions.
### Nice-to-have Requirements
- **Top Taker Markouts**: Once a day, identify the top N takers by number of txs (or volume) – and calculate their 30 seconds, 5 min, 30 min, 2 h, 12 h, 24h and 72 h markout over all their trades.
- **Saved file with potentially toxic takers**: Keep a file up to date, that anyone can programmatically download, with addresses and mean markout values – so makers can protect themselves from toxic takers.
## Not included
# Implementation
## Data Sources
- **Trades**: Could get trades from [Dune Dex Trades](https://docs.dune.com/data-catalog/curated/evm/DEX/dex-trades). Or index your own (likely much more effort). (Note caveats on duplicate counting [noted here](https://docs.dune.com/data-catalog/curated/evm/DEX/dex-trades) – but it's not too critical since we are looking for average markouts.)
- **Prices**: There are many APIs and historical datasets. But you can also build your own using the Tycho [Token Quoter](https://github.com/propeller-heads/tycho-x/blob/main/TAP-3.md) – and saving prices every block to cloud storage.

# Rationale

# References

# Risks
- **Gaming metrics**: Sophisticated traders might split trades across addresses or time to avoid detection.
- **Computational overhead**: Calculating markouts for millions of trades across multiple time horizons requires significant resources.