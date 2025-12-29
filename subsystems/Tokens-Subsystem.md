# Tokens Subsystem - DI Network

## Overview

The Tokens subsystem forms the foundation of DI Network, providing the native tokens, stablecoin infrastructure, and tokenomics that power all protocol operations.

## Core Components

### 1. DI Token (Native Governance Token)

**Contract**: `DIToken.sol`

```solidity
contract DIToken is ERC20, ERC20Votes, ERC20Permit, AccessControl {
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 1e18;
    
    constructor(address distributor) ERC20("DI Token", "DI") ERC20Permit("DI Token") {
        _mint(distributor, TOTAL_SUPPLY);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
}
```

**Key Features**:
- **Total Supply**: 1 billion tokens (fixed)
- **Governance**: ERC20Votes for delegation and voting
- **Gasless Approvals**: ERC20Permit for meta-transactions
- **Role-Based Access**: AccessControl for admin functions

**Utilities**:
- Primary collateral for DUSD minting (75% collateral factor)
- Governance voting rights in DAO
- Staking rewards (8-20% APY)
- Fee distribution from protocol revenue

### 2. DUSD Stablecoin

**Contract**: `DUSD.sol`

```solidity
contract DUSD is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    
    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
    
    function burn(address from, uint256 amount) external onlyRole(BURNER_ROLE) {
        _burn(from, amount);
    }
}
```

**Stability Mechanism**:
- **Over-collateralized**: 125% minimum collateral ratio
- **Multi-collateral**: DI, WBTC, WETH, USDT, USDC
- **Interest Rate**: 5% APR on borrowed DUSD
- **Liquidation**: 80% threshold with 5% penalty

**Use Cases**:
- Base currency for synthetic asset trading
- Cross-chain gas payments (gasless transactions)
- Liquidity provision in DPerp pools
- Bridge transfers between chains

### 3. Synthetic Tokens

**Contract**: `SyntheticToken.sol`

```solidity
contract SyntheticToken is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public priceId; // Oracle price feed ID
    
    constructor(
        string memory name,
        string memory symbol,
        bytes32 _priceId
    ) ERC20(name, symbol) {
        priceId = _priceId;
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
}
```

**Supported Assets**:
- **Crypto**: xBTC, xETH, xBNB, xADA, xSOL
- **Stocks**: xAAPL, xTSLA, xGOOG, xAMZN, xMSFT
- **Commodities**: xGold, xSilver, xOil, xGas
- **Forex**: xEUR, xGBP, xJPY, xCHF

### 4. DLP Token (Liquidity Provider)

**Contract**: `LiquidityPool.sol`

```solidity
contract LiquidityPool is ERC20 {
    IERC20 public dusd;
    uint256 public totalAUM; // Assets Under Management
    
    function addLiquidity(uint256 dusdAmount) external returns (uint256 dlpAmount) {
        dusd.transferFrom(msg.sender, address(this), dusdAmount);
        dlpAmount = (dusdAmount * totalSupply()) / totalAUM;
        _mint(msg.sender, dlpAmount);
        totalAUM += dusdAmount;
    }
    
    function removeLiquidity(uint256 dlpAmount) external returns (uint256 dusdAmount) {
        dusdAmount = (dlpAmount * totalAUM) / totalSupply();
        _burn(msg.sender, dlpAmount);
        totalAUM -= dusdAmount;
        dusd.transfer(msg.sender, dusdAmount);
    }
}
```

## Token Distribution & Vesting

### Distribution Breakdown

| Category | Amount | % | Vesting | TGE |
|----------|--------|---|---------|-----|
| Team | 150M | 15% | 48mo, 12mo cliff | 0% |
| Advisors | 50M | 5% | 24mo, 6mo cliff | 0% |
| Public Sale | 100M | 10% | 6mo linear | 30% |
| DI Liquidity | 150M | 15% | None | 100% |
| DUSD Liquidity | 100M | 10% | None | 100% |
| Treasury | 150M | 15% | 60mo | 0% |
| Ecosystem | 120M | 12% | 36mo | 15% |
| Private Sale | 120M | 12% | 18mo, 6mo cliff | 0% |
| Marketing | 50M | 5% | 24mo | 20% |
| KOL | 30M | 3% | 12mo, 3mo cliff | 0% |
| Development | 30M | 3% | 36mo | 10% |

### Vesting Implementation

**Contract**: `TokenVesting.sol`

```solidity
contract TokenVesting {
    struct VestingSchedule {
        uint256 totalAmount;
        uint256 startTime;
        uint256 cliffDuration;
        uint256 duration;
        uint256 released;
        bool revocable;
    }
    
    mapping(address => VestingSchedule) public vestingSchedules;
    
    function release() external {
        VestingSchedule storage schedule = vestingSchedules[msg.sender];
        uint256 releasable = _releasableAmount(schedule);
        require(releasable > 0, "No tokens to release");
        
        schedule.released += releasable;
        diToken.transfer(msg.sender, releasable);
    }
    
    function _releasableAmount(VestingSchedule memory schedule) private view returns (uint256) {
        if (block.timestamp < schedule.startTime + schedule.cliffDuration) {
            return 0;
        }
        
        uint256 timeFromStart = block.timestamp - schedule.startTime;
        if (timeFromStart >= schedule.duration) {
            return schedule.totalAmount - schedule.released;
        }
        
        uint256 vestedAmount = (schedule.totalAmount * timeFromStart) / schedule.duration;
        return vestedAmount - schedule.released;
    }
}
```

