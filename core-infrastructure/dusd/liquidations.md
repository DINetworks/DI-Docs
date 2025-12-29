# Liquidations

DUSD liquidations protect the protocol by automatically closing under-collateralized positions, ensuring system solvency and maintaining the stablecoin's backing. The liquidation system operates through automated mechanisms and incentivized liquidators.

## Liquidation Overview

### Liquidation Triggers
Positions become eligible for liquidation when:
- **Health Factor < 1.0**: Collateral value falls below liquidation threshold
- **Debt Exceeds Limit**: Outstanding debt surpasses maximum allowed
- **Interest Accumulation**: Unpaid interest pushes position underwater
- **Oracle Price Updates**: Collateral price drops trigger liquidations

### Liquidation Threshold
```solidity
function isLiquidatable(address borrower) external view returns (bool) {
    uint256 healthFactor = getHealthFactor(borrower);
    return healthFactor < 1e18; // Health factor below 1.0
}

function getHealthFactor(address borrower) public view returns (uint256) {
    uint256 totalCollateralValue = getTotalCollateralValue(borrower);
    uint256 totalDebt = getTotalDebt(borrower);
    
    if (totalDebt == 0) return type(uint256).max;
    
    uint256 liquidationThreshold = getWeightedLiquidationThreshold(borrower);
    return totalCollateralValue * liquidationThreshold / (totalDebt * 100);
}
```

## Liquidation Process

### Liquidation Execution
```solidity
function liquidate(
    address borrower,
    address collateralAsset,
    uint256 debtToCover
) external {
    require(isLiquidatable(borrower), "Position not liquidatable");
    
    uint256 maxDebtToCover = getMaxLiquidatableDebt(borrower, collateralAsset);
    require(debtToCover <= maxDebtToCover, "Debt amount too high");
    
    // Calculate collateral to seize
    uint256 collateralToSeize = calculateCollateralToSeize(
        collateralAsset,
        debtToCover
    );
    
    // Transfer DUSD from liquidator and burn it
    dusd.transferFrom(msg.sender, address(this), debtToCover);
    dusd.burn(debtToCover);
    
    // Reduce borrower's debt
    userDebt[borrower] -= debtToCover;
    
    // Transfer collateral to liquidator (including bonus)
    uint256 liquidationBonus = collateralToSeize * LIQUIDATION_BONUS / 10000;
    uint256 totalCollateral = collateralToSeize + liquidationBonus;
    
    userCollateral[borrower][collateralAsset] -= totalCollateral;
    
    if (collateralAsset == address(0)) {
        payable(msg.sender).transfer(totalCollateral);
    } else {
        IERC20(collateralAsset).transfer(msg.sender, totalCollateral);
    }
    
    emit Liquidation(
        borrower,
        msg.sender,
        collateralAsset,
        debtToCover,
        collateralToSeize,
        liquidationBonus
    );
}
```

### Collateral Calculation
```solidity
function calculateCollateralToSeize(
    address collateralAsset,
    uint256 debtToCover
) internal view returns (uint256) {
    uint256 collateralPrice = oracle.getPrice(collateralAsset);
    uint256 dusdPrice = oracle.getDUSDPrice();
    
    // Convert debt to collateral amount
    uint256 collateralAmount = debtToCover * dusdPrice / collateralPrice;
    
    return collateralAmount;
}
```

## Liquidation Parameters

### Asset-Specific Parameters
| Asset | Liquidation Threshold | Liquidation Bonus | Max Liquidation |
|-------|----------------------|-------------------|-----------------|
| **ETH** | 130% | 5% | 50% |
| **WBTC** | 130% | 5% | 50% |
| **USDC** | 105% | 2% | 100% |
| **DI** | 160% | 8% | 40% |

### Liquidation Limits
```solidity
mapping(address => uint256) public maxLiquidationRatio;

function getMaxLiquidatableDebt(address borrower, address collateralAsset) 
    public view returns (uint256) {
    uint256 totalDebt = getTotalDebt(borrower);
    uint256 maxRatio = maxLiquidationRatio[collateralAsset];
    
    return totalDebt * maxRatio / 100;
}
```

## Liquidation Incentives

### Liquidator Rewards
```javascript
const calculateLiquidationReward = (debtAmount, collateralAsset, collateralPrice) => {
  const liquidationBonus = getLiquidationBonus(collateralAsset) // e.g., 5%
  const collateralAmount = debtAmount / collateralPrice
  const bonusAmount = collateralAmount * liquidationBonus
  
  return {
    collateralReceived: collateralAmount + bonusAmount,
    bonusValue: bonusAmount * collateralPrice,
    profitMargin: (bonusAmount * collateralPrice) / debtAmount * 100
  }
}
```

### Gas Compensation
```solidity
uint256 public constant LIQUIDATION_GAS_COMPENSATION = 200e18; // 200 DUSD

function liquidateWithGasCompensation(
    address borrower,
    address collateralAsset,
    uint256 debtToCover
) external {
    liquidate(borrower, collateralAsset, debtToCover);
    
    // Compensate liquidator for gas costs
    dusd.mint(msg.sender, LIQUIDATION_GAS_COMPENSATION);
}
```

