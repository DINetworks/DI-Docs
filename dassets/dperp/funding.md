# Funding Rates

Funding rates in DPerp ensure that perpetual contract prices stay close to spot prices by creating incentives for traders to balance long and short positions. Funding is exchanged between long and short position holders every 8 hours.

## Funding Mechanism

### Funding Rate Calculation
```solidity
function getNextFundingRate(bytes32 assetId, uint256 poolAmount, uint256 reservedAmount) 
    external view returns (int256) {
    FundingRate memory rate = fundingRates[assetId];
    
    if (poolAmount == 0) return 0;
    
    int256 fundingRate = int256((rate.fundingRateFactor * reservedAmount) / poolAmount);
    
    uint256 maxRate = maxFundingRate[assetId];
    if (maxRate > 0 && uint256(fundingRate > 0 ? fundingRate : -fundingRate) > maxRate) {
        fundingRate = fundingRate > 0 ? int256(maxRate) : -int256(maxRate);
    }
    
    return fundingRate;
}
```

### Funding Rate Structure
```solidity
struct FundingRate {
    uint256 fundingRateFactor;      // Base funding rate factor
    uint256 stableFundingRateFactor; // Stable asset funding factor
    int256 cumulativeFundingRate;    // Cumulative funding rate
    uint256 lastFundingTime;         // Last funding update time
}
```

## Funding Intervals

### 8-Hour Funding Cycles
```solidity
uint256 public constant FUNDING_INTERVAL = 8 hours;

function updateCumulativeFundingRate(bytes32 assetId, uint256 poolAmount, uint256 reservedAmount) 
    external onlyRole(KEEPER_ROLE) {
    FundingRate storage rate = fundingRates[assetId];
    
    if (rate.lastFundingTime + FUNDING_INTERVAL > block.timestamp) {
        return; // Too early for next funding
    }
    
    uint256 intervals = (block.timestamp - rate.lastFundingTime) / FUNDING_INTERVAL;
    
    if (poolAmount > 0) {
        int256 fundingRate = int256((rate.fundingRateFactor * reservedAmount) / poolAmount);
        
        rate.cumulativeFundingRate += fundingRate * int256(intervals);
        rate.lastFundingTime = rate.lastFundingTime + (intervals * FUNDING_INTERVAL);
        
        emit UpdateFundingRate(assetId, fundingRate, rate.cumulativeFundingRate);
    }
}
```

### Funding Schedule
| Time (UTC) | Funding Event |
|------------|---------------|
| 00:00      | Funding Payment |
| 08:00      | Funding Payment |
| 16:00      | Funding Payment |

## Funding Rate Factors

### Utilization-Based Rates
Funding rates depend on pool utilization:
```
Funding Rate = (Reserved Amount / Pool Amount) × Funding Rate Factor
```

### Asset-Specific Factors
| Asset Category | Funding Rate Factor | Max Funding Rate |
|----------------|-------------------|------------------|
| Major Crypto   | 0.01%             | 0.75%           |
| Minor Crypto   | 0.02%             | 1.00%           |
| Commodities    | 0.015%            | 0.50%           |
| Equities       | 0.025%            | 0.30%           |
| Forex          | 0.005%            | 0.25%           |

## Funding Payments

### Payment Calculation
```solidity
function _getFundingFee(
    bytes32 assetId,
    uint256 size,
    uint256 entryFundingRate
) private view returns (uint256) {
    if (size == 0) return 0;
    
    int256 fundingRate = fundingRateManager.getFundingRate(assetId);
    uint256 currentRate = fundingRate > 0 ? uint256(fundingRate) : uint256(-fundingRate);
    
    if (currentRate <= entryFundingRate) return 0;
    
    uint256 fundingDelta = currentRate - entryFundingRate;
    return (size * fundingDelta) / FUNDING_RATE_PRECISION;
}
```

### Payment Direction
- **Positive Funding Rate**: Longs pay shorts
- **Negative Funding Rate**: Shorts pay longs
- **Zero Funding Rate**: No payments

### Example Calculation
```javascript
const calculateFundingPayment = (positionSize, fundingRate, timeHeld) => {
  const hourlyRate = fundingRate / (8 * 3600) // Convert 8-hour rate to per-second
  const fundingPayment = positionSize * hourlyRate * timeHeld
  
  return fundingPayment
}

// Example: $10,000 position, 0.1% funding rate, held for 8 hours
const payment = calculateFundingPayment(10000, 0.001, 8 * 3600) // $10
```

## Funding Rate Monitoring

### Real-time Funding Rates
```javascript
const useFundingRate = (assetId) => {
  return useQuery({
    queryKey: ['fundingRate', assetId],
    queryFn: async () => {
      const currentRate = await fundingRateManager.getFundingRate(keccak256(assetId))
      const poolAmount = await positionManager.poolAmounts(keccak256(assetId))
      const reservedAmount = await positionManager.reservedAmounts(keccak256(assetId))
      const nextRate = await fundingRateManager.getNextFundingRate(
        keccak256(assetId), 
        poolAmount, 
        reservedAmount
      )
      
      return {
        current: Number(currentRate) / 1e6,
        next: Number(nextRate) / 1e6,
        utilization: poolAmount > 0 ? Number(reservedAmount) / Number(poolAmount) : 0
      }
    },
    refetchInterval: 30000
  })
}
```

