# DPerp Subsystem - Perpetual Trading Protocol

## Overview

DPerp is the perpetual trading component within the DAssets ecosystem, providing GMX-style leveraged trading with up to 50x leverage. It uses liquidity pools as counterparty and implements advanced risk management mechanisms.

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        DPerp Protocol                           │
├─────────────────────────────────────────────────────────────────┤
│     Trading Engine     │    Position Management    │   Pricing   │
│  ┌─────────────────┐   │  ┌─────────────────────┐  │ ┌─────────┐ │
│  │ Order Execution │   │  │  Position Manager   │  │ │ Oracle  │ │
│  │ Leverage Control│   │  │  PnL Calculation    │  │ │ Pricing │ │
│  └─────────────────┘   │  └─────────────────────┘  │ └─────────┘ │
│  ┌─────────────────┐   │  ┌─────────────────────┐  │ ┌─────────┐ │
│  │ Risk Management │   │  │ Liquidation Engine  │  │ │ Mark    │ │
│  │ Global Limits   │   │  │ Health Monitoring   │  │ │ Price   │ │
│  └─────────────────┘   │  └─────────────────────┘  │ └─────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                    Liquidity Infrastructure                     │
│  DLP Pool (Counterparty)  │  Funding Rates  │  Fee Collection  │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. DPerp Main Contract

**Contract**: `DPerp.sol`

