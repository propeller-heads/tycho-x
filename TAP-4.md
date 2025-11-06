# Atomic Arbitrage
![Header](./assets/atomic-arbitrage.png)
# tldr;
A atomic arbitrage searcher quickstart.

> [!NOTE]  
> **Active Bounty**: This Tycho Application Proposal has an active bounty on it. Be the first to build this app and win a **$10,000 USD bounty**.
> 
> To work on the bounty reach out to **@tanay_j on Telegram**. To learn more read the [docs on Tycho bounties](https://docs.propellerheads.xyz/tycho/how-to-contribute/bounties).

# Implementations
- [**Triangle**](https://github.com/nonechuckX/tycho-arbitrage) – Atomic arbitrage bot
- [**Tycho Searcher**](https://github.com/jtapolcai/tycho-searcher) – Demo of a tycho searcher algorithm

# Motivation
**A good starting point for searching**: Make it easier to get started as a searcher – by showing a complete implementation of a searcher – that might miss some optimisations, but has all the necessary parts for executing atomic arbitrage.

**A base for further showcases**: This project can serve as a base for more specific showcases, such as a liquidation bot, or long-tail searcher.
# Background

**Dapps need searchers**: Searchers are the first and most important line of defense for the stability and functioning of many defi protocols. [More on how searchers are essential for dapp stability](https://www.propellerheads.xyz/blog/how-to-get-arbitrageurs-to-stabilize-your-protocol).

**Dapps are running their own searchers**: Many "bots" are now not run by independent third party teams, but by the dapp teams themselves, in order to guarantee the safety of their protocol. It insures them against the chance that there are no searchers searching on their protocol (yet).

**It's hard to get into searching**: Even if you have a strategy or an insight, and you want to go into competitive searching, it's hard to do so – the field is shrouded in misinformation (psyops) to distract newcomers.
# Goal
The goal of this project is to write a 
- **Showcase**: A minimal atomic arbitrage bot, that anyone can run and extend.
- **Docs**: Docs that onboard teams to the basics they need to understand to run a searcher or a keeper – and points the way to advanced topics.
# Condition for completion
A searcher that successfully finds all atomic arbitrage opportunities up to 3-hops on all chains that Tycho supports for a given token (and a max marketgraph of 1.000 pools).
# Specification
## Definitions
- **Marginal Price**: The marginal price of a pool is the incremental price you pay at a particular point on the swap curve (demand curve) of an AMM. It's the first derivative of the swap curve.
- **Spot price**: The spot price is the marginal price at the current state of the DEX (i.e. at swap amount = 0).
- **Cycle**: A cycle trade, or triangular arbitrage, is a trade that starts and ends in the same token. For example USDC -> WETH -> DAI -> USDC.
- **Candidate Cycle**: A cycle is a candidate for arbitrage if the product of all its spot prices is larger than 1. E.g. (USDC, WETH, 2000), (WETH, DAI, 1/1999), (DAI, USDC, 1) -> product is 2000 / 1999 > 1, so USDC -> WETH -> DAI -> USDC on these pools is a candidate cycle.
- **Profit**: The profit of an atomic arbitrage is the surplus in tokens minus the gas cost for the trade.
## Requirements
### Essential Requirements
- **Pre-calculate cycles**: Build a cycle file of the most relevant arbitrage cycles for a given set of start tokens.
	- **Market Graph**: Turn all pools and tokens into a graph, with pools as edges and nodes as tokens.
	- **Graph search for cycles:** Given a set of start tokens, enumerate all possible cycles of up to length N through the graph, from one of the start tokens. A cycle is a trade path that starts and ends in the same token. Use each pool at max once in a cycle.
		- **Limiting the search (Optional)**: Find good heuristics to limit the number of possible cycles in case there are too many. E.g. limit the set of possible bridge tokens to the most liquid N tokens.
- **Find optimal trades**: Given a cycle, find the trade amount that optimises the amount out (e.g. with binary search over the range (0, pool trade limit)). 
- **Check Cycles every block**:  For all cycles that have a spot price better than 1 (i.e. all cycles where the spot price indicates that you get more than 1 token out for token in at the margin) – calculate the optimal trade.
- **Profitable net gas**: Calculate gas in terms of out-token and consider a trade only as profitable if it is profitable net gas.
- **Profitability threshold**: Only execute trades that are profitable by at least X % (in BPS). (useful also for testing, you set it to slightly negative to find and test trade execution.)
- **Execute**: Execute the most profitable trade you found.
### Important Requirements
- **CLI Dashboard**: Implement a command line UI (or other UI component) to follow the searchers progress. Track at least: Number of cycles we monitor, current block, real-time counter for checked cycles this block, list of found arbitrage opportunities in current block, list of pending trades, list of succeeded arbitrage trades, profit in current run, user settings (start tokens, slippage setting (in BPS), bribe % (in BPS), chain, tracked protocols ("Uniswap v4, Balancer v2, etc."))
- **Recheck cycles only when a pool updates**: Calculate the cycle spot prices and optimal trade amounts once at the beginning. Then only recalculate a cycle spot price and optimal trade amount if one of the pools in the cycle had an update in the last block update.
- **Add slippage**: Add a slippage parameter (in basis points): Reduce the expected amount from both trades by the slippage when encoding the swaps. Only send trades that are also profitable *after* slippage.
- **Monitor trade execution**: Record and monitor pending trades. Block the pools involved in a trade from further trading until the previous trade either succeeded or failed. Record trade outcome (failed/succeeded, sell amount, buy amount, expected buy amount, gas cost, expected gas cost, token in, token out, profit net gas in out token).
- **Execution Options**: Give the user the option to pick one of several default execution options: Public mempool, [BuilderNet](https://buildernet.org/docs/api) through [Flashbots Protect](https://docs.flashbots.net/flashbots-protect/overview) via TEE builder, [MEVBlocker](https://cow.fi/mev-blocker). Pick a protected option by default.
- **Dynamic Bribe**: Bid a % of expected profit in gas (on chains where it's applicable).
- **Gas Safeguard**: Limit the amount of gas the searcher is allowed to use per e.g. hour – so that in case it bugs and sends non-profitable transactions you don't burn through your gas all at once.
### Nice-to-have requirements
- **Target Block**: Make trades only valid for a particular target block – so that you can consider trades that don't settle in the next block as failed.
- **Gas warning**: Notify when you're running out of gas.
### NOT included
- **Inventory Management**: Sell tokens automatically for gas to refill gas. Sell tokens automatically to treasure token (e.g. ETH or USDC).
### Advanced Topics
Include these topics in the docs, and guide how readers could add them to the current implementation:
- **Soft simulate**: Instead of simulating every amount out – interpolate the simulation with simpler functions. And then only simulate when you found a profit.
- **Immediately send**: Once you see a potentially profitable opportunity, immediately send it, do not wait for other cycle calculations to finish.
- **Heuristics to prioritise search**: How to further optimise the ordering in the cycle file by cycles that are most likely to have an opportunity (e.g. weight cycles by how often they created an opportunity in the last 7 days).
- **Analytical solutions v.s. Numerical**: Replace numerical solving (e.g. binary search over amount out) with analytical solvers – e.g. writing closed form solutions to the optimal trade amount given a reference price and a CFMM function. (we do NOT need to show how to do this, but maybe link to papers that do it for e.g. Univ2).
# Rationale
- **Why cycle files**: Prices (price of pools) change frequently, while the market structure (graph of pools) changes less frequently. So, given how computationally intensive the search for all arbitrage cycles is, it makes sense to asynchronously pre-cache the possible arbitrage paths – because they change infrequently.
	- **Gap in cycle files**: This also means that whenever there is a new pool – theoretically you should recalculate the cycle file. But this might still lead to too frequent re-calculations – so you might need to fill these gaps with a more targeted search (e.g. checking for up to 3-hop cycles starting with this pool only) until you trigger the next update of the whole cycle file.
- **Why optimise trade amounts and then account for gas**: Taking it more strictly, you should account for gas *during* the search for the optimal trade amount – as gas can also change with trade amount (e.g. when you cross a tick on Uniswap v3). However, this slows down the calculation significantly, and more importantly will make the objective non-convex, which makes the search for the optimal amount much harder.
# Considerations
- **Native v.s. VM implementations**: Native implementations simulate orders of magnitudes faster than VM implementations. So out of the [available protocols](https://docs.propellerheads.xyz/tycho/for-solvers/supported-protocols) you might need to limit yourself to the native implementations, or run cycles with VM protocols in separate threads so they don't block you from finding arbitrage on the other protocols.
# References
# Risks
- **Loss of funds**: It is fully in the user's responsibility to implement and run their searcher safely. All tokens held in the EOA are at risk. Risks include: Accidentally sending non-profitable trades and burning gas, and leaving trade profits in the router. To limit risk always transfer profits out of the trading hot wallet regularly, and limit the amount of gas in the wallet at anytime (and don't auto refill gas).
