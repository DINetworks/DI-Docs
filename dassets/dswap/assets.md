# Supported Assets

DSwap supports a diverse range of synthetic assets as virtual positions tracked in smart contracts, enabling efficient exposure to global markets without token deployment costs.

## Virtual Asset System

### No ERC20 Tokens
Unlike traditional synthetic protocols, DSwap uses virtual asset tracking:
- **Position Tracking**: Balances stored in smart contract mappings
- **No Token Deployment**: Eliminates gas costs and complexity
- **Instant Addition**: New assets can be added immediately
- **Efficient Operations**: Direct position updates without token transfers

### Asset Position Structure
```solidity
// Virtual asset positions
mapping(address => mapping(bytes32 => uint256)) public userPositions;

struct AssetInfo {
    bytes32 assetId;        // Unique asset identifier
    string name;            // Full asset name
    string symbol;          // Trading symbol
    bytes32 priceId;        // Oracle price feed ID
    uint256 totalValue;     // Total USD value of positions
    bool active;            // Trading enabled/disabled
}
```

## Asset Categories

### Equities (Primary Focus)
Major publicly traded companies:

| Symbol | Name | Market | Price Feed | Market Hours |
|--------|------|--------|------------|--------------|
| xAAPL | Apple Inc. | NASDAQ | AAPL/USD | 9:30-16:00 EST |
| xTSLA | Tesla Inc. | NASDAQ | TSLA/USD | 9:30-16:00 EST |
| xMSFT | Microsoft Corp. | NASDAQ | MSFT/USD | 9:30-16:00 EST |
| xAMZN | Amazon.com Inc. | NASDAQ | AMZN/USD | 9:30-16:00 EST |
| xGOOGL | Alphabet Inc. | NASDAQ | GOOGL/USD | 9:30-16:00 EST |

### Commodities
Physical assets and futures contracts:

| Symbol | Name | Contract | Price Feed | Trading Hours |
|--------|------|----------|------------|---------------|
| xGOLD | Gold Futures | GC | XAU/USD | 24/7 |
| xSILVER | Silver Futures | SI | XAG/USD | 24/7 |
| xOIL | Crude Oil Futures | CL | CL/USD | 24/7 |
| xCOPPER | Copper Futures | HG | HG/USD | 24/7 |

### Indices
Market indices and benchmarks:

| Symbol | Name | Components | Price Feed | Calculation |
|--------|------|------------|------------|-------------|
| xSP500 | S&P 500 Index | 500 stocks | SPX/USD | Market cap weighted |
| xNASDAQ | NASDAQ Composite | Tech-heavy | IXIC/USD | Market cap weighted |
| xDOW | Dow Jones Industrial | 30 blue chips | DJI/USD | Price weighted |

### Crypto Synthetics
Synthetic versions of major cryptocurrencies:

| Symbol | Name | Price Feed | Availability |
|--------|------|------------|--------------|
| xBTC | Synthetic Bitcoin | BTC/USD | 24/7 |
| xETH | Synthetic Ethereum | ETH/USD | 24/7 |

## Asset Properties

### Universal Features
All synthetic assets share common properties:
- **Oracle-based pricing**: Real-time price feeds
- **Virtual positions**: No token contracts required
- **Instant settlement**: Immediate position updates
- **Cross-asset swapping**: Universal compatibility
- **Settlement locks**: 1-minute cooldown after operations

### Asset Information Queries
```solidity
function getAssetInfo(bytes32 assetId) external view returns (AssetInfo memory) {
    return assetInfos[assetId];
}

function getAllAssets() external view returns (bytes32[] memory) {
    return assetIds;
}

function getUserPosition(address user, bytes32 assetId) external view returns (uint256) {
    return userPositions[user][assetId];
}
```

## Asset Addition Process

### Instant Asset Deployment
New assets can be added without token deployment:
```solidity
function addAsset(
    bytes32 assetId,
    string memory name,
    string memory symbol,
    bytes32 priceId
) external onlyRole(ADMIN_ROLE) {
    require(assetInfos[assetId].assetId == bytes32(0), "Asset exists");
    
    assetInfos[assetId] = AssetInfo({
        assetId: assetId,
        name: name,
        symbol: symbol,
        priceId: priceId,
        totalValue: 0,
        active: true
    });
    
    assetIds.push(assetId);
    
    emit AssetAdded(assetId, name, symbol, priceId);
}
```

