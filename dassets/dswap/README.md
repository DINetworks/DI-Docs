# DSwap - Spot Trading

DSwap enables spot trading of synthetic assets using DUSD as the base currency. The system uses oracle-based pricing with advanced risk management, dynamic fee structures, and virtual asset tracking for maximum protocol security and capital efficiency.

## Overview

DSwap provides:
- Virtual synthetic asset positions (no ERC20 tokens)
- Dynamic risk management with stress-based fees
- Settlement locks for MEV protection
- Gas-efficient clone factory for new assets
- Mathematical solvency guarantees

## Architecture

```
User Interface (SwapRouter)
         ↓
    Core Logic (DSwap)
         ↓
   Risk Management (Dynamic Backing + Fees)
         ↓
   Virtual Asset Tracking (SynthManager)
         ↓
   Clone Factory (Gas-Efficient Deployment)
```

### Core Components

- **SwapRouter**: User-facing interface with slippage protection
- **DSwap**: Core swap logic with dynamic fee management
- **SynthManager**: Clone factory and virtual asset registry
- **DUSDProvider**: Dynamic backing calculation and limits
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
// Note: Dynamic fee ranges from 0.3% to 2% based on stress ratio
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
// Note: 0.3% flat fee, no DUSD intermediate step
```

## Key Features

### Virtual Asset System
- **No ERC20 Tokens**: Positions tracked in SynthManager contract
- **Clone Factory**: 97.5% gas savings on new asset deployment
- **Instant Addition**: Add new assets without token deployment
- **Upgradeable Logic**: Update implementation without migration

### Advanced Risk Management
- **Hard Invariants**: Mathematical impossibility of protocol insolvency
- **Dynamic Backing**: Real-time collateral monitoring
- **Stress-Based Fees**: Dynamic burn fees (0.3-2%)
- **Settlement Locks**: 1-minute MEV protection

### Oracle-Based Pricing
- Real-time price feeds from Chainlink and Pyth
- Zero slippage on oracle price
- Instant execution without liquidity constraints
- Staleness and deviation protection

### Dynamic Fee Structure
- **Mint fee**: 0.3% flat (safe operation - no DUSD minting)
- **Burn fee**: 0.3-2% dynamic based on stress ratio (risky - mints DUSD)
- **Swap fee**: 0.3% flat (synthetic-to-synthetic, no DUSD involved)
- **Settlement lock**: 1-minute cooldown after all operations

### Clone Factory Benefits

#### Gas Efficiency
- **First deployment**: ~2M gas (implementation)
- **Subsequent deployments**: ~50K gas (clones)
- **Savings**: 97.5% gas reduction per new synthetic

#### Upgradeability
- Update implementation address in SynthManager
- New synthetics use new implementation
- Existing synthetics unchanged
- No migration needed

#### Consistency
- All synthetics share same logic
- Uniform behavior across assets
- Easier auditing and testing

## Trading Flow Example

### Mint xAAPL with 1000 DUSD (Virtual Position)
1. User approves 1000 DUSD to SwapRouter
2. Check: totalSupply + 1000 ≤ maxBorrowableDUSD ✓
3. Oracle provides AAPL price ($150)
4. Fee calculation: 1000 × 0.003 = 3 DUSD (flat mint fee)
5. Net amount: 997 DUSD
6. xAAPL amount: 997 ÷ 150 = 6.64 xAAPL
7. Burn 997 DUSD from user
8. SynthManager tracks 6.64 xAAPL virtual position
9. Settlement lock: 1 minute (no transfers/swaps)

### Burn xAAPL back to DUSD (Dynamic Fee)
1. User has 6.64 xAAPL virtual position (after settlement lock expires)
2. Oracle provides AAPL price ($160)
3. Value: 6.64 × 160 = 1,062 DUSD
4. Check: totalSupply + 1,062 ≤ maxBorrowableDUSD ✓
5. Calculate stress ratio: 0.7 (example)
6. Dynamic burn fee: 1.5% = 16 DUSD
7. Net amount: 1,046 DUSD
8. Mint 1,046 DUSD to user
9. Settlement lock: 1 minute (new lock period)

## Risk Management Architecture

### Hard Invariant Protection
```solidity
// Prevents protocol insolvency
require(
    totalDUSDSupply + dUSDAmount <= getMaxBorrowableDUSD(),
    "INSUFFICIENT_BACKING"
);
```

### Dynamic Fee Calculation
```solidity
// Stress-based burn fees
function getBurnFeeBps() public view returns (uint256) {
    uint256 stress = (totalDUSDSupply * 1e18) / totalSyntheticValue;
    uint256 fee = MIN_FEE_BPS + 
        (stress * (MAX_FEE_BPS - MIN_FEE_BPS)) / 1e18;
    return Math.min(fee, MAX_FEE_BPS);
}
```

### Settlement Lock Protection
```solidity
// Anti-MEV protection
mapping(address => uint256) public lastSwapTime;
uint256 public constant SETTLEMENT_LOCK = 1 minutes;

