# Orderbook
![Orderbook Header](./assets/orderbook-header.png)
# tldr;
All on-chain liquidity for a token pair presented in familiar orderbook format. Including the routes to execute at a price / depth.
# Implementations
NA - there are no implementations yet of this Tycho Extension.
## Dependencies
To complete this project you still need a simple 1-hop split solver and in order to provide the full call data for trade execution you need Tycho Execution.
These open dependencies should be available by the end of Feb 25 – feel free to mock them til then.
An more mature version of this orderbook interface will also need a fast, local, good routing solver – which is planned as a separate Tycho X project.
# Motivation
- **Bring more traders on-chain**: In the familiar format of an orderbook, on-chain liquidity will be easier to read and execute over for a wider set of traders. This can bring new traders, e.g. those with existing strategies for orderbook markets, to use on-chain liquidity.
- **Strengthen on-chain liquidity**: Making on-chain liquidity more accessible can increase on-chain volumes, and consequently revenues, and incentivise deeper on-chain liquidity, and the resulting deeper books further incentivises more on-chain trading through better prices.
- **Aggregate liquidity**: Another high barrier for traders to trade efficiently against on-chain liquidity is to use the liquidity fragmented over different pools and protocols. A unified orderbook also abstracts the different protocols and pools – and presents them in a unified form.
- **Save time**: Unless you have a specifically optimised setup – its hard to do all the simulations required for running sophisticated trading strategies (e.g. determining the supply for 1000 different tokens at 10 different trading volumes). A unified orderbook  includes pre-simulated trade amounts that traders can use directly, or as a starting point for "final mile" simulations.
# Background
- **Most traders are used to orderbooks**: Most liquidity historically and by volume, sits in orderbooks – and consequently most traders, their trading systems and strategies are uniquely adapted to the structure and interface of orderbooks. However, on-chain liquidity is largely not in classical two-sided limit orderbooks, but instead in unified liquidity pools governed by continuous supply curves (e.g. CFMMs) – which have a very different trading interface than orderbooks.
- **Liquidity is fragmented**: The liquidity for a given token or token pair, is commonly fragmented amongst several DEX protocols and pools. Moreover the liquidity useful to achieve the best price on a given trade extends beyond pools of just those pairs – to any path over any set of pools that starts and ends in the desired token pair.
# Goal
Provide all on-chain liquidity in a familiar limit orderbook interface to read (ticks and depth per tick) and write (execute, confirmation) to.
# Specification
## Definitions
- **Hop**: A single step in a trading route that moves from one token to another through a DEX pool. For example, trading ETH->USDC in a single Uniswap pool is 1 hop, while ETH->WBTC->USDC through two pools is 2 hops.
- **Price Level**: A point in the orderbook representing available liquidity at a specific price, including the route to execute that trade.
- **Route**: The optimal sequence of DEX interactions (including any splits across protocols) needed to execute a trade at a given price level.
- **Gas Costs**: Transaction fees required to execute a trade, which can be factored into price calculations in different ways (included (net), excluded (gross)).
## Requirements
### Essential Requirements
- **Unify liquidity of all "1-hop" pairs**: For a given token pair (A,B) the orderbook contains all liquidity of all permissionlessly and composable on-chain pools that contain tokens A and B.
- **Splitting**: The displayed liquidity includes optimal splits between pools.
- **Interface Feedback**: Get feedback and iterate on the interface (to simulate, execute, and track trades) with prospective users.
### Important Requirements
- **Orderbook Events**: Tycho already manages DEX state updates seamlessly, but to traders it's useful to have a stream of orderbook update events to react to. So emit an event for every change in the orderbook.  
- **Executable routes included**: For any point on the orderbook (depth or price) return the call data a trader would need to sign to execute that trade.
- **Include Gas Costs**: Prices are net gas costs (roughly). I.e. the cost to access the liquidity is calculated into the price.
- **Multi-hop paths**: The liquidity includes all liquidity accessible over muti-hop directed acyclical graph (DAG) paths,, up to n (e.g. 3) hops, and splits.
### Nice-to-have Requirements
- **Local fine-tuning**: Traders can request not only the provided points on the orderbook but also any arbitrary point in between. Those calls run a iterative search (solver) for the correct amount – within a certain window of tolerance.
## Not included
# Implementation
*Just a draft for inspiration – take and leave what you like.*
```ts
// Core Domain Types
type Side = 'buy' | 'sell'
type GasMode = 'include' | 'exclude'

type TokenPair = {
  token0: string  // Base token address
  token1: string  // Quote token address
}

type Route = {
  protocols: string[]  // DEX protocols used
  calldata: string    // Executable calldata including splits
}

type Level = {
  price: string
  size: string
  route: Route
  gasCost: {
    wei: string
    token0: string
    token1: string
  }
}

// Book State
type OrderbookState = {
  pair: TokenPair
  timestamp: number
  blockNumber: number
  bids: Level[]  // Sorted best to worst
  asks: Level[]  // Sorted best to worst
}

// Query Options
type OrderbookOptions = {
  maxHops?: number
  maxSplits?: number
  includedProtocols?: string[]
  pricePoints?: number
  gasMode: GasMode
  gasToken?: string
  maxGasPriceGwei?: string
}

type FinetuneOptions = OrderbookOptions & {
  tolerance: string
  maxIterations?: number
}

// Operation Results
type SizeQuote = {
  effectivePrice: string
  route: Route
  gasCost: Level['gasCost']
  priceImpact: string
}

type DepthAtPrice = {
  totalSize: string
  averagePrice: string
  route: Route
  gasCost: Level['gasCost']
}

// Event System
type OrderbookEvent = {
  type: 'price_level_changed'
  pair: TokenPair
  timestamp: number
  blockNumber: number
  side: Side
  price: string
  newSize: string
} | {
  type: 'book_reset'
  pair: TokenPair
  timestamp: number
  blockNumber: number
}

// Core Operations
type Operations = {
  getOrderbook: (pair: TokenPair, options?: OrderbookOptions) => Promise<OrderbookState>
  
  getQuoteForSize: (
    pair: TokenPair,
    size: string,
    side: Side,
    options?: OrderbookOptions
  ) => Promise<SizeQuote>
  
  getSizeAtLimit: (
    pair: TokenPair,
    limitPrice: string,
    side: Side,
    options?: OrderbookOptions
  ) => Promise<DepthAtPrice>
  
  findOptimalSize: (
    pair: TokenPair,
    targetPrice: string,
    options?: FinetuneOptions
  ) => Promise<SizeQuote>
  
  subscribeToEvents: (
    pair: TokenPair,
    onEvent: (event: OrderbookEvent) => void
  ) => { unsubscribe: () => void }
  
  getSupportedTokens: () => Promise<string[]>
  getSupportedPairs: () => Promise<TokenPair[]>
}
```
# Rationale
- **Pre-calculate v.s. real-time local calculation**: Should we pre-calculate liquidity (and therefore also execution paths) let users do so locally ad-hoc?
	- **Pre-calculation**: (+) provides fast responses and (-) wastes compute on unused price points
	- **Local calculation**: (+) computes only what's needed and (-) duplicates work across users
	- **Solution**: Provide both: 
		- Pre-calculated baseline orderbook for common price points
		- Ability to fine-tuning locally for precise sizes when needed
# References
- [Binance Spot API Docs](https://developers.binance.com/docs/binance-spot-api-docs)
- [CME Book Management Docs](https://cmegroupclientsite.atlassian.net/wiki/spaces/EPICSANDBOX/pages/457223312/MDP+3.0+-+Book+Management)
# Risks
