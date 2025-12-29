# Asset Minting

Asset minting allows users to create synthetic assets by depositing DUSD as collateral. The minting process uses oracle prices with advanced risk management to ensure protocol stability.

## Minting Process

### 1. Collateral Validation
Before minting, the system validates backing:
```javascript
// Check protocol solvency before minting
const maxBorrowable = await dusdProvider.getMaxBorrowableDUSD()
const currentSupply = await dusd.totalSupply()
const canMint = currentSupply + mintAmount <= maxBorrowable

if (!canMint) {
  throw new Error("Insufficient protocol backing")
}
```

### 2. Virtual Asset Creation
Unlike traditional tokens, synthetics are virtual positions:
```javascript
// Mint xAAPL with 1000 DUSD
await swapRouter.mintSynthetic(
  keccak256("xAAPL"),
  parseEther("1000"),
  parseEther("6.6") // Minimum xAAPL expected at $150/share
)
```

### 3. Price Calculation
The system uses oracle prices for exact calculations:
```
Oracle Price: $150 per AAPL share
DUSD Amount: 1000
Mint Fee (0.3%): 3 DUSD
Net Amount: 997 DUSD
xAAPL Minted: 997 ÷ 150 = 6.64 xAAPL
```

### 4. Settlement Lock
After minting, a settlement lock prevents immediate arbitrage:
```solidity
// 1-minute settlement lock applied
lastSwapTime[msg.sender] = block.timestamp;
// User cannot transfer, burn, or swap for 1 minute
```

## Supported Assets for Minting

### Equities (Primary Focus)
- **xAAPL**: Apple Inc. - Real-time NASDAQ pricing
- **xTSLA**: Tesla Inc. - High-volatility equity exposure
- **xMSFT**: Microsoft Corp. - Large-cap technology
- **xAMZN**: Amazon.com Inc. - E-commerce and cloud leader
- **xGOOGL**: Alphabet Inc. - Search and advertising giant

### Commodities
- **xGOLD**: Gold futures - Safe haven asset
- **xSILVER**: Silver futures - Industrial precious metal
- **xOIL**: Crude Oil futures - Energy commodity
- **xCOPPER**: Copper futures - Industrial metal

### Indices
- **xSP500**: S&P 500 Index - Broad market exposure
- **xNASDAQ**: NASDAQ Composite - Technology-heavy index
- **xDOW**: Dow Jones Industrial Average - Blue-chip stocks

### Crypto Synthetics
- **xBTC**: Synthetic Bitcoin - Digital gold exposure
- **xETH**: Synthetic Ethereum - Smart contract platform

## Minting Requirements

### Protocol-Level Requirements
- **Dynamic Backing**: Total DUSD supply must not exceed collateral-backed limit
- **Oracle Validity**: Price feeds must be fresh and valid
- **System Health**: Protocol must not be in emergency mode

### User-Level Requirements
- **Minimum Amount**: 10 DUSD minimum mint
- **DUSD Balance**: Sufficient DUSD for mint amount + fees
- **Settlement Status**: No active settlement lock from previous trades

### Risk Parameters
```javascript
const mintingLimits = {
  minMintAmount: parseEther("10"),     // 10 DUSD minimum
  maxSingleMint: parseEther("1000000"), // 1M DUSD maximum
  mintFee: 30, // 0.3% in basis points
  settlementLock: 60 // 1 minute in seconds
}
```

## Fee Structure

