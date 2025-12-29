# Collateral Management

DUSD is backed by a diversified portfolio of crypto assets that serve as collateral for minting the stablecoin. The system manages collateral through risk-adjusted parameters and automated liquidation mechanisms.

## Supported Collateral

### Collateral Assets
| Asset | Symbol | Min Ratio | Liquidation | Stability Fee | Max Exposure |
|-------|--------|-----------|-------------|---------------|--------------|
| **Ethereum** | ETH | 150% | 130% | 2.0% APR | 40% |
| **Bitcoin** | WBTC | 150% | 130% | 2.0% APR | 30% |
| **USD Coin** | USDC | 110% | 105% | 0.5% APR | 20% |
| **DI Token** | DI | 200% | 160% | 1.0% APR | 10% |

### Risk Parameters
```solidity
struct CollateralConfig {
    uint256 minCollateralRatio;    // Minimum collateralization ratio
    uint256 liquidationThreshold;  // Liquidation trigger point
    uint256 stabilityFee;         // Annual borrowing fee
    uint256 maxExposure;          // Maximum system exposure
    uint256 liquidationPenalty;   // Liquidation penalty fee
    bool isActive;                // Collateral enabled/disabled
}
```

## Collateral Deposit Process

### Depositing Collateral
```javascript
// Deposit ETH as collateral
await dusdManager.depositCollateral(
  ETH_ADDRESS,
  { value: parseEther("10") } // 10 ETH
)

// Deposit ERC20 collateral
await wbtc.approve(dusdManager.address, amount)
await dusdManager.depositCollateral(WBTC_ADDRESS, amount)
```

### Minting DUSD
```solidity
function mintDUSD(address collateralAsset, uint256 dusdAmount) external {
    CollateralConfig memory config = collateralConfigs[collateralAsset];
    require(config.isActive, "Collateral not supported");
    
    uint256 collateralValue = getCollateralValue(msg.sender, collateralAsset);
    uint256 existingDebt = userDebt[msg.sender];
    uint256 newDebt = existingDebt + dusdAmount;
    
    uint256 requiredCollateral = newDebt * config.minCollateralRatio / 100;
    require(collateralValue >= requiredCollateral, "Insufficient collateral");
    
    userDebt[msg.sender] = newDebt;
    _mint(msg.sender, dusdAmount);
    
    emit DUSDMinted(msg.sender, collateralAsset, dusdAmount);
}
```

## Collateral Valuation

### Price Oracle Integration
```solidity
function getCollateralValue(address user, address asset) public view returns (uint256) {
    uint256 balance = userCollateral[user][asset];
    uint256 price = oracle.getPrice(asset);
    uint256 decimals = IERC20Metadata(asset).decimals();
    
    return balance * price / (10 ** decimals);
}

function getTotalCollateralValue(address user) external view returns (uint256) {
    uint256 totalValue = 0;
    
    for (uint i = 0; i < supportedCollaterals.length; i++) {
        address asset = supportedCollaterals[i];
        totalValue += getCollateralValue(user, asset);
    }
    
    return totalValue;
}
```

### Risk-Adjusted Valuation
```javascript
const calculateRiskAdjustedValue = (collateralBalances, prices, riskFactors) => {
  let totalValue = 0
  
  for (const [asset, balance] of Object.entries(collateralBalances)) {
    const price = prices[asset]
    const riskFactor = riskFactors[asset] // 0.8 for volatile assets, 0.95 for stable
    const adjustedValue = balance * price * riskFactor
    totalValue += adjustedValue
  }
  
  return totalValue
}
```

## Collateral Ratios

### Health Factor Calculation
```solidity
function getHealthFactor(address user) external view returns (uint256) {
    uint256 totalCollateralValue = getTotalCollateralValue(user);
    uint256 totalDebt = userDebt[user];
    
    if (totalDebt == 0) return type(uint256).max;
    
    uint256 liquidationThreshold = getWeightedLiquidationThreshold(user);
    return totalCollateralValue * liquidationThreshold / (totalDebt * 100);
}

function getWeightedLiquidationThreshold(address user) internal view returns (uint256) {
    uint256 totalValue = 0;
    uint256 weightedThreshold = 0;
    
    for (uint i = 0; i < supportedCollaterals.length; i++) {
        address asset = supportedCollaterals[i];
        uint256 value = getCollateralValue(user, asset);
        uint256 threshold = collateralConfigs[asset].liquidationThreshold;
        
        totalValue += value;
        weightedThreshold += value * threshold;
    }
    
    return totalValue > 0 ? weightedThreshold / totalValue : 0;
}
```

### Collateral Ratio Monitoring
```javascript
const useCollateralRatio = (userAddress) => {
  return useQuery(
    ['collateralRatio', userAddress],
    async () => {
      const [totalCollateral, totalDebt, healthFactor] = await Promise.all([
        dusdManager.getTotalCollateralValue(userAddress),
        dusdManager.getUserDebt(userAddress),
        dusdManager.getHealthFactor(userAddress)
      ])
      
      const ratio = totalDebt > 0 ? Number(totalCollateral) / Number(totalDebt) : Infinity
      
      return {
        collateralValue: formatEther(totalCollateral),
        debtValue: formatEther(totalDebt),
        ratio: ratio,
        healthFactor: Number(healthFactor) / 1e18,
        isHealthy: Number(healthFactor) > 1e18,
        liquidationRisk: Number(healthFactor) < 1.2e18
      }
    },
    { refetchInterval: 30000 }
  )
}
```

## Collateral Operations

