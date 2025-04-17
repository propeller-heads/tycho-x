# tldr;
A onchain market making showcase, built on Tycho.
# Motivation
**Easy start for market makers**: Lower the barrier to entry for market makers – by implementing a minimal market maker on Tycho.

**Guide for how to market make onchain**: Guide users through all essential concepts they need to understand in market making onchain (even if some are not implemented in code) – especially from the perspective of a market maker used to centralised limit orderbook exchanges (CEXs).
# Background
**Market makers are essential**: Market makers make markets liquid and stable by adding liquidity and aligning prices between exchanges.

**DEX integrations are the largest hurdle to market makers**: Aspects specific to blockchains are a hurdle to market makers (e.g. bundling & bribing, nodes & RPCs, settlement periods, builder, validators, ...), but especially DEXs, as they are all different and lack a common interface. Tycho solves this.

**Data Market Makers care about**: To market make, you monitor price and depth, execute trades that adjust price, and move liquidity (maker orders) to change depth.
# Goal
The goal of this project is to build a minimal lovable market maker that showcases how to build a market maker with Tycho. Including:
- **Implementation**: A market maker that covers all essential features and that can serve as a starting point for professional market makers who want to market make onchain.
- **Docs**: Documentation on the core concepts, core concerns, and implementation of the market maker. As well as suggested ways to improve on the example.
# Condition for completion
A market maker that anyone can easily run, well documented and clean example implementation that helps the engineer understand the core concepts of Tycho as they are useful to market making, and complete documentation on how to solve the concerns of market making with Tycho and other tools onchain. 
# Specification
## Definitions
- **Market Maker**: A market maker uses inventory to place buy and sell orders on exchanges in order to reduce the spread, increase depth and stabilize prices.
- **Market Price**: The, theoretical, "correct" current market mid-price. In practice this could be a set target price (e.g. 1 DAI = 1 USDC), a price from a specific exchange (e.g. Binance WETH-USDC price) or a global weighted average price over many markets (e.g. Volume weighted average of different exchanges).
- **Profitable trade**: A trade is considered profitable (in our naive case) if you buy an asset, net all costs (trading fee, gas costs), below what you consider the market price.
- **Profit**: The profit of a trade is the value of what you bought minus the value of what you sold, minus all trading costs, at market price.
- **Marginal Price**: The marginal price of a pool is the incremental price you pay at a particular point on the swap curve (demand curve) of an AMM. It's the first derivative of the swap curve.
- **Spot price**: The spot price is the marginal price at the current state of the DEX (i.e. at swap amount = 0).
- **Optimal swap is to swap until marginal price = market price**: The optimal swap (regardless of whether it is profitable), without considering gas, and assuming monotonically increasing prices (which is true for all DEXs afaik) is always to buy until the marginal price is equal to the market price.
- **Swap to price**: Swap to price is the swap you need to make to move the marginal price of a pool to a particular price (e.g. the market price).
## Requirements
### Essential Requirements
- **Monitor inventory**: Monitor token balances in the EOA (inventory). Limit trades amounts to current inventory.
- **One token pair**: The showcase market maker watches all pools for one specific, set, token pair. (Pick a stable:stable pair, e.g. (USDC,DAI)).
- **Spread setting**: Let the user set a spread (in basis points - default is 0). This spread defines the market price for the buy (market price - spread) and sell side (market price + spread).
- **Hardcoded market price**: In this example the market price is hardcoded – and we trade on a pair with stable exchange rate. To trade on dynamic pairs users need to implement their own price provider (e.g. Binance feed).
	- **Mock Price Provider**: Mock a price provider so that users can easily replace the hard coded price with any price provider that they build.
- **Swap to price**: If the spot price of any pool deviates from market price – calculate the trade that moves the pool back to the market price. Limit the max swap amount to the current inventory.
- **Account for gas costs**: Convert the gas cost of the swap in out token denomination (e.g. using a price API) and subtract from the amount out to determine if a trade is profitable net gas. (Make it optional for the user to consider gas or not.)
- **Swap**: From all profitable swaps, make the ones that are the most profitable. Sort trades by profit, execute from the top down, block inventory and pools that are used by previously added trades, loop until the end. Bundle all swaps into one execution.
- **Python**: Implement the showcase in Python and use Tycho Simulation and Tycho Execution Python bindings.
### Important Requirements
- **Monitor trade execution**: Record and monitor pending trades. Block the pools involved in a trade from further trading until the previous trade either succeeded or failed. Record trade outcome (failed/succeeded, sell amount, buy amount, expected buy amount, gas cost, expected gas cost, token in, token out, market price).
- **Add slippage**: Add a slippage parameter (in basis points): Reduce the expected amount from both trades by the slippage when encoding the swaps. Only send trades that are also profitable *after* slippage.
- **Update aware calculations**: Only re-calculate swaps if the pool changed in the latest block update (otherwise use cached results).
- **Private Execution**: Instead of sending to the public mempool (where competing makers can observe and copy the trade) – send the trade via a private execution route (e.g. BuilderNet, or MEVBlocker on mainnet).
- **Monitor liquidity**: Let users define a set of sample depths (in one of the tokens the user market makes on in this example) and a set of percentage depths.
	- **Monitor price impact at depth**: Simulate the user defined depths on all pools of the user defined pair, and report the price impact (% difference in marginal price after and before the swaps – Tycho Simulation can gives you both for each pool).
	- **Report % depth**: Run a small binary search over trade amounts to find the trade amount needed to move the marginal price by a defined amount of BPS (%), report these amounts as percentage depths. 