### Flat Minting Fee
- **Rate**: 0.3% of DUSD amount
- **Rationale**: Minting is low-risk operation (doesn't drain DUSD supply)
- **Minimum**: 0.03 DUSD (0.3% of 10 DUSD minimum)
- **Maximum**: 3,000 DUSD (0.3% of 1M DUSD maximum)

### Fee Comparison
| Operation | Risk Level | Fee Structure |
|-----------|------------|---------------|
| **Mint Synthetic** | Low | 0.3% flat |
| **Swap Synthetic** | Low | 0.3% flat |
| **Burn Synthetic** | High | 0.3-2% dynamic |

**Why Flat Fee**: Minting converts DUSD to synthetic positions without creating new DUSD, posing minimal solvency risk.

## Minting Examples

### Example 1: Mint xAAPL
```javascript
const mintAmount = parseEther("5000") // 5000 DUSD
const assetId = keccak256("xAAPL")

// Get current AAPL price for estimation
const price = await oracle.getPrice("AAPL") // $150
const expectedShares = (5000 * 0.997) / 150 // 33.23 shares after 0.3% fee

// Execute mint with slippage protection
await swapRouter.mintSynthetic(
  assetId,
  mintAmount,
  parseEther("33.0") // Accept minimum 33 shares (1% slippage)
)

// Settlement lock: Cannot trade for 1 minute
console.log("Settlement lock active until:", Date.now() + 60000)
```

### Example 2: Mint Multiple Assets
```javascript
// Note: Each mint triggers separate settlement lock
// Better to mint one asset, then swap to others

// Mint xSP500
await swapRouter.mintSynthetic(
  keccak256("xSP500"),
  parseEther("2000"),
  parseEther("4.9") // ~5 index units at $400/unit
)

// Wait for settlement lock to expire (1 minute)
await new Promise(resolve => setTimeout(resolve, 61000))

// Then swap to other assets if desired
await swapRouter.swapSynthetic(
  keccak256("xSP500"),
  keccak256("xGOLD"),
  parseEther("2.5"), // Half position
  parseEther("2.4")  // Expected gold units
)
```

## Risk Management

### Protocol Solvency Protection
```solidity
// Hard invariant check before minting
function _checkSolvency(uint256 dUSDAmount) internal view {
    uint256 newTotalSupply = _dUSD().totalSupply() + dUSDAmount;
    uint256 maxBorrowable = _dusdProvider().getMaxBorrowableDUSD();
    
    require(
        newTotalSupply <= maxBorrowable,
        "Insufficient protocol backing"
    );
}
```

### Dynamic Backing Calculation
```solidity
// Real-time backing calculation
function getMaxBorrowableDUSD() external view returns (uint256) {
    uint256 totalCollateralValue = getTotalCollateralValue();
    uint256 maintenanceRatio = getMaintenanceRatio(); // e.g., 75%
    
    return (totalCollateralValue * maintenanceRatio) / 10000;
}
```

### Oracle Risk Mitigation
- **Staleness Checks**: Prices must be updated within last hour
- **Deviation Limits**: Prevent extreme price movements
- **Multiple Sources**: Chainlink + Pyth for redundancy
- **Circuit Breakers**: Pause minting during oracle issues

## Settlement Lock Mechanics

### Purpose
- **MEV Protection**: Prevents sandwich attacks
- **Arbitrage Dampening**: Reduces rapid price exploitation
- **Market Stability**: Allows oracle prices to stabilize

### Implementation
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

// Applied to: transfers, burns, swaps
// Not applied to: viewing balances, checking positions
```

### User Experience
```javascript
// Check settlement status
const lastSwap = await swapRouter.lastSwapTime(userAddress)
const lockExpiry = lastSwap + 60 // 1 minute
const isLocked = Date.now() / 1000 < lockExpiry

if (isLocked) {
  const remainingTime = lockExpiry - (Date.now() / 1000)
  console.log(`Settlement lock expires in ${remainingTime} seconds`)
}
```

## Integration Examples

### React Hook for Minting
```javascript
const useMintSynthetic = () => {
  const [isLoading, setIsLoading] = useState(false)
  const [settlementLock, setSettlementLock] = useState(null)
  
  const mint = async (asset, amount) => {
    setIsLoading(true)
    try {
      // Check protocol backing
      const maxBorrowable = await dusdProvider.getMaxBorrowableDUSD()
      const currentSupply = await dusd.totalSupply()
      
      if (currentSupply + amount > maxBorrowable) {
        throw new Error("Insufficient protocol backing")
      }
      
      // Execute mint
      const tx = await swapRouter.mintSynthetic(asset, amount, minOutput)
      await tx.wait()
      
      // Set settlement lock
      setSettlementLock(Date.now() + 60000)
      
      return tx
    } finally {
      setIsLoading(false)
    }
  }
  
  return { mint, isLoading, settlementLock }
}
```

### Minting Interface Component
```javascript
const MintInterface = ({ asset }) => {
  const [amount, setAmount] = useState('')
  const { mint, isLoading, settlementLock } = useMintSynthetic()
  const { data: price } = useOraclePrice(asset)
  const { data: backing } = useProtocolBacking()
  
  const estimatedShares = amount && price ? 
    (amount * 0.997) / price : 0 // After 0.3% fee
  
  const canMint = backing?.available >= amount
  
  return (
    <div className="mint-interface">
      <div className="asset-info">
        <h3>Mint {asset}</h3>
        <div>Current Price: ${price?.toFixed(2)}</div>
        <div>Protocol Backing: ${backing?.available?.toLocaleString()}</div>
      </div>
      
      <div className="mint-form">
        <input
          type="number"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          placeholder="DUSD Amount"
        />
        
        <div className="estimation">
          Estimated {asset}: {estimatedShares.toFixed(4)}
          <div className="fee">Fee: {(amount * 0.003).toFixed(2)} DUSD</div>
        </div>
        
        <button
          onClick={() => mint(asset, parseEther(amount))}
          disabled={!canMint || isLoading || settlementLock}
        >
          {settlementLock ? 'Settlement Lock Active' : 
           !canMint ? 'Insufficient Backing' :
           isLoading ? 'Minting...' : 'Mint'}
        </button>
        
        {settlementLock && (
          <div className="settlement-warning">
            Settlement lock expires in {Math.ceil((settlementLock - Date.now()) / 1000)}s
          </div>
        )}
      </div>
    </div>
  )
}
```

## Monitoring and Analytics

### Key Metrics
- **Total Synthetic Value**: Sum of all minted synthetic positions
- **Protocol Backing Ratio**: Available backing vs total DUSD supply
- **Mint Volume**: Daily/weekly minting activity
- **Asset Distribution**: Breakdown by synthetic asset type
- **Settlement Lock Activity**: Frequency and duration analysis

### Risk Indicators
```javascript
const getRiskIndicators = async () => {
  const totalSupply = await dusd.totalSupply()
  const maxBorrowable = await dusdProvider.getMaxBorrowableDUSD()
  const totalSyntheticValue = await getTotalSyntheticValue()
  
  return {
    backingRatio: maxBorrowable / totalSupply,
    utilizationRate: totalSupply / maxBorrowable,
    syntheticRatio: totalSyntheticValue / totalSupply,
    riskLevel: getRiskLevel(totalSupply, maxBorrowable)
  }
}
```

## Best Practices

### For Users
- **Check Backing**: Verify protocol has sufficient backing before large mints
- **Plan Timing**: Consider settlement locks when planning multiple operations
- **Monitor Fees**: Understand flat vs dynamic fee structure
- **Oracle Awareness**: Be aware of market hours for equity synthetics

### For Developers
- **Solvency Checks**: Always validate protocol backing before minting
- **Settlement Handling**: Account for 1-minute locks in UI/UX
- **Error Handling**: Gracefully handle backing insufficient errors
- **Real-time Updates**: Monitor backing ratios and settlement status

### Security Considerations
- **Oracle Dependency**: Understand reliance on price feed accuracy
- **Settlement Timing**: Don't assume immediate liquidity after mints
- **Protocol Limits**: Respect dynamic backing constraints
- **Fee Calculations**: Account for fees in all estimations