### Funding History
```javascript
const FundingRateChart = ({ assetId }) => {
  const { data: history } = useQuery(
    ['fundingHistory', assetId],
    () => getFundingRateHistory(assetId),
    { refetchInterval: 300000 } // 5 minutes
  )
  
  return (
    <div className="funding-chart">
      <h3>Funding Rate History - {assetId}</h3>
      <LineChart data={history} />
      <div className="funding-stats">
        <div>Average: {history?.average?.toFixed(4)}%</div>
        <div>Min: {history?.min?.toFixed(4)}%</div>
        <div>Max: {history?.max?.toFixed(4)}%</div>
      </div>
    </div>
  )
}
```

## Funding Rate Impact

### Position Cost Analysis
```javascript
const analyzeFundingCost = (position, fundingRate, holdingPeriod) => {
  const dailyFundingRate = fundingRate * 3 // 3 funding periods per day
  const totalDays = holdingPeriod / (24 * 3600)
  const totalFundingCost = position.size * dailyFundingRate * totalDays
  
  return {
    dailyCost: position.size * dailyFundingRate,
    totalCost: totalFundingCost,
    costPercentage: (totalFundingCost / position.collateral) * 100
  }
}
```

### Funding Rate Arbitrage
```javascript
const findFundingArbitrage = async (assets) => {
  const opportunities = []
  
  for (const asset of assets) {
    const fundingRate = await getFundingRate(asset)
    const spotPremium = await getSpotPremium(asset)
    
    // Look for funding rate arbitrage opportunities
    if (Math.abs(fundingRate) > 0.1 && Math.sign(fundingRate) !== Math.sign(spotPremium)) {
      opportunities.push({
        asset,
        fundingRate,
        spotPremium,
        strategy: fundingRate > 0 ? 'short_perp_long_spot' : 'long_perp_short_spot',
        expectedReturn: Math.abs(fundingRate) - Math.abs(spotPremium)
      })
    }
  }
  
  return opportunities.sort((a, b) => b.expectedReturn - a.expectedReturn)
}
```

## Funding Rate Strategies

### Funding Rate Farming
```javascript
const fundingRateFarm = async (asset, targetFundingRate) => {
  const currentRate = await getFundingRate(asset)
  
  if (Math.abs(currentRate) > targetFundingRate) {
    const strategy = currentRate > 0 ? 'short' : 'long'
    const positionSize = calculateOptimalSize(currentRate, targetFundingRate)
    
    await openPosition(asset, positionSize, strategy === 'long')
    
    return {
      strategy,
      expectedDailyReturn: Math.abs(currentRate) * 3 * positionSize,
      riskLevel: calculateRiskLevel(currentRate, asset)
    }
  }
}
```

### Delta-Neutral Strategies
```javascript
const deltaHedgedFunding = async (asset, perpSize) => {
  // Open perp position to collect funding
  await openPerpPosition(asset, perpSize, true) // Long perp
  
  // Hedge with spot or other derivative
  await hedgePosition(asset, perpSize, false) // Short hedge
  
  return {
    perpPosition: perpSize,
    hedgePosition: -perpSize,
    netDelta: 0,
    fundingIncome: await calculateFundingIncome(asset, perpSize)
  }
}
```

## Funding Rate Configuration

### Admin Functions
```solidity
function setFundingRate(bytes32 assetId, uint256 _fundingRateFactor, uint256 _stableFundingRateFactor) 
    external onlyRole(DEFAULT_ADMIN_ROLE) {
    fundingRates[assetId].fundingRateFactor = _fundingRateFactor;
    fundingRates[assetId].stableFundingRateFactor = _stableFundingRateFactor;
}

function setMaxFundingRate(bytes32 assetId, uint256 _maxFundingRate) 
    external onlyRole(DEFAULT_ADMIN_ROLE) {
    maxFundingRate[assetId] = _maxFundingRate;
}
```

### Dynamic Adjustments
```javascript
const adjustFundingRates = async (marketConditions) => {
  for (const [asset, conditions] of Object.entries(marketConditions)) {
    const currentUtilization = await getUtilization(asset)
    const volatility = await getVolatility(asset)
    
    let newFundingFactor = baseFundingFactors[asset]
    
    // Increase funding factor during high volatility
    if (volatility > 0.5) {
      newFundingFactor *= 1.5
    }
    
    // Adjust based on utilization
    if (currentUtilization > 0.8) {
      newFundingFactor *= 1.2
    }
    
    await setFundingRate(asset, newFundingFactor)
  }
}
```

## Integration Examples

### Funding Rate Display
```javascript
const FundingRateWidget = ({ asset }) => {
  const { data: fundingData } = useFundingRate(asset)
  const nextFundingTime = getNextFundingTime()
  
  return (
    <div className="funding-widget">
      <div className="funding-rate">
        <span>Funding Rate:</span>
        <span className={fundingData?.current > 0 ? 'positive' : 'negative'}>
          {(fundingData?.current * 100).toFixed(4)}%
        </span>
      </div>
      
      <div className="next-funding">
        <span>Next Funding:</span>
        <Countdown target={nextFundingTime} />
      </div>
      
      <div className="funding-prediction">
        <span>Predicted Rate:</span>
        <span>{(fundingData?.next * 100).toFixed(4)}%</span>
      </div>
    </div>
  )
}
```

### Position Funding Calculator
```javascript
const FundingCalculator = ({ position }) => {
  const { data: fundingRate } = useFundingRate(position.asset)
  
  const calculateFundingCost = (hours) => {
    const periods = Math.ceil(hours / 8)
    return position.size * fundingRate.current * periods
  }
  
  return (
    <div className="funding-calculator">
      <h4>Funding Cost Projection</h4>
      <div>8 hours: ${calculateFundingCost(8).toFixed(2)}</div>
      <div>1 day: ${calculateFundingCost(24).toFixed(2)}</div>
      <div>1 week: ${calculateFundingCost(168).toFixed(2)}</div>
    </div>
  )
}
```