### Asset Lifecycle Management
```javascript
// Add new asset (governance-controlled)
const addNewAsset = async (assetDetails) => {
  const proposal = {
    title: `Add ${assetDetails.name} Synthetic Asset`,
    description: `Add ${assetDetails.symbol} for trading`,
    targets: [DSWAP_ADDRESS],
    values: [0],
    calldatas: [
      dswap.interface.encodeFunctionData("addAsset", [
        keccak256(assetDetails.symbol),
        assetDetails.name,
        assetDetails.symbol,
        assetDetails.priceId
      ])
    ]
  }
  
  return await governance.propose(proposal)
}
```

## Asset Queries and Analytics

### Position Tracking
```javascript
// Get user's synthetic asset positions
const getUserPositions = async (userAddress) => {
  const allAssets = await dswap.getAllAssets()
  const positions = []
  
  for (const assetId of allAssets) {
    const balance = await dswap.getUserPosition(userAddress, assetId)
    if (balance > 0) {
      const assetInfo = await dswap.getAssetInfo(assetId)
      const price = await oracle.getPrice(assetInfo.priceId)
      
      positions.push({
        assetId,
        symbol: assetInfo.symbol,
        balance: formatEther(balance),
        value: (Number(balance) * Number(price)) / 1e26, // Convert to USD
        price: Number(price) / 1e8
      })
    }
  }
  
  return positions
}
```

### Asset Analytics
```javascript
const getAssetAnalytics = async (assetId) => {
  const assetInfo = await dswap.getAssetInfo(assetId)
  const totalValue = assetInfo.totalValue
  const holders = await getAssetHolders(assetId)
  const volume = await getAssetVolume(assetId, '24h')
  
  return {
    symbol: assetInfo.symbol,
    totalValue: formatEther(totalValue),
    holders: holders.length,
    volume24h: formatEther(volume),
    marketShare: (Number(totalValue) / await getTotalSyntheticValue()) * 100
  }
}
```

## Market Data Integration

### Real-Time Price Feeds
```javascript
const AssetPriceWidget = ({ assetId }) => {
  const { data: priceData } = useQuery(
    ['assetPrice', assetId],
    () => oracle.getPriceData(assetId),
    { refetchInterval: 30000 }
  )
  
  const price = priceData ? Number(priceData.price) / 1e8 : 0
  const change24h = calculatePriceChange(priceData, '24h')
  
  return (
    <div className="price-widget">
      <div className="current-price">${price.toFixed(2)}</div>
      <div className={`price-change ${change24h >= 0 ? 'positive' : 'negative'}`}>
        {change24h >= 0 ? '+' : ''}{change24h.toFixed(2)}%
      </div>
      <div className="last-update">
        Updated: {formatTime(priceData?.timestamp)}
      </div>
    </div>
  )
}
```

### Market Hours Display
```javascript
const MarketStatus = ({ assetId }) => {
  const { data: marketHours } = useQuery(
    ['marketHours', assetId],
    () => getMarketHours(assetId)
  )
  
  const isOpen = marketHours?.isOpen
  const nextEvent = isOpen ? marketHours.closeTime : marketHours.openTime
  
  return (
    <div className={`market-status ${isOpen ? 'open' : 'closed'}`}>
      <div className="status-indicator">
        {isOpen ? '🟢 Market Open' : '🔴 Market Closed'}
      </div>
      <div className="next-event">
        {isOpen ? 'Closes' : 'Opens'} at {formatTime(nextEvent)}
      </div>
    </div>
  )
}
```

## Asset Risk Parameters

### Risk Classification
```javascript
const assetRiskProfiles = {
  equities: {
    volatility: 'medium-high',
    liquidity: 'high',
    marketHours: 'limited',
    oracleRisk: 'low'
  },
  commodities: {
    volatility: 'high',
    liquidity: 'medium',
    marketHours: '24/7',
    oracleRisk: 'medium'
  },
  indices: {
    volatility: 'medium',
    liquidity: 'high',
    marketHours: 'limited',
    oracleRisk: 'low'
  },
  crypto: {
    volatility: 'very-high',
    liquidity: 'high',
    marketHours: '24/7',
    oracleRisk: 'low'
  }
}
```

### Dynamic Risk Adjustments
```solidity
// Adjust parameters based on market conditions
function updateAssetRiskParameters(bytes32 assetId) external onlyKeeper {
    uint256 volatility = calculateVolatility(assetId);
    uint256 liquidity = assessLiquidity(assetId);
    
    if (volatility > HIGH_VOLATILITY_THRESHOLD) {
        // Increase fees during high volatility
        assetFeeMultipliers[assetId] = 150; // 1.5x
    }
    
    if (liquidity < LOW_LIQUIDITY_THRESHOLD) {
        // Reduce position limits for illiquid assets
        maxPositionSizes[assetId] = maxPositionSizes[assetId] * 50 / 100;
    }
}
```

