# Risk Management

DPerp implements comprehensive risk management systems to protect traders, liquidity providers, and the protocol itself. The system uses multiple layers of risk controls, monitoring, and mitigation strategies.

## Risk Framework

### Multi-Layer Risk Controls
1. **Position Level**: Individual position limits and validation
2. **User Level**: Per-user exposure and leverage limits  
3. **Asset Level**: Per-asset position caps and parameters
4. **Protocol Level**: Global exposure limits and circuit breakers

### Risk Parameters
```solidity
struct RiskParameters {
    uint256 maxLeverage;           // Maximum allowed leverage
    uint256 maxPositionSize;       // Maximum position size per user
    uint256 maxGlobalLong;         // Maximum global long exposure
    uint256 maxGlobalShort;        // Maximum global short exposure
    uint256 maintenanceMargin;     // Minimum maintenance margin
    uint256 liquidationFee;        // Liquidation penalty
    bool tradingEnabled;           // Trading pause flag
}
```

## Position Risk Controls

### Leverage Limits
```solidity
function _validateLeverage(address account, bytes32 assetId, bool isLong) private view {
    Position memory position = positionManager.getPosition(account, assetId, isLong);
    require(position.size >= position.collateral, "Invalid leverage");
    
    uint256 leverage = (position.size * BASIS_POINTS) / position.collateral;
    require(leverage >= MIN_LEVERAGE, "Leverage too low");
    require(leverage <= maxLeverage[assetId], "Leverage too high");
}
```

### Position Size Limits
```solidity
mapping(bytes32 => uint256) public maxPositionSizes;
mapping(address => mapping(bytes32 => uint256)) public userPositionSizes;

function validatePositionSize(address user, bytes32 assetId, uint256 sizeDelta) internal view {
    uint256 newSize = userPositionSizes[user][assetId] + sizeDelta;
    require(newSize <= maxPositionSizes[assetId], "Position size limit exceeded");
}
```

### Margin Requirements
```javascript
const calculateMarginRequirements = (assetId, positionSize, leverage) => {
  const maxLeverage = getMaxLeverage(assetId)
  const maintenanceMargin = positionSize / maxLeverage
  const liquidationFee = getLiquidationFee(assetId)
  
  return {
    initial: positionSize / leverage,
    maintenance: maintenanceMargin,
    liquidation: maintenanceMargin + liquidationFee,
    buffer: maintenanceMargin * 1.2 // 20% safety buffer
  }
}
```

## Global Risk Limits

### Exposure Caps
```solidity
function setMaxGlobalSize(bytes32 assetId, uint256 longSize, uint256 shortSize) 
    external onlyRole(ADMIN_ROLE) {
    maxGlobalLongSizes[assetId] = longSize;
    maxGlobalShortSizes[assetId] = shortSize;
}

function validateGlobalExposure(bytes32 assetId, uint256 sizeDelta, bool isLong) internal view {
    if (isLong) {
        require(globalLongSizes[assetId] + sizeDelta <= maxGlobalLongSizes[assetId], "Max longs exceeded");
    } else {
        require(globalShortSizes[assetId] + sizeDelta <= maxGlobalShortSizes[assetId], "Max shorts exceeded");
    }
}
```

### Concentration Risk
```javascript
const monitorConcentrationRisk = async () => {
  const assets = await getAllAssets()
  const concentrationReport = {}
  
  for (const asset of assets) {
    const totalLongs = await dPerp.globalLongSizes(keccak256(asset))
    const totalShorts = await dPerp.globalShortSizes(keccak256(asset))
    const totalExposure = totalLongs + totalShorts
    const netExposure = Math.abs(totalLongs - totalShorts)
    
    concentrationReport[asset] = {
      totalExposure,
      netExposure,
      longShortRatio: totalShorts > 0 ? totalLongs / totalShorts : Infinity,
      concentrationRisk: netExposure / totalExposure,
      riskLevel: getRiskLevel(netExposure / totalExposure)
    }
  }
  
  return concentrationReport
}
```

## Oracle Risk Management

### Price Validation
```solidity
function validatePrice(bytes32 assetId, uint256 price) internal view {
    require(price > 0, "Invalid price");
    
    uint256 lastPrice = historicalPrices[assetId];
    if (lastPrice > 0) {
        uint256 priceChange = price > lastPrice ? 
            ((price - lastPrice) * BASIS_POINTS) / lastPrice :
            ((lastPrice - price) * BASIS_POINTS) / lastPrice;
        
        require(priceChange <= maxPriceDeviation[assetId], "Price deviation too high");
    }
    
    require(block.timestamp - priceTimestamps[assetId] <= maxPriceAge[assetId], "Price too stale");
}
```

