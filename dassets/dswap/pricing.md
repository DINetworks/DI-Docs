# Pricing Model

DSwap uses an oracle-based pricing model with dynamic fee structures and advanced risk management to ensure protocol stability while providing accurate, real-time pricing for synthetic assets.

## Oracle-Based Pricing

### Price Sources
DSwap integrates multiple oracle providers for robust price feeds:

**Primary Sources:**
- **Chainlink**: Decentralized oracle network with proven reliability
- **Pyth Network**: High-frequency price updates for real-time trading

**Price Feed Structure:**
```solidity
struct PriceData {
    int256 price;      // Asset price in USD (8 decimals)
    uint256 timestamp; // Last update timestamp
    bool valid;        // Price validity flag
}
```

### Price Validation
All prices undergo validation before use:
```solidity
function getValidatedPrice(bytes32 assetId) external view returns (uint256) {
    (int256 price, uint256 timestamp, bool valid) = oracle.getPrice(assetId);
    
    require(valid, "Invalid price feed");
    require(price > 0, "Price must be positive");
    require(timestamp > block.timestamp - MAX_PRICE_AGE, "Price too stale");
    
    return uint256(price);
}
```

## Pricing Mechanics

### Mint Pricing (DUSD → Synthetic)
When minting synthetic assets from DUSD:
```
Synthetic Amount = (DUSD Amount - Flat Fee) / Oracle Price
Fee = 0.3% flat rate (low risk operation)
```

**Example:**
```
DUSD Input: 1000
Mint Fee (0.3%): 3 DUSD
Net Amount: 997 DUSD
AAPL Price: $150
xAAPL Minted: 997 / 150 = 6.64 xAAPL
```

### Burn Pricing (Synthetic → DUSD)
When burning synthetic assets for DUSD with dynamic fees:
```
DUSD Amount = (Synthetic Amount × Oracle Price) - Dynamic Fee
Dynamic Fee = 0.3% to 2% based on stress ratio
```

**Example:**
```
xAAPL Input: 6.64 shares
AAPL Price: $160
Gross Value: 6.64 × 160 = 1,062 DUSD
Stress Ratio: 0.7 (example)
Dynamic Fee (1.5%): 16 DUSD
Net DUSD: 1,046 DUSD
```

### Swap Pricing (Synthetic ↔ Synthetic)
Direct swaps between synthetic assets:
```
Output Amount = (Input Amount × Input Price / Output Price) - Flat Fee
Flat Fee = 0.3% (no DUSD supply impact)
```

**Example:**
```
Input: 6.6 xAAPL at $150 = $990
Swap Fee (0.3%): $2.97
Net Value: $987.03
Output Price: $200 (xTSLA)
xTSLA Received: $987.03 / $200 = 4.935 xTSLA
```

## Dynamic Fee System

### Risk-Based Fee Structure
Different operations pose different risks to protocol solvency:

| Operation | Risk Level | Fee Type | Rationale |
|-----------|------------|----------|-----------|
| **Mint Synthetic** | Low | 0.3% flat | Converts DUSD to position |
| **Swap Synthetic** | Low | 0.3% flat | Position reallocation only |
| **Burn Synthetic** | High | 0.3-2% dynamic | Creates new DUSD supply |

### Dynamic Burn Fee Calculation
```solidity
function getBurnFeeBps() public view returns (uint256) {
    uint256 totalSupply = _dUSD().totalSupply();
    uint256 totalSyntheticValue = getTotalSyntheticValue();
    
    // Calculate stress ratio (how close to backing limit)
    uint256 stress = (totalSupply * 1e18) / totalSyntheticValue;
    
    // Linear fee curve from MIN_FEE_BPS to MAX_FEE_BPS
    uint256 fee = MIN_FEE_BPS + 
        (stress * (MAX_FEE_BPS - MIN_FEE_BPS)) / 1e18;
    
    return Math.min(fee, MAX_FEE_BPS);
}
```

### Fee Schedule
| Stress Ratio | Burn Fee | Protocol State |
|--------------|----------|----------------|
| 0.1 | 0.47% | Very healthy |
| 0.3 | 0.81% | Healthy |
| 0.6 | 1.32% | Moderate stress |
| 0.9 | 1.83% | High stress |
| 1.0 | BLOCKED | Critical (prevented by hard cap) |

### Stress Ratio Calculation
```javascript
const calculateStressRatio = async () => {
  const totalDUSDSupply = await dusd.totalSupply()
  const totalSyntheticValue = await getTotalSyntheticValue()
  const maxBorrowable = await dusdProvider.getMaxBorrowableDUSD()
  
  return {
    currentStress: totalDUSDSupply / totalSyntheticValue,
    maxStress: maxBorrowable / totalSyntheticValue,
    utilizationRate: totalDUSDSupply / maxBorrowable
  }
}
```

## Protocol Solvency Protection

### Hard Invariant Enforcement
```solidity
// Mathematical impossibility of insolvency
require(
    _dUSD().totalSupply() + dUSDAmount <= getMaxBorrowableDUSD(),
    "INSUFFICIENT_BACKING"
);
```