```solidity
contract DPerp is AccessControl, ReentrancyGuard, Pausable {
    IERC20 public dusd;
    IPositionManager public positionManager;
    IVaultPriceFeed public priceFeed;
    ILiquidityPool public liquidityPool;
    IFundingRateManager public fundingManager;
    
    uint256 public constant BASIS_POINTS = 10000;
    uint256 public constant TRADING_FEE = 10; // 0.1%
    uint256 public constant LIQUIDATION_FEE_USD = 5e18; // $5
    uint256 public constant LIQUIDATION_FEE_BASIS_POINTS = 50; // 0.5%
    uint256 public constant MIN_PROFIT_TIME = 3 hours;
    
    mapping(bytes32 => uint256) public maxLeverage;
    mapping(bytes32 => uint256) public maxGlobalLongSizes;
    mapping(bytes32 => uint256) public maxGlobalShortSizes;
    mapping(bytes32 => uint256) public globalLongSizes;
    mapping(bytes32 => uint256) public globalShortSizes;
    mapping(address => bool) public isLiquidator;
    mapping(address => bool) public isManager;
    
    bool public includeAmmPrice = true;
    bool public inManagerMode = false;
    
    function increasePosition(
        address[] memory _path,
        bytes32 _indexToken,
        uint256 _amountIn,
        uint256 _minOut,
        uint256 _sizeDelta,
        bool _isLong,
        uint256 _acceptablePrice
    ) external nonReentrant whenNotPaused {
        if (inManagerMode) {
            require(isManager[msg.sender], "Manager only");
        }
        
        require(_sizeDelta > 0, "Invalid size delta");
        
        // Update funding rates
        fundingManager.updateFundingRate(_indexToken);
        
        // Validate leverage
        uint256 leverage = (_sizeDelta * BASIS_POINTS) / _amountIn;
        require(leverage >= 11000, "Min leverage 1.1x"); // 1.1x min
        require(leverage <= maxLeverage[_indexToken], "Max leverage exceeded");
        
        // Check global limits
        if (_isLong) {
            require(globalLongSizes[_indexToken] + _sizeDelta <= maxGlobalLongSizes[_indexToken], "Max global longs");
        } else {
            require(globalShortSizes[_indexToken] + _sizeDelta <= maxGlobalShortSizes[_indexToken], "Max global shorts");
        }
        
        // Get execution price
        uint256 price = priceFeed.getPrice(_indexToken, _isLong, includeAmmPrice);
        require(_isLong ? price <= _acceptablePrice : price >= _acceptablePrice, "Slippage exceeded");
        
        // Transfer collateral
        dusd.transferFrom(msg.sender, address(this), _amountIn);
        
        // Calculate and collect fees
        uint256 fee = _collectMarginFees(msg.sender, _path, _indexToken, _isLong, _sizeDelta, positionManager.getPositionSize(msg.sender, _indexToken, _isLong));
        uint256 collateralDelta = _amountIn - fee;
        
        // Update position
        positionManager.increasePosition(msg.sender, _indexToken, _sizeDelta, collateralDelta, _isLong, price);
        
        // Update global sizes
        if (_isLong) {
            globalLongSizes[_indexToken] += _sizeDelta;
        } else {
            globalShortSizes[_indexToken] += _sizeDelta;
        }
        
        // Reserve tokens from pool
        liquidityPool.reserveTokens(_indexToken, _sizeDelta);
        
        emit IncreasePosition(
            msg.sender,
            _indexToken,
            _amountIn,
            _sizeDelta,
            _isLong,
            price,
            fee
        );
    }
    
    function decreasePosition(
        address _account,
        bytes32 _indexToken,
        uint256 _collateralDelta,
        uint256 _sizeDelta,
        bool _isLong,
        address _receiver,
        uint256 _acceptablePrice
    ) external nonReentrant whenNotPaused returns (uint256) {
        if (inManagerMode) {
            require(isManager[msg.sender], "Manager only");
        }
        
        require(_account == msg.sender || isLiquidator[msg.sender], "Invalid account");
        
        // Update funding rates
        fundingManager.updateFundingRate(_indexToken);
        
        // Get mark price
        uint256 price = priceFeed.getPrice(_indexToken, !_isLong, includeAmmPrice);
        require(_isLong ? price >= _acceptablePrice : price <= _acceptablePrice, "Slippage exceeded");
        
        // Check minimum profit time for profits
        uint256 lastIncreasedTime = positionManager.getPositionLastIncreasedTime(_account, _indexToken, _isLong);
        if (block.timestamp < lastIncreasedTime + MIN_PROFIT_TIME) {
            (bool hasProfit, ) = positionManager.getDelta(_indexToken, positionManager.getPositionSize(_account, _indexToken, _isLong), positionManager.getPositionEntryPrice(_account, _indexToken, _isLong), _isLong, price);
            require(!hasProfit, "Min profit time not reached");
        }
        
        // Execute position decrease
        uint256 usdOut = positionManager.decreasePosition(_account, _indexToken, _collateralDelta, _sizeDelta, _isLong, _receiver, price);
        
        // Update global sizes
        if (_isLong) {
            globalLongSizes[_indexToken] -= _sizeDelta;
        } else {
            globalShortSizes[_indexToken] -= _sizeDelta;
        }
        
        // Release tokens from pool
        liquidityPool.releaseTokens(_indexToken, _sizeDelta);
        
        // Transfer payout
        if (usdOut > 0) {
            dusd.transfer(_receiver, usdOut);
        }
        
        emit DecreasePosition(
            _account,
            _indexToken,
            _collateralDelta,
            _sizeDelta,
            _isLong,
            _receiver,
            price,
            usdOut
        );
        
        return usdOut;
    }
    
    function liquidatePosition(
        address _account,
        bytes32 _indexToken,
        bool _isLong,
        address _feeReceiver
    ) external nonReentrant whenNotPaused {
        require(isLiquidator[msg.sender] || msg.sender == _account, "Invalid liquidator");
        
        // Update funding rates
        fundingManager.updateFundingRate(_indexToken);
        
        // Validate liquidation
        (uint256 liquidationState, uint256 marginFees) = positionManager.validateLiquidation(_account, _indexToken, _isLong, true);
        require(liquidationState != 0, "Position cannot be liquidated");
        
        // Get position info
        (uint256 size, uint256 collateral, , , , , ) = positionManager.getPosition(_account, _indexToken, _isLong);
        
        // Calculate liquidation fee
        uint256 liquidationFeeUsd = LIQUIDATION_FEE_USD;
        if (collateral > liquidationFeeUsd) {
            uint256 liquidationFeeBasisPoints = (collateral - liquidationFeeUsd) * LIQUIDATION_FEE_BASIS_POINTS / BASIS_POINTS;
            liquidationFeeUsd += liquidationFeeBasisPoints;
        }
        
        // Execute liquidation
        positionManager.liquidatePosition(_account, _indexToken, _isLong, _feeReceiver);
        
        // Update global sizes
        if (_isLong) {
            globalLongSizes[_indexToken] -= size;
        } else {
            globalShortSizes[_indexToken] -= size;
        }
        
        // Release tokens from pool
        liquidityPool.releaseTokens(_indexToken, size);
        
        // Pay liquidation fee
        dusd.transfer(_feeReceiver, liquidationFeeUsd);
        
        emit LiquidatePosition(
            _account,
            _indexToken,
            _isLong,
            size,
            collateral,
            liquidationFeeUsd,
            _feeReceiver
        );
    }
    
    function _collectMarginFees(
        address _account,
        address[] memory _path,
        bytes32 _indexToken,
        bool _isLong,
        uint256 _sizeDelta,
        uint256 _size
    ) private returns (uint256) {
        uint256 feeUsd = _getPositionFee(_account, _indexToken, _isLong, _sizeDelta);
        
        uint256 fundingFee = fundingManager.getFundingFee(_account, _indexToken, _isLong, _size);
        feeUsd += fundingFee;
        
        dusd.transfer(feeCollector, feeUsd);
        
        emit CollectMarginFees(feeUsd, fundingFee);
        
        return feeUsd;
    }
    
    function _getPositionFee(
        address _account,
        bytes32 _indexToken,
        bool _isLong,
        uint256 _sizeDelta
    ) private view returns (uint256) {
        if (_sizeDelta == 0) return 0;
        
        uint256 afterFeeUsd = (_sizeDelta * (BASIS_POINTS - TRADING_FEE)) / BASIS_POINTS;
        return _sizeDelta - afterFeeUsd;
    }
}
```

