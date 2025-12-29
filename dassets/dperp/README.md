# DPerp - Perpetual Trading

DPerp enables leveraged perpetual trading of synthetic assets using a CEX-style order book model with trader-to-trader matching. The system provides up to 50x leverage with cross-margin support and direct counterparty trading without liquidity pools.

## Overview

DPerp provides:
- Central limit order book (CLOB) trading like traditional exchanges
- Trader-to-trader counterparty matching (no LP pools)
- Oracle-anchored mark pricing for fair liquidations
- Cross-margin system for capital efficiency
- Price-deviation based funding rates
- Professional trading features and order types

## Architecture

```
User Interface (Trading API)
         ↓
    Order Book Engine (Off-chain)
         ↓
    Trade Matching & Settlement (On-chain)
         ↓
┌─────────────────────────────────────────┐
│  MarketRegistry │ PositionManager       │
│  OracleModule   │ MarginAccount         │
│  FundingEngine  │ OrderBook             │
└─────────────────────────────────────────┘
```

### Core Components

- **OrderBook**: CEX-style order matching with price-time priority
- **PositionManager**: Cross-margin position tracking and PnL
- **MarginAccount**: Unified collateral management across positions
- **FundingEngine**: Oracle-anchored funding between traders
- **MarketRegistry**: Market configuration and trading parameters
- **TradeEngine**: Batch settlement of matched trades

## Key Features

### CEX-Style Order Book Trading
- **Central Limit Order Book**: Price-time priority matching like Binance/Bybit
- **Trader-to-Trader**: Direct counterparty matching without LP intermediaries
- **Professional Orders**: Limit, Market, Stop, Take-Profit, IOC, FOK
- **Partial Fills**: Large orders filled incrementally across multiple counterparties
- **Real-Time Matching**: Sub-second order execution and matching

### No Liquidity Pool Model
- **Direct Trading**: Traders trade directly against each other
- **No LP Risk**: No liquidity providers acting as counterparties
- **Market-Driven Pricing**: Order book determines execution prices
- **Zero Protocol Directional Risk**: Protocol never takes trading positions
- **Pure Matching**: Protocol only facilitates trade matching and settlement

### Oracle-Anchored Risk Management
- **Mark Price = Oracle Price**: Liquidations and funding use oracle prices
- **Fair Liquidations**: Prevent manipulation through order book pricing
- **Funding Mechanism**: Based on price deviation from oracle
- **Risk Calculations**: All margin and health checks use oracle data

### Cross-Margin System
- **Shared Collateral**: Single margin pool across all positions
- **Portfolio Margining**: Offsetting positions reduce margin requirements
- **Capital Efficiency**: Maximize leverage with available margin
- **Unified Risk**: Holistic portfolio risk management

## Supported Markets

### Cryptocurrency Perpetuals
- **BTC/USD**: Bitcoin perpetual (up to 50x leverage)
- **ETH/USD**: Ethereum perpetual (up to 50x leverage)
- **BNB/USD**: Binance Coin perpetual (up to 25x leverage)
- **SOL/USD**: Solana perpetual (up to 25x leverage)
- **AVAX/USD**: Avalanche perpetual (up to 25x leverage)

### Equity Perpetuals
- **AAPL/USD**: Apple perpetual (up to 10x leverage)
- **TSLA/USD**: Tesla perpetual (up to 5x leverage)
- **GOOGL/USD**: Google perpetual (up to 10x leverage)
- **AMZN/USD**: Amazon perpetual (up to 10x leverage)
- **MSFT/USD**: Microsoft perpetual (up to 10x leverage)

### Commodity Perpetuals
- **GOLD/USD**: Gold perpetual (up to 20x leverage)
- **SILVER/USD**: Silver perpetual (up to 20x leverage)
- **OIL/USD**: Crude Oil perpetual (up to 20x leverage)

### Forex Perpetuals
- **EUR/USD**: Euro perpetual (up to 100x leverage)
- **GBP/USD**: British Pound perpetual (up to 100x leverage)
- **JPY/USD**: Japanese Yen perpetual (up to 100x leverage)

## Position Operations

### Opening Positions via Order Book
```javascript
// Submit limit order to CEX-style order book
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: "BUY",           // BUY or SELL
  size: parseEther("10000"), // $10,000 position
  price: parseEther("50000"), // $50,000 limit price
  orderType: "LIMIT",
  timeInForce: "GTC",    // Good Till Cancel
  reduceOnly: false      // Open new position
})

// Market order for immediate execution
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: "BUY",
  size: parseEther("5000"),
  orderType: "MARKET",
  slippageTolerance: 100 // 1% max slippage
})
```