## Collateral Management

### DUSD Provider System

**Contract**: `DUSDProvider.sol`

```solidity
contract DUSDProvider {
    struct BorrowPosition {
        uint256 collateralAmount;
        uint256 borrowedAmount;
        uint256 interestAccrued;
        uint256 lastUpdated;
        uint256 collateralFactor; // 75% for DI
    }
    
    mapping(address => BorrowPosition) public positions;
    
    function depositCollateral(uint256 amount) external {
        diToken.transferFrom(msg.sender, address(this), amount);
        positions[msg.sender].collateralAmount += amount;
        emit CollateralDeposited(msg.sender, amount);
    }
    
    function borrowDUSD(uint256 amount) external {
        BorrowPosition storage position = positions[msg.sender];
        _accrueInterest(position);
        
        uint256 maxBorrow = _getMaxBorrowAmount(msg.sender);
        require(position.borrowedAmount + amount <= maxBorrow, "Insufficient collateral");
        
        position.borrowedAmount += amount;
        dusd.mint(msg.sender, amount);
        emit DUSDBorrowed(msg.sender, amount);
    }
    
    function repayDUSD(uint256 amount) external {
        BorrowPosition storage position = positions[msg.sender];
        _accrueInterest(position);
        
        dusd.burnFrom(msg.sender, amount);
        position.borrowedAmount -= amount;
        emit DUSDRepaid(msg.sender, amount);
    }
    
    function _getMaxBorrowAmount(address user) private view returns (uint256) {
        uint256 collateralValue = _getCollateralValue(user);
        return (collateralValue * positions[user].collateralFactor) / 10000;
    }
}
```

### Liquidation System

```solidity
contract Liquidator {
    uint256 public constant LIQUIDATION_THRESHOLD = 8000; // 80%
    uint256 public constant LIQUIDATION_BONUS = 500; // 5%
    
    function liquidate(address user) external {
        BorrowPosition memory position = dusdProvider.getPosition(user);
        require(_isLiquidatable(position), "Position healthy");
        
        uint256 collateralValue = _getCollateralValue(user);
        uint256 debtValue = position.borrowedAmount + position.interestAccrued;
        
        // Calculate liquidation amounts
        uint256 liquidationAmount = debtValue;
        uint256 collateralToSeize = (liquidationAmount * (10000 + LIQUIDATION_BONUS)) / 10000;
        
        // Execute liquidation
        dusd.burnFrom(msg.sender, liquidationAmount);
        diToken.transfer(msg.sender, collateralToSeize);
        
        // Update position
        dusdProvider.liquidatePosition(user, collateralToSeize, liquidationAmount);
    }
    
    function _isLiquidatable(BorrowPosition memory position) private pure returns (bool) {
        if (position.borrowedAmount == 0) return false;
        
        uint256 collateralValue = _getCollateralValue(position.collateralAmount);
        uint256 debtValue = position.borrowedAmount + position.interestAccrued;
        
        return (collateralValue * 10000) / debtValue < LIQUIDATION_THRESHOLD;
    }
}
```

## Staking System

### Staking Contract

**Contract**: `DIStaking.sol`

```solidity
contract DIStaking {
    struct StakeInfo {
        uint256 amount;
        uint256 lockPeriod;
        uint256 startTime;
        uint256 rewardDebt;
        uint256 multiplier;
    }
    
    mapping(address => StakeInfo) public stakes;
    uint256 public totalStaked;
    uint256 public rewardRate = 1000; // 10% base APY
    
    function stake(uint256 amount, uint256 lockPeriod) external {
        require(lockPeriod >= 0 && lockPeriod <= 24 * 30 days, "Invalid lock period");
        
        diToken.transferFrom(msg.sender, address(this), amount);
        
        uint256 multiplier = _getLockMultiplier(lockPeriod);
        stakes[msg.sender] = StakeInfo({
            amount: amount,
            lockPeriod: lockPeriod,
            startTime: block.timestamp,
            rewardDebt: 0,
            multiplier: multiplier
        });
        
        totalStaked += amount;
        emit Staked(msg.sender, amount, lockPeriod);
    }
    
    function claimRewards() external {
        StakeInfo storage stakeInfo = stakes[msg.sender];
        uint256 rewards = _calculateRewards(stakeInfo);
        
        stakeInfo.rewardDebt += rewards;
        diToken.transfer(msg.sender, rewards);
        emit RewardsClaimed(msg.sender, rewards);
    }
    
    function unstake() external {
        StakeInfo storage stakeInfo = stakes[msg.sender];
        require(block.timestamp >= stakeInfo.startTime + stakeInfo.lockPeriod, "Still locked");
        
        uint256 amount = stakeInfo.amount;
        uint256 rewards = _calculateRewards(stakeInfo);
        
        delete stakes[msg.sender];
        totalStaked -= amount;
        
        diToken.transfer(msg.sender, amount + rewards);
        emit Unstaked(msg.sender, amount, rewards);
    }
    
    function _getLockMultiplier(uint256 lockPeriod) private pure returns (uint256) {
        if (lockPeriod == 0) return 10000; // 1.0x
        if (lockPeriod <= 3 * 30 days) return 12000; // 1.2x
        if (lockPeriod <= 6 * 30 days) return 15000; // 1.5x
        if (lockPeriod <= 12 * 30 days) return 20000; // 2.0x
        return 25000; // 2.5x for 24 months
    }
}
```

