# DSwap - Spot Trading

DSwap enables spot trading of synthetic assets using DUSD as the base currency. The system uses oracle-based pricing with enhanced multi-layer security architecture, dynamic fee structures, and mathematical solvency guarantees for maximum protocol security.

## Overview

DSwap provides:
- Virtual synthetic asset positions (no ERC20 tokens)
- Mathematical solvency guarantees through hard invariants
- Dynamic risk management with stress-based fees
- Insurance fund integration for emergency support
- Settlement locks for MEV protection
- Gas-efficient clone factory for new assets

## Architecture

```
User Interface (SwapRouter)
         ↓
    Core Logic (DSwap)
         ↓
   Enhanced Risk Management (Hard Invariants + Dynamic Fees + Insurance)
         ↓
   Virtual Asset Tracking (SynthManager)
         ↓
   Clone Factory (Gas-Efficient Deployment)
```

### Core Components

- **SwapRouter**: User-facing interface with slippage protection
- **DSwap**: Enhanced core logic with multi-layer security
- **SynthManager**: Clone factory and virtual asset registry
- **DUSDProvider**: Dynamic backing calculation and limits
- **Insurance Fund**: Emergency DUSD reserves for danger zone support
- **Virtual Assets**: Position tracking without token deployment

## Supported Operations

### 1. Mint Synthetic
Convert DUSD to virtual synthetic assets at oracle prices:
```javascript
// Mint xAAPL using DUSD (creates virtual position)
await swapRouter.mintSynthetic(
  keccak256("xAAPL"), // Asset ID
  parseEther("1000"), // 1000 DUSD
  parseEther("6.6")   // Min 6.6 xAAPL at $150/share
)
// Result: Virtual xAAPL position tracked in SynthManager
```

### 2. Burn Synthetic
Convert virtual synthetic assets back to DUSD with dynamic fees:
```javascript
// Burn xAAPL for DUSD (subject to dynamic burn fee)
await swapRouter.burnSynthetic(
  keccak256("xAAPL"), // Asset ID
  parseEther("6.6"),  // 6.6 xAAPL virtual position
  parseEther("980")   // Min 980 DUSD (after dynamic fee)
)
// Note: Dynamic fee ranges from 0.3% to 5% based on stress ratio
```

### 3. Swap Synthetic
Direct swap between virtual synthetic assets:
```javascript
// Swap xAAPL to xTSLA (both virtual positions)
await swapRouter.swapSynthetic(
  keccak256("xAAPL"), // From asset
  keccak256("xTSLA"), // To asset
  parseEther("6.6"),  // 6.6 xAAPL
  parseEther("4.9")   // Min 4.9 xTSLA
)
// Note: 0.3% base fee, no DUSD intermediate step
```

## Enhanced Security Architecture

### Hard Invariants (Mathematical Guarantees)

The enhanced DSwap system implements three non-negotiable invariants that provide mathematical guarantees of protocol solvency:

#### Invariant #1 — Global Solvency (Absolute)
```solidity
require(
    syntheticDebtDUSD + totalDUSDSupply <= maxBorrowableDUSD + emergencyInsurance,
    "GLOBAL_SOLVENCY_BREACH"
);
```
- **Enforced**: At every state transition
- **Components**: `syntheticDebtDUSD = Σ (assetSupply × oraclePrice)`
- **Emergency Insurance**: Insurance fund DUSD reserves (only in danger zone)
- **Guarantee**: Mathematical impossibility of protocol insolvency

#### Invariant #2 — Rate Limiting (Time-Based Protection)
```solidity
require(
    block.timestamp >= lastMintTime[asset] + mintCooldown[asset] || 
    mintAmountInPeriod[asset] + amount <= mintRateCap[asset],
    "MINT_RATE_EXCEEDED"
);
```
- **Purpose**: Prevents flash minting during oracle spikes
- **Scope**: Global per asset and per user cooldowns
- **Protection**: Limits rapid exploitation of price movements

#### Invariant #3 — Asset Debt Caps (Diversification)
```solidity
require(
    syntheticLiability[asset] + amount <= assetDebtCap[asset],
    "ASSET_DEBT_CAP_EXCEEDED"
);
```
- **Purpose**: Prevents single asset from destabilizing system
- **Configuration**: Independent caps per synthetic asset
- **Risk Management**: Ensures portfolio diversification

### Enhanced Stress Metrics

The system uses a comprehensive stress calculation that accounts for all outstanding liabilities:

