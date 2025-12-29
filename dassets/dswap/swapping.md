# Asset Swapping

Asset swapping enables direct exchange between synthetic assets with oracle-based pricing, dynamic fee structures, and advanced risk management to maintain protocol stability.

## Swap Mechanisms

### 1. Direct Synthetic Swaps
Swap between synthetic assets without DUSD intermediary:
```javascript
// Swap xAAPL to xTSLA directly
await swapRouter.swapSynthetic(
  keccak256("xAAPL"),  // From asset
  keccak256("xTSLA"),  // To asset  
  parseEther("6.6"),   // 6.6 xAAPL shares
  parseEther("4.9")    // Min 4.9 xTSLA shares
)
```

### 2. DUSD-Mediated Swaps
For complex swaps or when direct pairs aren't optimal:
```javascript
// Two-step swap via DUSD (if needed)
// Step 1: Burn synthetic to DUSD (dynamic fee applies)
await swapRouter.burnSynthetic(
  keccak256("xAAPL"),
  parseEther("6.6"),
  parseEther("980") // Min DUSD after dynamic burn fee
)

// Wait for settlement lock (1 minute)
// Step 2: Mint new synthetic from DUSD (flat fee)
await swapRouter.mintSynthetic(
  keccak256("xGOLD"),
  parseEther("980"),
  parseEther("0.48") // Min gold units
)
```

## Swap Calculation

### Direct Swap Pricing
Direct swaps use oracle prices with flat fees:
```
Source Asset: 6.6 xAAPL at $150 = $990
Swap Fee (0.3%): $990 × 0.003 = $2.97
Net Value: $987.03
Destination Price: $200 (xTSLA)
Output Amount: $987.03 ÷ $200 = 4.935 xTSLA
```

### Implementation
```solidity
function swapSynthetic(
    bytes32 fromAssetId,
    bytes32 toAssetId,
    uint256 amount
) external settlementLock {
    // Get oracle prices
    uint256 fromPrice = oracle.getPrice(fromAssetId);
    uint256 toPrice = oracle.getPrice(toAssetId);
    
    // Calculate USD value
    uint256 usdValue = (amount * fromPrice) / 1e18;
    
    // Apply flat swap fee (0.3%)
    uint256 fee = (usdValue * swapFeeBps) / 10000;
    uint256 netValue = usdValue - fee;
    
    // Calculate output amount
    uint256 outputAmount = (netValue * 1e18) / toPrice;
    
    // Update positions (virtual assets)
    userPositions[msg.sender][fromAssetId] -= amount;
    userPositions[msg.sender][toAssetId] += outputAmount;
    
    // Apply settlement lock
    lastSwapTime[msg.sender] = block.timestamp;
    
    emit SyntheticSwapped(msg.sender, fromAssetId, toAssetId, amount, outputAmount, fee);
}
```

## Supported Swap Pairs

### Popular Direct Pairs
| From | To | Description | Fee Type |
|------|----|-----------| ---------|
| xAAPL | xTSLA | Tech stocks | 0.3% flat |
| xGOLD | xSILVER | Precious metals | 0.3% flat |
| xSP500 | xNASDAQ | Index rotation | 0.3% flat |
| xBTC | xETH | Crypto majors | 0.3% flat |

### Cross-Category Swaps
All synthetic assets can swap to any other synthetic asset:
- **Equity to Commodity**: xAAPL ↔ xGOLD
- **Commodity to Index**: xGOLD ↔ xSP500
- **Index to Crypto**: xSP500 ↔ xBTC
- **Any to Any**: Universal compatibility

## Fee Structure

### Flat Swap Fees
Direct synthetic swaps use flat 0.3% fees:
```javascript
const swapFee = {
  rate: 0.003, // 0.3%
  rationale: "Low risk - no DUSD supply impact",
  application: "All direct synthetic swaps"
}
```

### Fee Comparison by Operation
| Operation | Risk Level | Fee Structure | Reason |
|-----------|------------|---------------|---------|
| **Mint Synthetic** | Low | 0.3% flat | Converts DUSD to position |
| **Swap Synthetic** | Low | 0.3% flat | Position-to-position only |
| **Burn Synthetic** | High | 0.3-2% dynamic | Creates new DUSD supply |

