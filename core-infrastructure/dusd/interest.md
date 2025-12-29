# Interest Rates

DUSD borrowing involves stability fees (interest rates) that adjust dynamically based on market conditions, system utilization, and collateral risk to maintain protocol stability and incentivize proper behavior.

## Interest Rate Model

### Base Rate Calculation
```solidity
contract InterestRateModel {
    uint256 public baseRate = 2e16; // 2% base rate
    uint256 public multiplier = 5e16; // 5% multiplier
    uint256 public jumpMultiplier = 109e16; // 109% jump multiplier
    uint256 public kink = 8e17; // 80% utilization kink
    
    function getBorrowRate(uint256 cash, uint256 borrows, uint256 reserves) 
        external view returns (uint256) {
        uint256 util = utilizationRate(cash, borrows, reserves);
        
        if (util <= kink) {
            return baseRate + (util * multiplier / 1e18);
        } else {
            uint256 normalRate = baseRate + (kink * multiplier / 1e18);
            uint256 excessUtil = util - kink;
            return normalRate + (excessUtil * jumpMultiplier / 1e18);
        }
    }
    
    function utilizationRate(uint256 cash, uint256 borrows, uint256 reserves) 
        public pure returns (uint256) {
        if (borrows == 0) return 0;
        return borrows * 1e18 / (cash + borrows - reserves);
    }
}
```

### Rate Components
```
Total Interest Rate = Base Rate + Utilization Rate + Risk Premium + Stability Adjustment
```

## Collateral-Specific Rates

### Risk-Based Pricing
| Collateral | Base Rate | Risk Premium | Current Rate | Max Rate |
|------------|-----------|--------------|--------------|----------|
| **USDC** | 0.5% | 0% | 0.5% | 5% |
| **ETH** | 2% | 1% | 3% | 15% |
| **WBTC** | 2% | 1% | 3% | 15% |
| **DI** | 1% | 2% | 3% | 20% |

### Rate Calculation by Asset
```solidity
function getAssetBorrowRate(address asset) external view returns (uint256) {
    AssetConfig memory config = assetConfigs[asset];
    uint256 utilization = getAssetUtilization(asset);
    uint256 dusdPrice = oracle.getDUSDPrice();
    
    uint256 baseRate = config.baseRate;
    uint256 utilizationRate = utilization * config.multiplier / 1e18;
    uint256 riskPremium = config.riskPremium;
    
    // Stability adjustment based on DUSD price
    uint256 stabilityAdjustment = 0;
    if (dusdPrice < 0.99e18) {
        // DUSD below peg - increase rates to reduce supply
        stabilityAdjustment = (0.99e18 - dusdPrice) * 10; // 10x multiplier
    } else if (dusdPrice > 1.01e18) {
        // DUSD above peg - decrease rates to increase supply
        stabilityAdjustment = (dusdPrice - 1.01e18) * 5; // 5x reduction
    }
    
    return baseRate + utilizationRate + riskPremium + stabilityAdjustment;
}
```

## Dynamic Rate Adjustments

### Market-Responsive Rates
```javascript
const updateInterestRates = async () => {
  const marketConditions = await getMarketConditions()
  
  for (const asset of supportedAssets) {
    const currentRate = await getRateForAsset(asset)
    const volatility = marketConditions.volatility[asset]
    const liquidity = marketConditions.liquidity[asset]
    const dusdPrice = marketConditions.dusdPrice
    
    let newRate = currentRate
    
    // Adjust for volatility
    if (volatility > 0.5) {
      newRate *= 1.2 // +20% for high volatility
    }
    
    // Adjust for liquidity
    if (liquidity < 1000000) { // $1M threshold
      newRate *= 1.1 // +10% for low liquidity
    }
    
    // Adjust for DUSD peg
    if (dusdPrice < 0.995) {
      newRate *= 1.15 // +15% to reduce supply
    } else if (dusdPrice > 1.005) {
      newRate *= 0.9 // -10% to increase supply
    }
    
    await updateAssetRate(asset, newRate)
  }
}
```

