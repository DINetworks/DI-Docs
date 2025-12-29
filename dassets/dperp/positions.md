# Position Management

Position management in DPerp handles the creation, modification, and closure of leveraged trading positions. The system tracks position state, calculates P&L, and manages collateral requirements.

## Position Structure

### Core Position Data
```solidity
struct Position {
    uint256 size;              // Position size in USD
    uint256 collateral;        // Collateral deposited
    uint256 entryPrice;        // Average entry price
    uint256 entryFundingRate;  // Funding rate at entry
    uint256 reserveAmount;     // Reserved liquidity tokens
    int256 realisedPnl;        // Cumulative realized P&L
    uint256 lastIncreasedTime; // Last position increase
    bool isLong;               // Long or short position
}
```

### Position Identification
Positions are uniquely identified by:
```solidity
function getPositionKey(address account, bytes32 assetId, bool isLong) 
    returns (bytes32) {
    return keccak256(abi.encodePacked(account, assetId, isLong));
}
```

## Opening Positions

### Increase Position
```javascript
// Open or increase existing position
await dPerp.increasePosition(
  keccak256("BTC"),    // Asset ID
  parseEther("500"),   // Collateral amount (DUSD)
  parseEther("5000"),  // Position size (USD)
  true                 // isLong
)
```

### Position Creation Logic
```solidity
function increasePosition(
    bytes32 assetId,
    uint256 amountIn,
    uint256 sizeDelta,
    bool isLong
) external {
    // Burn DUSD collateral
    dUSD.burn(msg.sender, amountIn);
    
    // Get current price
    uint256 price = isLong ? 
        priceFeed.getPrice(assetId, true) : 
        priceFeed.getPrice(assetId, false);
    
    // Calculate fees
    uint256 fee = _collectMarginFees(assetId, sizeDelta, position.size, position.entryFundingRate);
    uint256 collateralDelta = amountIn > fee ? amountIn - fee : 0;
    
    // Update position
    positionManager.increasePosition(msg.sender, assetId, sizeDelta, collateralDelta, isLong, price);
}
```

## Modifying Positions

### Adding Collateral
```javascript
// Add more collateral to existing position
await dPerp.increasePosition(
  keccak256("BTC"),
  parseEther("200"),  // Additional collateral
  0,                  // No size increase
  true
)
```

### Increasing Size
```javascript
// Increase position size with same collateral ratio
await dPerp.increasePosition(
  keccak256("BTC"),
  parseEther("100"),   // Additional collateral
  parseEther("1000"),  // Additional size
  true
)
```

### Entry Price Calculation
When increasing positions, entry price is recalculated:
```solidity
if (position.size == 0) {
    position.entryPrice = price;
} else {
    position.entryPrice = ((position.size * position.entryPrice) + (sizeDelta * price)) 
                         / (position.size + sizeDelta);
}
```

## Closing Positions

### Partial Close
```javascript
// Close 50% of position
const position = await positionManager.getPosition(account, keccak256("BTC"), true)
const halfSize = position.size / 2n

await dPerp.decreasePosition(
  keccak256("BTC"),
  0,           // No collateral withdrawal
  halfSize,    // Close half the size
  true
)
```

### Full Close
```javascript
// Close entire position
await dPerp.decreasePosition(
  keccak256("BTC"),
  position.collateral, // Withdraw all collateral
  position.size,       // Close entire size
  true
)
```

### Close Logic
```solidity
function decreasePosition(
    bytes32 assetId,
    uint256 collateralDelta,
    uint256 sizeDelta,
    bool isLong
) external returns (uint256) {
    // Calculate P&L
    int256 pnl = isLong 
        ? int256((sizeDelta * price) / 1e18) - int256((sizeDelta * position.entryPrice) / 1e18)
        : int256((sizeDelta * position.entryPrice) / 1e18) - int256((sizeDelta * price) / 1e18);
    
    // Update position
    position.realisedPnl += pnl;
    position.size -= sizeDelta;
    
    // Calculate payout
    uint256 usdOut = position.size == 0 ? position.collateral : collateralDelta;
    
    return usdOut;
}
```

## P&L Calculations

### Unrealized P&L
```solidity
function getDelta(
    bytes32 assetId,
    uint256 size,
    uint256 entryPrice,
    bool isLong,
    uint256 lastIncreasedTime
) public view returns (bool hasProfit, uint256 delta) {
    uint256 price = isLong ? 
        priceFeed.getPrice(assetId, false) : 
        priceFeed.getPrice(assetId, true);
    
    uint256 priceDelta = entryPrice > price ? entryPrice - price : price - entryPrice;
    delta = (size * priceDelta) / entryPrice;
    
    hasProfit = isLong ? price > entryPrice : entryPrice > price;
    
    // Apply minimum profit time
    uint256 minBps = block.timestamp > lastIncreasedTime + MIN_PROFIT_TIME ? 
        0 : minProfitBasisPoints[assetId];
    if (hasProfit && delta * BASIS_POINTS <= size * minBps) {
        delta = 0;
    }
}
```