### Circuit Breakers
```javascript
const checkCircuitBreakers = async (assetId, priceChange, volumeSpike) => {
  const triggers = {
    priceDeviation: Math.abs(priceChange) > 0.1, // 10% price change
    volumeSpike: volumeSpike > 5, // 5x normal volume
    liquidationCascade: await detectLiquidationCascade(assetId)
  }
  
  if (Object.values(triggers).some(Boolean)) {
    await pauseTrading(assetId, triggers)
    await notifyRiskTeam(assetId, triggers)
    
    return {
      paused: true,
      triggers,
      resumeConditions: getResumeConditions(triggers)
    }
  }
  
  return { paused: false }
}
```

## Liquidity Risk

### Pool Utilization Monitoring
```javascript
const monitorPoolUtilization = async (assetId) => {
  const poolAmount = await positionManager.poolAmounts(keccak256(assetId))
  const reservedAmount = await positionManager.reservedAmounts(keccak256(assetId))
  const utilization = poolAmount > 0 ? reservedAmount / poolAmount : 0
  
  const riskMetrics = {
    utilization,
    availableLiquidity: poolAmount - reservedAmount,
    utilizationRisk: getUtilizationRisk(utilization),
    fundingPressure: utilization > 0.8 ? 'high' : utilization > 0.6 ? 'medium' : 'low'
  }
  
  // Adjust parameters based on utilization
  if (utilization > 0.9) {
    await increaseFundingRates(assetId)
    await reduceMaxLeverage(assetId)
  }
  
  return riskMetrics
}
```

### Liquidity Stress Testing
```javascript
const stressTestLiquidity = async (scenarios) => {
  const results = {}
  
  for (const scenario of scenarios) {
    const { name, priceShock, liquidationVolume, withdrawals } = scenario
    
    // Simulate price shock
    const liquidationsTriggered = await simulateLiquidations(priceShock)
    
    // Calculate required liquidity
    const liquidityNeeded = liquidationsTriggered.reduce((sum, liq) => sum + liq.size, 0)
    
    // Check if pool can handle stress
    const availableLiquidity = await getTotalPoolLiquidity()
    const liquidityShortfall = Math.max(0, liquidityNeeded - availableLiquidity)
    
    results[name] = {
      liquidationsTriggered: liquidationsTriggered.length,
      liquidityNeeded,
      availableLiquidity,
      liquidityShortfall,
      passed: liquidityShortfall === 0
    }
  }
  
  return results
}
```

## User Risk Management

### Risk Scoring
```javascript
const calculateUserRiskScore = async (userAddress) => {
  const positions = await getUserPositions(userAddress)
  const tradingHistory = await getTradingHistory(userAddress)
  
  let riskScore = 0
  
  // Position concentration risk
  const totalExposure = positions.reduce((sum, pos) => sum + pos.size, 0)
  const maxPosition = Math.max(...positions.map(pos => pos.size))
  const concentrationRatio = maxPosition / totalExposure
  riskScore += concentrationRatio * 30
  
  // Leverage risk
  const avgLeverage = positions.reduce((sum, pos) => sum + pos.leverage, 0) / positions.length
  riskScore += Math.min(avgLeverage / 10, 1) * 25
  
  // Liquidation history
  const liquidationRate = tradingHistory.liquidations / tradingHistory.totalTrades
  riskScore += liquidationRate * 20
  
  // P&L volatility
  const pnlVolatility = calculatePnLVolatility(tradingHistory.pnlHistory)
  riskScore += Math.min(pnlVolatility / 0.5, 1) * 25
  
  return {
    score: Math.min(riskScore, 100),
    level: getRiskLevel(riskScore),
    recommendations: generateRiskRecommendations(riskScore, positions)
  }
}
```

### Position Limits by Risk Level
```javascript
const getPositionLimits = (userRiskScore, assetCategory) => {
  const baseLimits = {
    crypto: { maxLeverage: 50, maxSize: 1000000 },
    commodities: { maxLeverage: 20, maxSize: 500000 },
    equities: { maxLeverage: 10, maxSize: 250000 },
    forex: { maxLeverage: 100, maxSize: 2000000 }
  }
  
  const riskMultipliers = {
    low: 1.0,      // 0-30 risk score
    medium: 0.7,   // 31-60 risk score  
    high: 0.4,     // 61-80 risk score
    extreme: 0.2   // 81-100 risk score
  }
  
  const riskLevel = getRiskLevel(userRiskScore)
  const multiplier = riskMultipliers[riskLevel]
  const baseLimitSet = baseLimits[assetCategory]
  
  return {
    maxLeverage: Math.floor(baseLimitSet.maxLeverage * multiplier),
    maxSize: Math.floor(baseLimitSet.maxSize * multiplier),
    riskLevel
  }
}
```

## Risk Monitoring Dashboard