**Key Insight**: Swaps only change position allocations without affecting DUSD supply, hence the flat fee structure.

## Settlement Lock System

### Lock Mechanics
Every swap triggers a 1-minute settlement lock:
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

### Lock Applications
**Blocked During Lock**:
- Additional swaps
- Burning synthetics to DUSD
- Transferring synthetic positions
- Any position-changing operations

**Allowed During Lock**:
- Viewing positions and balances
- Checking prices and quotes
- Reading contract state
- Planning next moves

### MEV Protection
Settlement locks prevent:
- **Sandwich Attacks**: Can't immediately reverse trades
- **Flash Arbitrage**: Eliminates rapid price exploitation
- **Oracle Front-running**: Reduces timing-based advantages

## Swap Examples

### Example 1: Tech Stock Rotation
```javascript
// Rotate from Apple to Tesla
const fromAmount = parseEther("10") // 10 xAAPL shares
const fromPrice = 150 // $150 per AAPL share
const toPrice = 200   // $200 per TSLA share

// Calculate expected output
const usdValue = 10 * 150 // $1,500
const fee = usdValue * 0.003 // $4.50
const netValue = usdValue - fee // $1,495.50
const expectedTSLA = netValue / 200 // 7.4775 xTSLA

await swapRouter.swapSynthetic(
  keccak256("xAAPL"),
  keccak256("xTSLA"),
  parseEther("10"),
  parseEther("7.4") // 1% slippage tolerance
)

// Settlement lock: 1 minute before next operation
```

### Example 2: Diversification Swap
```javascript
// Diversify from single stock to index
await swapRouter.swapSynthetic(
  keccak256("xAAPL"),
  keccak256("xSP500"),
  parseEther("20"), // 20 AAPL shares
  parseEther("7.4") // Expected S&P 500 units
)

// Note: Must wait 1 minute before additional swaps
```

### Example 3: Cross-Asset Class Swap
```javascript
// Move from equities to commodities
await swapRouter.swapSynthetic(
  keccak256("xTSLA"),
  keccak256("xGOLD"),
  parseEther("5"),    // 5 TSLA shares
  parseEther("0.49")  // Expected gold units
)
```

## Advanced Swapping

### Batch Swap Strategy
Since each swap triggers settlement lock, plan efficiently:
```javascript
// Inefficient: Multiple swaps with locks
await swap("xAAPL", "xTSLA", amount1) // Lock 1 minute
// Wait 1 minute...
await swap("xMSFT", "xGOLD", amount2) // Lock 1 minute
// Wait 1 minute...

// Efficient: Plan single optimal swap
// Calculate best final allocation and execute once
await swap("xAAPL", "xSP500", totalAmount) // Single lock
```

### Portfolio Rebalancing
```javascript
const rebalancePortfolio = async (targetAllocations) => {
  // Calculate current positions
  const currentPositions = await getUserPositions(userAddress)
  
  // Determine optimal swap to reach target
  const optimalSwap = calculateOptimalRebalance(currentPositions, targetAllocations)
  
  // Execute single swap to minimize settlement locks
  if (optimalSwap.required) {
    await swapRouter.swapSynthetic(
      optimalSwap.fromAsset,
      optimalSwap.toAsset,
      optimalSwap.amount,
      optimalSwap.minOutput
    )
  }
}
```

## Integration Patterns

### React Hook for Swapping
```javascript
const useSwapSynthetic = () => {
  const [isSwapping, setIsSwapping] = useState(false)
  const [settlementLock, setSettlementLock] = useState(null)
  
  const swap = async (fromAsset, toAsset, amount, minOutput) => {
    setIsSwapping(true)
    try {
      // Check settlement lock
      const lastSwap = await swapRouter.lastSwapTime(userAddress)
      const lockExpiry = lastSwap + 60
      
      if (Date.now() / 1000 < lockExpiry) {
        throw new Error("Settlement lock still active")
      }
      
      // Execute swap
      const tx = await swapRouter.swapSynthetic(fromAsset, toAsset, amount, minOutput)
      await tx.wait()
      
      // Set new settlement lock
      setSettlementLock(Date.now() + 60000)
      
      return tx
    } finally {
      setIsSwapping(false)
    }
  }
  
  return { swap, isSwapping, settlementLock }
}
```

