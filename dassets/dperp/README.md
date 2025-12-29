# DPerp - Perpetual Trading

DPerp enables leveraged perpetual trading of synthetic assets using an order book model with oracle-anchored pricing. The system provides up to 50x leverage with cross-margin support and trader-to-trader counterparty matching.

## Overview

DPerp provides:
- Central limit order book (CLOB) trading
- Oracle-anchored mark pricing (never order book pricing)
- Cross-margin system for capital efficiency
- Price-deviation based funding rates
- Equity-aware market hours and risk controls

## Architecture

```
User Interface (OrderBook API)
         ↓
    Matching Engine (Off-chain)
         ↓
    Settlement Layer (On-chain)
         ↓
┌─────────────────────────────────────────┐
│  MarketRegistry │ PositionManager       │
│  OracleModule   │ MarginAccount         │
│  FundingEngine  │ LiquidationEngine     │
└─────────────────────────────────────────┘
```

### Core Components

- **MarketRegistry**: Market configuration and status management
- **PositionManager**: Position lifecycle and PnL tracking
- **MarginAccount**: Cross-margin collateral management
- **FundingEngine**: Oracle-anchored funding rate system
- **OrderBook**: Off-chain order matching with on-chain settlement
- **LiquidationEngine**: Automated liquidation system

## Key Features

### Order Book Trading
- **Central Limit Order Book**: Price-time priority matching
- **Order Types**: Limit, Market, Post-only, Reduce-only, IOC/FOK
- **Partial Fills**: Support for partial order execution
- **Batch Settlement**: Gas-efficient trade commits

### Oracle-Anchored Pricing
- **Mark Price = Oracle Price**: Never uses order book mid price
- **Funding Mechanism**: Based on price deviation from oracle
- **Risk Management**: All margin and liquidation use oracle prices
- **Market Hours**: Equity-aware funding adjustments

### Cross-Margin System
- **Shared Collateral**: Single margin pool across all positions
- **Capital Efficiency**: Maximize leverage with available margin
- **Portfolio Risk**: Unified risk management
- **Netting**: Offsetting positions reduce margin requirements

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
// Submit limit order to order book
await orderBook.submitOrder({
  market: keccak256("BTC"),
  side: "BUY",           // BUY or SELL
  size: parseEther("10000"), // $10,000 position
  price: parseEther("50000"), // $50,000 limit price
  orderType: "LIMIT",
  timeInForce: "GTC"     // Good Till Cancel
})
```

### Position Management
```javascript
// Get position info
const position = await positionManager.getPosition(
  userAddress,
  keccak256("BTC")
)

// Close position at market
await positionManager.closePosition(
  keccak256("BTC"),
  parseEther("50000") // acceptable price
)
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

## Funding Mechanism

### Oracle-Anchored Funding
Unlike traditional perp funding based on open interest skew, DPerp uses price deviation:

```
Funding Rate = clamp(
    (Perp Price - Oracle Price) / Oracle Price,
    ±maxFundingRate
)
```

### Funding Flow
- **Direct Payment**: Longs ↔ Shorts (no protocol capture)
- **Continuous Accrual**: Real-time funding accumulation
- **Hourly Settlement**: Applied every hour
- **Market Hours Aware**: Reduced funding when equity markets closed

### Funding Rate Caps
- **Crypto**: ±10% daily maximum
- **Equities**: ±5% daily maximum (market hours), ±1% (after hours)
- **Commodities**: ±8% daily maximum
- **Forex**: ±15% daily maximum

## Order Types & Execution

### Supported Order Types
1. **Limit Orders**: Execute at specified price or better
2. **Market Orders**: Execute immediately at best available price
3. **Post-Only**: Only add liquidity (reject if would take)
4. **Reduce-Only**: Only reduce existing position size
5. **IOC (Immediate or Cancel)**: Execute immediately or cancel
6. **FOK (Fill or Kill)**: Execute completely or cancel

### Order Matching
- **Price-Time Priority**: Best price first, then time priority
- **Partial Fills**: Large orders can be filled incrementally
- **Self-Trade Prevention**: Prevent user trading with themselves
- **Minimum Size**: Prevent dust orders

### Execution Flow
```
1. User submits order to OrderBook API
2. Off-chain matching engine processes order
3. Matched trades batched for settlement
4. On-chain settlement updates positions
5. Margin and funding calculations updated
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

### Order Submission
```javascript
import { OrderBook } from '@di-network/sdk'

const orderBook = new OrderBook({
  provider: web3Provider,
  network: 'mainnet'
})

// Submit limit order
const order = await orderBook.submitOrder({
  market: 'BTC',
  side: 'BUY',
  size: '10000', // $10,000 position
  price: '50000', // $50,000 limit
  orderType: 'LIMIT',
  timeInForce: 'GTC',
  reduceOnly: false
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
- **High Leverage**: Up to 100x on forex, 50x on crypto
- **Capital Efficiency**: Cross-margin system maximizes capital use
- **Fair Pricing**: Oracle-anchored mark prices prevent manipulation
- **24/7 Trading**: Access to global markets around the clock
- **Advanced Orders**: Professional trading tools and order types

### For Market Makers
- **Maker Rebates**: Earn fees for providing liquidity
- **Professional Tools**: Advanced order types and APIs
- **Risk Management**: Sophisticated position and risk controls
- **Institutional Features**: High-volume trading support

### For the Protocol
- **Sustainable Model**: Trader-to-trader matching, no directional risk
- **Scalable Architecture**: Order book model supports high throughput
- **Revenue Generation**: Trading fees and liquidation penalties
- **Community Alignment**: DUSD staking aligns incentives