modifier settlementLock() {
    require(
        block.timestamp >= lastSwapTime[msg.sender] + SETTLEMENT_LOCK,
        "Settlement locked"
    );
    _;
}
```

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

**Key Functions:**
- `updateImplementation()`: Upgrade token implementation
- `deploySynthetic()`: Clone and initialize new synthetic
- `setSyntheticStatus()`: Enable/disable trading
- `mint()`: Mint synthetic tokens (virtual positions)
- `burn()`: Burn synthetic tokens (virtual positions)

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

// Check dynamic fees and protocol limits
const burnFee = await dswap.getBurnFeeBps()
const stressRatio = await dswap.getStressRatio()
const canMint = await dswap.canMint('1000')

// Get virtual asset info
const synthInfo = await dswap.synthManager.getSynthetic(keccak256('xAAPL'))
console.log('Virtual asset address:', synthInfo.token)
console.log('Total virtual supply:', synthInfo.totalSupply)

// Execute swap with settlement awareness
const tx = await dswap.mintSynthetic('xAAPL', '1000')
// Note: 1-minute settlement lock applies after trade
```

### Smart Contract Integration
```solidity
interface ISwapRouter {
    function mintSynthetic(
        bytes32 assetId,
        uint256 dUSDAmount,
        uint256 minAmountOut
    ) external;
    
    function burnSynthetic(
        bytes32 assetId,
        uint256 synthAmount,
        uint256 minAmountOut
    ) external;
    
    function swapSynthetic(
        bytes32 fromAssetId,
        bytes32 toAssetId,
        uint256 amountIn,
        uint256 minAmountOut
    ) external;
}

interface IDSwap {
    // Risk management functions
    function getBurnFeeBps() external view returns (uint256);
    function getStressRatio() external view returns (uint256);
    function getMaxBorrowableDUSD() external view returns (uint256);
    function getTotalSyntheticValue() external view returns (uint256);
    function canMint(uint256 dUSDAmount) external view returns (bool);
    function lastSwapTime(address user) external view returns (uint256);
}

interface ISynthManager {
    function getSynthetic(bytes32 assetId) external view returns (SynthInfo memory);
    function getAllSynthetics() external view returns (bytes32[] memory);
    function deploySynthetic(bytes32 assetId, string memory name, string memory symbol, bytes32 priceId) external;
    function updateImplementation(address newImpl) external;
}

// Settlement lock modifier example
modifier checkSettlementLock() {
    require(
        block.timestamp >= lastSwapTime[msg.sender] + SETTLEMENT_LOCK,
        "Settlement period active"
    );
    _;
}
```

## Deployment Sequence

1. Deploy `SyntheticToken` implementation
2. Deploy `SynthManager` with implementation address
3. Deploy `DSwap` with oracle, DUSD, and SynthManager
4. Deploy `SwapRouter` with DSwap, SynthManager, and DUSD
5. Grant DSwap permission to call SynthManager
6. Deploy synthetics via `SynthManager.deploySynthetic()`
7. Configure fees in DSwap
8. Set fee collector address

### Example Deployment Script

```solidity
// 1. Deploy implementation
SyntheticToken implementation = new SyntheticToken();

// 2. Deploy SynthManager
SynthManager synthManager = new SynthManager(address(implementation));

// 3. Deploy DSwap
DSwap dSwap = new DSwap(oracle, dUSD, address(synthManager));

// 4. Deploy SwapRouter
SwapRouter router = new SwapRouter(address(dSwap), address(synthManager), dUSD);

// 5. Grant roles
synthManager.grantRole(ADMIN_ROLE, address(dSwap));

// 6. Deploy synthetics (virtual assets)
synthManager.deploySynthetic(
    keccak256("xBTC"),
    "Synthetic Bitcoin",
    "xBTC",
    pythBTCPriceId
);

// 7. Configure fees
dSwap.setFees(30); // 0.3%
dSwap.setFeeCollector(treasury);
```

## Advanced Security Features

### Protocol Solvency Protection
- **Dynamic Backing Limits**: Real-time calculation prevents over-issuance
- **Stress Monitoring**: Continuous ratio tracking and fee adjustment
- **Hard Caps**: Mathematical impossibility of insolvency
- **Emergency Controls**: Governance pause and parameter adjustment

### MEV and Arbitrage Protection
- **Settlement Locks**: 1-minute cooldown prevents rapid arbitrage
- **Oracle Anchoring**: No AMM manipulation possible
- **Fee Curves**: Economic disincentives for destabilizing behavior

### Operational Security
- Multi-signature governance for parameter changes
- Oracle staleness and validity checks
- Role-based access control
- Comprehensive event logging

## Benefits

### For Traders
- Access to global markets 24/7
- Zero slippage oracle-based pricing
- Instant settlement with MEV protection
- No counterparty risk
- Dynamic fees reflect market conditions

### For the Protocol
- Guaranteed solvency through hard limits
- Self-balancing through dynamic fees
- Scalable virtual asset system
- Minimal operational overhead
- Advanced risk management

## Security Features

### Four-Layer Defense System
1. **Hard Invariant**: Dynamic backing limits prevent insolvency
2. **Dynamic Fees**: Economic pressure maintains balance
3. **Settlement Locks**: MEV and arbitrage protection
4. **Real-time Monitoring**: Continuous risk assessment

### Risk Classification
| Operation | Risk Level | Fee Type |
|-----------|------------|----------|
| **Mint Synthetic** | Low | 0.3% flat |
| **Swap Synthetic** | Low | 0.3% flat |
| **Burn Synthetic** | High | 0.3-2% dynamic |

**Key Insight**: Only burning synthetics (synthetic → DUSD) creates solvency risk by minting new DUSD, hence the dynamic fee structure. Minting and swapping are safe operations with flat fees.