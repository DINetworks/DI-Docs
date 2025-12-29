# Liquidations

Liquidations in DPerp protect the protocol and other traders by automatically closing positions that fall below maintenance margin requirements. The system uses a multi-tier liquidation approach to minimize losses and maintain system stability.

## Liquidation Conditions

### Liquidation Triggers
Positions are liquidated when:
1. **Maintenance Margin Breach**: Remaining collateral falls below maintenance requirements
2. **Loss Exceeds Collateral**: Unrealized losses exceed available collateral
3. **Fee Accumulation**: Funding fees exceed remaining margin
4. **Leverage Limit Breach**: Position leverage exceeds maximum allowed

### Liquidation Validation
```solidity
function _validateLiquidation(
    address account,
    bytes32 assetId,
    bool isLong,
    bool raise
) private view returns (uint256, uint256) {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    
    (bool hasProfit, uint256 delta) = getDelta(assetId, position.size, position.entryPrice, isLong, position.lastIncreasedTime);
    
    uint256 marginFees = _getFundingFee(assetId, position.size, position.entryFundingRate);
    marginFees += (position.size * marginFeeBasisPoints[assetId]) / BASIS_POINTS;
    
    // Check if losses exceed collateral
    if (!hasProfit && position.collateral < delta) {
        return (1, marginFees); // Liquidation state 1: Full liquidation
    }
    
    uint256 remainingCollateral = position.collateral;
    if (!hasProfit) {
        remainingCollateral = position.collateral - delta;
    }
    
    // Check if fees exceed remaining collateral
    if (remainingCollateral < marginFees + liquidationFeeUsd[assetId]) {
        return (1, marginFees); // Liquidation state 1: Full liquidation
    }
    
    // Check leverage limit
    if (remainingCollateral * maxLeverage[assetId] < position.size * BASIS_POINTS) {
        return (2, marginFees); // Liquidation state 2: Partial liquidation
    }
    
    return (0, marginFees); // Healthy position
}
```

## Liquidation Types

### Full Liquidation (State 1)
Complete position closure when:
- Losses exceed available collateral
- Remaining margin insufficient for fees
- Position becomes insolvent

### Partial Liquidation (State 2)
Partial position reduction when:
- Leverage exceeds maximum allowed
- Position can be saved by reducing size
- Sufficient collateral remains after reduction

### Liquidation Process
```solidity
function liquidatePosition(
    address account,
    bytes32 assetId,
    bool isLong
) external nonReentrant onlyRole(LIQUIDATOR_ROLE) {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    require(position.size > 0, "No position");
    
    (uint256 liquidationState, uint256 marginFees) = _validateLiquidation(account, assetId, isLong, true);
    require(liquidationState != 0, "Position cannot be liquidated");
    
    if (liquidationState == 2) {
        // Partial liquidation - reduce position to acceptable leverage
        _decreasePosition(account, assetId, 0, position.size, isLong, account);
        return;
    }
    
    // Full liquidation
    uint256 markPrice = isLong ? priceFeed.getPrice(assetId, false) : priceFeed.getPrice(assetId, true);
    
    positionManager.liquidatePosition(account, assetId, isLong, markPrice);
    
    // Update global sizes
    if (isLong) {
        globalLongSizes[assetId] = globalLongSizes[assetId] > position.size ? globalLongSizes[assetId] - position.size : 0;
    } else {
        globalShortSizes[assetId] = globalShortSizes[assetId] > position.size ? globalShortSizes[assetId] - position.size : 0;
    }
    
    // Pay liquidator fee
    uint256 liquidatorFee = (position.collateral * 50) / BASIS_POINTS; // 0.5%
    if (liquidatorFee > 0) {
        dUSD.mint(msg.sender, liquidatorFee);
    }
}
```

## Liquidation Pricing

### Mark Price Calculation
Liquidations use mark prices to prevent manipulation:
```solidity
uint256 markPrice = isLong ? 
    priceFeed.getPrice(assetId, false) :  // Use bid price for long liquidations
    priceFeed.getPrice(assetId, true);    // Use ask price for short liquidations
```