### Real-time Risk Metrics
```javascript
const RiskDashboard = () => {
  const { data: riskMetrics } = useQuery(
    ['riskMetrics'],
    fetchRiskMetrics,
    { refetchInterval: 30000 }
  )
  
  return (
    <div className="risk-dashboard">
      <div className="risk-overview">
        <RiskGauge 
          label="Protocol Risk"
          value={riskMetrics?.protocolRisk}
          threshold={70}
        />
        <RiskGauge 
          label="Liquidity Risk"
          value={riskMetrics?.liquidityRisk}
          threshold={80}
        />
        <RiskGauge 
          label="Concentration Risk"
          value={riskMetrics?.concentrationRisk}
          threshold={60}
        />
      </div>
      
      <div className="asset-risks">
        {riskMetrics?.assetRisks?.map(asset => (
          <AssetRiskCard key={asset.id} asset={asset} />
        ))}
      </div>
      
      <div className="risk-alerts">
        {riskMetrics?.alerts?.map(alert => (
          <RiskAlert key={alert.id} alert={alert} />
        ))}
      </div>
    </div>
  )
}
```

### Automated Risk Responses
```javascript
const automatedRiskResponse = async (riskEvent) => {
  const { type, severity, assetId, details } = riskEvent
  
  switch (type) {
    case 'HIGH_UTILIZATION':
      if (severity === 'critical') {
        await pauseNewPositions(assetId)
        await increaseFundingRates(assetId, 2.0) // 2x multiplier
      }
      break
      
    case 'PRICE_DEVIATION':
      if (severity === 'critical') {
        await pauseTrading(assetId)
        await triggerEmergencyOracle(assetId)
      }
      break
      
    case 'LIQUIDATION_CASCADE':
      await pauseTrading(assetId)
      await activateInsuranceFund()
      await notifyEmergencyTeam(riskEvent)
      break
      
    case 'ORACLE_FAILURE':
      await pauseAllTrading()
      await switchToBackupOracle(assetId)
      break
  }
  
  // Log all responses
  await logRiskResponse(riskEvent, responses)
}
```

## Insurance and Backstops

### Insurance Fund
```solidity
contract InsuranceFund {
    mapping(bytes32 => uint256) public assetInsurance;
    uint256 public totalInsurance;
    
    function coverLoss(bytes32 assetId, uint256 amount) external onlyRole(PERP_ROLE) {
        require(assetInsurance[assetId] >= amount, "Insufficient insurance");
        assetInsurance[assetId] -= amount;
        totalInsurance -= amount;
        
        dUSD.mint(msg.sender, amount);
        emit InsuranceClaim(assetId, amount);
    }
    
    function addInsurance(bytes32 assetId, uint256 amount) external {
        dUSD.burn(msg.sender, amount);
        assetInsurance[assetId] += amount;
        totalInsurance += amount;
        
        emit InsuranceAdded(assetId, amount);
    }
}
```

### Emergency Procedures
```javascript
const emergencyProcedures = {
  async pauseAllTrading() {
    await dPerp.setIsLeverageEnabled(false)
    await notifyAllUsers('Trading paused due to emergency')
  },
  
  async liquidateAllPositions(assetId) {
    const positions = await getAllPositions(assetId)
    for (const position of positions) {
      if (await isLiquidatable(position)) {
        await dPerp.liquidatePosition(position.account, assetId, position.isLong)
      }
    }
  },
  
  async activateInsuranceFund() {
    const shortfall = await calculateSystemShortfall()
    if (shortfall > 0) {
      await insuranceFund.coverLoss(shortfall)
    }
  }
}
```

## Risk Reporting

### Daily Risk Report
```javascript
const generateDailyRiskReport = async () => {
  const report = {
    date: new Date().toISOString().split('T')[0],
    protocolMetrics: await getProtocolRiskMetrics(),
    assetMetrics: await getAssetRiskMetrics(),
    userMetrics: await getUserRiskMetrics(),
    liquidations: await getDailyLiquidations(),
    alerts: await getRiskAlerts(),
    recommendations: await generateRiskRecommendations()
  }
  
  await saveRiskReport(report)
  await notifyRiskTeam(report)
  
  return report
}
```

### Risk Alerts
```javascript
const RiskAlert = ({ alert }) => {
  const getSeverityColor = (severity) => {
    const colors = {
      low: 'green',
      medium: 'yellow', 
      high: 'orange',
      critical: 'red'
    }
    return colors[severity] || 'gray'
  }
  
  return (
    <div className={`risk-alert ${alert.severity}`}>
      <div className="alert-header">
        <span className={`severity-badge ${alert.severity}`}>
          {alert.severity.toUpperCase()}
        </span>
        <span className="alert-time">
          {formatTime(alert.timestamp)}
        </span>
      </div>
      
      <div className="alert-content">
        <h4>{alert.title}</h4>
        <p>{alert.description}</p>
        
        {alert.actions && (
          <div className="alert-actions">
            {alert.actions.map(action => (
              <button key={action.id} onClick={() => executeAction(action)}>
                {action.label}
              </button>
            ))}
          </div>
        )}
      </div>
    </div>
  )
}
```