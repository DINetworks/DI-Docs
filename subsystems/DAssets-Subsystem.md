# DAssets Subsystem - Synthetic Assets Protocol

## Overview

The DAssets subsystem enables trading of synthetic versions of real-world assets (stocks, commodities, forex, crypto) using DUSD as collateral. It includes both spot trading (DSwap) and perpetual trading (DPerp) capabilities.

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      DAssets Protocol                           │
├─────────────────────────────────────────────────────────────────┤
│        DSwap (Spot Trading)        │     DPerp (Perpetuals)     │
│  ┌─────────────┐ ┌─────────────┐   │  ┌─────────────────────┐   │
│  │    Mint     │ │    Burn     │   │  │   Position Manager  │   │
│  │   Synths    │ │   Synths    │   │  │   Leverage Trading  │   │
│  └─────────────┘ └─────────────┘   │  └─────────────────────┘   │
│  ┌─────────────┐ ┌─────────────┐   │  ┌─────────────────────┐   │
│  │    Swap     │ │   Oracle    │   │  │  Liquidity Pool     │   │
│  │   Assets    │ │   Pricing   │   │  │   (DLP Tokens)      │   │
│  └─────────────┘ └─────────────┘   │  └─────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                    Shared Infrastructure                        │
│  DUSD Base Currency  │  Oracle Module  │  Synthetic Registry   │
└─────────────────────────────────────────────────────────────────┘
```

## DSwap - Spot Trading

### Core Contract

**Contract**: `DSwap.sol`

```solidity
contract DSwap is AccessControl, ReentrancyGuard {
    IERC20 public dusd;
    IOracleModule public oracle;
    ISynthManager public synthManager;
    
    uint256 public constant MINT_FEE = 10; // 0.1%
    uint256 public constant BURN_FEE = 10; // 0.1%
    uint256 public constant SWAP_FEE = 30; // 0.3%
    
    function mintSynthetic(
        bytes32 assetId,
        uint256 dusdAmount
    ) external nonReentrant returns (uint256 synthAmount) {
        require(synthManager.isAssetActive(assetId), "Asset not active");
        
        // Get current price from oracle
        (int256 price, uint256 timestamp, bool valid) = oracle.getPrice(assetId);
        require(valid && block.timestamp - timestamp <= 3600, "Invalid price");
        
        // Calculate synthetic amount
        uint256 fee = (dusdAmount * MINT_FEE) / 10000;
        uint256 netAmount = dusdAmount - fee;
        synthAmount = (netAmount * 1e18) / uint256(price);
        
        // Transfer DUSD and mint synthetic
        dusd.transferFrom(msg.sender, address(this), dusdAmount);
        dusd.transfer(feeCollector, fee);
        
        address synthToken = synthManager.getSynthToken(assetId);
        ISyntheticToken(synthToken).mint(msg.sender, synthAmount);
        
        emit SyntheticMinted(msg.sender, assetId, dusdAmount, synthAmount, fee);
    }
    
    function burnSynthetic(
        bytes32 assetId,
        uint256 synthAmount
    ) external nonReentrant returns (uint256 dusdAmount) {
        require(synthManager.isAssetActive(assetId), "Asset not active");
        
        // Get current price from oracle
        (int256 price, uint256 timestamp, bool valid) = oracle.getPrice(assetId);
        require(valid && block.timestamp - timestamp <= 3600, "Invalid price");
        
        // Calculate DUSD amount
        dusdAmount = (synthAmount * uint256(price)) / 1e18;
        uint256 fee = (dusdAmount * BURN_FEE) / 10000;
        uint256 netAmount = dusdAmount - fee;
        
        // Burn synthetic and transfer DUSD
        address synthToken = synthManager.getSynthToken(assetId);
        ISyntheticToken(synthToken).burnFrom(msg.sender, synthAmount);
        
        dusd.transfer(msg.sender, netAmount);
        dusd.transfer(feeCollector, fee);
        
        emit SyntheticBurned(msg.sender, assetId, synthAmount, dusdAmount, fee);
    }
    
    function swapSynthetic(
        bytes32 fromAssetId,
        bytes32 toAssetId,
        uint256 fromAmount
    ) external nonReentrant returns (uint256 toAmount) {
        require(synthManager.isAssetActive(fromAssetId), "From asset not active");
        require(synthManager.isAssetActive(toAssetId), "To asset not active");
        
        // Get prices from oracle
        (int256 fromPrice, uint256 fromTimestamp, bool fromValid) = oracle.getPrice(fromAssetId);
        (int256 toPrice, uint256 toTimestamp, bool toValid) = oracle.getPrice(toAssetId);
        
        require(fromValid && block.timestamp - fromTimestamp <= 3600, "Invalid from price");
        require(toValid && block.timestamp - toTimestamp <= 3600, "Invalid to price");
        
        // Calculate swap amounts
        uint256 dusdValue = (fromAmount * uint256(fromPrice)) / 1e18;
        uint256 fee = (dusdValue * SWAP_FEE) / 10000;
        uint256 netDusdValue = dusdValue - fee;
        toAmount = (netDusdValue * 1e18) / uint256(toPrice);
        
        // Execute swap
        address fromToken = synthManager.getSynthToken(fromAssetId);
        address toToken = synthManager.getSynthToken(toAssetId);
        
        ISyntheticToken(fromToken).burnFrom(msg.sender, fromAmount);
        ISyntheticToken(toToken).mint(msg.sender, toAmount);
        
        // Collect fees in DUSD equivalent
        dusd.transferFrom(msg.sender, feeCollector, fee);
        
        emit SyntheticSwapped(msg.sender, fromAssetId, toAssetId, fromAmount, toAmount, fee);
    }
}
```

### Synthetic Asset Manager

**Contract**: `SynthManager.sol`

```solidity
contract SynthManager is AccessControl {
    struct SyntheticAsset {
        string name;
        string symbol;
        bytes32 priceId;
        address tokenAddress;
        uint256 totalSupply;
        uint256 maxSupply;
        bool isActive;
        uint256 mintFee;
        uint256 burnFee;
        uint256 lastPriceUpdate;
    }
    
    mapping(bytes32 => SyntheticAsset) public assets;
    mapping(address => bytes32) public tokenToAssetId;
    bytes32[] public assetIds;
    
    function addAsset(
        bytes32 assetId,
        string memory name,
        string memory symbol,
        bytes32 priceId,
        uint256 maxSupply
    ) external onlyRole(ADMIN_ROLE) {
        require(assets[assetId].tokenAddress == address(0), "Asset exists");
        
        // Deploy synthetic token
        address tokenAddress = address(new SyntheticToken(name, symbol, priceId));
        
        assets[assetId] = SyntheticAsset({
            name: name,
            symbol: symbol,
            priceId: priceId,
            tokenAddress: tokenAddress,
            totalSupply: 0,
            maxSupply: maxSupply,
            isActive: true,
            mintFee: 10, // 0.1%
            burnFee: 10, // 0.1%
            lastPriceUpdate: block.timestamp
        });
        
        tokenToAssetId[tokenAddress] = assetId;
        assetIds.push(assetId);
        
        emit AssetAdded(assetId, name, symbol, tokenAddress);
    }
    
    function updateAssetStatus(bytes32 assetId, bool isActive) external onlyRole(ADMIN_ROLE) {
        require(assets[assetId].tokenAddress != address(0), "Asset not found");
        assets[assetId].isActive = isActive;
        emit AssetStatusUpdated(assetId, isActive);
    }
    
    function getSynthToken(bytes32 assetId) external view returns (address) {
        return assets[assetId].tokenAddress;
    }
    
    function isAssetActive(bytes32 assetId) external view returns (bool) {
        return assets[assetId].isActive;
    }
    
    function getAssetInfo(bytes32 assetId) external view returns (SyntheticAsset memory) {
        return assets[assetId];
    }
}
```

## DPerp - Perpetual Trading

### Core Contract

**Contract**: `DPerp.sol`

```solidity
contract DPerp is AccessControl, ReentrancyGuard {
    IERC20 public dusd;
    IPositionManager public positionManager;
    IVaultPriceFeed public priceFeed;
    ILiquidityPool public liquidityPool;
    IFundingRateManager public fundingManager;
    
    uint256 public constant BASIS_POINTS = 10000;
    uint256 public constant TRADING_FEE = 10; // 0.1%
    uint256 public constant LIQUIDATION_FEE = 500; // 5%
    uint256 public constant MIN_LEVERAGE = 11000; // 1.1x
    uint256 public constant MAX_LEVERAGE = 500000; // 50x
    
    mapping(bytes32 => uint256) public maxLeverage; // Per asset
    mapping(bytes32 => uint256) public maxOpenInterest; // Per asset
    mapping(bytes32 => uint256) public globalLongSizes;
    mapping(bytes32 => uint256) public globalShortSizes;
    
    function increasePosition(
        bytes32 assetId,
        uint256 sizeDelta,
        bool isLong,
        uint256 acceptablePrice
    ) external nonReentrant {
        require(sizeDelta > 0, "Invalid size");
        
        // Update funding rates
        fundingManager.updateFundingRate(assetId);
        
        // Get execution price
        uint256 price = priceFeed.getPrice(assetId, isLong);
        require(isLong ? price <= acceptablePrice : price >= acceptablePrice, "Price slippage");
        
        // Calculate collateral needed
        uint256 leverage = (sizeDelta * BASIS_POINTS) / msg.value;
        require(leverage >= MIN_LEVERAGE && leverage <= maxLeverage[assetId], "Invalid leverage");
        
        // Check global limits
        if (isLong) {
            require(globalLongSizes[assetId] + sizeDelta <= maxOpenInterest[assetId], "Max long OI");
        } else {
            require(globalShortSizes[assetId] + sizeDelta <= maxOpenInterest[assetId], "Max short OI");
        }
        
        // Calculate fees
        uint256 tradingFee = (sizeDelta * TRADING_FEE) / BASIS_POINTS;
        uint256 collateralAfterFees = msg.value - tradingFee;
        
        // Transfer collateral
        dusd.transferFrom(msg.sender, address(this), msg.value);
        dusd.transfer(feeCollector, tradingFee);
        
        // Update position
        positionManager.increasePosition(
            msg.sender,
            assetId,
            sizeDelta,
            collateralAfterFees,
            isLong,
            price
        );
        
        // Update global sizes
        if (isLong) {
            globalLongSizes[assetId] += sizeDelta;
        } else {
            globalShortSizes[assetId] += sizeDelta;
        }
        
        // Reserve tokens from liquidity pool
        liquidityPool.reserveTokens(assetId, sizeDelta);
        
        emit PositionIncreased(
            msg.sender,
            assetId,
            sizeDelta,
            collateralAfterFees,
            isLong,
            price,
            tradingFee
        );
    }
    
    function decreasePosition(
        bytes32 assetId,
        uint256 sizeDelta,
        uint256 collateralDelta,
        bool isLong,
        address receiver,
        uint256 acceptablePrice
    ) external nonReentrant returns (uint256) {
        require(sizeDelta > 0 || collateralDelta > 0, "Invalid deltas");
        
        // Update funding rates
        fundingManager.updateFundingRate(assetId);
        
        // Get mark price
        uint256 price = priceFeed.getPrice(assetId, !isLong);
        require(isLong ? price >= acceptablePrice : price <= acceptablePrice, "Price slippage");
        
        // Calculate PnL and fees
        (uint256 usdOut, uint256 fee) = positionManager.decreasePosition(
            msg.sender,
            assetId,
            sizeDelta,
            collateralDelta,
            isLong,
            receiver,
            price
        );
        
        // Update global sizes
        if (isLong) {
            globalLongSizes[assetId] -= sizeDelta;
        } else {
            globalShortSizes[assetId] -= sizeDelta;
        }
        
        // Release tokens from liquidity pool
        liquidityPool.releaseTokens(assetId, sizeDelta);
        
        // Transfer payout
        if (usdOut > 0) {
            dusd.transfer(receiver, usdOut);
        }
        
        emit PositionDecreased(
            msg.sender,
            assetId,
            sizeDelta,
            collateralDelta,
            isLong,
            receiver,
            price,
            usdOut,
            fee
        );
        
        return usdOut;
    }
    
    function liquidatePosition(
        address account,
        bytes32 assetId,
        bool isLong
    ) external nonReentrant {
        // Update funding rates
        fundingManager.updateFundingRate(assetId);
        
        // Check if position can be liquidated
        require(positionManager.isLiquidatable(account, assetId, isLong), "Position healthy");
        
        // Get position info
        (uint256 size, uint256 collateral, , , , , ) = positionManager.getPosition(account, assetId, isLong);
        
        // Calculate liquidation fee
        uint256 liquidationFee = (collateral * LIQUIDATION_FEE) / BASIS_POINTS;
        uint256 liquidatorReward = liquidationFee / 2;
        uint256 poolReward = liquidationFee - liquidatorReward;
        
        // Execute liquidation
        positionManager.liquidatePosition(account, assetId, isLong);
        
        // Update global sizes
        if (isLong) {
            globalLongSizes[assetId] -= size;
        } else {
            globalShortSizes[assetId] -= size;
        }
        
        // Release tokens and distribute rewards
        liquidityPool.releaseTokens(assetId, size);
        dusd.transfer(msg.sender, liquidatorReward);
        dusd.transfer(address(liquidityPool), poolReward);
        
        emit PositionLiquidated(
            account,
            assetId,
            isLong,
            size,
            collateral,
            liquidationFee,
            msg.sender
        );
    }
}
```

### Position Manager

**Contract**: `PositionManager.sol`

```solidity
contract PositionManager is AccessControl {
    struct Position {
        uint256 size;
        uint256 collateral;
        uint256 entryPrice;
        uint256 entryFundingRate;
        uint256 reserveAmount;
        int256 realisedPnl;
        uint256 lastIncreasedTime;
    }
    
    mapping(bytes32 => Position) public positions; // keccak256(account, assetId, isLong)
    mapping(bytes32 => uint256) public reservedAmounts;
    mapping(bytes32 => uint256) public poolAmounts;
    mapping(bytes32 => uint256) public guaranteedUsd;
    
    function increasePosition(
        address account,
        bytes32 assetId,
        uint256 sizeDelta,
        uint256 collateralDelta,
        bool isLong,
        uint256 price
    ) external onlyRole(VAULT_ROLE) {
        bytes32 key = getPositionKey(account, assetId, isLong);
        Position storage position = positions[key];
        
        uint256 fee = _collectMarginFees(account, assetId, sizeDelta, position.size, position.entryFundingRate);
        collateralDelta -= fee;
        
        if (position.size == 0) {
            position.entryPrice = price;
            position.entryFundingRate = fundingManager.getCumulativeFundingRate(assetId);
        } else {
            position.entryPrice = _getNextAveragePrice(
                assetId,
                position.size,
                position.entryPrice,
                isLong,
                sizeDelta,
                price
            );
        }
        
        position.size += sizeDelta;
        position.collateral += collateralDelta;
        position.lastIncreasedTime = block.timestamp;
        
        _validatePosition(position, price);
        
        emit PositionIncreased(account, assetId, sizeDelta, collateralDelta, isLong, price);
    }
    
    function decreasePosition(
        address account,
        bytes32 assetId,
        uint256 sizeDelta,
        uint256 collateralDelta,
        bool isLong,
        address receiver,
        uint256 price
    ) external onlyRole(VAULT_ROLE) returns (uint256, uint256) {
        bytes32 key = getPositionKey(account, assetId, isLong);
        Position storage position = positions[key];
        
        require(position.size >= sizeDelta, "Position size exceeded");
        require(position.collateral >= collateralDelta, "Position collateral exceeded");
        
        uint256 fee = _collectMarginFees(account, assetId, sizeDelta, position.size, position.entryFundingRate);
        
        (uint256 usdOut, int256 delta) = _reduceCollateral(
            account,
            assetId,
            collateralDelta,
            sizeDelta,
            isLong,
            price
        );
        
        if (position.size != sizeDelta) {
            position.entryFundingRate = fundingManager.getCumulativeFundingRate(assetId);
            position.size -= sizeDelta;
            position.collateral -= collateralDelta;
            
            _validatePosition(position, price);
        } else {
            delete positions[key];
        }
        
        if (delta > 0) {
            position.realisedPnl += delta;
        }
        
        emit PositionDecreased(account, assetId, sizeDelta, collateralDelta, isLong, receiver, price, usdOut);
        
        return (usdOut, fee);
    }
    
    function isLiquidatable(
        address account,
        bytes32 assetId,
        bool isLong
    ) external view returns (bool) {
        bytes32 key = getPositionKey(account, assetId, isLong);
        Position memory position = positions[key];
        
        if (position.size == 0) return false;
        
        (bool hasProfit, uint256 delta) = _getDelta(assetId, position.size, position.entryPrice, isLong);
        uint256 marginFees = _getMarginFees(account, assetId, position.size, position.entryFundingRate);
        
        if (!hasProfit && position.collateral < delta) {
            return true;
        }
        
        uint256 remainingCollateral = hasProfit ? position.collateral + delta : position.collateral - delta;
        
        return remainingCollateral < marginFees + LIQUIDATION_FEE_USD;
    }
    
    function getPositionKey(address account, bytes32 assetId, bool isLong) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(account, assetId, isLong));
    }
}
```

### Liquidity Pool (DLP)

**Contract**: `LiquidityPool.sol`

```solidity
contract LiquidityPool is ERC20, AccessControl, ReentrancyGuard {
    IERC20 public dusd;
    IVaultPriceFeed public priceFeed;
    
    uint256 public constant COOLDOWN_DURATION = 15 minutes;
    uint256 public constant MAX_UTILIZATION = 8000; // 80%
    
    mapping(address => uint256) public lastAddedAt;
    mapping(bytes32 => uint256) public reservedAmounts;
    uint256 public totalReserved;
    
    constructor(address _dusd) ERC20("DI Liquidity Pool", "DLP") {
        dusd = IERC20(_dusd);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
    
    function addLiquidity(uint256 dusdAmount) external nonReentrant returns (uint256 dlpAmount) {
        require(dusdAmount > 0, "Invalid amount");
        
        uint256 aum = getAUM();
        uint256 supply = totalSupply();
        
        if (supply == 0) {
            dlpAmount = dusdAmount;
        } else {
            dlpAmount = (dusdAmount * supply) / aum;
        }
        
        dusd.transferFrom(msg.sender, address(this), dusdAmount);
        _mint(msg.sender, dlpAmount);
        
        lastAddedAt[msg.sender] = block.timestamp;
        
        emit LiquidityAdded(msg.sender, dusdAmount, dlpAmount);
    }
    
    function removeLiquidity(uint256 dlpAmount) external nonReentrant returns (uint256 dusdAmount) {
        require(dlpAmount > 0, "Invalid amount");
        require(balanceOf(msg.sender) >= dlpAmount, "Insufficient DLP");
        require(block.timestamp >= lastAddedAt[msg.sender] + COOLDOWN_DURATION, "Cooldown period");
        
        uint256 aum = getAUM();
        dusdAmount = (dlpAmount * aum) / totalSupply();
        
        // Check utilization after withdrawal
        uint256 newAum = aum - dusdAmount;
        require(totalReserved <= (newAum * MAX_UTILIZATION) / BASIS_POINTS, "Max utilization exceeded");
        
        _burn(msg.sender, dlpAmount);
        dusd.transfer(msg.sender, dusdAmount);
        
        emit LiquidityRemoved(msg.sender, dusdAmount, dlpAmount);
    }
    
    function reserveTokens(bytes32 assetId, uint256 amount) external onlyRole(VAULT_ROLE) {
        reservedAmounts[assetId] += amount;
        totalReserved += amount;
        
        require(totalReserved <= (getAUM() * MAX_UTILIZATION) / BASIS_POINTS, "Max utilization exceeded");
    }
    
    function releaseTokens(bytes32 assetId, uint256 amount) external onlyRole(VAULT_ROLE) {
        require(reservedAmounts[assetId] >= amount, "Insufficient reserved");
        reservedAmounts[assetId] -= amount;
        totalReserved -= amount;
    }
    
    function getAUM() public view returns (uint256) {
        return dusd.balanceOf(address(this));
    }
    
    function getDLPPrice() external view returns (uint256) {
        uint256 supply = totalSupply();
        if (supply == 0) return 1e18;
        return (getAUM() * 1e18) / supply;
    }
    
    function getUtilization() external view returns (uint256) {
        uint256 aum = getAUM();
        if (aum == 0) return 0;
        return (totalReserved * BASIS_POINTS) / aum;
    }
}
```

## Supported Assets

### Asset Categories

| Category | Assets | Price Feed |
|----------|--------|------------|
| **Crypto** | xBTC, xETH, xBNB, xADA, xSOL | Chainlink + Pyth |
| **Stocks** | xAAPL, xTSLA, xGOOG, xAMZN, xMSFT | Chainlink |
| **Commodities** | xGold, xSilver, xOil, xGas | Chainlink |
| **Forex** | xEUR, xGBP, xJPY, xCHF | Chainlink |

### Asset Configuration

```solidity
struct AssetConfig {
    bytes32 assetId;
    string name;
    string symbol;
    bytes32 priceId;
    uint256 maxSupply;
    uint256 maxLeverage;
    uint256 maxOpenInterest;
    uint256 fundingRateFactor;
    bool isActive;
}
```

## Integration Examples

### Spot Trading (DSwap)

```javascript
// Mint synthetic Bitcoin
const dusdAmount = ethers.parseEther("1000");
await dusd.approve(dswap.address, dusdAmount);
const synthAmount = await dswap.mintSynthetic("xBTC", dusdAmount);

// Swap synthetic assets
await dswap.swapSynthetic("xBTC", "xETH", synthAmount);

// Burn synthetic for DUSD
await dswap.burnSynthetic("xETH", synthAmount);
```

### Perpetual Trading (DPerp)

```javascript
// Open long position on BTC
const collateral = ethers.parseEther("1000");
const size = ethers.parseEther("10000"); // 10x leverage
const acceptablePrice = ethers.parseEther("50000");

await dusd.approve(dperp.address, collateral);
await dperp.increasePosition("xBTC", size, true, acceptablePrice);

// Close position
await dperp.decreasePosition("xBTC", size, 0, true, userAddress, acceptablePrice);
```

### Liquidity Provision

```javascript
// Add liquidity to DLP pool
const dusdAmount = ethers.parseEther("10000");
await dusd.approve(liquidityPool.address, dusdAmount);
const dlpAmount = await liquidityPool.addLiquidity(dusdAmount);

// Remove liquidity (after cooldown)
await liquidityPool.removeLiquidity(dlpAmount);
```

## Risk Management

### Position Limits

- **Max Leverage**: 50x for BTC/ETH, 30x for other assets
- **Max Open Interest**: $10M per asset per side
- **Maintenance Margin**: 0.5-1% of position size
- **Liquidation Fee**: $5 + 0.5% of collateral

### Pool Protection

- **Max Utilization**: 80% of pool can be reserved
- **Cooldown Period**: 15 minutes for liquidity removal
- **Dynamic Fees**: Fees increase with utilization
- **Insurance Fund**: 5% of fees for extreme losses

## Monitoring & Analytics

### Key Metrics

- **Total Synthetic Supply**: Value of all minted synthetics
- **Trading Volume**: Daily/weekly trading volume
- **Open Interest**: Total perpetual positions
- **Pool Utilization**: Percentage of pool reserved
- **Funding Rates**: Current funding rates per asset

### Events for Tracking

```solidity
event SyntheticMinted(address indexed user, bytes32 assetId, uint256 dusdAmount, uint256 synthAmount, uint256 fee);
event SyntheticBurned(address indexed user, bytes32 assetId, uint256 synthAmount, uint256 dusdAmount, uint256 fee);
event PositionIncreased(address indexed account, bytes32 assetId, uint256 sizeDelta, uint256 collateralDelta, bool isLong, uint256 price, uint256 fee);
event PositionDecreased(address indexed account, bytes32 assetId, uint256 sizeDelta, uint256 collateralDelta, bool isLong, address receiver, uint256 price, uint256 usdOut, uint256 fee);
event PositionLiquidated(address indexed account, bytes32 assetId, bool isLong, uint256 size, uint256 collateral, uint256 liquidationFee, address liquidator);
event LiquidityAdded(address indexed provider, uint256 dusdAmount, uint256 dlpAmount);
event LiquidityRemoved(address indexed provider, uint256 dusdAmount, uint256 dlpAmount);
```

---

*DAssets Subsystem - Trade the World's Assets On-Chain*