### Utilization-Based Adjustments
```solidity
function updateUtilizationRates() external {
    for (uint i = 0; i < supportedAssets.length; i++) {
        address asset = supportedAssets[i];
        uint256 utilization = getAssetUtilization(asset);
        
        if (utilization > 90e16) { // >90% utilization
            // Increase rates aggressively
            assetConfigs[asset].multiplier = assetConfigs[asset].multiplier * 150 / 100;
        } else if (utilization < 50e16) { // <50% utilization
            // Decrease rates to encourage borrowing
            assetConfigs[asset].multiplier = assetConfigs[asset].multiplier * 90 / 100;
        }
        
        emit RateUpdated(asset, assetConfigs[asset].multiplier);
    }
}
```

## Interest Accrual

### Compound Interest Calculation
```solidity
contract InterestAccrual {
    mapping(address => uint256) public borrowIndex;
    mapping(address => uint256) public accrualBlockNumber;
    
    function accrueInterest(address asset) public returns (uint256) {
        uint256 currentBlockNumber = block.number;
        uint256 accrualBlockNumberPrior = accrualBlockNumber[asset];
        
        if (accrualBlockNumberPrior == currentBlockNumber) {
            return NO_ERROR;
        }
        
        uint256 cashPrior = getCash(asset);
        uint256 borrowsPrior = totalBorrows[asset];
        uint256 reservesPrior = totalReserves[asset];
        uint256 borrowIndexPrior = borrowIndex[asset];
        
        uint256 borrowRateMantissa = interestRateModel.getBorrowRate(
            cashPrior, 
            borrowsPrior, 
            reservesPrior
        );
        
        uint256 blockDelta = currentBlockNumber - accrualBlockNumberPrior;
        uint256 simpleInterestFactor = borrowRateMantissa * blockDelta;
        uint256 interestAccumulated = simpleInterestFactor * borrowsPrior / 1e18;
        
        uint256 totalBorrowsNew = interestAccumulated + borrowsPrior;
        uint256 borrowIndexNew = simpleInterestFactor * borrowIndexPrior / 1e18 + borrowIndexPrior;
        
        borrowIndex[asset] = borrowIndexNew;
        totalBorrows[asset] = totalBorrowsNew;
        accrualBlockNumber[asset] = currentBlockNumber;
        
        emit AccrueInterest(asset, interestAccumulated, borrowIndexNew, totalBorrowsNew);
        
        return NO_ERROR;
    }
}
```

### User Interest Calculation
```javascript
const calculateUserInterest = async (userAddress, asset) => {
  const userBorrow = await getUserBorrowBalance(userAddress, asset)
  const borrowIndex = await getBorrowIndex(asset)
  const userBorrowIndex = await getUserBorrowIndex(userAddress, asset)
  
  const accruedInterest = userBorrow * (borrowIndex - userBorrowIndex) / 1e18
  
  return {
    principal: userBorrow,
    accruedInterest,
    totalOwed: userBorrow + accruedInterest,
    currentRate: await getCurrentBorrowRate(asset)
  }
}
```

## Interest Payment

### Automatic Accrual
Interest accrues automatically on every interaction:
```solidity
modifier accrueInterestForAsset(address asset) {
    require(accrueInterest(asset) == 0, "Interest accrual failed");
    _;
}

function borrow(address asset, uint256 amount) 
    external 
    accrueInterestForAsset(asset) 
{
    // Borrowing logic with updated interest
}

function repay(address asset, uint256 amount) 
    external 
    accrueInterestForAsset(asset) 
{
    // Repayment logic with updated interest
}
```

### Manual Interest Payment
```javascript
// Pay accrued interest without reducing principal
const payInterest = async (asset) => {
  const interestOwed = await getAccruedInterest(userAddress, asset)
  
  await dusd.approve(dusdManager.address, interestOwed)
  await dusdManager.payInterest(asset, interestOwed)
}

// Get interest payment quote
const getInterestQuote = async (asset, timeframe) => {
  const currentRate = await getCurrentBorrowRate(asset)
  const principal = await getUserBorrowBalance(userAddress, asset)
  
  const dailyRate = currentRate / 365
  const interestForPeriod = principal * dailyRate * timeframe
  
  return {
    principal: formatEther(principal),
    dailyInterest: formatEther(principal * dailyRate),
    periodInterest: formatEther(interestForPeriod),
    annualRate: (currentRate * 100).toFixed(2) + '%'
  }
}
```

## Rate Monitoring

