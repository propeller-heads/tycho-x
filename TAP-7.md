# Funding Rate Arbitrage
<img width="1494" height="444" alt="image" src="https://github.com/user-attachments/assets/5cff7017-3b3d-46e4-a225-3a0dda50e93c" />

# tldr;

A quickstart for a delta-neutral funding rate arbitrage bot.

> [!NOTE]
> 
> **WIP Bounty:** This Tycho Application Proposal is currently being built by a bounty hunter. Succesful completion will receive a $10,000 USD bounty.
> 
> To work on the bounty reach out to **@tanay_j on Telegram**. To learn more read the [docs on Tycho bounties](https://docs.propellerheads.xyz/tycho/how-to-contribute/bounties).

# Implementations
- [**Tycho Hedge**](https://github.com/sr33j/tycho-hedge) – [fundingrate.wtf](https://fundingrate.wtf/)

# Motivation

**Create an Extensible Base**: Provide a free, locally-runnable bot that serves as a foundation for developers to build a range of customized delta-neutral strategies.

**Make Funding Rate Arbs Accessible**: Lower the barrier to entry to funding rate arbitrage with an open source easy-to-run example.

# Background

**Funding Rate Opportunity**: Venues like Hyperliquid often have annualized funding rates of ~40% on many assets.
    
**Collateral Management**: Funding rate arbitrage is cumbersome due to the need to actively manage positions, collateral, and the spot hedge. Maximizing returns requires tight margins, and precise and automated rebalancing to avoid liquidation or one-sided exposure. 

For example, deposit $125 in collateral and take a $200 short on PEPE at $1 (200 tokens short). Also hold $200 of spot PEPE (200 tokens long). This is delta-neutral at ~1.6x target leverage. If PEPE price rises to $1.50:
- Spot: 200 tokens now worth $300 (gain +$100).
- Perp: Short 200 tokens, notional $300; unrealized loss -$100; collateral drops to $25.
- Leverage is now 12x ($300 notional / $25 collateral), significantly higher than the ~1.6x target and risking liquidation.
To rebalance: Sell ~66.67 tokens of spot for $100 USDC (new spot: 133.33 tokens worth $200). Bridge $100 to Hyperliquid, adding to collateral (now $125). Reduce perp short by ~66.67 tokens (new short: 133.33 tokens, notional $200). Result: Exact ~1.6x leverage restored, delta-neutral.

# Goal

The goal of this project is to build a funding rate arbitrage bot between Hyperliquid perps (short) and spot on-chain (long). Well documented, easy to run and customise (change assets/chain/collateral), and run locally. Including a UI to monitor the strategy in real time.

# Condition for completion
A bot that, given a perpetual contract ticker, its corresponding spot token address, and a target leverage, can establish a delta-neutral position by shorting the perp on Hyperliquid and going long the spot asset on a low-cost chain (e.g., Unichain) using Tycho; automatically rebalance and bridge assets to maintain its leverage target; and report its status and collected yield on a local dashboard.

# Specification

## User Scenarios
- **Yield Farmers/Traders** who want to run this strategy autonomously but need a starting point to do so.
- **Developers** learning how to build bots that interact with both a perpetuals exchange like Hyperliquid and use Tycho for spot execution.
- **Protocol Teams** who may want to adapt the logic to build treasury management tools for their own assets.

## Definitions

- **Funding Rate**: A periodic payment made between long and short holders of perps to enforce price convergence between the perpetual and spot markets. When the funding rate is positive, the longs pay the shorts.    
- **Delta Neutral**: A state where the portfolio's value is hedged against changes in the underlying asset's price. In this case, achieved by balancing a long spot position with an equally valued short perpetuals position (matching notional values at current price).   
- **Rebalancing**: The action of buying or selling the spot asset to adjust the hedge and return the perpetual position's leverage to its target and keeping it delta neutral.
- **Target Leverage**: A user-defined ratio of the perpetual position's notional value to its collateral. It determines the strategy's risk level and serves as the primary trigger for rebalancing actions.

## Requirements

### Essential Requirements

- **Configuration**: The user can specify perp ticker, spot token address, and target leverage.
- **Perpetual Exchange Integration**: Manage a perpetuals position on **Hyperliquid**.
- **Spot Exchange Integration**: The strategy executes spot swaps on a low-cost chain (e.g., **Unichain**) using **Tycho**.
- **Rebalancing Logic**: The bot automatically triggers a rebalancing trade when the current leverage deviates from the target leverage by a defined buffer or when spot value and perp notional deviate by a defined delta buffer. This check must run on a defined schedule (e.g., every minute).
- **Local Dashboard**: The local UI displays:
    - Total PnL ($ and % return).
    - Charts for AUM and annualized funding rate.
    - A table of current spot and perp positions, collateral, and leverage.
    - Tables for historical trades, deposits, and withdrawals.

### Important Requirements

- **Automated Bridging**: Bridge collateral to the target chain as part of its rebalancing and collateral management flow.
- **Negative Funding Protection**: Unwind the position if the funding rate is no longer profitable (e.g., when the 30-day average minus one standard deviation is negative).
- **Liquidation Recovery**: Handle an unexpected liquidation (e.g., sell remaining spot, resupply collateral, re-establish the position).
- **Transaction Resiliency**: Retry logic for failed spot transactions.


### Nice-to-have Requirements

- **Cost Analysis**: Display a graph in the UI that shows the costs of recent rebalancing trades.
- **State Recovery**: Implement a more robust state recovery mechanism (like a local SQL database) for the dashboard than a local text file. Text files can be corrupted and a better mechanism can ensure reliable continuity in case of restarts  

# Implementation

The bot operates in a continuous loop with the following logical flow:

1. **Configuration**: On startup, the bot reads user settings (ticker, leverage targets, API keys).
    
2. **Initialization**:
    - Connects to Hyperliquid and Tycho.
    - Fetches current state: wallet balances, open perpetual positions, and spot asset holdings.  
    - Determines if it needs to establish a new position or use an existing one.
        
3. **Monitoring Loop** (runs on a fixed schedule, e.g., every minute):
    - Fetches the current mark price of the perpetual contract. 
    - Calculates the current notional value of the perp and spot positions. 
    - Calculates the current leverage of the perp position.
        
4. **Rebalancing Logic**:
    - Compares current leverage to the user-defined target leverage. It should also check the difference between the spot postion's value and the perp's notional value.
    - If the deviation in either exceeds a buffer, it calculates the required rebalancing trade.
    - Example: If price increases, spot holdings gain value. The bot calculates how much spot asset to sell for USDC.
        
5. **Execution**:
    - The spot trade (e.g., selling PEPE for USDC) executes with Tycho.  
    - Upon successful execution, we bridge the bought USDC (if necessary) and transfer it to Hyperliquid account and add it as collateral, reducing leverage back to the target.
        
6. **Risk Management**:
    - The bot periodically checks the historical funding rate. If it meets the negative funding criteria, it unwinds the spot and perpetual positions.
    - Log all actions, trades, and errors are to a file and display on the dashboard.
   
# Future Work

- **Multi-Asset & Multi-Exchange Strategy**: Monitor multiple tickers and exchanges. 
- **LP Hedging Strategy**: Adapt the logic to hedge the impermanent loss of a concentrated liquidity LP position.   
- **Perp DEX & Tycho Integration**: Explore how perpetual exchanges could natively integrate Tycho to offer managed hedging products.   

# Risks

- **Negative funding rates**: If the perpetual price falls below the spot price, the basis becomes negative and funding rates can flip. You risk paying fees instead of collecting them.  
- **Liquidation Risk**: Extreme market volatility, API failures, or network congestion could prevent a timely rebalance, leading to the loss of collateral in the perpetuals position.   
- **Execution Risk**: The strategy relies on third-party exchanges and bridges. API downtime, high slippage, or bridge failures can compromise the strategy. The user is fully responsible for all funds.