### Dynamic Backing Calculation
```solidity
function getMaxBorrowableDUSD() external view returns (uint256) {
    uint256 totalCollateralValue = getTotalCollateralValue();
    uint256 maintenanceRatio = getMaintenanceRatio(); // e.g., 75%
    
    return (totalCollateralValue * maintenanceRatio) / 10000;
}
```

### Real-Time Monitoring
```javascript
const monitorProtocolHealth = async () => {
  const health = await getProtocolHealth()
  
  return {
    backingRatio: health.maxBorrowable / health.totalSupply,
    stressLevel: getStressLevel(health.stressRatio),
    burnFeeRate: await getBurnFeeBps() / 100, // Convert to percentage
    emergencyMode: health.backingRatio < 1.1 // 110% minimum
  }
}
```

## Settlement Lock Impact on Pricing

### MEV Protection Mechanism
```solidity
mapping(address => uint256) public lastSwapTime;
uint256 public constant SETTLEMENT_LOCK = 1 minutes;

modifier settlementLock() {
    require(
        block.timestamp >= lastSwapTime[msg.sender] + SETTLEMENT_LOCK,
        "Settlement period active"
    );
    _;
}
```

### Price Stability Benefits
- **Prevents Sandwich Attacks**: Cannot immediately reverse trades
- **Reduces Arbitrage Pressure**: 1-minute cooldown dampens rapid exploitation
- **Oracle Price Stabilization**: Allows prices to settle between operations

### User Experience Considerations
```javascript
// Price quote with settlement awareness
const getPriceQuoteWithLock = async (user, operation) => {
  const quote = await getBasicQuote(operation)
  const lockStatus = await getSettlementLockStatus(user)
  
  return {
    ...quote,
    settlementLock: {
      isActive: lockStatus.isLocked,
      expiresAt: lockStatus.expiresAt,
      canExecute: !lockStatus.isLocked
    }
  }
}
```

## Price Feed Management

### Oracle Integration
```solidity
interface IOracleModule {
    function getPrice(bytes32 assetId) external view returns (
        int256 price,
        uint256 timestamp,
        bool valid
    );
    
    function getPriceData(bytes32 assetId) external view returns (
        int256 price,
        uint256 timestamp,
        bool valid,
        uint256 confidence
    );
}
```

### Multi-Source Aggregation
```solidity
function getAggregatedPrice(bytes32 assetId) external view returns (uint256) {
    uint256[] memory prices = new uint256[](sources.length);
    uint256 validCount = 0;
    
    for (uint i = 0; i < sources.length; i++) {
        (int256 price,, bool valid) = sources[i].getPrice(assetId);
        if (valid && price > 0) {
            prices[validCount] = uint256(price);
            validCount++;
        }
    }
    
    require(validCount >= minSources, "Insufficient valid sources");
    return calculateMedian(prices, validCount);
}
```

## Price Precision and Calculations

### Decimal Handling
All prices use 8 decimal precision (matching Chainlink standard):
```solidity
uint256 constant PRICE_DECIMALS = 8;
uint256 constant PRICE_PRECISION = 10**PRICE_DECIMALS;

// Convert price to asset amount
function priceToAmount(uint256 price, uint256 usdValue) pure returns (uint256) {
    return (usdValue * PRICE_PRECISION) / price;
}
```

### Rounding Protection
```solidity
function safeDivision(uint256 a, uint256 b) pure returns (uint256) {
    return (a + b / 2) / b; // Round to nearest
}
```

## Real-Time Price Updates

### Price Freshness Requirements
Maximum allowed price age varies by asset type:
- **Equities**: 1 hour (during market hours), 24 hours (market closed)
- **Commodities**: 30 minutes
- **Crypto**: 5 minutes
- **Indices**: 15 minutes

### Update Mechanisms
```solidity
mapping(bytes32 => uint256) public maxPriceAge;

function isPriceValid(bytes32 assetId, uint256 timestamp) view returns (bool) {
    uint256 maxAge = maxPriceAge[assetId];
    return block.timestamp - timestamp <= maxAge;
}
```

## Market Hours Handling

### Trading Hours Integration
```solidity
struct MarketHours {
    uint256 openTime;   // Market open (UTC)
    uint256 closeTime;  // Market close (UTC)
    bool isOpen;        // Current status
    uint256 lastPrice;  // Price at market close
}

function isMarketOpen(bytes32 assetId) view returns (bool) {
    MarketHours memory hours = marketSchedule[assetId];
    uint256 currentTime = block.timestamp % 86400; // Seconds in day
    
    return hours.isOpen && 
           currentTime >= hours.openTime && 
           currentTime <= hours.closeTime;
}
```

### After-Hours Pricing
- **Crypto Assets**: 24/7 pricing available
- **Equity Assets**: Last market price used when closed
- **Commodity Assets**: Futures pricing may continue
- **Index Assets**: Calculated from component prices

## Price Deviation Protection