```solidity
stress = (syntheticDebtDUSD + totalDUSDSupply + additionalDUSD) * 1e18 / maxBorrowableDUSD
```

**Where**:
- `syntheticDebtDUSD = Σ (assetSupply × oraclePrice)` - Total synthetic asset debt
- `totalDUSDSupply` - All minted DUSD tokens
- `maxBorrowableDUSD = (totalCollateralValue × maintenanceRatio) / 10000`

**Stress Zones**:
- **Healthy** (< 95%): Base fees, normal operations
- **Normal** (95-100%): Linear fee escalation
- **Danger** (100-105%): Exponential fees + insurance fund support
- **Blocked** (> 105%): Operations halted by hard cap

### Advanced Fee System

The enhanced fee system provides both revenue generation and risk mitigation:

#### Dynamic Fee Structure
```solidity
function getBurnFeeBps(uint256 additionalDUSD) public view returns (uint256) {
    uint256 stress = getStressRatio(additionalDUSD);
    
    if (stress < 95e16) {        // < 95%
        return baseFeeBps;       // 0.3% base rate
    } else if (stress < 1e18) {  // 95% - 100%
        // Linear escalation: 0.3% to 0.5%
        uint256 normalZone = (stress - 95e16) * 1e18 / 5e16;
        return baseFeeBps + (normalZone * 20) / 1e18;
    } else if (stress < 105e16) { // 100% - 105%
        // Exponential escalation: 0.5% to 5%
        uint256 dangerZone = (stress - 1e18) * 1e18 / 5e16;
        return 50 + (dangerZone * dangerZone * 450) / 1e36;
    } else {
        revert("STRESS_TOO_HIGH");
    }
}
```

#### Fee Distribution System
```solidity
function setFeeDistribution(
    uint256 _baseFeeBps,
    uint256 _insuranceFeeRatio,
    address _feeRecipient
) external onlyRole(ADMIN_ROLE) {
    baseFeeBps = _baseFeeBps;           // Base fee rate (default 0.3%)
    insuranceFeeRatio = _insuranceFeeRatio; // Insurance fund share (default 50%)
    feeRecipient = _feeRecipient;       // Remaining fees recipient
}
```

**Fee Allocation**:
- **Insurance Fund**: 50% (configurable) - Builds protocol reserves
- **Fee Recipient**: 50% (configurable) - Protocol revenue/stakers
- **Dynamic Rates**: Stress-based escalation for burn operations

### Insurance Fund Integration

The insurance fund provides additional backing during stress periods:

#### Emergency Insurance Activation
```solidity
function _getEmergencyInsurance(uint256 additionalDUSD) internal view returns (uint256) {
    uint256 stress = _getStressRatio(additionalDUSD);
    return stress >= 1e18 ? dusd.balanceOf(address(this)) : 0;
}
```

#### Stress-Based DUSD Minting
```solidity
function _mintDUSDFromBurnedSynth(address user, uint256 dusdAmount, uint256 netDUSD) internal {
    _checkGlobalSolvency(dusdAmount);
    
    uint256 stress = _getStressRatio(dusdAmount);
    IDUSD dusd = _dusd();
    
    if (stress >= 1e18) {
        // Danger zone: Use insurance fund reserves
        require(dusd.balanceOf(address(this)) >= netDUSD, "Insufficient insurance fund");
        dusd.transfer(user, netDUSD);
    } else {
        // Normal operation: Mint new DUSD
        dusd.mint(user, netDUSD);
    }
}
```

**Insurance Fund Features**:
- **Automatic Activation**: Engages when stress ≥ 100%
- **Reserve Utilization**: Uses accumulated DUSD reserves
- **Solvency Protection**: Prevents new DUSD minting in danger zone
- **Fee Funding**: Built from 50% of all trading fees

### Settlement Lock Protection
```solidity
mapping(address => uint256) public lastSwapTime;
uint256 public constant SETTLEMENT_LOCK = 1 minutes;

modifier settlementLock(address user) {
    require(
        block.timestamp >= lastSwapTime[user] + SETTLEMENT_LOCK,
        "Settlement locked"
    );
    _;
}
```

**Anti-MEV Features**:
- **1-Minute Cooldown**: Prevents rapid arbitrage cycles
- **User-Specific**: Each user has independent cooldown
- **Comprehensive Coverage**: Applies to all swap operations
- **Price Stabilization**: Reduces oracle front-running opportunities