### Position Management
```javascript
// Get position info
const position = await positionManager.getPosition(
  userAddress,
  keccak256("BTC")
)

// Close position with market order
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: position.isLong ? "SELL" : "BUY",
  size: position.size,
  orderType: "MARKET",
  reduceOnly: true
})
```

### Position Structure
```solidity
struct Position {
    int256 size;              // Position size (+long, -short)
    uint256 entryPrice;       // Average entry price
    uint256 lastFundingIndex; // Last funding index
    uint256 timestamp;        // Position open time
}
```

## CEX-Style Trading Model

### Order Book Mechanics
- **Price-Time Priority**: Best price first, then earliest timestamp
- **Continuous Matching**: Real-time order matching as orders arrive
- **Depth Aggregation**: Multiple orders at same price level
- **Spread Management**: Natural bid-ask spread from trader orders
- **Market Impact**: Large orders move through multiple price levels

### Trade Execution Flow
```
1. Trader A submits BUY order: 1 BTC at $50,000
2. Trader B submits SELL order: 1 BTC at $50,000
3. Orders match at $50,000
4. On-chain settlement:
   - Trader A: Long position opened
   - Trader B: Short position opened
   - Both pay trading fees
   - Margin reserved for both positions
```

### No Liquidity Pool Counterparty
Unlike GMX/GLP model:
- ❌ No LP tokens or liquidity providers
- ❌ No protocol acting as counterparty
- ❌ No pool-based PnL calculations
- ✅ Direct trader-to-trader matching
- ✅ Market-driven price discovery
- ✅ Zero protocol directional exposure

## Margin System

### Cross-Margin Benefits
- **Capital Efficiency**: Use same collateral for multiple positions
- **Risk Netting**: Offsetting positions reduce margin requirements
- **Simplified Management**: Single margin balance to monitor
- **Portfolio Approach**: Holistic risk assessment

### Margin Calculations
```solidity
// Initial margin requirement
initialMargin = positionNotional / maxLeverage

// Maintenance margin requirement  
maintenanceMargin = positionNotional * maintenanceMarginBps / 10000

// Available margin
availableMargin = totalCollateral + unrealizedPnL - usedMargin
```

### Leverage Limits by Asset Category
| Asset Category | Max Leverage | Maintenance Margin |
|----------------|--------------|--------------------|
| Major Crypto   | 50x          | 2%                 |
| Minor Crypto   | 25x          | 4%                 |
| Major Equities | 10x          | 10%                |
| Volatile Equities | 5x        | 20%                |
| Commodities    | 20x          | 5%                 |
| Forex          | 100x         | 1%                 |

## Funding Mechanism (Trader-to-Trader)

### CEX-Style Funding
Funding flows directly between traders, not through liquidity pools:

```
Funding Rate = clamp(
    (Order Book Mid Price - Oracle Price) / Oracle Price,
    ±maxFundingRate
)
```

### Direct Payment System
- **Long Positions**: Pay funding when order book price > oracle price
- **Short Positions**: Receive funding when order book price > oracle price
- **No Protocol Capture**: 100% of funding flows between traders
- **Hourly Settlement**: Funding applied every hour automatically

### Funding Calculation Example
```
Oracle BTC Price: $50,000
Order Book Mid Price: $50,100 (0.2% premium)
Funding Rate: +0.2% / 24 = +0.0083% per hour

Long Position (1 BTC): Pays $4.17 per hour
Short Position (1 BTC): Receives $4.17 per hour
```

### Funding Rate Caps
- **Crypto**: ±10% daily maximum
- **Equities**: ±5% daily maximum (market hours), ±1% (after hours)
- **Commodities**: ±8% daily maximum
- **Forex**: ±15% daily maximum

## Order Types & Execution

### Professional Order Types
1. **Limit Orders**: Execute at specified price or better
2. **Market Orders**: Execute immediately against best available orders
3. **Stop Orders**: Trigger market order when price reaches stop level
4. **Take Profit**: Close position when profit target reached
5. **IOC (Immediate or Cancel)**: Execute immediately or cancel remainder
6. **FOK (Fill or Kill)**: Execute completely or cancel entire order
7. **Post-Only**: Only add liquidity (reject if would match immediately)
8. **Reduce-Only**: Only reduce existing position size

### Order Matching Engine
- **Price Priority**: Better prices matched first
- **Time Priority**: Earlier orders at same price matched first
- **Pro-Rata**: Large orders split across multiple counterparties
- **Self-Trade Prevention**: Users cannot trade against themselves
- **Minimum Size**: Prevent dust orders and spam