### Circuit Breakers
```solidity
uint256 constant MAX_PRICE_DEVIATION = 1000; // 10%

function checkPriceDeviation(bytes32 assetId, uint256 newPrice) view {
    uint256 lastPrice = historicalPrices[assetId];
    uint256 deviation = abs(newPrice - lastPrice) * 10000 / lastPrice;
    
    require(deviation <= MAX_PRICE_DEVIATION, "Price deviation too high");
}
```

### Volatility Adjustments
```solidity
function getVolatilityAdjustedFee(bytes32 assetId) view returns (uint256) {
    uint256 baseFee = getBaseFee(assetId);
    uint256 volatility = calculateVolatility(assetId);
    
    if (volatility > HIGH_VOLATILITY_THRESHOLD) {
        return baseFee * 150 / 100; // +50% during high volatility
    }
    
    return baseFee;
}
```

## Integration Examples

### Price Display Component
```javascript
const PriceDisplay = ({ assetId }) => {
  const { data: priceData } = useQuery(
    ['price', assetId],
    () => oracle.getPriceData(assetId),
    { 
      refetchInterval: 30000,
      staleTime: 25000
    }
  )
  
  const price = priceData ? Number(priceData.price) / 1e8 : 0
  const isStale = Date.now() - priceData?.timestamp * 1000 > 300000
  const confidence = priceData?.confidence || 0
  
  return (
    <div className={`price ${isStale ? 'stale' : 'fresh'}`}>
      <div className="price-value">${price.toFixed(2)}</div>
      <div className="price-meta">
        {isStale && <span className="stale-indicator">⚠️ Stale</span>}
        <span className="confidence">Confidence: {confidence}%</span>
      </div>
    </div>
  )
}
```

### Dynamic Fee Calculator
```javascript
const useDynamicFee = (operation, assetId, amount) => {
  return useQuery({
    queryKey: ['dynamicFee', operation, assetId, amount],
    queryFn: async () => {
      if (operation === 'burn') {
        const burnFeeBps = await swapRouter.getBurnFeeBps()
        const feeAmount = (amount * burnFeeBps) / 10000
        return {
          feeRate: burnFeeBps / 100, // Convert to percentage
          feeAmount,
          isDynamic: true
        }
      } else {
        // Mint or swap - flat 0.3% fee
        const feeAmount = amount * 0.003
        return {
          feeRate: 0.3,
          feeAmount,
          isDynamic: false
        }
      }
    },
    enabled: !!operation && !!assetId && !!amount
  })
}
```

### Protocol Health Monitor
```javascript
const ProtocolHealthWidget = () => {
  const { data: health } = useQuery(
    ['protocolHealth'],
    getProtocolHealth,
    { refetchInterval: 30000 }
  )
  
  const getHealthColor = (ratio) => {
    if (ratio > 1.5) return 'green'
    if (ratio > 1.2) return 'yellow'
    return 'red'
  }
  
  return (
    <div className="protocol-health">
      <div className="health-metric">
        <span>Backing Ratio:</span>
        <span className={getHealthColor(health?.backingRatio)}>
          {(health?.backingRatio * 100).toFixed(1)}%
        </span>
      </div>
      <div className="health-metric">
        <span>Current Burn Fee:</span>
        <span>{health?.burnFeeRate?.toFixed(2)}%</span>
      </div>
      <div className="health-metric">
        <span>Stress Level:</span>
        <span className={health?.stressLevel}>{health?.stressLevel}</span>
      </div>
    </div>
  )
}
```

## Risk Management

### Oracle Failure Handling
```solidity
function getEmergencyPrice(bytes32 assetId) view returns (uint256) {
    // Use last known good price with time decay
    uint256 lastPrice = emergencyPrices[assetId];
    uint256 timeSinceUpdate = block.timestamp - lastPriceUpdate[assetId];
    
    // Gradually reduce confidence in stale prices
    if (timeSinceUpdate > EMERGENCY_PRICE_DECAY) {
        revert("No valid price available");
    }
    
    return lastPrice;
}
```

### Price Manipulation Protection
Multiple validation layers prevent manipulation:
1. **Multi-source aggregation**: Median of multiple oracles
2. **Deviation limits**: Maximum allowed price changes
3. **Time-weighted averages**: Smooth out temporary spikes
4. **Circuit breakers**: Halt trading during anomalies
5. **Settlement locks**: Prevent rapid arbitrage cycles

## Performance Optimization

### Gas-Efficient Price Queries
```solidity
// Batch price queries for multiple assets
function getBatchPrices(bytes32[] calldata assetIds) 
    external view returns (uint256[] memory prices) {
    prices = new uint256[](assetIds.length);
    
    for (uint i = 0; i < assetIds.length; i++) {
        (int256 price,, bool valid) = oracle.getPrice(assetIds[i]);
        require(valid, "Invalid price");
        prices[i] = uint256(price);
    }
}
```

### Caching Strategies
```javascript
// Price caching with smart invalidation
const useCachedPrices = (assetIds) => {
  return useQuery({
    queryKey: ['batchPrices', assetIds],
    queryFn: () => getBatchPrices(assetIds),
    staleTime: 30000, // 30 seconds
    cacheTime: 300000, // 5 minutes
    refetchInterval: 60000 // 1 minute
  })
}
```