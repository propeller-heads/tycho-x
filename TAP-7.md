# Funding Rate Strategy

# tldr;

A quickstart of a delta-neutral basis trading bot.

> [!NOTE]
> 
> **Active Bounty:** This Tycho Application Proposal has an active bounty on it. Be the first to build this app and win a $10,000 USD bounty.
> 
> To work on the bounty reach out to **@tanay_j on Telegram**. To learn more read the [docs on Tycho bounties](https://docs.propellerheads.xyz/tycho/how-to-contribute/bounties).

# Implementations

NA - there are no implementations yet of this Tycho Application.

# Motivation

**Create an Extensible Base**: Provide a free, locally-runnable bot that serves as a foundation for developers to build a range of customized delta-neutral strategies.

**Make Funding Rate Arbs Accessible**: Lower the barrier to entry to funding rate arbitrage with an open source easy-to-run example.

# Background

**Funding Rate Opportunity**: Venues like Hyperliquid often have annualized funding rates of ~40% on many assets.
    
**Collateral Management**: What makes funding rate arb cumbersome is to actively manage and balance positions, collateral and the spot hedge. To maximize returns requires tight margins, and precise and automated rebalancing to avoid liquidation or one-sided exposure. 

For example, let's say you deposit $125 in collateral and take a $200 short on PEPE at $1. You also hold $200 of spot PEPE. Great, you are delta neutral. However, if PEPE price goes up to $1.50, you are $300 short with $25 of collateral. You are 12x leveraged, likely at risk of liquidation if not already liquidated. However, you have made a $100 gain on your spot position. So you can sell $100 of your PEPE for USD, (with $200 left in your spot), move $100 for your perp collateral (back to $125 of collateral), and then size down your perp position from $300 to $200.

# Goal

The goal of this project is to build a well funding rate arbitrage bot between Hyerpliquid perps (short) and spot on-chain (long). Well documented, easy to run and customise (change assets/chain/collateral), and run locally. Including a UI to monitor the strategy in real time.

# Condition for completion
A bot that, given a perpetual contract ticker, its corresponding spot token address, and a target leverage, can establish a delta-neutral position by shorting the perp on Hyperliquid and going long the spot asset on a low-cost chain (e.g., Unichain) using Tycho; automatically rebalance and bridge assets to maintain its leverage target; and report its status and collected yield on a local dashboard.

# Specification

## Definitions

- **Funding Rate Arbitrage**: A specific delta-neutral strategy of shorting a perpetual contract while holding its underlying spot asset. The goal is to profit from the price difference (the 'basis') by collecting the resulting funding rate payments.
- **Basis**: The price difference between a perpetual futures contract and its underlying spot asset. 
- **Funding Rate**: A periodic payment made between long and short holders of perps to enforce price convergence between the perpetual and spot markets. When the funding rate is positive, the longs pay the shorts.    
- **Delta Neutral**: A state where the portfolio's value is hedged against changes in the underlying asset's price. In this case, achieved by balancing a long spot position with an equally valued short perpetuals position.   
- **Rebalancing**: The action of buying or selling the spot asset to adjust the hedge and return the perpetual position's leverage to its target.
- **Target Leverage**: A user-defined ratio of the perpetual position's notional value to its collateral. It determines the strategy's risk level and serves as the primary trigger for rebalancing actions.

## User Scenarios
- **Yield Farmers/Traders** who want to run this strategy autonomously but need a starting point to do so.
- **Developers** learning how to build bots that interact with both a perpetuals exchange like Hyperliquid and use Tycho for spot execution.
- **Protocol Teams** who may want to adapt the logic to build treasury management tools or peg-keeping modules for their own assets.

## Requirements

### Essential Requirements

- **Configuration**: The user can specify perp ticker, spot token address, and target leverage.
- **Perpetual Exchange Integration**: The strategy manages a perpetuals position on **Hyperliquid**.
- **Spot Exchange Integration**: The strategy executes spot swaps on a low-cost chain (e.g., **Unichain**) using **Tycho**.
- **Rebalancing Logic**: The bot automatically triggers a rebalancing trade when the current leverage deviates from the target leverage by a defined buffer. This check must run on a defined schedule (e.g., every minute).
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

- **Cost Analysis**: Display a graph in the UI that maps the gas costs of recent rebalancing events.
- **State Persistence**: Implement a more robust state recovery mechanism for the dashboard than a local text file.

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
    - Compares current leverage to the user-defined target leverage.
    - If the deviation exceeds a buffer, it calculates the required rebalancing trade.
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

- **Basis Risk**: If the perpetual price falls below the spot price, the basis becomes negative and funding rates can flip. You risk paying fees instead of collecting them.  
- **Liquidation Risk**: Extreme market volatility, API failures, or network congestion could prevent a timely rebalance, leading to the loss of collateral in the perpetuals position.   
- **Execution Risk**: The strategy relies on third-party exchanges and bridges. API downtime, high slippage, or bridge failures can compromise the strategy. The user is fully responsible for all funds.