### Real-time Rate Tracking
```javascript
const useInterestRates = () => {
  return useQuery(
    ['interestRates'],
    async () => {
      const rates = {}
      
      for (const asset of supportedAssets) {
        const [currentRate, utilization, nextRate] = await Promise.all([
          dusdManager.getCurrentBorrowRate(asset),
          dusdManager.getUtilizationRate(asset),
          dusdManager.getNextBorrowRate(asset)
        ])
        
        rates[asset] = {
          current: Number(currentRate) / 1e18 * 100,
          utilization: Number(utilization) / 1e18 * 100,
          next: Number(nextRate) / 1e18 * 100,
          trend: nextRate > currentRate ? 'increasing' : 'decreasing'
        }
      }
      
      return rates
    },
    { refetchInterval: 30000 }
  )
}
```

### Interest Rate Dashboard
```javascript
const InterestRateDashboard = () => {
  const { data: rates } = useInterestRates()
  const { data: userPositions } = useUserBorrowPositions()
  
  return (
    <div className="interest-dashboard">
      <div className="market-rates">
        <h3>Current Market Rates</h3>
        {Object.entries(rates || {}).map(([asset, data]) => (
          <div key={asset} className="rate-item">
            <div className="asset-info">
              <span>{asset}</span>
              <span className={`trend ${data.trend}`}>
                {data.current.toFixed(2)}%
              </span>
            </div>
            <div className="utilization">
              Utilization: {data.utilization.toFixed(1)}%
            </div>
          </div>
        ))}
      </div>
      
      <div className="user-positions">
        <h3>Your Borrowing Costs</h3>
        {userPositions?.map(position => (
          <div key={position.asset} className="position-item">
            <div>Asset: {position.asset}</div>
            <div>Borrowed: {formatNumber(position.borrowed)} DUSD</div>
            <div>Rate: {position.rate.toFixed(2)}%</div>
            <div>Daily Interest: ${formatNumber(position.dailyInterest)}</div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

## Rate Optimization

### Borrowing Strategy
```javascript
const optimizeBorrowingCosts = async (targetAmount, maxRate) => {
  const assets = await getSupportedAssets()
  const opportunities = []
  
  for (const asset of assets) {
    const rate = await getCurrentBorrowRate(asset)
    const available = await getAvailableToBorrow(asset)
    const collateralRequired = await getCollateralRequired(asset, targetAmount)
    
    if (rate <= maxRate && available >= targetAmount) {
      opportunities.push({
        asset,
        rate: Number(rate) / 1e18 * 100,
        collateralRequired: formatEther(collateralRequired),
        costPerYear: targetAmount * Number(rate) / 1e18
      })
    }
  }
  
  return opportunities.sort((a, b) => a.rate - b.rate)
}
```

### Rate Arbitrage
```javascript
const findRateArbitrage = async () => {
  const opportunities = []
  
  for (const asset of supportedAssets) {
    const borrowRate = await getBorrowRate(asset)
    const supplyRate = await getSupplyRate(asset) // If lending is available
    const externalRate = await getExternalLendingRate(asset)
    
    // Borrow from DI Network, lend externally
    if (externalRate > borrowRate * 1.1) { // 10% margin for safety
      opportunities.push({
        asset,
        borrowRate: Number(borrowRate) / 1e18 * 100,
        lendRate: Number(externalRate) / 1e18 * 100,
        spread: (Number(externalRate) - Number(borrowRate)) / 1e18 * 100,
        strategy: 'borrow_and_lend'
      })
    }
  }
  
  return opportunities.sort((a, b) => b.spread - a.spread)
}
```

## Integration Examples

### Interest Calculator
```javascript
const InterestCalculator = ({ asset, amount, duration }) => {
  const [interestData, setInterestData] = useState(null)
  
  useEffect(() => {
    const calculateInterest = async () => {
      const rate = await getCurrentBorrowRate(asset)
      const dailyRate = Number(rate) / 1e18 / 365
      const totalInterest = amount * dailyRate * duration
      
      setInterestData({
        dailyInterest: amount * dailyRate,
        totalInterest,
        effectiveRate: (totalInterest / amount) * (365 / duration) * 100
      })
    }
    
    if (asset && amount && duration) {
      calculateInterest()
    }
  }, [asset, amount, duration])
  
  return (
    <div className="interest-calculator">
      <div>Daily Interest: ${interestData?.dailyInterest?.toFixed(2)}</div>
      <div>Total Interest: ${interestData?.totalInterest?.toFixed(2)}</div>
      <div>Effective Rate: {interestData?.effectiveRate?.toFixed(2)}%</div>
    </div>
  )
}
```