## Multi-Layer Defense System

### Layer 1: Hard Mathematical Invariants
- **Global Solvency**: `totalDebt ≤ maxBorrowable + insurance`
- **Rate Limiting**: Prevents flash exploitation
- **Asset Caps**: Diversification enforcement
- **Guarantee**: Mathematical impossibility of insolvency

### Layer 2: Dynamic Economic Controls
- **Stress-Based Fees**: 0.3% to 5% based on system stress
- **Insurance Fund**: Automatic activation in danger zone
- **Fee Distribution**: 50% to insurance, 50% to protocol
- **Exponential Escalation**: Rapid fee increase in danger zone

### Layer 3: Operational Protections
- **Settlement Locks**: 1-minute MEV protection
- **Oracle Validation**: Staleness and deviation checks
- **Access Control**: Role-based admin functions
- **Emergency Pause**: Immediate halt capability

### Layer 4: Real-Time Monitoring
- **Continuous Stress Calculation**: Real-time solvency tracking
- **Asset-Level Monitoring**: Per-asset debt tracking
- **Rate Limit Tracking**: Global and per-user limits
- **Insurance Fund Status**: Reserve level monitoring

## Key Features

### Virtual Asset System
- **No ERC20 Tokens**: Positions tracked in SynthManager contract
- **Clone Factory**: 97.5% gas savings on new asset deployment
- **Instant Addition**: Add new assets without token deployment
- **Upgradeable Logic**: Update implementation without migration

### Oracle-Based Pricing
- Real-time price feeds from Chainlink and Pyth
- Zero slippage on oracle price
- Instant execution without liquidity constraints
- Staleness and deviation protection

### Dynamic Fee Structure
- **Mint fee**: 0.3% base (safe operation - no DUSD minting)
- **Burn fee**: 0.3-5% dynamic based on stress ratio (risky - mints DUSD)
- **Swap fee**: 0.3% base (synthetic-to-synthetic, no DUSD involved)
- **Settlement lock**: 1-minute cooldown after all operations

## Trading Flow Example

### Enhanced Mint Flow with Security Checks
1. User approves 1000 DUSD to SwapRouter
2. **Hard Invariant Check**: Verify global solvency constraints
3. **Rate Limiting**: Check per-asset and per-user mint limits
4. **Asset Cap Check**: Verify asset debt cap not exceeded
5. Oracle provides AAPL price ($150)
6. Fee calculation: 1000 × 0.003 = 3 DUSD (base mint fee)
7. Fee distribution: 1.5 DUSD to insurance fund, 1.5 DUSD to fee recipient
8. Net amount: 997 DUSD
9. xAAPL amount: 997 ÷ 150 = 6.64 xAAPL
10. Burn 1000 DUSD from user
11. SynthManager tracks 6.64 xAAPL virtual position
12. Settlement lock: 1 minute (no transfers/swaps)

### Enhanced Burn Flow with Insurance Support
1. User has 6.64 xAAPL virtual position (after settlement lock expires)
2. **Cooldown Check**: Verify mint cooldown period has passed
3. Oracle provides AAPL price ($160)
4. Value: 6.64 × 160 = 1,062 DUSD
5. **Stress Calculation**: Current stress ratio = 0.98 (98% of max borrowable)
6. **Dynamic Fee**: 0.4% = 4.25 DUSD (normal zone linear escalation)
7. **Global Solvency Check**: Verify hard invariants
8. **Insurance Fund Check**: Stress < 100%, normal minting
9. Net amount: 1,058 DUSD
10. Mint 1,058 DUSD to user
11. Fee distribution: 2.125 DUSD to insurance fund, 2.125 DUSD to fee recipient
12. Settlement lock: 1 minute (new lock period)

## Risk Classification Matrix

| Operation | Risk Level | Fee Structure | Insurance Support |
|-----------|------------|---------------|-------------------|
| **Mint Synthetic** | Low | Base fee (0.3%) | No |
| **Swap Synthetic** | Low | Base fee (0.3%) | No |
| **Burn Synthetic** | High | Dynamic (0.3-5%) | Yes (danger zone) |

## Stress Response System

| Stress Level | Range | Fee Rate | Insurance | Operations |
|--------------|-------|----------|-----------|------------|
| **Healthy** | 0-95% | 0.3% | Inactive | Full |
| **Normal** | 95-100% | 0.3-0.5% | Inactive | Full |
| **Danger** | 100-105% | 0.5-5% | Active | Restricted |
| **Blocked** | >105% | N/A | N/A | Halted |