### Realized P&L Tracking
```javascript
const getPositionPnL = async (account, asset, isLong) => {
  const position = await positionManager.getPosition(account, keccak256(asset), isLong)
  const [hasProfit, unrealizedPnL] = await dPerp.getDelta(
    keccak256(asset),
    position.size,
    position.entryPrice,
    isLong,
    position.lastIncreasedTime
  )
  
  return {
    realized: position.realisedPnl,
    unrealized: hasProfit ? unrealizedPnL : -unrealizedPnL,
    total: position.realisedPnl + (hasProfit ? unrealizedPnL : -unrealizedPnL)
  }
}
```

## Position Monitoring

### Position Health
```javascript
const getPositionHealth = async (account, asset, isLong) => {
  const position = await positionManager.getPosition(account, keccak256(asset), isLong)
  const [hasProfit, delta] = await dPerp.getDelta(
    keccak256(asset),
    position.size,
    position.entryPrice,
    isLong,
    position.lastIncreasedTime
  )
  
  // Calculate remaining collateral after P&L
  const remainingCollateral = hasProfit ? 
    position.collateral + delta : 
    position.collateral - delta
  
  // Calculate current leverage
  const leverage = position.size / remainingCollateral
  
  // Get liquidation price
  const liquidationPrice = getLiquidationPrice(position, asset)
  
  return {
    leverage,
    remainingCollateral,
    liquidationPrice,
    healthFactor: remainingCollateral / (position.size / maxLeverage[asset])
  }
}
```

### Liquidation Price Calculation
```javascript
const getLiquidationPrice = (position, asset) => {
  const maintenanceMargin = liquidationFeeUsd[asset]
  const maxLoss = position.collateral - maintenanceMargin
  
  if (position.isLong) {
    return position.entryPrice * (1 - maxLoss / position.size)
  } else {
    return position.entryPrice * (1 + maxLoss / position.size)
  }
}
```

## Position Limits

### Size Validation
```solidity
function _validatePosition(address account, bytes32 assetId, bool isLong) private view {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    require(position.size > 0, "Invalid position size");
    require(position.collateral > 0, "Invalid collateral");
    
    uint256 liquidationFee = liquidationFeeUsd[assetId];
    require(position.collateral > liquidationFee, "Collateral below liquidation fee");
}
```

### Global Limits
```solidity
// Check global exposure limits
if (isLong) {
    require(globalLongSizes[assetId] + sizeDelta <= maxGlobalLongSizes[assetId], "Max longs exceeded");
    globalLongSizes[assetId] += sizeDelta;
} else {
    require(globalShortSizes[assetId] + sizeDelta <= maxGlobalShortSizes[assetId], "Max shorts exceeded");
    globalShortSizes[assetId] += sizeDelta;
}
```

## Integration Examples

### Position Management Hook
```javascript
const usePositionManager = (account) => {
  const [positions, setPositions] = useState([])
  
  const openPosition = async (asset, collateral, size, isLong) => {
    await dPerp.increasePosition(
      keccak256(asset),
      parseEther(collateral),
      parseEther(size),
      isLong
    )
    await refreshPositions()
  }
  
  const closePosition = async (asset, isLong) => {
    const position = await positionManager.getPosition(account, keccak256(asset), isLong)
    await dPerp.decreasePosition(
      keccak256(asset),
      position.collateral,
      position.size,
      isLong
    )
    await refreshPositions()
  }
  
  const refreshPositions = async () => {
    // Fetch all user positions
    const userPositions = await getUserPositions(account)
    setPositions(userPositions)
  }
  
  return { positions, openPosition, closePosition, refreshPositions }
}
```

### Position Display Component
```javascript
const PositionCard = ({ position, asset }) => {
  const { data: pnl } = useQuery(
    ['position-pnl', position.account, asset, position.isLong],
    () => getPositionPnL(position.account, asset, position.isLong),
    { refetchInterval: 5000 }
  )
  
  const leverage = position.size / position.collateral
  const isProfit = pnl?.total > 0
  
  return (
    <div className="position-card">
      <div className="position-header">
        <span>{asset}</span>
        <span className={position.isLong ? 'long' : 'short'}>
          {position.isLong ? 'LONG' : 'SHORT'}
        </span>
      </div>
      
      <div className="position-details">
        <div>Size: ${formatNumber(position.size)}</div>
        <div>Collateral: ${formatNumber(position.collateral)}</div>
        <div>Leverage: {leverage.toFixed(2)}x</div>
        <div className={isProfit ? 'profit' : 'loss'}>
          P&L: ${formatNumber(pnl?.total || 0)}
        </div>
      </div>
    </div>
  )
}
```

## Risk Management

### Position Validation
- Minimum collateral requirements
- Maximum leverage enforcement
- Position size limits
- Global exposure caps

### Monitoring Systems
- Real-time P&L tracking
- Liquidation risk alerts
- Funding fee calculations
- Position health scoring