### Liquidation Price Formula
```javascript
const calculateLiquidationPrice = (position, asset) => {
  const maintenanceMargin = liquidationFeeUsd[asset]
  const fundingFees = getFundingFee(asset, position.size, position.entryFundingRate)
  const marginFees = (position.size * marginFeeBasisPoints[asset]) / BASIS_POINTS
  const totalFees = maintenanceMargin + fundingFees + marginFees
  
  const maxLoss = position.collateral - totalFees
  
  if (position.isLong) {
    // Long liquidation price = entry price - (max loss / position size) * entry price
    return position.entryPrice * (1 - maxLoss / position.size)
  } else {
    // Short liquidation price = entry price + (max loss / position size) * entry price
    return position.entryPrice * (1 + maxLoss / position.size)
  }
}
```

## Liquidation Fees

### Fee Structure
| Component | Amount | Recipient |
|-----------|--------|-----------|
| Liquidation Fee | $5-100 | Protocol |
| Liquidator Reward | 0.5% of collateral | Liquidator |
| Gas Compensation | Variable | Liquidator |

### Fee Calculation
```solidity
mapping(bytes32 => uint256) public liquidationFeeUsd;

function setLiquidationFee(bytes32 assetId, uint256 feeUsd) external onlyRole(ADMIN_ROLE) {
    require(feeUsd <= MAX_LIQUIDATION_FEE_USD, "Fee too high");
    liquidationFeeUsd[assetId] = feeUsd;
}
```

## Liquidation Monitoring

### Position Health Monitoring
```javascript
const usePositionHealth = (account, asset, isLong) => {
  return useQuery({
    queryKey: ['positionHealth', account, asset, isLong],
    queryFn: async () => {
      const position = await positionManager.getPosition(account, keccak256(asset), isLong)
      if (position.size === 0n) return null
      
      const [hasProfit, delta] = await dPerp.getDelta(
        keccak256(asset),
        position.size,
        position.entryPrice,
        isLong,
        position.lastIncreasedTime
      )
      
      const fundingFee = await getFundingFee(asset, position.size, position.entryFundingRate)
      const marginFee = (position.size * marginFeeBasisPoints[asset]) / BASIS_POINTS
      const totalFees = fundingFee + marginFee
      
      const remainingCollateral = hasProfit ? 
        position.collateral + delta : 
        position.collateral - delta
      
      const liquidationPrice = calculateLiquidationPrice(position, asset)
      const healthFactor = (remainingCollateral - totalFees) / liquidationFeeUsd[asset]
      
      return {
        healthFactor,
        liquidationPrice,
        remainingCollateral,
        totalFees,
        isAtRisk: healthFactor < 1.5,
        isCritical: healthFactor < 1.1
      }
    },
    refetchInterval: 5000
  })
}
```

### Liquidation Alerts
```javascript
const LiquidationAlert = ({ positionHealth, currentPrice }) => {
  if (!positionHealth?.isAtRisk) return null
  
  const priceDistance = Math.abs(currentPrice - positionHealth.liquidationPrice) / currentPrice
  
  return (
    <div className={`liquidation-alert ${positionHealth.isCritical ? 'critical' : 'warning'}`}>
      {positionHealth.isCritical ? (
        <div className="critical-alert">
          🚨 CRITICAL: Position may be liquidated soon!
          <div>Liquidation Price: ${positionHealth.liquidationPrice.toFixed(2)}</div>
          <div>Distance: {(priceDistance * 100).toFixed(2)}%</div>
        </div>
      ) : (
        <div className="warning-alert">
          ⚠️ WARNING: Position at risk
          <div>Health Factor: {positionHealth.healthFactor.toFixed(2)}</div>
          <div>Consider adding collateral or reducing position size</div>
        </div>
      )}
    </div>
  )
}
```

## Liquidation Protection

### Auto-Deleveraging
```javascript
const autoDeleverage = async (account, asset, isLong, targetLeverage) => {
  const position = await positionManager.getPosition(account, keccak256(asset), isLong)
  const currentLeverage = position.size / position.collateral
  
  if (currentLeverage > targetLeverage) {
    const targetSize = position.collateral * targetLeverage
    const sizeReduction = position.size - targetSize
    
    await dPerp.decreasePosition(
      keccak256(asset),
      0, // No collateral withdrawal
      sizeReduction,
      isLong
    )
    
    return {
      oldLeverage: currentLeverage,
      newLeverage: targetLeverage,
      sizeReduced: sizeReduction
    }
  }
}
```