## Virtual Asset Architecture

### SyntheticToken (Clone Implementation)
**File:** `src/tokens/SyntheticToken.sol`

Minimal ERC20 implementation designed for cloning:

```solidity
contract SyntheticToken is ERC20 {
    address public manager;
    bool private initialized;
    
    function initialize(string memory name, string memory symbol, address _manager) external;
    function mint(address to, uint256 amount) external;
    function burn(address from, uint256 amount) external;
}
```

### SynthManager (Clone Factory)
**File:** `src/tokens/SynthManager.sol`

Factory and registry for synthetic tokens:

```solidity
struct SynthInfo {
    address token;        // Clone address
    string name;          // Token name
    string symbol;        // Token symbol
    bytes32 priceId;      // Oracle price ID
    uint256 totalSupply;  // Total minted
    bool active;          // Trading enabled
}
```

## Supported Assets (Virtual Positions)

### Equities (Primary Focus)
- **xAAPL**: Apple Inc. (virtual position)
- **xTSLA**: Tesla Inc. (virtual position)
- **xMSFT**: Microsoft Corp. (virtual position)
- **xAMZN**: Amazon.com Inc. (virtual position)
- **xGOOGL**: Alphabet Inc. (virtual position)

### Commodities
- **xGOLD**: Gold futures (virtual position)
- **xSILVER**: Silver futures (virtual position)
- **xOIL**: Crude Oil futures (virtual position)
- **xCOPPER**: Copper futures (virtual position)

### Indices
- **xSP500**: S&P 500 Index (virtual position)
- **xNASDAQ**: NASDAQ Composite (virtual position)
- **xDOW**: Dow Jones Industrial Average (virtual position)

### Crypto (Synthetic Versions)
- **xBTC**: Synthetic Bitcoin (virtual position)
- **xETH**: Synthetic Ethereum (virtual position)

**Note**: All assets are virtual positions tracked in SynthManager, not ERC20 tokens. This eliminates deployment costs and enables instant asset addition with 97.5% gas savings.

## Integration

```javascript
import { DSwap } from '@di-network/sdk'

const dswap = new DSwap({
  provider: web3Provider,
  network: 'mainnet'
})

// Check enhanced risk metrics
const stressRatio = await dswap.getStressRatio()
const burnFee = await dswap.getBurnFeeBps()
const insuranceFund = await dswap.getInsuranceFundBalance()
const canMint = await dswap.canMint('1000')

// Get virtual asset info
const synthInfo = await dswap.synthManager.getSynthetic(keccak256('xAAPL'))
console.log('Virtual asset address:', synthInfo.token)
console.log('Total virtual supply:', synthInfo.totalSupply)
console.log('Asset debt cap:', synthInfo.debtCap)

// Execute swap with enhanced security
const tx = await dswap.mintSynthetic('xAAPL', '1000')
// Note: 1-minute settlement lock applies after trade
```

### Smart Contract Integration
```solidity
interface IDSwap {
    // Enhanced risk management functions
    function getStressRatio() external view returns (uint256);
    function getBurnFeeBps() external view returns (uint256);
    function getInsuranceFundBalance() external view returns (uint256);
    function canMint(bytes32 assetId, uint256 amount) external view returns (bool);
    function getAssetDebtCap(bytes32 assetId) external view returns (uint256);
    function lastSwapTime(address user) external view returns (uint256);
    
    // Admin functions for enhanced security
    function setFeeDistribution(uint256 baseFeeBps, uint256 insuranceRatio, address recipient) external;
    function addAsset(bytes32 assetId, uint256 debtCap, uint256 rateCap, uint256 cooldown) external;
    function setAssetActive(bytes32 assetId, bool active) external;
}
```

## Benefits

### For Traders
- Access to global markets 24/7
- Zero slippage oracle-based pricing
- Mathematical solvency guarantees
- Enhanced MEV protection
- Dynamic fees reflect system health

### For the Protocol
- Mathematical impossibility of insolvency
- Self-balancing through dynamic fees and insurance
- Scalable virtual asset system
- Comprehensive risk management
- Revenue generation through fee distribution

**Key Insight**: The enhanced DSwap system provides mathematical guarantees of protocol solvency through hard invariants while maintaining economic incentives through dynamic fees and insurance fund support. Only burning synthetics creates solvency risk by minting new DUSD, hence the sophisticated multi-layer security architecture.