## Partial vs Full Liquidations

### Partial Liquidation
```solidity
function partialLiquidate(
    address borrower,
    address collateralAsset,
    uint256 debtToCover
) external {
    require(isLiquidatable(borrower), "Position not liquidatable");
    
    uint256 maxPartialDebt = getTotalDebt(borrower) * 50 / 100; // Max 50%
    require(debtToCover <= maxPartialDebt, "Exceeds partial liquidation limit");
    
    // Execute partial liquidation
    _executeLiquidation(borrower, collateralAsset, debtToCover);
    
    // Ensure position remains healthy after partial liquidation
    require(getHealthFactor(borrower) >= 1.1e18, "Position still unhealthy");
}
```

### Full Liquidation
```solidity
function fullLiquidate(address borrower) external {
    require(isLiquidatable(borrower), "Position not liquidatable");
    require(getHealthFactor(borrower) < 0.95e18, "Use partial liquidation");
    
    uint256 totalDebt = getTotalDebt(borrower);
    address[] memory collateralAssets = getUserCollateralAssets(borrower);
    
    // Liquidate all collateral proportionally
    for (uint i = 0; i < collateralAssets.length; i++) {
        address asset = collateralAssets[i];
        uint256 assetValue = getCollateralValue(borrower, asset);
        uint256 debtShare = totalDebt * assetValue / getTotalCollateralValue(borrower);
        
        if (debtShare > 0) {
            _executeLiquidation(borrower, asset, debtShare);
        }
    }
}
```

## Liquidation Protection

### Grace Period
```solidity
mapping(address => uint256) public liquidationGracePeriod;

function setLiquidationGracePeriod(address borrower, uint256 period) external {
    require(msg.sender == borrower || isAuthorized(msg.sender), "Unauthorized");
    require(period <= MAX_GRACE_PERIOD, "Grace period too long");
    
    liquidationGracePeriod[borrower] = block.timestamp + period;
}

function isInGracePeriod(address borrower) public view returns (bool) {
    return block.timestamp < liquidationGracePeriod[borrower];
}
```

### Auto-Repayment
```javascript
const setupAutoRepayment = async (borrower, repaymentAmount, triggerRatio) => {
  await dusd.approve(autoRepaymentContract.address, repaymentAmount)
  
  await autoRepaymentContract.setupAutoRepayment({
    borrower,
    repaymentAmount,
    triggerRatio, // e.g., 1.2 (120% health factor)
    maxGasPrice: parseUnits("50", "gwei")
  })
}
```

## Liquidation Monitoring

### Health Factor Tracking
```javascript
const useLiquidationRisk = (userAddress) => {
  return useQuery(
    ['liquidationRisk', userAddress],
    async () => {
      const [healthFactor, liquidationPrice, timeToLiquidation] = await Promise.all([
        dusdManager.getHealthFactor(userAddress),
        calculateLiquidationPrice(userAddress),
        estimateTimeToLiquidation(userAddress)
      ])
      
      const risk = Number(healthFactor) / 1e18
      
      return {
        healthFactor: risk,
        liquidationPrice,
        timeToLiquidation,
        riskLevel: getRiskLevel(risk),
        isAtRisk: risk < 1.2,
        isCritical: risk < 1.05
      }
    },
    { refetchInterval: 10000 }
  )
}

const getRiskLevel = (healthFactor) => {
  if (healthFactor < 1.05) return 'critical'
  if (healthFactor < 1.2) return 'high'
  if (healthFactor < 1.5) return 'medium'
  return 'low'
}
```

### Liquidation Alerts
```javascript
const LiquidationAlert = ({ riskData, currentPrices }) => {
  if (!riskData?.isAtRisk) return null
  
  const priceDistance = Math.abs(
    currentPrices.ETH - riskData.liquidationPrice.ETH
  ) / currentPrices.ETH
  
  return (
    <div className={`liquidation-alert ${riskData.riskLevel}`}>
      {riskData.isCritical ? (
        <div className="critical-warning">
          🚨 CRITICAL: Position may be liquidated!
          <div>Health Factor: {riskData.healthFactor.toFixed(3)}</div>
          <div>Time to Liquidation: {riskData.timeToLiquidation}</div>
        </div>
      ) : (
        <div className="warning">
          ⚠️ WARNING: Low health factor
          <div>Current: {riskData.healthFactor.toFixed(3)}</div>
          <div>Liquidation Price: ${riskData.liquidationPrice.ETH.toFixed(2)}</div>
          <div>Distance: {(priceDistance * 100).toFixed(1)}%</div>
        </div>
      )}
      
      <div className="actions">
        <button onClick={() => addCollateral()}>Add Collateral</button>
        <button onClick={() => repayDebt()}>Repay Debt</button>
      </div>
    </div>
  )
}
```