### Execution Examples
```javascript
// Stop-loss order
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: "SELL",
  size: parseEther("10000"),
  orderType: "STOP",
  stopPrice: parseEther("48000"), // Trigger at $48,000
  reduceOnly: true
})

// Take-profit order
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: "SELL",
  size: parseEther("10000"),
  orderType: "TAKE_PROFIT",
  stopPrice: parseEther("55000"), // Trigger at $55,000
  reduceOnly: true
})
```

## Liquidation System

### Liquidation Triggers
A position is liquidatable when:
- **Margin Ratio**: Below maintenance margin requirement
- **Oracle-Based**: Uses mark price (oracle) for calculations
- **Cross-Margin**: Considers entire portfolio health

### Liquidation Process
1. **Health Monitoring**: Continuous position health checks
2. **Liquidation Call**: Liquidator calls liquidatePosition()
3. **Price Validation**: Confirm position is liquidatable at oracle price
4. **Position Closure**: Close position at mark price
5. **Liquidator Reward**: 0.5% of position size + fixed fee

### Auto-Deleveraging (ADL)
When liquidations cannot be filled:
1. **Ranking**: Rank profitable positions by PnL and leverage
2. **Selection**: Choose highest-ranked positions for ADL
3. **Execution**: Force close positions at mark price
4. **Compensation**: ADL participants receive small compensation

## Risk Management

### Oracle Security
- **Primary Source**: Pyth Network for real-time prices
- **Fallback**: Chainlink for established assets
- **Validation**: Staleness and deviation checks
- **Circuit Breakers**: Pause trading on extreme price moves

### Position Limits
- **Max Position Size**: Per-market position size limits
- **Open Interest Caps**: Maximum total long/short per market
- **Concentration Limits**: Prevent excessive exposure to single asset
- **User Limits**: Per-user position size restrictions

### Market Hours Logic (Equity-Specific)
| Market State | Funding Behavior | Trading |
|--------------|------------------|---------|
| Market Open | Normal funding rates | Full trading |
| Market Closed | Capped funding (±1%) | Limited trading |
| Trading Halt | Funding paused | Trading paused |
| Extreme Volatility | Emergency clamp | Circuit breaker |

## Fee Structure

### Trading Fees (Maker/Taker Model)
| Asset Category | Maker Fee | Taker Fee |
|----------------|-----------|-----------|
| Major Crypto | -0.01% (rebate) | 0.05% |
| Minor Crypto | 0.00% | 0.08% |
| Equities | 0.02% | 0.10% |
| Commodities | 0.01% | 0.07% |
| Forex | -0.005% (rebate) | 0.03% |

### Other Fees
- **Funding Fees**: Variable, paid between traders
- **Liquidation Fee**: 0.5% of position + $5 fixed
- **ADL Compensation**: 0.1% of ADL'd position

## DUSD Staking Integration

### Protocol Backstop
DUSD staking provides the economic backbone:
- **Insurance Fund**: Staked DUSD backs the insurance fund
- **Bad Debt Coverage**: Stakers absorb losses after insurance fund
- **Revenue Sharing**: Stakers earn from trading fees and liquidations
- **Governance Rights**: Participate in risk parameter decisions

### Staking Tiers
| Lock Period | Reward Multiplier | Risk Level |
|-------------|-------------------|------------|
| Flexible | 1.0x | Low |
| 3 months | 1.1x | Low |
| 6 months | 1.25x | Medium |
| 12 months | 1.5x | High |

### Loss Waterfall
```
Trader Loss → Insurance Fund → Junior DUSD Stakers → Senior DUSD Stakers
```

## Integration Examples

### Perpetual Trading

```javascript
// Submit limit order to CEX-style order book
await di.orderBook.submitOrder({
  market: 'BTC',
  side: 'BUY',
  size: '10000', // $10,000 position
  price: '50000', // $50,000 limit
  orderType: 'LIMIT',
  timeInForce: 'GTC'
})

// Market order for immediate execution
await di.orderBook.submitOrder({
  market: 'BTC',
  side: 'SELL',
  size: '5000',
  orderType: 'MARKET'
})

// Advanced order types
await di.orderBook.submitOrder({
  market: 'BTC',
  side: 'SELL',
  size: '10000',
  orderType: 'STOP',
  stopPrice: '48000',
  reduceOnly: true // Stop-loss
})
```

