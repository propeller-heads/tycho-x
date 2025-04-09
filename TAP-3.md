# Price Quoter
![Price Quoter Header](./assets/price-quoter-header.png)
# tldr;
A fast, trustless price oracle library for all tokens.
# Implementations
NA - there are no implementations yet of this Tycho Extension.
# Motivation
**High speed, broad coverage, 0 downtime price oracle**: Calculate prices for tens of thousands of tokens in milliseconds, cover every token that is traded anywhere on-chain, as soon as there is a single pool – and have the data availability and reliability guarantees as the chain itself.

**Highly reliable and adaptive - based on routing**: Calculate prices on exactly where liquidity for this token is right now – and adapt the calculation logic dynamically to any movements in liquidity to different pools.

**Transparent and trustless**: Make pricing logic simple, transparent and let clients verify instead of trust the data integrity - and hence the derived prices

**Flexible**: Make it easy to modify the price quoter to individual demands. Adjusting the token coverage, or price derivation logic.
# Background
**Token Price APIs are not enough**: Today, the only way to get the current price of a token is to use one of many token price APIs. However they are limiting for several reasons:
- **High latency**: To get a price for a token, you need to add at least one network roundtrip. In the context of many users (traders or solvers) this is slow, and especially so if you want to know the prices for thousands or even hundreds of thousand of tokens.
- **Limited coverage**: Every token price provider makes tradeoffs and does not cover *every* token. But many users have special needs – e.g. needing to know prices for tokens that launched *this block* immediately (meme coin), or for tokens with low liquidity, or for tokens that only exist on long-tail DEXs (e.g. only on Curve) that the price API does not cover.
- **Trusted**: Consumers need to trust the third party provider on the integrity of the data. Any mistake (e.g. block delay, or bug in aggregation logic) can have serious consequences for some users (e.g. those that trade based on the price information). 
- **Lack of transparency**: How prices are derived is rarely disclosed (e.g. is it a median or mean of Univ3 pools at mid price, or does it include / exclude spread) – and there is no, or very limited, flexibility for consumers of the API to change how price is derived.
- **Downtime**: All APIs have downtime, which translates to economic loss (for traders), or at least bad UX (e.g. for wallets). And providers might make changes, or discontinue the service at any time.
- **Costly**: For production usage, most APIs also charge and can become costly, excluding many use cases that require frequent and broad coverage of tokens. (e.g. pairs trading that covers thousands of tokens).

**Oracles are not enough**: Oracles suffer from many of the same limitations – they are entirely dependent on third parties, the aggregation logic is frequently not public, and they can have downtime at any time that consumers have no control over as they rely on off-chain infrastructure. And they only cover a very limited set of tokens. Additionally: 
- **Mostly cover tokens traded on CEXs**: Most oracles rely on market maker feeds based *purely* on CeFi liquidity. Meaning that tokens that are traded only or primarily on-chain are not supported by many oracles or oracle price providers.
- **Do not provide realistic liquidity**: Most oracles are used for on-chain use cases. But here, e.g. for a lending protocol's liquidation margin, what often counts more is liquidity that is available now on-chain, not off-chain.

**Single pool on-chain oracles are also not enough**: Relying on a single, often hard-coded, pool for price quotes is also not reliable: The price on a single pool can easily vary by its spread from the true market mid-price, and, more importantly, liquidity can entirely move, in a single block, to another pool – making the price on the original pool outdated and easily manipulable.

**Use Cases for token quoting**: A trustless, broad coverage and low latency token quoter is useful to many. For example: 
- Convert gas fees into out-token amount as a solver/searcher/marketmaker/trade/relayer/paymaster or dapp UI.
- Build a price provider to oracles like Pyth
- Track portfolio value for wallets and wallet trackers
- Build a real-time price chart to track on-chain token prices, or
- Calculate effective spread / price impact and slippage on quotes from exchanges / RFQ or OTC.
# Goal
The goal of this project is to build a trustless, super low-latency, highly reliably and flexible price quoting library that anyone who want to have real time prices on any token traded on-chain can run themselves and use freely.

The quoter is fast enough to update the price for every token that was traded in the last 30 days, within 500ms after new block deltas arrived.