## Integration Examples

### Asset Selection Interface
```javascript
const AssetSelector = ({ onSelect, category }) => {
  const { data: assets } = useQuery(
    ['assets', category],
    () => getAssetsByCategory(category)
  )
  
  return (
    <div className="asset-grid">
      {assets?.map(asset => (
        <AssetCard
          key={asset.assetId}
          asset={asset}
          onClick={() => onSelect(asset)}
        />
      ))}
    </div>
  )
}

const AssetCard = ({ asset, onClick }) => {
  const { data: price } = useAssetPrice(asset.assetId)
  const { data: volume } = useAssetVolume(asset.assetId)
  
  return (
    <div className="asset-card" onClick={onClick}>
      <div className="asset-header">
        <span className="symbol">{asset.symbol}</span>
        <span className="price">${price?.toFixed(2)}</span>
      </div>
      <div className="asset-details">
        <div className="name">{asset.name}</div>
        <div className="volume">24h Volume: ${volume?.toLocaleString()}</div>
      </div>
    </div>
  )
}
```

### Portfolio Overview
```javascript
const SyntheticPortfolio = ({ userAddress }) => {
  const { data: positions } = useQuery(
    ['userPositions', userAddress],
    () => getUserPositions(userAddress)
  )
  
  const totalValue = positions?.reduce((sum, pos) => sum + pos.value, 0) || 0
  
  return (
    <div className="synthetic-portfolio">
      <div className="portfolio-header">
        <h3>Synthetic Assets Portfolio</h3>
        <div className="total-value">${totalValue.toLocaleString()}</div>
      </div>
      
      <div className="position-list">
        {positions?.map(position => (
          <div key={position.assetId} className="position-item">
            <div className="asset-info">
              <span className="symbol">{position.symbol}</span>
              <span className="balance">{position.balance}</span>
            </div>
            <div className="value-info">
              <span className="price">${position.price.toFixed(2)}</span>
              <span className="value">${position.value.toLocaleString()}</span>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

## Future Asset Expansion

### Planned Additions
Assets under consideration for future deployment:
- **More Equities**: xNVDA, xMETA, xNFLX, xDIS
- **International Equities**: xNIKKEI, xFTSE, xDAX
- **Additional Commodities**: xNATGAS, xCORN, xSOY
- **Crypto Indices**: xDEFI, xNFT, xMEME
- **Forex Pairs**: xEUR, xGBP, xJPY

### Community Asset Proposals
```javascript
// Propose new synthetic asset
const proposeNewAsset = async (assetDetails) => {
  const proposal = {
    title: `Add ${assetDetails.name} Synthetic Asset`,
    description: `
      Asset: ${assetDetails.name} (${assetDetails.symbol})
      Oracle: ${assetDetails.oracleProvider}
      Market Cap: $${assetDetails.marketCap.toLocaleString()}
      Liquidity: $${assetDetails.dailyVolume.toLocaleString()} daily
      Rationale: ${assetDetails.rationale}
    `,
    targets: [DSWAP_ADDRESS],
    values: [0],
    calldatas: [
      dswap.interface.encodeFunctionData("addAsset", [
        keccak256(assetDetails.symbol),
        assetDetails.name,
        assetDetails.symbol,
        assetDetails.priceId
      ])
    ]
  }
  
  return await governance.propose(proposal)
}
```

## Asset Management

### Activation/Deactivation
```solidity
function setAssetStatus(bytes32 assetId, bool active) external onlyRole(ADMIN_ROLE) {
    require(assetInfos[assetId].assetId != bytes32(0), "Asset not found");
    assetInfos[assetId].active = active;
    
    emit AssetStatusUpdated(assetId, active);
}
```

### Asset Metrics Tracking
```javascript
const trackAssetMetrics = async (assetId) => {
  const metrics = {
    totalPositions: await getTotalPositions(assetId),
    uniqueHolders: await getUniqueHolders(assetId),
    averagePosition: await getAveragePositionSize(assetId),
    dailyVolume: await getDailyVolume(assetId),
    priceVolatility: await calculateVolatility(assetId, '30d')
  }
  
  return metrics
}
```

## Oracle Integration

### Price Feed Requirements
Each asset requires reliable oracle integration:
- **Primary Feed**: Chainlink or Pyth price feed
- **Backup Feed**: Secondary oracle for redundancy
- **Update Frequency**: Minimum update frequency requirements
- **Deviation Limits**: Maximum allowed price deviations

### Oracle Configuration
```solidity
struct OracleConfig {
    bytes32 primaryFeed;    // Primary oracle feed ID
    bytes32 backupFeed;     // Backup oracle feed ID
    uint256 maxAge;         // Maximum price age in seconds
    uint256 maxDeviation;   // Maximum price deviation (basis points)
    bool requiresBoth;      // Require both oracles to agree
}

