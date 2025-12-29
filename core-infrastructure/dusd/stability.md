# Stability Mechanism

DUSD maintains its $1 peg through a combination of algorithmic mechanisms, collateral backing, and market incentives designed to ensure price stability under various market conditions.

## Stability Overview

### Peg Maintenance
DUSD uses multiple mechanisms to maintain its $1.00 USD peg:
- **Collateral Backing**: Over-collateralized positions
- **Arbitrage Incentives**: Profit opportunities for peg restoration
- **Stability Pool**: Community-funded price support
- **Redemption Mechanism**: Direct DUSD-to-collateral exchange
- **Interest Rate Adjustments**: Dynamic borrowing costs

### Target Metrics
- **Price Range**: $0.995 - $1.005 (0.5% deviation)
- **Collateralization Ratio**: 150% minimum
- **Stability Pool Size**: 10-20% of DUSD supply
- **Redemption Fee**: 0.5-2% based on system state

## Collateral-Backed Stability

### Over-Collateralization
```solidity
function mintDUSD(uint256 collateralAmount) external {
    uint256 collateralValue = getCollateralValue(collateralAmount);
    uint256 maxMintable = collateralValue * 100 / COLLATERAL_RATIO; // 150% = 66.67% LTV
    
    require(dusdAmount <= maxMintable, "Insufficient collateral");
    
    collateralBalances[msg.sender] += collateralAmount;
    dusdBalances[msg.sender] += dusdAmount;
    
    _mint(msg.sender, dusdAmount);
}
```

### Collateral Types & Ratios
| Collateral | Min Ratio | Liquidation Threshold | Stability Fee |
|------------|-----------|----------------------|---------------|
| **ETH** | 150% | 130% | 2% APR |
| **BTC** | 150% | 130% | 2% APR |
| **USDC** | 110% | 105% | 0.5% APR |
| **DI Token** | 200% | 160% | 1% APR |

## Redemption Mechanism

### Direct Redemption
Users can redeem DUSD for underlying collateral at face value:
```solidity
function redeemDUSD(uint256 dusdAmount) external {
    uint256 fee = calculateRedemptionFee(dusdAmount);
    uint256 netAmount = dusdAmount - fee;
    
    // Burn DUSD
    _burn(msg.sender, dusdAmount);
    
    // Release collateral proportionally
    uint256 collateralAmount = (netAmount * totalCollateral) / totalDUSDSupply;
    transferCollateral(msg.sender, collateralAmount);
    
    emit Redemption(msg.sender, dusdAmount, collateralAmount, fee);
}
```

### Redemption Fee Structure
```javascript
const calculateRedemptionFee = (systemState) => {
  const baseRate = 0.005 // 0.5%
  const maxRate = 0.05   // 5%
  
  // Increase fee when system is under stress
  const stressMultiplier = systemState.collateralRatio < 1.5 ? 2.0 : 1.0
  const demandMultiplier = systemState.redemptionVolume > systemState.avgVolume ? 1.5 : 1.0
  
  return Math.min(baseRate * stressMultiplier * demandMultiplier, maxRate)
}
```

## Arbitrage Mechanisms

### Price Deviation Response
When DUSD trades off-peg, arbitrage opportunities emerge:

**DUSD > $1.00 (Premium)**
1. Mint DUSD with collateral
2. Sell DUSD on market for premium
3. Profit = Premium - Stability Fee

**DUSD < $1.00 (Discount)**
1. Buy DUSD at discount
2. Redeem for full collateral value
3. Profit = Discount - Redemption Fee

### Arbitrage Example
```javascript
// DUSD trading at $0.98 (2% discount)
const arbitrageOpportunity = async () => {
  const dusdPrice = 0.98
  const redemptionFee = 0.01 // 1%
  const profit = (1.00 - dusdPrice - redemptionFee) // 1% profit
  
  if (profit > 0) {
    // Buy DUSD at discount
    await buyDUSD(parseEther("10000"), dusdPrice)
    
    // Redeem for full value
    await dusd.redeem(parseEther("10000"))
    
    // Net profit: ~$100 on $10,000 trade
  }
}
```

## Interest Rate Mechanism

### Dynamic Rate Adjustment
Interest rates adjust based on DUSD price and system utilization:
```solidity
function updateInterestRate() external {
    uint256 currentPrice = oracle.getDUSDPrice();
    uint256 utilizationRate = totalBorrowed * 1e18 / totalCollateralValue;
    
    if (currentPrice < TARGET_PRICE) {
        // DUSD below peg - increase rates to reduce supply
        baseRate = baseRate * 110 / 100; // +10%
    } else if (currentPrice > TARGET_PRICE) {
        // DUSD above peg - decrease rates to increase supply
        baseRate = baseRate * 95 / 100; // -5%
    }
    
    // Adjust for utilization
    uint256 utilizationAdjustment = utilizationRate > 80e16 ? 120 : 100; // +20% if >80% utilized
    currentRate = baseRate * utilizationAdjustment / 100;
}
```

### Rate Calculation
```
Final Rate = Base Rate × Price Adjustment × Utilization Adjustment × Risk Premium
```