### Adding Collateral
```javascript
const addCollateral = async (asset, amount) => {
  if (asset === ETH_ADDRESS) {
    await dusdManager.depositCollateral(asset, { value: amount })
  } else {
    await erc20.approve(dusdManager.address, amount)
    await dusdManager.depositCollateral(asset, amount)
  }
}
```

### Withdrawing Collateral
```solidity
function withdrawCollateral(address asset, uint256 amount) external {
    require(userCollateral[msg.sender][asset] >= amount, "Insufficient collateral");
    
    userCollateral[msg.sender][asset] -= amount;
    
    // Check health factor after withdrawal
    require(getHealthFactor(msg.sender) >= 1e18, "Would cause liquidation");
    
    if (asset == ETH_ADDRESS) {
        payable(msg.sender).transfer(amount);
    } else {
        IERC20(asset).transfer(msg.sender, amount);
    }
    
    emit CollateralWithdrawn(msg.sender, asset, amount);
}
```

### Collateral Swapping
```javascript
// Swap one collateral type for another
const swapCollateral = async (fromAsset, toAsset, amount) => {
  // Get swap quote
  const quote = await getSwapQuote(fromAsset, toAsset, amount)
  
  // Execute atomic swap
  await dusdManager.swapCollateral(
    fromAsset,
    toAsset,
    amount,
    quote.minAmountOut,
    quote.deadline
  )
}
```

## Risk Management

### Diversification Limits
```solidity
mapping(address => uint256) public maxCollateralExposure;

function checkExposureLimit(address asset, uint256 additionalAmount) internal view {
    uint256 currentExposure = totalCollateralByAsset[asset];
    uint256 totalSystemCollateral = getTotalSystemCollateral();
    uint256 newExposure = currentExposure + additionalAmount;
    
    uint256 exposurePercentage = newExposure * 100 / totalSystemCollateral;
    require(exposurePercentage <= maxCollateralExposure[asset], "Exposure limit exceeded");
}
```

### Dynamic Risk Parameters
```javascript
const adjustRiskParameters = async (marketConditions) => {
  for (const asset of supportedAssets) {
    const volatility = await getVolatility(asset)
    const liquidity = await getLiquidity(asset)
    
    // Increase collateral requirements for volatile assets
    if (volatility > 0.5) {
      await dusdManager.updateCollateralRatio(asset, 180) // 180% instead of 150%
    }
    
    // Reduce exposure limits for illiquid assets
    if (liquidity < 1000000) { // $1M liquidity threshold
      await dusdManager.updateMaxExposure(asset, 5) // 5% max exposure
    }
  }
}
```

## Liquidation Integration

### Liquidation Triggers
```solidity
function isLiquidatable(address user) external view returns (bool) {
    return getHealthFactor(user) < 1e18;
}

function liquidate(address user, address collateralAsset, uint256 debtToCover) external {
    require(isLiquidatable(user), "User not liquidatable");
    
    uint256 collateralToLiquidate = calculateCollateralAmount(
        collateralAsset,
        debtToCover
    );
    
    // Transfer DUSD from liquidator
    dusd.transferFrom(msg.sender, address(this), debtToCover);
    dusd.burn(debtToCover);
    
    // Transfer collateral to liquidator (with bonus)
    uint256 liquidationBonus = collateralToLiquidate * LIQUIDATION_BONUS / 100;
    uint256 totalCollateral = collateralToLiquidate + liquidationBonus;
    
    userCollateral[user][collateralAsset] -= totalCollateral;
    userDebt[user] -= debtToCover;
    
    transferCollateral(msg.sender, collateralAsset, totalCollateral);
}
```

## Integration Examples

### Collateral Dashboard
```javascript
const CollateralDashboard = ({ userAddress }) => {
  const { data: collateralData } = useQuery(
    ['userCollateral', userAddress],
    () => getUserCollateralData(userAddress)
  )
  
  return (
    <div className="collateral-dashboard">
      <div className="health-indicator">
        <div className={`health-factor ${collateralData?.healthFactor < 1.2 ? 'danger' : 'safe'}`}>
          Health Factor: {collateralData?.healthFactor?.toFixed(2)}
        </div>
        <div>Collateral Ratio: {(collateralData?.ratio * 100).toFixed(1)}%</div>
      </div>
      
      <div className="collateral-breakdown">
        {collateralData?.assets?.map(asset => (
          <div key={asset.symbol} className="asset-item">
            <div>{asset.symbol}: {formatNumber(asset.balance)}</div>
            <div>Value: ${formatNumber(asset.value)}</div>
            <div>Weight: {asset.weight}%</div>
          </div>
        ))}
      </div>
      
      <div className="actions">
        <button onClick={() => openAddCollateralModal()}>Add Collateral</button>
        <button onClick={() => openWithdrawModal()}>Withdraw</button>
        <button onClick={() => openSwapModal()}>Swap Collateral</button>
      </div>
    </div>
  )
}
```

### Risk Monitoring
```javascript
const useRiskMonitoring = (userAddress) => {
  return useQuery(
    ['riskMonitoring', userAddress],
    async () => {
      const healthFactor = await dusdManager.getHealthFactor(userAddress)
      const liquidationPrice = await calculateLiquidationPrice(userAddress)
      
      return {
        healthFactor: Number(healthFactor) / 1e18,
        liquidationPrice,
        riskLevel: getRiskLevel(Number(healthFactor) / 1e18),
        timeToLiquidation: estimateTimeToLiquidation(userAddress)
      }
    },
    { refetchInterval: 10000 }
  )
}

const getRiskLevel = (healthFactor) => {
  if (healthFactor < 1.1) return 'critical'
  if (healthFactor < 1.3) return 'high'
  if (healthFactor < 1.5) return 'medium'
  return 'low'
}
```