## Liquidation Bots

### Bot Implementation
```javascript
class LiquidationBot {
  constructor(provider, privateKey) {
    this.provider = provider
    this.wallet = new ethers.Wallet(privateKey, provider)
    this.dusdManager = new ethers.Contract(DUSD_MANAGER_ADDRESS, ABI, this.wallet)
  }
  
  async scanForLiquidations() {
    const borrowers = await this.getAllBorrowers()
    const liquidatablePositions = []
    
    for (const borrower of borrowers) {
      const isLiquidatable = await this.dusdManager.isLiquidatable(borrower)
      
      if (isLiquidatable) {
        const profitability = await this.calculateProfitability(borrower)
        
        if (profitability.profit > profitability.gasCost) {
          liquidatablePositions.push({
            borrower,
            ...profitability
          })
        }
      }
    }
    
    return liquidatablePositions.sort((a, b) => b.profit - a.profit)
  }
  
  async executeLiquidation(position) {
    const gasPrice = await this.provider.getGasPrice()
    const gasLimit = 500000 // Estimated gas limit
    
    try {
      const tx = await this.dusdManager.liquidate(
        position.borrower,
        position.collateralAsset,
        position.debtToCover,
        { gasPrice, gasLimit }
      )
      
      return await tx.wait()
    } catch (error) {
      console.error('Liquidation failed:', error)
      throw error
    }
  }
  
  async calculateProfitability(borrower) {
    const debt = await this.dusdManager.getTotalDebt(borrower)
    const collateral = await this.dusdManager.getTotalCollateralValue(borrower)
    const liquidationBonus = await this.dusdManager.getLiquidationBonus()
    
    const maxLiquidatable = debt * 0.5 // 50% max
    const collateralToReceive = maxLiquidatable * (1 + liquidationBonus)
    const profit = collateralToReceive - maxLiquidatable
    
    const gasPrice = await this.provider.getGasPrice()
    const gasCost = gasPrice * 500000 // Estimated gas
    
    return {
      debtToCover: maxLiquidatable,
      collateralReceived: collateralToReceive,
      profit,
      gasCost,
      netProfit: profit - gasCost
    }
  }
}
```

### Liquidation Dashboard
```javascript
const LiquidationDashboard = () => {
  const [liquidations, setLiquidations] = useState([])
  const [opportunities, setOpportunities] = useState([])
  
  useEffect(() => {
    const contract = new ethers.Contract(DUSD_MANAGER_ADDRESS, ABI, provider)
    
    const handleLiquidation = (borrower, liquidator, collateralAsset, debtCovered, collateralSeized, bonus) => {
      const liquidationEvent = {
        borrower,
        liquidator,
        collateralAsset,
        debtCovered: formatEther(debtCovered),
        collateralSeized: formatEther(collateralSeized),
        bonus: formatEther(bonus),
        timestamp: Date.now()
      }
      
      setLiquidations(prev => [liquidationEvent, ...prev.slice(0, 99)])
    }
    
    contract.on('Liquidation', handleLiquidation)
    
    return () => {
      contract.off('Liquidation', handleLiquidation)
    }
  }, [])
  
  return (
    <div className="liquidation-dashboard">
      <div className="opportunities">
        <h3>Liquidation Opportunities</h3>
        {opportunities.map(opp => (
          <div key={opp.borrower} className="opportunity-item">
            <div>Borrower: {opp.borrower.slice(0, 8)}...</div>
            <div>Debt: ${formatNumber(opp.debtToCover)}</div>
            <div>Profit: ${formatNumber(opp.netProfit)}</div>
            <button onClick={() => executeLiquidation(opp)}>
              Liquidate
            </button>
          </div>
        ))}
      </div>
      
      <div className="recent-liquidations">
        <h3>Recent Liquidations</h3>
        {liquidations.map(liq => (
          <div key={`${liq.borrower}-${liq.timestamp}`} className="liquidation-item">
            <div>{liq.borrower.slice(0, 8)}...</div>
            <div>${liq.debtCovered}</div>
            <div>${liq.bonus} bonus</div>
            <div>{formatTime(liq.timestamp)}</div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

## Integration Examples

### Liquidation Protection Service
```javascript
const LiquidationProtectionService = ({ userAddress }) => {
  const { data: riskData } = useLiquidationRisk(userAddress)
  
  const protectPosition = async () => {
    if (riskData?.healthFactor < 1.2) {
      // Calculate required collateral to reach safe ratio
      const requiredCollateral = calculateRequiredCollateral(userAddress, 1.5)
      
      // Attempt to add collateral automatically
      await addCollateralAutomatically(requiredCollateral)
    }
  }
  
  useEffect(() => {
    if (riskData?.isCritical) {
      protectPosition()
    }
  }, [riskData])
  
  return (
    <div className="protection-service">
      <div>Protection Status: {riskData?.riskLevel}</div>
      <button onClick={protectPosition}>
        Auto-Protect Position
      </button>
    </div>
  )
}
```