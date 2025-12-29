# Leverage & Margin

DPerp uses an isolated margin system that allows traders to use leverage up to 50x while maintaining strict risk controls. Each position is isolated, preventing losses from affecting other positions.

## Leverage System

### Leverage Calculation
```solidity
function getPositionLeverage(address account, bytes32 assetId, bool isLong) 
    public view returns (uint256) {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    require(position.collateral > 0, "Invalid collateral");
    return (position.size * BASIS_POINTS) / position.collateral;
}
```

### Leverage Limits
| Asset Category | Max Leverage | Maintenance Margin |
|----------------|--------------|-------------------|
| Major Crypto   | 50x          | 2.5%             |
| Minor Crypto   | 25x          | 5%               |
| Commodities    | 20x          | 6%               |
| Equities       | 10x          | 12%              |
| Forex          | 100x         | 1.5%             |

### Leverage Validation
```solidity
function _validateLeverage(address account, bytes32 assetId, bool isLong) private view {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    require(position.size >= position.collateral, "Invalid leverage");
    
    uint256 leverage = (position.size * BASIS_POINTS) / position.collateral;
    require(leverage >= MIN_LEVERAGE, "Leverage too low");
    require(leverage <= maxLeverage[assetId], "Leverage too high");
}
```

## Margin Requirements

### Initial Margin
Required collateral to open a position:
```
Initial Margin = Position Size / Max Leverage
```

**Example:**
```
Position Size: $10,000
Max Leverage: 20x
Initial Margin: $10,000 / 20 = $500
```

### Maintenance Margin
Minimum collateral to keep position open:
```
Maintenance Margin = Position Size / Max Leverage + Liquidation Fee
```

### Margin Calculation
```javascript
const calculateMarginRequirements = (positionSize, maxLeverage, liquidationFee) => {
  const initialMargin = positionSize / maxLeverage
  const maintenanceMargin = initialMargin + liquidationFee
  
  return {
    initial: initialMargin,
    maintenance: maintenanceMargin,
    available: initialMargin - maintenanceMargin
  }
}
```

## Isolated Margin Model

### Position Isolation
Each position maintains separate collateral:
```solidity
struct Position {
    uint256 size;       // Position size in USD
    uint256 collateral; // Isolated collateral
    // ... other fields
}
```

### Cross-Margin vs Isolated
| Feature | Cross-Margin | Isolated Margin |
|---------|--------------|-----------------|
| Risk Isolation | No | Yes |
| Margin Efficiency | Higher | Lower |
| Liquidation Risk | Portfolio-wide | Position-specific |
| Implementation | Complex | Simple |

## Margin Calls and Liquidation

### Liquidation Conditions
Position liquidated when:
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
        return (1, marginFees); // Liquidation state 1
    }
    
    uint256 remainingCollateral = position.collateral;
    if (!hasProfit) {
        remainingCollateral = position.collateral - delta;
    }
    
    // Check if fees exceed remaining collateral
    if (remainingCollateral < marginFees + liquidationFeeUsd[assetId]) {
        return (1, marginFees); // Liquidation state 1
    }
    
    // Check leverage limit
    if (remainingCollateral * maxLeverage[assetId] < position.size * BASIS_POINTS) {
        return (2, marginFees); // Liquidation state 2 (partial)
    }
    
    return (0, marginFees); // Healthy position
}
```

### Liquidation Price Calculation
```javascript
const calculateLiquidationPrice = (position, asset) => {
  const maintenanceMargin = liquidationFeeUsd[asset]
  const fundingFees = getFundingFee(asset, position.size, position.entryFundingRate)
  const totalFees = maintenanceMargin + fundingFees
  
  const maxLoss = position.collateral - totalFees
  
  if (position.isLong) {
    // Long liquidation: entry price - (max loss / position size) * entry price
    return position.entryPrice * (1 - maxLoss / position.size)
  } else {
    // Short liquidation: entry price + (max loss / position size) * entry price
    return position.entryPrice * (1 + maxLoss / position.size)
  }
}
```

## Leverage Examples

### Opening Leveraged Position
```javascript
// Open 10x leveraged BTC long position
const collateral = parseEther("1000")  // $1000 DUSD
const leverage = 10
const positionSize = collateral * BigInt(leverage)  // $10,000 position

await dPerp.increasePosition(
  keccak256("BTC"),
  collateral,
  positionSize,
  true  // isLong
)
```

### Adjusting Leverage
```javascript
// Reduce leverage by adding collateral
const currentPosition = await positionManager.getPosition(account, keccak256("BTC"), true)
const currentLeverage = currentPosition.size / currentPosition.collateral
const targetLeverage = 5
const additionalCollateral = (currentPosition.size / targetLeverage) - currentPosition.collateral

