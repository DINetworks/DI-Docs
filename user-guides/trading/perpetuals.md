# Perpetual Trading Guide

Master leveraged trading on DPerp, DI Network's perpetual futures platform offering up to 50x leverage on synthetic assets with no expiration dates.

## Getting Started

### Prerequisites
- Connected wallet with DUSD for collateral
- Understanding of leverage and margin trading
- Familiarity with perpetual futures concepts

### Account Setup
1. Connect your wallet to DPerp
2. Deposit DUSD as collateral
3. Review risk parameters and position limits
4. Start with small positions to learn the platform

## Opening Your First Position

### Step 1: Choose Market
Select from available perpetual markets:
- **Crypto**: BTC/USD, ETH/USD, SOL/USD
- **Commodities**: GOLD/USD, OIL/USD
- **Equities**: AAPL/USD, TSLA/USD
- **Forex**: EUR/USD, GBP/USD

### Step 2: Set Position Parameters
```
Market: BTC/USD
Side: Long ☑ / Short ☐
Collateral: 1000 DUSD
Leverage: 10x
Position Size: 10,000 USD
Entry Price: ~$40,000
```

### Step 3: Risk Management
- Set stop-loss levels
- Define take-profit targets
- Monitor liquidation price
- Understand funding costs

### Step 4: Execute Trade
1. Review all parameters
2. Click "Open Position"
3. Confirm transaction in wallet
4. Monitor position in dashboard

## Position Management

### Monitoring Positions
Your position dashboard shows:
- **Current P&L**: Unrealized profit/loss
- **Entry Price**: Average entry price
- **Mark Price**: Current market price
- **Liquidation Price**: Price at which position gets liquidated
- **Funding Rate**: Current funding rate
- **Margin Ratio**: Remaining margin percentage

### Adjusting Positions
**Add Collateral**:
```
Current Position: 10,000 USD with 1000 DUSD collateral (10x)
Add Collateral: 500 DUSD
New Leverage: 6.67x (safer position)
```

**Increase Size**:
```
Current Position: 10,000 USD
Add Size: 5,000 USD with 500 DUSD collateral
New Position: 15,000 USD with 1,500 DUSD total collateral
```

**Partial Close**:
```
Current Position: 10,000 USD
Close: 3,000 USD (30% of position)
Remaining: 7,000 USD position
```

## Leverage and Margin

### Leverage Levels
| Asset Category | Max Leverage | Recommended for Beginners |
|----------------|--------------|---------------------------|
| Major Crypto   | 50x          | 2-5x                     |
| Minor Crypto   | 25x          | 2-3x                     |
| Commodities    | 20x          | 2-4x                     |
| Equities       | 10x          | 2-3x                     |
| Forex          | 100x         | 5-10x                    |

### Margin Requirements
**Initial Margin**: Minimum collateral to open position
```
Position Size: $10,000
Leverage: 20x
Initial Margin Required: $500 (5%)
```

**Maintenance Margin**: Minimum to avoid liquidation
```
Maintenance Margin: 2.5% of position size
For $10,000 position: $250 minimum
```

### Liquidation Protection
- Monitor health factor (should stay > 1.2)
- Add collateral when approaching liquidation
- Use stop-losses to limit downside
- Consider reducing leverage during volatility

## Funding Rates

### Understanding Funding
Funding payments occur every 8 hours between long and short traders:
- **Positive Rate**: Longs pay shorts
- **Negative Rate**: Shorts pay longs
- **Rate Based On**: Market demand and utilization

### Funding Schedule
| Time (UTC) | Funding Payment |
|------------|-----------------|
| 00:00      | 8-hour funding  |
| 08:00      | 8-hour funding  |
| 16:00      | 8-hour funding  |

### Funding Calculation
```
Position Size: $10,000
Funding Rate: 0.1% (8-hour rate)
Funding Payment: $10,000 × 0.001 = $10
```

### Funding Strategies
- **Funding Arbitrage**: Collect funding when rates are high
- **Rate Monitoring**: Close positions before expensive funding
- **Delta Hedging**: Hedge spot positions to collect funding