## Governance Integration

### Voting Power & Delegation

```solidity
// Delegate voting power
diToken.delegate(delegateAddress);

// Check voting power
uint256 votes = diToken.getVotes(account);

// Create proposal (requires 1% of supply)
governance.propose(targets, values, calldatas, description);

// Vote on proposal
governance.vote(proposalId, support);
```

### Governance Parameters

- **Proposal Threshold**: 10M DI (1% of supply)
- **Voting Period**: 7 days
- **Quorum**: 40M DI (4% of supply)
- **Timelock**: 48 hours before execution

## Token Economics

### Value Accrual Mechanisms

1. **Collateral Demand**: DI needed for DUSD minting
2. **Staking Rewards**: High APY creates buying pressure
3. **Governance Premium**: Voting rights add value
4. **Fee Distribution**: Revenue sharing with stakers

### Fee Distribution

```
Protocol Fees (100%)
├─ Stakers: 50%
├─ Treasury: 30%
├─ Insurance Fund: 15%
└─ Development: 5%
```

### Supply Schedule

| Month | Circulating | % | Events |
|-------|-------------|---|--------|
| 0 | 311M | 31.1% | TGE + Liquidity |
| 6 | 424M | 42.4% | Public sale complete |
| 12 | 544M | 54.4% | Team cliff ends |
| 24 | 704M | 70.4% | Major vestings complete |
| 48 | 1,000M | 100% | All tokens released |

## Integration Examples

### Basic Token Operations

```javascript
// Stake DI tokens
await diStaking.stake(ethers.parseEther("1000"), 6 * 30 * 24 * 3600); // 6 months

// Deposit collateral and borrow DUSD
await diToken.approve(dusdProvider.address, ethers.parseEther("1000"));
await dusdProvider.depositCollateral(ethers.parseEther("1000"));
await dusdProvider.borrowDUSD(ethers.parseEther("750")); // 75% collateral factor

// Delegate voting power
await diToken.delegate(delegateAddress);
```

### Advanced Operations

```javascript
// Add liquidity to DPerp pool
await dusd.approve(liquidityPool.address, ethers.parseEther("10000"));
const dlpAmount = await liquidityPool.addLiquidity(ethers.parseEther("10000"));

// Check position health
const position = await dusdProvider.getPosition(userAddress);
const isHealthy = await liquidator.isPositionHealthy(userAddress);

// Claim staking rewards
const rewards = await diStaking.calculateRewards(userAddress);
await diStaking.claimRewards();
```

## Security Features

### Access Control

```solidity
// Role-based permissions
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");

modifier onlyRole(bytes32 role) {
    require(hasRole(role, msg.sender), "Access denied");
    _;
}
```

### Emergency Controls

```solidity
// Emergency pause
bool public paused;

modifier whenNotPaused() {
    require(!paused, "Contract paused");
    _;
}

function emergencyPause() external onlyRole(ADMIN_ROLE) {
    paused = true;
    emit EmergencyPaused();
}
```

### Oracle Integration

```solidity
// Price validation
function _validatePrice(int256 price, uint256 timestamp) private view {
    require(price > 0, "Invalid price");
    require(block.timestamp - timestamp <= MAX_PRICE_AGE, "Price too old");
}
```

## Monitoring & Analytics

### Key Metrics

- **Total Supply**: 1B DI tokens
- **Circulating Supply**: Dynamic based on vesting
- **Staked Amount**: Tokens locked in staking
- **DUSD Supply**: Total minted stablecoin
- **Collateralization Ratio**: System health metric

### Events for Tracking

```solidity
event CollateralDeposited(address indexed user, uint256 amount);
event DUSDBorrowed(address indexed user, uint256 amount);
event DUSDRepaid(address indexed user, uint256 amount);
event Liquidation(address indexed user, uint256 collateralSeized, uint256 debtRepaid);
event Staked(address indexed user, uint256 amount, uint256 lockPeriod);
event RewardsClaimed(address indexed user, uint256 rewards);
```

---

*Tokens Subsystem - The Foundation of DI Network*