await dPerp.increasePosition(
  keccak256("BTC"),
  additionalCollateral,
  0,  // No size increase
  true
)
```

### Maximum Position Size
```javascript
const calculateMaxPositionSize = (collateral, maxLeverage) => {
  return collateral * maxLeverage
}

// Example: $500 collateral with 20x leverage
const maxSize = calculateMaxPositionSize(500, 20)  // $10,000 max position
```

## Margin Monitoring

### Real-time Margin Tracking
```javascript
const useMarginMonitor = (account, asset, isLong) => {
  return useQuery({
    queryKey: ['margin', account, asset, isLong],
    queryFn: async () => {
      const position = await positionManager.getPosition(account, keccak256(asset), isLong)
      const [hasProfit, delta] = await dPerp.getDelta(
        keccak256(asset),
        position.size,
        position.entryPrice,
        isLong,
        position.lastIncreasedTime
      )
      
      const remainingCollateral = hasProfit ? 
        position.collateral + delta : 
        position.collateral - delta
      
      const currentLeverage = position.size / remainingCollateral
      const liquidationPrice = calculateLiquidationPrice(position, asset)
      const marginRatio = remainingCollateral / position.size
      
      return {
        collateral: position.collateral,
        remainingCollateral,
        leverage: currentLeverage,
        liquidationPrice,
        marginRatio,
        isAtRisk: marginRatio < 0.1  // 10% margin ratio threshold
      }
    },
    refetchInterval: 5000
  })
}
```

### Margin Alerts
```javascript
const MarginAlert = ({ marginData }) => {
  const getAlertLevel = (marginRatio) => {
    if (marginRatio < 0.05) return 'critical'
    if (marginRatio < 0.1) return 'warning'
    return 'safe'
  }
  
  const alertLevel = getAlertLevel(marginData.marginRatio)
  
  return (
    <div className={`margin-alert ${alertLevel}`}>
      {alertLevel === 'critical' && (
        <span>⚠️ Position at risk of liquidation</span>
      )}
      {alertLevel === 'warning' && (
        <span>⚡ Low margin - consider adding collateral</span>
      )}
      <div>Margin Ratio: {(marginData.marginRatio * 100).toFixed(2)}%</div>
      <div>Liquidation Price: ${marginData.liquidationPrice.toFixed(2)}</div>
    </div>
  )
}
```

## Risk Management

### Leverage Limits by Experience
```javascript
const getLeverageLimit = (userTier, asset) => {
  const baseLimits = {
    'BTC': 50,
    'ETH': 50,
    'GOLD': 20,
    'AAPL': 10
  }
  
  const tierMultipliers = {
    'beginner': 0.2,    // 20% of max leverage
    'intermediate': 0.5, // 50% of max leverage
    'advanced': 1.0     // Full leverage
  }
  
  return baseLimits[asset] * tierMultipliers[userTier]
}
```

### Position Size Limits
```solidity
mapping(bytes32 => uint256) public maxGlobalLongSizes;
mapping(bytes32 => uint256) public maxGlobalShortSizes;
mapping(bytes32 => uint256) public globalLongSizes;
mapping(bytes32 => uint256) public globalShortSizes;

function setMaxGlobalSize(bytes32 assetId, uint256 longSize, uint256 shortSize) 
    external onlyRole(ADMIN_ROLE) {
    maxGlobalLongSizes[assetId] = longSize;
    maxGlobalShortSizes[assetId] = shortSize;
}
```

## Fee Impact on Leverage

### Margin Fees
```solidity
function _collectMarginFees(
    bytes32 assetId,
    uint256 sizeDelta,
    uint256 size,
    uint256 entryFundingRate
) private view returns (uint256) {
    uint256 feeUsd = (sizeDelta * marginFeeBasisPoints[assetId]) / BASIS_POINTS;
    
    uint256 fundingFee = _getFundingFee(assetId, size, entryFundingRate);
    feeUsd += fundingFee;
    
    return feeUsd;
}
```

### Effective Leverage After Fees
```javascript
const calculateEffectiveLeverage = (collateral, positionSize, fees) => {
  const netCollateral = collateral - fees
  return positionSize / netCollateral
}

// Example: $1000 collateral, $10000 position, $10 fees
const effectiveLeverage = calculateEffectiveLeverage(1000, 10000, 10)  // 10.1x
```

## Best Practices

### Leverage Management
- Start with lower leverage (2-5x)
- Monitor margin ratio continuously
- Set stop-losses to prevent liquidation
- Add collateral during adverse moves

### Risk Controls
- Never use maximum leverage
- Maintain margin buffer (>20%)
- Diversify across multiple positions
- Use position sizing rules