## Stability Pool

### Pool Mechanics
The stability pool provides DUSD price support through:
- **Liquidation Absorption**: Pool covers bad debt
- **Price Floor**: Creates buying pressure below peg
- **Yield Generation**: Rewards for stability providers

### Pool Operations
```solidity
contract StabilityPool {
    function provideToSP(uint256 amount) external {
        dusd.transferFrom(msg.sender, address(this), amount);
        
        deposits[msg.sender] += amount;
        totalDeposits += amount;
        
        // Update reward snapshots
        updateRewardSnapshots(msg.sender);
    }
    
    function offset(uint256 debtToOffset, uint256 collToAdd) external onlyTroveManager {
        uint256 totalDUSD = totalDeposits;
        
        // Reduce each deposit proportionally
        for (address depositor in depositors) {
            uint256 depositLoss = deposits[depositor] * debtToOffset / totalDUSD;
            deposits[depositor] -= depositLoss;
            
            // Compensate with collateral gain
            uint256 collateralGain = collToAdd * deposits[depositor] / totalDUSD;
            collateralGains[depositor] += collateralGain;
        }
        
        totalDeposits -= debtToOffset;
    }
}
```

## Emergency Mechanisms

### Circuit Breakers
Automatic responses to extreme market conditions:
```solidity
function checkCircuitBreakers() internal {
    uint256 price = oracle.getDUSDPrice();
    uint256 deviation = price > 1e18 ? price - 1e18 : 1e18 - price;
    
    if (deviation > MAX_DEVIATION) { // 5% deviation
        // Pause new minting
        mintingPaused = true;
        
        // Increase redemption incentives
        emergencyRedemptionBonus = true;
        
        // Notify governance
        emit EmergencyActivated(price, deviation);
    }
}
```

### Recovery Mode
When system collateral ratio falls below 150%:
```solidity
modifier recoveryMode() {
    require(getSystemCollateralRatio() >= RECOVERY_THRESHOLD, "System in recovery mode");
    _;
}

function enterRecoveryMode() internal {
    // Halt new borrowing
    borrowingPaused = true;
    
    // Reduce liquidation threshold
    liquidationThreshold = RECOVERY_LIQUIDATION_THRESHOLD;
    
    // Increase stability incentives
    stabilityPoolRewards *= 2;
}
```

## Monitoring & Analytics

### Stability Metrics
```javascript
const useStabilityMetrics = () => {
  return useQuery(['stabilityMetrics'], async () => {
    const [price, collateralRatio, poolSize, redemptionRate] = await Promise.all([
      oracle.getDUSDPrice(),
      getSystemCollateralRatio(),
      stabilityPool.getTotalDeposits(),
      getRedemptionRate()
    ])
    
    return {
      price: Number(price) / 1e18,
      deviation: Math.abs(Number(price) / 1e18 - 1.0),
      collateralRatio: Number(collateralRatio) / 1e18,
      poolSize: formatEther(poolSize),
      redemptionRate: Number(redemptionRate) / 1e18,
      stabilityScore: calculateStabilityScore(price, collateralRatio, poolSize)
    }
  })
}
```

### Stability Dashboard
```javascript
const StabilityDashboard = () => {
  const { data: metrics } = useStabilityMetrics()
  
  return (
    <div className="stability-dashboard">
      <div className="price-widget">
        <div className="current-price">
          ${metrics?.price?.toFixed(4)}
        </div>
        <div className={`deviation ${metrics?.deviation > 0.005 ? 'warning' : 'normal'}`}>
          Deviation: {(metrics?.deviation * 100).toFixed(2)}%
        </div>
      </div>
      
      <div className="system-health">
        <div>Collateral Ratio: {(metrics?.collateralRatio * 100).toFixed(1)}%</div>
        <div>Stability Pool: ${formatNumber(metrics?.poolSize)}</div>
        <div>Redemption Rate: {(metrics?.redemptionRate * 100).toFixed(2)}%</div>
      </div>
      
      <StabilityChart data={metrics} />
    </div>
  )
}
```

## Integration Examples

### Stability Monitoring
```javascript
const monitorStability = async () => {
  const price = await oracle.getDUSDPrice()
  const deviation = Math.abs(Number(price) / 1e18 - 1.0)
  
  if (deviation > 0.01) { // 1% deviation
    // Alert stability providers
    await notifyStabilityProviders({
      price: Number(price) / 1e18,
      deviation,
      action: price > 1e18 ? 'mint_and_sell' : 'buy_and_redeem'
    })
  }
}
```

### Automated Arbitrage
```javascript
const arbitrageBot = async () => {
  const marketPrice = await getMarketPrice('DUSD')
  const redemptionFee = await dusd.getRedemptionFee()
  
  if (marketPrice < 0.995 && (1.0 - marketPrice) > redemptionFee) {
    // Profitable arbitrage opportunity
    const amount = calculateOptimalArbitrageSize(marketPrice, redemptionFee)
    
    await buyDUSDFromMarket(amount, marketPrice)
    await dusd.redeem(amount)
  }
}
```