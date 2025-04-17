# tldr;
A bot to find and execute profitable pairs of mirrored trades over two chains.
# Motivation
- **Starting point to enter into Cross-Chain Arbitrage**: Give a practical starting point for cross-chain arbitrage – and guide readers on how they can improve the bot further.
- **Provide a backup searcher**: When there are no strong searchers yet, many DEX and chain teams need a backup searcher that pegs prices.
- **Couple Defi Markets**: Cross-chain arbitrage bots link liquidity across chains –which improves depth an price accuracy on every chain. This makes it more viable to launch specialised chains and still effectively maintain access to the liquidity of all other chains (and CEXs). 
# User Stories
- **New Chains**: New chains want DEX prices to be inline with the wider market. This gives traders & LPs confidence and avoids losses (to LVR arbitrage). They can run this bot to peg prices before other searchers arrive.
- **Arbitrage with liquidity showcase**: Many teams have liquidity (market makers, relayers, and decentralised vaults), but few know how to build a searcher. This bot serves them as a starting point to bring their strength (liquidity management) and market make across-chains. 
# Background
- **New chains are at high risk of wrong prices**: When chains and DEXs launch it usually takes a while until searchers integrate them – and during that period prices are volatile. Wrong prices hurt confidence in the chain, and LPs suffer much higher losses to LVR.
- **Cross-Chain arbitrage is not obvious**: There is little shared knowledge on the actual mechanics of cross-chain arbitrage, and there are no open source repositories to get started. This increases the barrier to entry for cross-chain arbitrage unnecessarily.
# Goal
The goal is to find pairs of pools on two chains that differ in price, calculate optimal swap amounts, and if profitable, execute them.
# Condition for completion
Monitor all pools for a particular token pair on both chains (with Tycho), find all net-gas profitable arbitrage trades and execute them in the next block. Bounded by your token inventory on each chain. Monitor and log the executions (and failures).
# Specification
## Definitions
## Requirements
### Essential Requirements
- **Settings**: User defined settings.
	- **Set token pairs**: User can set token pairs they want to arbitrage. (include defaults).
	- **EOA**: User sets an EOA and private key that holds the assets to be used for arbitrage trades (same EOA on both chains).
	- **Chains**: Configure the chains across which to arbitrage.