### Stop-Loss Orders
```javascript
const setStopLoss = async (account, asset, isLong, stopPrice) => {
  const position = await positionManager.getPosition(account, keccak256(asset), isLong)
  const liquidationPrice = calculateLiquidationPrice(position, asset)
  
  // Ensure stop-loss is before liquidation price
  const safeStopPrice = isLong ? 
    Math.max(stopPrice, liquidationPrice * 1.05) : // 5% buffer for longs
    Math.min(stopPrice, liquidationPrice * 0.95)   // 5% buffer for shorts
  
  await createStopLossOrder({
    account,
    asset: keccak256(asset),
    isLong,
    triggerPrice: safeStopPrice,
    size: position.size,
    orderType: 'market'
  })
}
```

## Liquidation Bots

### Liquidator Implementation
```javascript
class LiquidationBot {
  constructor(provider, privateKey) {
    this.provider = provider
    this.wallet = new ethers.Wallet(privateKey, provider)
    this.dPerp = new ethers.Contract(DPERP_ADDRESS, DPERP_ABI, this.wallet)
  }
  
  async scanForLiquidations() {
    const positions = await this.getAllPositions()
    const liquidatablePositions = []
    
    for (const position of positions) {
      const [liquidationState] = await this.dPerp._validateLiquidation(
        position.account,
        position.assetId,
        position.isLong,
        false
      )
      
      if (liquidationState > 0) {
        liquidatablePositions.push({
          ...position,
          liquidationState,
          estimatedReward: await this.calculateLiquidationReward(position)
        })
      }
    }
    
    return liquidatablePositions.sort((a, b) => b.estimatedReward - a.estimatedReward)
  }
  
  async executeLiquidation(position) {
    try {
      const gasEstimate = await this.dPerp.estimateGas.liquidatePosition(
        position.account,
        position.assetId,
        position.isLong
      )
      
      const tx = await this.dPerp.liquidatePosition(
        position.account,
        position.assetId,
        position.isLong,
        { gasLimit: gasEstimate.mul(120).div(100) } // 20% buffer
      )
      
      return await tx.wait()
    } catch (error) {
      console.error('Liquidation failed:', error)
      throw error
    }
  }
  
  async calculateLiquidationReward(position) {
    const liquidatorFee = position.collateral * 0.005 // 0.5%
    const gasPrice = await this.provider.getGasPrice()
    const gasCost = gasPrice.mul(200000) // Estimated gas
    
    return liquidatorFee - gasCost
  }
}
```

### Liquidation Monitoring
```javascript
const LiquidationMonitor = () => {
  const [liquidations, setLiquidations] = useState([])
  
  useEffect(() => {
    const contract = new ethers.Contract(DPERP_ADDRESS, DPERP_ABI, provider)
    
    const handleLiquidation = (key, account, assetId, isLong, size, collateral, reserveAmount, realisedPnl, markPrice) => {
      const liquidationEvent = {
        key,
        account,
        assetId,
        isLong,
        size: formatEther(size),
        collateral: formatEther(collateral),
        markPrice: formatEther(markPrice),
        timestamp: Date.now()
      }
      
      setLiquidations(prev => [liquidationEvent, ...prev.slice(0, 99)]) // Keep last 100
    }
    
    contract.on('LiquidatePosition', handleLiquidation)
    
    return () => {
      contract.off('LiquidatePosition', handleLiquidation)
    }
  }, [])
  
  return (
    <div className="liquidation-monitor">
      <h3>Recent Liquidations</h3>
      {liquidations.map(liquidation => (
        <div key={liquidation.key} className="liquidation-item">
          <div>{liquidation.account.slice(0, 8)}...</div>
          <div>{liquidation.assetId}</div>
          <div className={liquidation.isLong ? 'long' : 'short'}>
            {liquidation.isLong ? 'LONG' : 'SHORT'}
          </div>
          <div>${liquidation.size}</div>
          <div>${liquidation.markPrice}</div>
        </div>
      ))}
    </div>
  )
}
```

## Risk Management

### Liquidation Prevention
- Monitor position health continuously
- Set appropriate stop-losses
- Maintain margin buffers
- Use position sizing rules

### Protocol Protection
- Maximum liquidation fees
- Liquidator incentives
- Insurance fund coverage
- Emergency pause mechanisms