### 2. Position Manager

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
    
    mapping(bytes32 => Position) public positions;
    mapping(bytes32 => uint256) public reservedAmounts;
    mapping(bytes32 => uint256) public poolAmounts;
    mapping(bytes32 => uint256) public guaranteedUsd;
    
    IVaultPriceFeed public priceFeed;
    IFundingRateManager public fundingManager;
    
    uint256 public constant BASIS_POINTS = 10000;
    uint256 public constant FUNDING_RATE_PRECISION = 1000000;
    uint256 public constant LIQUIDATION_FEE_USD = 5e18;
    uint256 public constant MARGIN_FEE_BASIS_POINTS = 10;
    
    function increasePosition(
        address _account,
        bytes32 _indexToken,
        uint256 _sizeDelta,
        uint256 _collateralDelta,
        bool _isLong,
        uint256 _price
    ) external onlyRole(VAULT_ROLE) {
        bytes32 key = getPositionKey(_account, _indexToken, _isLong);
        Position storage position = positions[key];
        
        uint256 fee = _collectMarginFees(_account, _indexToken, _sizeDelta, position.size, position.entryFundingRate);
        _collateralDelta = _collateralDelta - fee;
        
        if (position.size == 0) {
            position.entryPrice = _price;
            position.entryFundingRate = fundingManager.getCumulativeFundingRate(_indexToken);
        } else {
            position.entryPrice = getNextAveragePrice(_indexToken, position.size, position.entryPrice, _isLong, _sizeDelta, _price);
        }
        
        position.size += _sizeDelta;
        position.collateral += _collateralDelta;
        position.lastIncreasedTime = block.timestamp;
        
        require(position.size > 0, "Position size must be greater than 0");
        
        _validatePosition(position, _price);
        
        // Update reserved amounts
        uint256 reserveDelta = (_sizeDelta * position.entryPrice) / priceFeed.PRICE_PRECISION();
        position.reserveAmount += reserveDelta;
        reservedAmounts[_indexToken] += reserveDelta;
        
        if (_isLong) {
            guaranteedUsd[_indexToken] += _sizeDelta;
        }
        
        emit IncreasePosition(_account, _indexToken, _collateralDelta, _sizeDelta, _isLong, _price, fee);
    }
    
    function decreasePosition(
        address _account,
        bytes32 _indexToken,
        uint256 _collateralDelta,
        uint256 _sizeDelta,
        bool _isLong,
        address _receiver,
        uint256 _price
    ) external onlyRole(VAULT_ROLE) returns (uint256) {
        bytes32 key = getPositionKey(_account, _indexToken, _isLong);
        Position storage position = positions[key];
        
        require(position.size >= _sizeDelta, "Position size exceeded");
        require(position.collateral >= _collateralDelta, "Position collateral exceeded");
        
        uint256 fee = _collectMarginFees(_account, _indexToken, _sizeDelta, position.size, position.entryFundingRate);
        
        (uint256 usdOut, int256 delta) = _reduceCollateral(_account, _indexToken, _collateralDelta, _sizeDelta, _isLong, _price);
        
        if (position.size != _sizeDelta) {
            position.entryFundingRate = fundingManager.getCumulativeFundingRate(_indexToken);
            position.size -= _sizeDelta;
            position.collateral -= _collateralDelta;
            
            _validatePosition(position, _price);
        } else {
            delete positions[key];
        }
        
        // Update reserved amounts
        uint256 reserveDelta = (position.reserveAmount * _sizeDelta) / (position.size + _sizeDelta);
        position.reserveAmount -= reserveDelta;
        reservedAmounts[_indexToken] -= reserveDelta;
        
        if (_isLong) {
            guaranteedUsd[_indexToken] -= _sizeDelta;
        }
        
        if (delta > 0) {
            position.realisedPnl += delta;
        }
        
        emit DecreasePosition(_account, _indexToken, _collateralDelta, _sizeDelta, _isLong, _receiver, _price, usdOut);
        
        return usdOut;
    }
    
    function liquidatePosition(
        address _account,
        bytes32 _indexToken,
        bool _isLong,
        address _feeReceiver
    ) external onlyRole(VAULT_ROLE) {
        bytes32 key = getPositionKey(_account, _indexToken, _isLong);
        Position storage position = positions[key];
        
        require(position.size > 0, "Position does not exist");
        
        (uint256 liquidationState, uint256 marginFees) = validateLiquidation(_account, _indexToken, _isLong, true);
        require(liquidationState != 0, "Position cannot be liquidated");
        
        uint256 feeTokens = (marginFees * poolAmounts[_indexToken]) / guaranteedUsd[_indexToken];
        
        // Update reserved amounts
        reservedAmounts[_indexToken] -= position.reserveAmount;
        
        if (_isLong) {
            guaranteedUsd[_indexToken] -= position.size;
        }
        
        delete positions[key];
        
        emit LiquidatePosition(_account, _indexToken, _isLong, position.size, position.collateral, position.reserveAmount, position.realisedPnl, marginFees);
    }
    
    function validateLiquidation(
        address _account,
        bytes32 _indexToken,
        bool _isLong,
        bool _raise
    ) public view returns (uint256, uint256) {
        bytes32 key = getPositionKey(_account, _indexToken, _isLong);
        Position memory position = positions[key];
        
        (bool hasProfit, uint256 delta) = getDelta(_indexToken, position.size, position.entryPrice, _isLong, priceFeed.getLatestPrimaryPrice(_indexToken));
        uint256 marginFees = getFundingFee(_account, _indexToken, _isLong, position.size, position.entryFundingRate);
        marginFees += getPositionFee(position.size);
        
        if (!hasProfit && position.collateral < delta) {
            if (_raise) revert("Losses exceed collateral");
            return (1, marginFees);
        }
        
        uint256 remainingCollateral = hasProfit ? position.collateral + delta : position.collateral - delta;
        
        if (remainingCollateral < marginFees + LIQUIDATION_FEE_USD) {
            if (_raise) revert("Fees exceed collateral");
            return (1, marginFees);
        }
        
        if (remainingCollateral < (marginFees + LIQUIDATION_FEE_USD)) {
            if (_raise) revert("Liquidation fees exceed collateral");
            return (1, marginFees);
        }
        
        if (remainingCollateral * maxLeverage[_indexToken] < position.size * BASIS_POINTS) {
            if (_raise) revert("Max leverage exceeded");
            return (2, marginFees);
        }
        
        return (0, marginFees);
    }
    
    function getDelta(
        bytes32 _indexToken,
        uint256 _size,
        uint256 _entryPrice,
        bool _isLong,
        uint256 _lastIncreasedTime
    ) public view returns (bool, uint256) {
        require(_entryPrice > 0, "Invalid entry price");
        
        uint256 price = priceFeed.getLatestPrimaryPrice(_indexToken);
        uint256 priceDelta = _entryPrice > price ? _entryPrice - price : price - _entryPrice;
        uint256 delta = (_size * priceDelta) / _entryPrice;
        
        bool hasProfit;
        if (_isLong) {
            hasProfit = price > _entryPrice;
        } else {
            hasProfit = _entryPrice > price;
        }
        
        return (hasProfit, delta);
    }
    
    function getPositionKey(address _account, bytes32 _indexToken, bool _isLong) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(_account, _indexToken, _isLong));
    }
    
    function getPosition(address _account, bytes32 _indexToken, bool _isLong) public view returns (uint256, uint256, uint256, uint256, uint256, int256, uint256) {
        bytes32 key = getPositionKey(_account, _indexToken, _isLong);
        Position memory position = positions[key];
        return (position.size, position.collateral, position.entryPrice, position.entryFundingRate, position.reserveAmount, position.realisedPnl, position.lastIncreasedTime);
    }
}
```

### 3. Funding Rate Manager

**Contract**: `FundingRateManager.sol`

```solidity
contract FundingRateManager is AccessControl {
    uint256 public constant FUNDING_RATE_PRECISION = 1000000;
    uint256 public constant MAX_FUNDING_RATE = FUNDING_RATE_PRECISION / 100; // 1%
    
    mapping(bytes32 => uint256) public fundingRateFactor;
    mapping(bytes32 => uint256) public stableFundingRateFactor;
    mapping(bytes32 => uint256) public cumulativeFundingRates;
    mapping(bytes32 => uint256) public lastFundingTimes;
    
    IVault public vault;
    
    uint256 public fundingInterval = 8 hours;
    
    function updateFundingRate(bytes32 _token) external {
        if (lastFundingTimes[_token] == 0) {
            lastFundingTimes[_token] = (block.timestamp / fundingInterval) * fundingInterval;
            return;
        }
        
        if (lastFundingTimes[_token] + fundingInterval > block.timestamp) {
            return;
        }
        
        uint256 fundingRate = getNextFundingRate(_token);
        cumulativeFundingRates[_token] += fundingRate;
        lastFundingTimes[_token] = (block.timestamp / fundingInterval) * fundingInterval;
        
        emit UpdateFundingRate(_token, cumulativeFundingRates[_token]);
    }
    
    function getNextFundingRate(bytes32 _token) public view returns (uint256) {
        if (lastFundingTimes[_token] + fundingInterval > block.timestamp) {
            return 0;
        }
        
        uint256 intervals = (block.timestamp - lastFundingTimes[_token]) / fundingInterval;
        uint256 poolAmount = vault.poolAmounts(_token);
        
        if (poolAmount == 0) {
            return 0;
        }
        
        uint256 reservedAmount = vault.reservedAmounts(_token);
        uint256 fundingRate = (fundingRateFactor[_token] * reservedAmount) / poolAmount;
        
        return fundingRate * intervals;
    }
    
    function getFundingFee(
        address _account,
        bytes32 _indexToken,
        bool _isLong,
        uint256 _size,
        uint256 _entryFundingRate
    ) external view returns (uint256) {
        if (_size == 0) return 0;
        
        uint256 fundingRate = cumulativeFundingRates[_indexToken] - _entryFundingRate;
        if (fundingRate == 0) return 0;
        
        return (_size * fundingRate) / FUNDING_RATE_PRECISION;
    }
    
    function getCumulativeFundingRate(bytes32 _token) external view returns (uint256) {
        return cumulativeFundingRates[_token];
    }
    
    function setFundingRate(
        bytes32 _token,
        uint256 _fundingRateFactor,
        uint256 _stableFundingRateFactor
    ) external onlyRole(ADMIN_ROLE) {
        require(_fundingRateFactor <= MAX_FUNDING_RATE, "Invalid funding rate factor");
        require(_stableFundingRateFactor <= MAX_FUNDING_RATE, "Invalid stable funding rate factor");
        
        fundingRateFactor[_token] = _fundingRateFactor;
        stableFundingRateFactor[_token] = _stableFundingRateFactor;
        
        emit SetFundingRate(_token, _fundingRateFactor, _stableFundingRateFactor);
    }
}
```

### 4. Vault Price Feed

**Contract**: `VaultPriceFeed.sol`

```solidity
contract VaultPriceFeed is AccessControl {
    IOracleModule public oracle;
    
    mapping(bytes32 => uint256) public priceSpreadBasisPoints;
    mapping(bytes32 => uint256) public adjustmentBasisPoints;
    mapping(bytes32 => bool) public isAdjustmentAdditive;
    mapping(bytes32 => uint256) public lastUpdatedAt;
    
    uint256 public constant PRICE_PRECISION = 1e30;
    uint256 public constant ONE_USD = PRICE_PRECISION;
    uint256 public constant BASIS_POINTS_DIVISOR = 10000;
    uint256 public constant MAX_SPREAD_BASIS_POINTS = 50; // 0.5%
    uint256 public constant MAX_ADJUSTMENT_BASIS_POINTS = 20; // 0.2%
    
    bool public includeAmmPrice = true;
    bool public useV2Pricing = false;
    
    function getPrice(bytes32 _token, bool _maximise, bool _includeAmmPrice) external view returns (uint256) {
        uint256 price = getPrimaryPrice(_token, _maximise);
        
        if (_includeAmmPrice && includeAmmPrice) {
            uint256 ammPrice = getAmmPrice(_token);
            if (ammPrice > 0) {
                if (_maximise && ammPrice > price) {
                    price = ammPrice;
                }
                if (!_maximise && ammPrice < price) {
                    price = ammPrice;
                }
            }
        }
        
        uint256 _priceSpreadBasisPoints = priceSpreadBasisPoints[_token];
        if (_priceSpreadBasisPoints > 0) {
            if (_maximise) {
                price = (price * (BASIS_POINTS_DIVISOR + _priceSpreadBasisPoints)) / BASIS_POINTS_DIVISOR;
            } else {
                price = (price * (BASIS_POINTS_DIVISOR - _priceSpreadBasisPoints)) / BASIS_POINTS_DIVISOR;
            }
        }
        
        return price;
    }
    
    function getPrimaryPrice(bytes32 _token, bool _maximise) public view returns (uint256) {
        (int256 price, uint256 timestamp, bool valid) = oracle.getPrice(_token);
        require(valid, "Invalid price");
        require(block.timestamp - timestamp <= 3600, "Price too old");
        
        uint256 normalizedPrice = uint256(price) * PRICE_PRECISION / 1e8; // Assuming 8 decimals from oracle
        
        uint256 _adjustmentBasisPoints = adjustmentBasisPoints[_token];
        if (_adjustmentBasisPoints > 0) {
            bool isAdditive = isAdjustmentAdditive[_token];
            if (isAdditive) {
                normalizedPrice = (normalizedPrice * (BASIS_POINTS_DIVISOR + _adjustmentBasisPoints)) / BASIS_POINTS_DIVISOR;
            } else {
                normalizedPrice = (normalizedPrice * (BASIS_POINTS_DIVISOR - _adjustmentBasisPoints)) / BASIS_POINTS_DIVISOR;
            }
        }
        
        return normalizedPrice;
    }
    
    function getAmmPrice(bytes32 _token) public view returns (uint256) {
        // AMM price calculation based on pool reserves and utilization
        // This would integrate with the liquidity pool to get dynamic pricing
        return 0; // Placeholder
    }
    
    function getLatestPrimaryPrice(bytes32 _token) external view returns (uint256) {
        return getPrimaryPrice(_token, false);
    }
    
    function setPriceSpreadBasisPoints(bytes32 _token, uint256 _priceSpreadBasisPoints) external onlyRole(ADMIN_ROLE) {
        require(_priceSpreadBasisPoints <= MAX_SPREAD_BASIS_POINTS, "Invalid spread");
        priceSpreadBasisPoints[_token] = _priceSpreadBasisPoints;
    }
    
    function setAdjustment(
        bytes32 _token,
        bool _isAdditive,
        uint256 _adjustmentBasisPoints
    ) external onlyRole(ADMIN_ROLE) {
        require(_adjustmentBasisPoints <= MAX_ADJUSTMENT_BASIS_POINTS, "Invalid adjustment");
        isAdjustmentAdditive[_token] = _isAdditive;
        adjustmentBasisPoints[_token] = _adjustmentBasisPoints;
    }
}
```

## Trading Mechanics

### Position Opening Flow

1. **Validation**: Check leverage limits, global OI limits
2. **Price Execution**: Get execution price with spread
3. **Fee Collection**: Trading fee (0.1%) + funding fee
4. **Position Update**: Update position in PositionManager
5. **Pool Reservation**: Reserve tokens from liquidity pool
6. **Global Updates**: Update global long/short sizes

### Position Closing Flow

1. **Price Calculation**: Get mark price for PnL calculation
2. **PnL Settlement**: Calculate realized profit/loss
3. **Fee Deduction**: Deduct trading and funding fees
4. **Position Update**: Update or close position
5. **Pool Release**: Release reserved tokens
6. **Payout**: Transfer net amount to user

### Liquidation Flow

1. **Health Check**: Validate liquidation conditions
2. **Price Update**: Get current mark price
3. **Fee Calculation**: Liquidation fee + remaining collateral
4. **Position Closure**: Close position completely
5. **Reward Distribution**: Pay liquidator bonus
6. **Pool Update**: Release reserved tokens

## Risk Management

### Position Limits

```solidity
// Per-asset configuration
struct AssetConfig {
    uint256 maxLeverage;        // 50x for BTC/ETH, 30x others
    uint256 maxGlobalLong;      // $10M max long OI
    uint256 maxGlobalShort;     // $10M max short OI
    uint256 fundingRateFactor;  // 0.01% per hour
    uint256 priceSpread;        // 0.1-0.3% spread
}
```

### Liquidation Conditions

A position can be liquidated if:
1. **Underwater**: `losses >= collateral`
2. **Insufficient Margin**: `remaining_collateral < fees + liquidation_fee`
3. **Over-leveraged**: `leverage > max_leverage`

### Global Risk Controls

- **Max Open Interest**: $10M per asset per side
- **Utilization Limits**: 80% max pool utilization
- **Price Deviation**: 2% max price movement per update
- **Circuit Breakers**: Automatic pause on anomalies

## Integration Examples

### Opening a Position

```javascript
// Open 10x long BTC position
const collateral = ethers.parseEther("1000"); // $1000 DUSD
const size = ethers.parseEther("10000"); // $10000 position size
const acceptablePrice = ethers.parseEther("50000"); // Max $50k BTC price