- **Find 1-hop arbitrage trade pairs**: Find all trade pairs (one on each chain) with positive surplus (net gas cost).
- **Monitor inventory**: Monitor token balances in the EAO (inventory). Limit trades amounts to current inventory.
- **Execute on next block**: Execute every profitable pair of trades on the very next block of their respective chains.
- **Monitor trade execution**: Block the pools involved in an active arbitrage trade from further trading until both trades either succeeded or failed. Then unblock those pools and add them again to the list of pools to consider for arbitrage.
### Important Requirements
- **Only check if you need to**: Only re-check pairs for arbitrage where at least one of the pools had a change (pool delta event) in the last block. This will drastically reduce your search space.
- **Txs target only the next block**: Target trades to a specific block. E.g. implement a block check in the Tycho router contract at the beginning of the trade (which if it fails reverts the trade) – and encode the target block in the transaction.
- **Add slippage**: Add a slippage parameter (in basis points): Reduce the expected amount from both trades by the slippage when encoding the swaps. Only send trades that are also profitable *after* slippage.
- **Add risk parameter**: Add a risk parameter (in basis points). Account for settlement risk (that only one trade settles) by discounting the expected amount out by a fixed percentage. Only settle trades that are profitable *after* slippage and risk discount. (but *don't* discount the expected amount out for the encoding and execution of the swaps by the risk discount.)
- **Target Block**: Make trades only valid for a particular target block – so that you can consider trades that don't settle in the next block as failed.
### Nice-to-have Requirements
- **Consider multi-hop paths**: Build a graph (tokens as nodes, pools as edges), and find all N-hop (e.g. 2-hop) paths from A to B (e.g. A->C->B). Add these paths to the list of candidates for arbitrage trades, in addition to the single pool options.
- **Profit Driven Gas Bribes**: Add a percentage (in basis points) of the expected profit as an execution bribe (e.g. as tip to builders on mainnet).
- **Private Execution**: Instead of sending to the public mempool (where competing bots can observe and outbid your trade) – send the trade via a private execution route (e.g. BuilderNet, or MEVBlocker on mainnet).
- **Execution Strategies**: Implement and test different execution strategy options: 
	- **Block Time Aware Execution**: Don't send if you expect at least one more block update from the faster chain before the deadline until you need to submit your transaction on the slower blockchain. Instead wait until the last update of the faster blockchain, and only then allow the sending of arbitrage trades.
	- **Sequential Execution – Slow then fast**: Another strategy is to first execute on the slow chain, and only after successful trade on the slow chain, execute the second leg on the fast chain.
## Not included
Things, that, for now, we expect users to do manually or implement it themselves.
- **Inventory management**: If the inventory becomes imbalanced, rebalance it by bridging assets between the chains.
- **Automated gas refill**: Refill gas by selling some inventory for gas tokens.
# Implementation
*This is mostly a recommendation – implement as you think best.*
Split into three stages. After every new block you receive (from either chain): 
- **Update State**: Updating you knowledge of your state (DEX prices, inventory, and settled trades).
- **Identify Arbitrage**: Find the list of possible arbitrage trades.
- **Execute**: Execute the most profitable trades
## Update State
- **Get updated DEX state**: Get all DEX updates from the latest block (Tycho does this for you automatically).
- **Monitor Inventory**: Update token balances of the wallet on both chains.
- **Monitor Trades**: Note which trades that you previously sent settled, and whether they succeeded or failed (reverted).
- **Monitor Spot Prices**: Record spotprices (net trading fee) on all pools (Tycho Simulation does this for you).
## Identify Arbitrage
To identify profitable arbitrage opportunities: First filter for pools that have crossing spot prices, then optimise trade amounts, and then deduct gas costs – all pairs that still have a positive trade surplus are candidates for arbitrage.
- **Identify Pools with Crossing Spot Price**: Identify all pool pairs (one pool on each chain) that trade the same pair of assets (e.g. `(USDC,WETH)`), where, in the margin, you can sell higher on one than you can buy on the other. (i.e. all pools where net fee spot prices "cross").
- **Calculate optimal trade amount**: For every crossing pair, the amount you need to trade to achieve the best price (net gas).
	- **Buy and then sell the amount you bought on the other pool**:
		- **Buy**: Sell amount `N_0` of asset `X` on `pool_0` on `chain_0`, record the buy amount `M` of asset `Y`.
		- **Sell**: Sell `M` of asset `Y` on `pool_1` on `chain_1`, and record the buy amount `N_1`.
		- **Calculate surplus**: The difference between the amount you sold and the amount you got after completing the "loop" is your trade surplus. `trade_surplus = N_1 - N_0`.
	- **Numerical Optimisation**: Iterate over amounts `N_0` to find the amount that maximises `trade_surplus`. (e.g. using binary search, starting with the bound 0 and what you can trade (`min(inventory, pool.max_trade_amount)`).
- **Deduct Gas Costs**: For all trade pairs with a positive trade surplus: Calculate the gas cost of both trades:
	- **Convert the gas cost** (in ETH) into asset X – using the exchange rate of the arbitrage trades itself. 
	- **Deduct gas**: Deduct the gas cost in asset X from the trade surplus: `profit = trade_surplus - gas`
- **List profitable trades**: All trade pairs that are net gas profitable (i.e. that have a positive `profit`) are candidates to settle. 
## Execute trades
- **Execute trades in order of profitability and drop conflicting trades**: 
	- Sort trades by surplus, execute from the top down
	- Block pools involved in a trade from further trades, discard any trade that uses a blocked pool
	- Until you reach the end of the list.
- **Block pools part of active trades**: Block all pools involved in active arbitrage trades until both pools either settle or fail. If a trade doesn't settle in the target block, consider it failed (consider single block as final). 
# Rationale
- **Account for risk proportional to trade amount**: The larger your trade, the more you are exposed to the risk of adverse selection (only one of your trades settles and at a price worse than global market price). Which is why we suggest to add a risk parameter that automatically scales your risk buffer with volume (i.e. discounting for risk by a percentage of trade volume).
- **Avoid stale trades with single-block expiry**: By default, trades don't have an expiry, so they stay in the mempool, and can still settle in the future. The more time passes, the more risk we have of one-sided settlements – so we ideally want trades to either settle in the next block or expire. This doesn't eliminate the problem entirely, since on many chains you can only enforce this onchain (e.g. by checking for the target block in your contract) – and it still costs gas to settle this reverted trade.
# Considerations
- **Improve asset prices**: For convenience we use the exchange rate of the pool we arbitrage on to convert gas into out token denomination. Since your gas calculation doesn't have to be so exact, this is fine. However, there could possibly be outliers where the arbitrage is so significant (or the pool so faulty) that this leads to a wrong conclusion.
- **Multi-hop trades**: We only search for exactly matching pairs. However, you might need more than one hop to mirror a trade on another chain. E.g. on one chain you have pool (A,C) and on another pools (A,B) and (B,C). To arbitrage between these two chains, you need to consider two-hop trades like A -> B -> C.
- **Nonce issues**: Stale trades in the mempool can cause issues with the nonces you set for new trades.
# Risks
- **Risk of one sided settlement**: The pair of trades execute independently. So one could settle, while the other does not (e.g. due to slippage or re-org). This leaves the bot with an unbalanced exposure to one asset. IF that trades settlement price was worse than the global market price at the time (i.e. it has a bad markout) then such a trade causes us a loss.
	- **Lots of failures in short time and at high volume**: It is likely that settlements fail more frequently when markets are volatile. Which is exactly when arbitrage opportunities are more likely, and the resulting trades are larger. So failures likely occur in sudden bursts.
	- **Adverse selection in failed trades**: If markets move fast (e.g. price jumps 1% on Binance) and other searchers (e.g. CEX-DEX searches) compete to arb the pools you want to arb – then you can suffer adverse selection: On the direction (and chain) where you would have gotten a good price your trade fails (because someone else outbid us on trading on the pool), and on the other direction (where we are buying for a price thats now above market) your trade goes through.