*Note that this is meant to be a library, not an API or UI.*
# Specification
## Definitions
## Requirements
### Essential Requirements
- **Price in ETH**: The ability to get the price for any token (given the token address) in terms of ETH.
- **Arbitrary Numeraire**: The ability to get the price of any token in terms of an arbitrary numeraire token (defined by its address) – e.g. USDC, EURC, etc. (and the preset option to get quotes in "native" token (e.g. ETH))
- **Arbitrary depth**: Let users define the depth at which to probe the orderbook to define the mid-price. Denominated in the numeraire.
- **Gas Price**: Include the cost of gas to execute the trade in the net price (and hence in the spread).
- **Calculate and cache all prices**: Include a default setting where the app calculates and holds in memory all prices for all tokens it knows of. And updates them with every block continuously.
- **Language**: Preference for Rust, ideally with additional Python bindings.
### Important Requirements
- **Cache for faster queries**: Cache all paths you calculated. And then only re-check the paths where at least on pool had an update since the block that this path was last updated.
- **Spread at depth**: Return also spread at chosen depth, in basis points. Gross and net gas (if gas feature is implemented otherwise just gross gas).
- **Save price history**: Let users define tokens to track and calculate and save the price of these tokens to file at every block update, as long as the indexer is running.
### Nice-to-have requirements
- **Percentage Depth**: Calculate the amount of tokens needed to move the price on this token pair (numeraire, quote) by 0.5, 1, and 2%. (e.g. Binary search simulate multiple swap amounts until you get within an acceptable margin of the target price impact.)
- **Split across non-overlapping paths**: Instead of using the single path with the best price, use all parallel, non-overlapping, paths and split the trade efficiently between them, to receive a more precise quote.
- **N-hop parameter**: Let users define the maximum number of hops from token to numeraire token that the network search is allowed to take. Less hops will not cover some tokens, and might overlook some more liquid paths, but will be much faster. (absolute max should be no higher than 4 as the network will unreasonably explode here and might lead to bad UX if someone doesn't appreciate the implications of this setting).
- **Find best net price**: Instead of using a fixed depth, find the depth at which you can achieve the best net price (`net_price = (amount_out - gas_interms_of_token_out) / amount_in`) in both directions. Then take the mid-point between those prices.
## NOT included
- **Hosted Service**: A hosted endpoint for token prices
- **UI**: A UI to display token prices, or price history.
# Implementation
- User sets parameters (numeraire token, probing depth in terms of numeraire, TVL filter).
- Quoter keeps up to date with latest changes with Tycho
- Quoter keeps in memory all paths from numeraire to all tokens (within TVL).
- Quoter recalculates the price for every path where at least one pool had an update in the last block.
- Quoter holds in memory spot prices for all tokens
- Quoter responds to user query for any spot price.
- You can derive the price for any token in terms of any other token in the following steps:


To calculate the price for any token:
1. Build a network from all available liquidity pools (nodes are tokens, pools are edges).
3. Find all paths of max depth N (N-hop parameter) from numeraire to the target token.
4. Swap M tokens (probing depth parameter) of numeraire to target token on all paths.
5. Account for gas cost of the path (deduct from amount out) – you can use the route's own price for that.
6. Pick the path with best (net gas) price.
7. Swap the amount out given by the first swap, back into the other direction on the same path.
  - (Optional improvement) Swap the amount back on all paths, and pick the one that give the best net gas price.
8. Token Price: The mean of the two prices from the two swaps, denominated in the numeraire token, is the tokens price.

# Rationale
- **Uniquely possible with Tycho**: You can only follow liquidity to new pools and protocols, and to cover the widest range of tokens, if the liquidity indexer covers the broadest range of protocols and pool types, and constantly updates to newly deployed protocols. This is Tycho's strength, making it possible to support a token quoter that adapts to protocols and liquidity automatically.
- **Depth is a relevant metric**: Many applications of token quotes require not only the mid-price, but also the effective spread and depth (e.g. assessing collateral margins for lending pools).
# Future Considerations
- **State verification**: Verify the data is coherent and up to date with latest block with a state proof. (Tycho Indexer will provide the feature in the future.)
# References
# Risks
- Bugs in the token quoter can lead to losses if used as a trade or risk parameter – this warrants a high degree of caution in testing, and potentially auditing. As well as providing clear change logs and informing users about the assumptions the implementation makes.
- Faulty Tokens: Tokens with unexpected behaviour can produce strange prices that would not execute on-chain. If this happens you might have to restrict the set of allowed "in-between" tokens that can act as a bridge between numeraire and quote token.