## Trading Strategies

### Trend Following
1. Identify strong trends using technical analysis
2. Enter positions in trend direction
3. Use trailing stops to protect profits
4. Scale out of positions gradually

### Mean Reversion
1. Identify overbought/oversold conditions
2. Enter counter-trend positions
3. Target return to mean prices
4. Use tight stops due to trend risk

### Scalping
1. Make many small, quick trades
2. Target small price movements
3. Use high leverage for amplified returns
4. Requires constant monitoring

### Hedging
1. Hedge existing spot positions
2. Reduce portfolio volatility
3. Protect against adverse moves
4. Maintain market exposure

## Risk Management

### Position Sizing
**Kelly Criterion**: Optimal position size based on win rate
```
Position Size = (Win Rate × Average Win - Loss Rate × Average Loss) / Average Win
```

**Fixed Percentage**: Risk fixed percentage per trade
```
Account Size: $10,000
Risk per Trade: 2%
Maximum Loss: $200 per position
```

### Stop Losses
**Percentage-Based**:
```
Entry: $40,000
Stop Loss: 5% = $38,000
Risk per Unit: $2,000
```

**Technical-Based**:
- Support/resistance levels
- Moving averages
- Trend lines
- Previous highs/lows

### Portfolio Limits
- Maximum 20% of portfolio in perpetuals
- No more than 5 open positions simultaneously
- Diversify across asset categories
- Monitor correlation between positions

## Advanced Features

### Order Types
**Market Orders**: Execute immediately at current price
**Limit Orders**: Execute at specific price or better
**Stop Orders**: Trigger market order at stop price
**Take Profit**: Close position at profit target

### Conditional Orders
Set up automated trading rules:
1. **If-Then Orders**: Execute based on conditions
2. **Trailing Stops**: Dynamic stop-loss levels
3. **OCO Orders**: One-cancels-other orders
4. **Time-Based**: Execute at specific times

### Portfolio Margin
Advanced users can use portfolio margin:
- Cross-margin multiple positions
- Offset long/short positions
- Improved capital efficiency
- Higher complexity and risk

## Performance Tracking

### Key Metrics
- **Total P&L**: Overall profit/loss
- **Win Rate**: Percentage of profitable trades
- **Average Win/Loss**: Size of wins vs losses
- **Sharpe Ratio**: Risk-adjusted returns
- **Maximum Drawdown**: Largest peak-to-trough decline

### Trade Journal
Record for each trade:
- Entry/exit prices and times
- Position size and leverage
- Reasoning for trade
- Outcome and lessons learned

## Common Mistakes

### Overleveraging
- Using maximum leverage available
- Not accounting for volatility
- Ignoring liquidation risk
- Chasing losses with higher leverage

### Poor Risk Management
- No stop-losses
- Position sizes too large
- Ignoring correlation
- Emotional decision making

### Funding Ignorance
- Not understanding funding costs
- Holding positions through expensive funding
- Ignoring funding rate trends
- Not factoring funding into strategy

## Mobile Trading

### Mobile App Features
- Full trading functionality
- Real-time price alerts
- Position monitoring
- Quick order execution

### Mobile Best Practices
- Use limit orders for better execution
- Set up price alerts for key levels
- Monitor positions regularly
- Have backup internet connection

## Troubleshooting

### Position Not Opening
- Check collateral balance
- Verify leverage limits
- Ensure market is active
- Check for system maintenance

### Unexpected Liquidation
- Monitor health factor closely
- Add collateral proactively
- Understand funding impact
- Use position size calculator

### Funding Confusion
- Check funding rate history
- Understand 8-hour schedule
- Calculate funding costs
- Consider funding in strategy

## Getting Advanced

### API Trading
- Automated trading strategies
- Real-time data feeds
- Programmatic order management
- Backtesting capabilities

### Institutional Features
- Higher position limits
- Dedicated support
- Custom fee structures
- Advanced reporting

### Community
- Join trading discussions
- Share strategies and insights
- Learn from experienced traders
- Participate in competitions