- **Maintaining a stable connection**: Make the showcase robust against dropped websocket connections, and auto-reconnect in case the connection drops.
- **Simulation Speed Benchmark**: Let users run a simulation benchmark to check the speed of Tycho Simulation swaps and get-price on a set of pools the user cares about.
- **Target Block**: Make trades only valid for a particular target block – so that you can consider trades that don't settle in the next block as failed.
### Nice-to-have requirements
 - **Consider multi-hop paths**: Build a graph (tokens as nodes, pools as edges), and find all N-hop (e.g. 2-hop) paths from A to B (e.g. A->C->B). Add these paths to the list of candidate trades, in addition to the single pool options.
 - **Profit Driven Gas Bribes**: Add a percentage (in basis points) of the expected profit as an execution bribe (e.g. as tip to builders on mainnet).
### Essential Requirements for the Documentation
The documentation should document both the implemented and not implement concepts.
In this list here we define the *additional* topics that we have to include in the documentation, but NOT necessarily in the code:
- **Integrate external price feeds**: How to integrate external price feeds, for example the Binance price.
- **Overview of safe execution options**: An overview of the options how market makers can submit their trades for either / or both maximum speed and security (e.g. TEE Builder, Echo, Flashbots Protect etc.).
- **Auto topping up of gas**: Automatically buy some ETH when you run low on it.
- **Timing submission with block times**: Purposely wait with submission until a certain delta of the expected deadline builders accept trades for the next block, in order to listen to updates to the market price (if dynamic) to the last moment.
- **Bidding gas and rebates**: How to bid gas and how to get rebates from builders.
	- **Dynamic gas bids**: How to bid dynamically according to the surplus on the trade (if that's possible).
- **Orderbook**: Reference the Tycho Orderbook implementation and how they can use it to monitor all liquidity for a token pair in a single orderbook. -> https://github.com/propeller-heads/tycho-x/blob/main/TAP-2.md, https://www.orderbook.wtf/
# Implementation
## Strategy options
Options for strategies we can run, from simple to more interesting:
**Stable coin market maker**: Pick a stable coin pair that is supposed to have 1:1 price. Then arbitrage whenever its profitable from one into the other. 1-hop only.
Advanced: Redeem the coins when you have a disbalance.
# Rationale
- **Why make gas optional**: Depending on the agreement, it might be more important to the market maker to keep a pools price within a certain spread of the market price, than to make a profit on every trade (e.g. when they are compensated by protocol teams to do this). 
# Future Considerations
- **Rust implementation**: Some teams might prefer to work in Rust. Tycho is built in Rust, and we expect a Rust implementation to be more performant. However, we decided for this showcase to start with Python, as it makes the showcase accessible to more teams (and several market making teams specifically requested an example in Python.)
- **Liquidity Provisioning**: This example deliberately omits managing liquidity positions in AMMs (important for many market makers) – we could add this with future versions of Tycho.
- **RFQ**: This example also omits providing quotes into RFQ protocols, or hosting an endpoint for others to request quotes on – another frequent activity of market makers.
- **(Cross-Chain) Arbitrage**: CEX-DEX or cross-chain arbitrage are also omitted from this example because it has some specific and additional considerations. A separate Tycho Application Proposal covers a showcase for cross-chain arbitrage:
- **Price Provider**: Many market makers are also price providers. A separate Tycho Application Proposal covers a token quoter that builds token prices entirely from onchain pools: 
- **Intent Auctions**: Market makers also frequently participate in intent auctions. Auction templates for market makers will also be in separate examples.
- **Inventory management**: Managing and moving inventory between venues, and accounting for cost of inventory, is an important part of market making – but outside the scope of Tycho.
# References
# Risks
- **Loss of funds**: It is fully in the users responsibility to implement their market maker safely. There are many possibilities to lose funds, not limited to: Setting the wrong market price, executing many trades with high gas cost, and adverse selection of stale orders in the mempool.