mapping(bytes32 => OracleConfig) public oracleConfigs;
```

## Asset Lifecycle

### Addition Process
1. **Community Proposal**: Governance proposal for new asset
2. **Technical Review**: Oracle availability and reliability assessment
3. **Risk Assessment**: Volatility and liquidity analysis
4. **Governance Vote**: Community approval required
5. **Oracle Integration**: Price feed setup and testing
6. **Activation**: Asset becomes available for trading

### Deprecation Process
Assets may be deprecated due to:
- **Oracle Issues**: Unreliable or discontinued price feeds
- **Low Usage**: Insufficient trading volume or interest
- **Regulatory Concerns**: Legal or compliance issues
- **Technical Problems**: Smart contract or integration issues

### Asset Status Management
```javascript
const manageAssetStatus = async (assetId, newStatus, reason) => {
  // Only governance can change asset status
  const proposal = {
    title: `${newStatus ? 'Activate' : 'Deactivate'} ${assetId}`,
    description: `Reason: ${reason}`,
    targets: [DSWAP_ADDRESS],
    values: [0],
    calldatas: [
      dswap.interface.encodeFunctionData("setAssetStatus", [assetId, newStatus])
    ]
  }
  
  return await governance.propose(proposal)
}
```

## Integration Examples

### Asset Discovery
```javascript
const useAssetDiscovery = (searchTerm, category) => {
  return useQuery({
    queryKey: ['assetDiscovery', searchTerm, category],
    queryFn: async () => {
      const allAssets = await dswap.getAllAssets()
      const assetDetails = await Promise.all(
        allAssets.map(id => dswap.getAssetInfo(id))
      )
      
      return assetDetails
        .filter(asset => asset.active)
        .filter(asset => !category || getAssetCategory(asset) === category)
        .filter(asset => !searchTerm || 
          asset.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
          asset.symbol.toLowerCase().includes(searchTerm.toLowerCase())
        )
    }
  })
}
```

### Asset Performance Tracking
```javascript
const AssetPerformanceTracker = ({ assetId, timeframe }) => {
  const { data: performance } = useQuery(
    ['assetPerformance', assetId, timeframe],
    () => getAssetPerformance(assetId, timeframe)
  )
  
  return (
    <div className="performance-tracker">
      <div className="performance-metrics">
        <div>Price Change: {performance?.priceChange?.toFixed(2)}%</div>
        <div>Volume: ${performance?.volume?.toLocaleString()}</div>
        <div>Volatility: {performance?.volatility?.toFixed(2)}%</div>
        <div>Holders: {performance?.holders}</div>
      </div>
      
      <div className="performance-chart">
        <PriceChart data={performance?.priceHistory} />
      </div>
    </div>
  )
}
```

## Risk Considerations

### Asset-Specific Risks
**Equity Assets**:
- Market hours limitations
- Earnings and news impact
- Sector correlation risks
- Regulatory changes

**Commodity Assets**:
- High volatility
- Supply/demand shocks
- Geopolitical impacts
- Storage and delivery complexities

**Index Assets**:
- Component stock impacts
- Rebalancing effects
- Market-wide movements
- Calculation methodology changes

**Crypto Assets**:
- Extreme volatility
- Regulatory uncertainty
- Technical risks
- Market manipulation

### Risk Mitigation
- **Diversification**: Spread exposure across asset categories
- **Position Limits**: Respect maximum position sizes
- **Oracle Monitoring**: Track price feed health and accuracy
- **Market Awareness**: Understand asset-specific market dynamics

## Future Developments

### Planned Features
- **More Asset Categories**: REITs, bonds, currencies
- **Enhanced Analytics**: Advanced charting and analysis tools
- **Social Trading**: Copy trading and strategy sharing
- **Mobile Optimization**: Native mobile app integration

### Community Involvement
- Propose new assets through governance
- Participate in asset risk assessments
- Provide feedback on asset performance
- Help educate users about different asset classes