await dusd.approve(dperp.address, collateral);
await dperp.increasePosition(
    [], // path (not used in this implementation)
    ethers.keccak256(ethers.toUtf8Bytes("BTC")), // indexToken
    collateral, // amountIn
    0, // minOut
    size, // sizeDelta
    true, // isLong
    acceptablePrice
);
```

### Closing a Position

```javascript
// Close half of the position
const sizeDelta = ethers.parseEther("5000"); // Close $5000
const collateralDelta = ethers.parseEther("500"); // Withdraw $500 collateral
const acceptablePrice = ethers.parseEther("49000"); // Min $49k BTC price

await dperp.decreasePosition(
    userAddress, // account
    ethers.keccak256(ethers.toUtf8Bytes("BTC")), // indexToken
    collateralDelta, // collateralDelta
    sizeDelta, // sizeDelta
    true, // isLong
    userAddress, // receiver
    acceptablePrice
);
```

### Liquidating a Position

```javascript
// Liquidate unhealthy position
await dperp.liquidatePosition(
    targetAccount, // account to liquidate
    ethers.keccak256(ethers.toUtf8Bytes("BTC")), // indexToken
    true, // isLong
    liquidatorAddress // fee receiver
);
```

## Monitoring & Analytics

### Key Metrics

- **Open Interest**: Total long/short positions per asset
- **Utilization Rate**: Percentage of pool reserved
- **Funding Rates**: Current funding rates per asset
- **Liquidation Rate**: Percentage of positions liquidated
- **Trading Volume**: Daily/weekly trading volume

### Position Health Monitoring

```javascript
// Check position health
const positionKey = await positionManager.getPositionKey(userAddress, assetId, isLong);
const position = await positionManager.getPosition(userAddress, assetId, isLong);
const [liquidationState, marginFees] = await positionManager.validateLiquidation(userAddress, assetId, isLong, false);

if (liquidationState > 0) {
    console.log("Position can be liquidated");
}
```

### Events for Tracking

```solidity
event IncreasePosition(address indexed account, bytes32 indexed indexToken, uint256 amountIn, uint256 sizeDelta, bool isLong, uint256 price, uint256 fee);
event DecreasePosition(address indexed account, bytes32 indexed indexToken, uint256 collateralDelta, uint256 sizeDelta, bool isLong, address receiver, uint256 price, uint256 usdOut);
event LiquidatePosition(address indexed account, bytes32 indexed indexToken, bool isLong, uint256 size, uint256 collateral, uint256 liquidationFee, address liquidator);
event UpdateFundingRate(bytes32 indexed token, uint256 fundingRate);
event CollectMarginFees(uint256 feeUsd, uint256 fundingFee);
```

---

*DPerp Subsystem - Advanced Perpetual Trading with Maximum Capital Efficiency*