### Swap Quote Calculator
```javascript
const useSwapQuote = (fromAsset, toAsset, amount) => {
  return useQuery({
    queryKey: ['swapQuote', fromAsset, toAsset, amount],
    queryFn: async () => {
      if (!fromAsset || !toAsset || !amount) return null
      
      const [fromPrice, toPrice] = await Promise.all([
        oracle.getPrice(fromAsset),
        oracle.getPrice(toAsset)
      ])
      
      const usdValue = amount * fromPrice
      const fee = usdValue * 0.003 // 0.3% flat fee
      const netValue = usdValue - fee
      const outputAmount = netValue / toPrice
      
      return {
        inputAmount: amount,
        outputAmount,
        fee,
        exchangeRate: fromPrice / toPrice,
        priceImpact: 0, // Always 0 for oracle-based pricing
        settlementLock: 60 // 1 minute lock after swap
      }
    },
    enabled: !!fromAsset && !!toAsset && !!amount
  })
}
```

## Risk Management

### Oracle Risk
**Price Feed Dependencies**:
- All swaps rely on accurate oracle prices
- Stale prices can cause unfavorable swaps
- Multiple oracle sources provide redundancy

**Mitigation Strategies**:
```javascript
// Validate oracle freshness
const validateOraclePrice = async (assetId) => {
  const priceData = await oracle.getPriceData(assetId)
  const maxAge = 3600 // 1 hour
  
  if (Date.now() / 1000 - priceData.timestamp > maxAge) {
    throw new Error("Oracle price too stale")
  }
  
  return priceData.price
}
```

### Settlement Lock Risk
**Liquidity Constraints**:
- Cannot immediately reverse positions
- Must wait 1 minute between operations
- Plan swap sequences carefully

**Risk Mitigation**:
```javascript
// Check settlement status before swapping
const checkSettlementStatus = async (userAddress) => {
  const lastSwap = await swapRouter.lastSwapTime(userAddress)
  const lockExpiry = lastSwap + 60
  const isLocked = Date.now() / 1000 < lockExpiry
  
  return {
    isLocked,
    expiresAt: lockExpiry,
    remainingTime: isLocked ? lockExpiry - (Date.now() / 1000) : 0
  }
}
```

## Monitoring and Analytics

### Swap Metrics
```javascript
const getSwapAnalytics = async () => {
  const events = await getSwapEvents()
  
  return {
    totalVolume: calculateTotalVolume(events),
    popularPairs: getMostTradedPairs(events),
    averageSwapSize: getAverageSwapSize(events),
    feeRevenue: calculateFeeRevenue(events),
    settlementLockUtilization: getSettlementLockStats(events)
  }
}
```

### User Swap History
```javascript
const SwapHistory = ({ userAddress }) => {
  const { data: swaps } = useQuery(
    ['swapHistory', userAddress],
    () => getUserSwapHistory(userAddress)
  )
  
  return (
    <div className="swap-history">
      {swaps?.map(swap => (
        <div key={swap.txHash} className="swap-item">
          <div>{swap.fromAsset} → {swap.toAsset}</div>
          <div>{swap.amountIn} → {swap.amountOut}</div>
          <div>Fee: {swap.fee}</div>
          <div>{formatTime(swap.timestamp)}</div>
        </div>
      ))}
    </div>
  )
}
```

## Best Practices

### Efficient Swapping
- **Plan Sequences**: Minimize settlement locks through strategic planning
- **Batch Thinking**: Consider final target allocation vs multiple small swaps
- **Timing Awareness**: Account for 1-minute locks in user experience
- **Oracle Monitoring**: Verify price freshness before large swaps

### Risk Management
- **Slippage Protection**: Always set reasonable minimum output amounts
- **Settlement Planning**: Don't assume immediate liquidity after swaps
- **Oracle Validation**: Check price feed status before swapping
- **Position Sizing**: Consider swap fees in position sizing decisions

### User Experience
- **Lock Indicators**: Show settlement lock status clearly in UI
- **Fee Transparency**: Display exact fees before swap execution
- **Price Updates**: Refresh quotes regularly for accuracy
- **Error Handling**: Gracefully handle settlement lock errors