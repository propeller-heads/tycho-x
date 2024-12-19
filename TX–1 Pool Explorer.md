# tldr;
A local UI to explore DEX pools. Low-latency, uncompromising pool coverage and data availability.
# Implementations
NA - there are no implementations yet of this Tycho Extension.

# Motivation
- **Speed, precision and breadth**: Today, there are only 3rd party web UIs to explore available DEX pools. However, they are slow (network latency), limiting (do not share the entire dataset), and incomplete (do not list all tokens, all pools, or only with delay after launch). – Instead lets build an uncompromising UI on Tycho: Verifiable data reliability, complete data (no pool or token is missing), and very low latency (local, efficiently implemented). So you can find any pool, filter fast, and explore the entire set of DEX pools visually.
- **Trustless source on DEX liquidity for better decisions**: A trustless, robust, explorer to read and know the on-chain liquidity in real-time is valuable for decision: Such as competitive analysis (DEX / token), trading, yield optimisation, calculating collateral factors, and market reports.
- **On-ramp to DIY use-cases**: A good UI helps understand the data – and can inspire users to build implement their own tools against it – such as automated LP, producing liquidation risk reports, trading, solving or many other.
- **Open Information about DEX liquidity**:  The block data on DEX liquidity is theoretically open to anyone. But its very hard to parse, especially across different DEXs. Today no-one but a few analysis firms, solvers and routers have an accurate, complete and real-time the on-chain market. This UI will open a complete view to anyone who's interested.
# Goal
Make on-chain liquidity easy to observe and explore, directly, through a local, filterable pool list and pool graph.
# Specification
## Requirements
### Essential Requirements
- **Pool list view**: Columns: Token1, Token2, TVL (in USDC), Protocol, Last Tx (last update) block number, Last update time
	- **Filter option**: Let the user sort and filter token, protocol and TVL.
-  **Graph View**: Show pools (of current filter view) in graph view (nodes are tokens, edges are pools).
### Important Requirements
- **Graph View**
	- **Graph View Detail**: Click on a node, see total TVL for the token, number of pools, top 5 pools sorted by TVL with TVL, protocol name, last trade timestamp.
	- **Scale nodes by TVL**: Scale node volume by TVL (or by log TVL).
- **Overall Metrics**: 
	- **Total number of pools indexed**
	- **Total number of contracts / protocols indexed**
	- **Total TVL indexed**
- **Filter view Metrics:**
	- **Total number of pools in current filter view**
	- **Total TVL in current filter view**
### Nice-to-have requirements
- **Path finder**: Given an in-token and an out-token find all paths of length 2-4 through the graph and highlight those paths.
	- **Most liquid**: Highlight the path that goes through the sequence of most liquid pools (the path that has the most liquid lowest-liquidity leg).
### Not included
## Definitions
- **TVL**: Total value locked in a liquidity pool. The sum total of all tokens, denominated in a common numeraire (commonly USDC or ETH).
- **Protocol**: The set of smart contracts comprising a protocol which also contain one or multiple liquidity pools. (Can be a DEX, but also e.g. a lending protocol).
# Implementation
*A proposal for the implementation.*
## Tools
- **UI Data Analysis Framework**: Consider a UI framework that you can easily host locally: Like [Streamlit](https://streamlit.io/).
## Design
*A proposal! for the design.*
```
// Core Types
type Token = {
  address: Address
  symbol: string
  decimals: number
}

type Protocol = {
  name: string
}

type Pool = {
  token1: Token
  token2: Token
  tvl: USDCAmount
  protocol: Protocol
  lastBlockUpdate: BlockNumber
  lastUpdateTime: Timestamp
}

type Metrics = {
  totalPools: number
  totalProtocols: number
  totalTvl: USDCAmount
  filteredPools?: number
  filteredTvl?: USDCAmount
}

// Filter & Sort Types
type SortField = 'tvl' | 'lastUpdateTime' | 'protocol'
type SortDirection = 'asc' | 'desc'

type FilterCriteria = {
  tokens: Set<Token>
  protocols: Set<Protocol>
  tvlRange: {
    min: USDCAmount
    max: USDCAmount
  }
  sortBy: SortField
  sortDirection: SortDirection
}

// Graph Types
type TokenNode = {
  token: Token
  totalTvl: USDCAmount
  poolCount: number
  topPools: Pool[]  // Limited to top 5 by TVL
}

type PoolEdge = {
  pool: Pool
  source: Token
  target: Token
}

type MarketGraph = {
  nodes: Set<TokenNode>
  edges: Set<PoolEdge>
}

type DirectedPath = {
  startToken: Token
  endToken: Token
  pools: Pool[]
  minLiquidity: USDCAmount
  totalTvl: USDCAmount
  direction: 'forward' | 'reverse'
}

// UI State Types
type ViewType = 'list' | 'graph' | 'pathfinder'

type AppState = {
  pools: Set<Pool>
  currentFilters: FilterCriteria
  selectedView: ViewType
  graphState: {
    selectedNode?: Token
    nodeDetails?: TokenNode
    layout: ForceDirectedLayout
    scaling: TVLScaling
  }
  pathFinderState: {
    inputToken?: Token
    outputToken?: Token
    paths: Set<DirectedPath>
    mostLiquidPath?: DirectedPath
    pathLengthRange: {
      min: number  // 2
      max: number  // 4
    }
  }
}

// Operation Types
type Operations = {
  filterPools: (pools: Set<Pool>, criteria: FilterCriteria) => Set<Pool>
  getMetrics: (pools: Set<Pool>) => Metrics
  sortPools: (pools: Set<Pool>, field: SortField, direction: SortDirection) => Pool[]
  
  poolsToGraph: (pools: Set<Pool>) => MarketGraph
  getNodeDetails: (graph: MarketGraph, token: Token) => TokenNode
  findPaths: (
    graph: MarketGraph, 
    fromToken: Token, 
    toToken: Token, 
    lengthRange: {min: number, max: number}
  ) => Set<DirectedPath>
  findMostLiquidPath: (paths: Set<DirectedPath>) => DirectedPath | undefined
}

// View Components
type PoolListView = {
  pools: Pool[]
  filters: FilterCriteria
  metrics: Metrics
  sortState: [SortField, SortDirection]
  onFilterChange: (filters: FilterCriteria) => void
  onSortChange: (field: SortField) => void
}

type GraphView = {
  graph: MarketGraph
  selectedNode?: Token
  nodeDetails?: TokenNode
  layout: ForceDirectedLayout
  scaling: TVLScaling
  onNodeSelect: (token: Token) => void
}

type PathExplorerView = {
  inputToken?: Token
  outputToken?: Token
  paths: Set<DirectedPath>
  mostLiquidPath?: DirectedPath
  pathLengthRange: {min: number, max: number}
  onTokenSelect: (position: 'input' | 'output', token: Token) => void
  onPathLengthChange: (range: {min: number, max: number}) => void
}
```
# Rationale
- **Criticial use-cases don't want to rely on third party APIs**: Users who make financial decisions based on this data – do not want to rely on third party data.
- **Many users have higher throughput and lower latency requirements**: Many users need, or at least significantly benefit, from being able to do much faster analyses, on more complete and larger scale pool data.
- **UIs onramp programmatic users**: The UI makes it easier to imagine ways to use the data, and to validate the breadth and accuracy of the data. So a UI can also onboard developers who will use the underlying data programatically.
# References
# Risks
- **Meaningful difference to DEX explorers**: Web-based, third-party, DEX explorers already exist. Data is not verifiable, complete or openly accessible – but users might not value lower latency, more reliability, accuracy and larger breadth enough.