### Position Monitoring
```javascript
const usePosition = (account, market) => {
  return useQuery({
    queryKey: ['position', account, market],
    queryFn: async () => {
      const position = await positionManager.getPosition(account, keccak256(market))
      const pnl = await positionManager.calculatePnL(account, keccak256(market))
      const marginRatio = await marginAccount.getMarginRatio(account)
      
      return {
        ...position,
        unrealizedPnL: pnl,
        marginRatio,
        liquidationPrice: calculateLiquidationPrice(position, marginRatio)
      }
    },
    refetchInterval: 5000
  })
}
```

### Order Book Monitoring
```javascript
// Real-time order book data
const useOrderBook = (market) => {
  return useQuery({
    queryKey: ['orderbook', market],
    queryFn: async () => {
      const orderbook = await orderBookAPI.getOrderBook(market)
      return {
        bids: orderbook.bids, // Buy orders
        asks: orderbook.asks, // Sell orders
        spread: orderbook.asks[0].price - orderbook.bids[0].price,
        midPrice: (orderbook.asks[0].price + orderbook.bids[0].price) / 2
      }
    },
    refetchInterval: 100 // Update every 100ms
  })
}

// Trade history and fills
const useTradeHistory = (account) => {
  return useQuery({
    queryKey: ['trades', account],
    queryFn: async () => {
      return await orderBookAPI.getTradeHistory(account)
    },
    refetchInterval: 1000
  })
}
```

### Liquidation Bot
```javascript
// Monitor positions for liquidation opportunities
const liquidationBot = async () => {
  const positions = await positionManager.getAllPositions()
  
  for (const position of positions) {
    const canLiquidate = await positionManager.canLiquidatePosition(
      position.user,
      position.market
    )
    
    if (canLiquidate) {
      await positionManager.liquidatePosition(
        position.user,
        position.market
      )
    }
  }
}
```

## Smart Contract Architecture

### Core Contracts
```solidity
// Market configuration
interface IMarketRegistry {
    function getMarket(bytes32 market) external view returns (MarketConfig memory);
    function isValidMarket(bytes32 market) external view returns (bool);
    function isMarketOpen(bytes32 market) external view returns (bool);
}

// Position management
interface IPositionManager {
    function openPosition(TradeParams calldata params) external payable;
    function closePosition(bytes32 market, uint256 acceptablePrice) external payable;
    function liquidatePosition(address user, bytes32 market) external;
    function getPosition(address user, bytes32 market) external view returns (Position memory);
}

// Margin accounting
interface IMarginAccount {
    function deposit(uint256 amount) external;
    function withdraw(uint256 amount) external;
    function getAvailableMargin(address user) external view returns (uint256);
    function getMarginRatio(address user) external view returns (uint256);
}

// Order book
interface IOrderBook {
    function submitOrder(OrderParams calldata params) external payable;
    function cancelOrder(uint256 orderId) external;
    function getOrderBook(bytes32 market) external view returns (OrderBookData memory);
    function getUserOrders(address user) external view returns (Order[] memory);
}
```

## Security Features

### Access Control
- **Role-Based Permissions**: Admin, Liquidator, Keeper roles
- **Multi-Signature**: Critical functions require multiple signatures
- **Timelock**: Parameter changes have execution delay
- **Emergency Pause**: Immediate trading halt capability

### Oracle Protection
- **Multiple Sources**: Pyth + Chainlink redundancy
- **Staleness Checks**: Reject outdated price data
- **Deviation Limits**: Maximum price movement per update
- **Circuit Breakers**: Automatic pause on extreme moves

### Economic Security
- **Insurance Fund**: Protocol-owned reserves for bad debt
- **DUSD Staking**: Community-backed insurance mechanism
- **Position Limits**: Prevent excessive concentration risk
- **Liquidation Incentives**: Ensure timely liquidation execution

## Benefits

### For Traders
- **CEX-Like Experience**: Familiar order book trading interface
- **Direct Counterparty**: Trade directly against other traders
- **Professional Tools**: Advanced order types and execution options
- **Fair Pricing**: Market-driven price discovery through order book
- **High Leverage**: Up to 100x on forex, 50x on crypto
- **Cross-Margin**: Efficient capital utilization across positions

### For Market Makers
- **Maker Rebates**: Earn fees for providing liquidity to order book
- **Professional APIs**: High-frequency trading support
- **Risk Management**: Sophisticated position and risk controls
- **Institutional Features**: Large order handling and execution

### For the Protocol
- **Zero Directional Risk**: Never acts as counterparty to trades
- **Sustainable Model**: Earn fees from trade facilitation only
- **Scalable Architecture**: Order book model supports unlimited throughput
- **Market Neutral**: Protocol success independent of trader PnL