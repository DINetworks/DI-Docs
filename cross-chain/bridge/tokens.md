# Bridged Tokens

Bridged tokens are representations of assets from other blockchains that exist on DI Network. They maintain 1:1 backing with their original assets through the bridge's lock-and-mint mechanism.

## Token Standards

### Naming Convention
Bridged tokens follow a consistent naming pattern:
- **Prefix**: Chain identifier (e.g., "e" for Ethereum)
- **Symbol**: Original token symbol
- **Example**: eBTC (Ethereum Bitcoin), pUSDC (Polygon USDC)

### Contract Implementation
```solidity
contract BridgedToken is ERC20, IBridgedToken {
    address public bridge;
    string public originChain;
    address public originContract;
    
    modifier onlyBridge() {
        require(msg.sender == bridge, "Only bridge can mint/burn");
        _;
    }
    
    function mint(address to, uint256 amount) external onlyBridge {
        _mint(to, amount);
    }
    
    function burn(address from, uint256 amount) external onlyBridge {
        _burn(from, amount);
    }
}
```

## Supported Bridged Tokens

### Major Cryptocurrencies
| Token | Symbol | Origin Chain | DI Network Symbol |
|-------|--------|--------------|-------------------|
| Bitcoin | BTC | Bitcoin | eBTC |
| Ethereum | ETH | Ethereum | eETH |
| USD Coin | USDC | Ethereum | eUSDC |
| Tether | USDT | Ethereum | eUSDT |

### DeFi Tokens
| Token | Symbol | Origin Chain | DI Network Symbol |
|-------|--------|--------------|-------------------|
| Uniswap | UNI | Ethereum | eUNI |
| Aave | AAVE | Ethereum | eAAVE |
| Compound | COMP | Ethereum | eCOMP |
| SushiSwap | SUSHI | Ethereum | eSUSHI |

## Token Lifecycle

### Minting Process
1. User locks original tokens on source chain
2. Bridge validators verify the lock transaction
3. Bridged tokens are minted on DI Network
4. Tokens are transferred to user's address

```javascript
// Mint bridged tokens
const mintTx = await bridge.mint({
  originTx: lockTransactionHash,
  token: 'USDC',
  amount: '1000',
  recipient: userAddress,
  proof: merkleProof
})
```

### Burning Process
1. User initiates withdrawal request
2. Bridged tokens are burned on DI Network
3. Unlock message sent to origin chain
4. Original tokens released to user

```javascript
// Burn bridged tokens
const burnTx = await bridge.burn({
  token: 'eUSDC',
  amount: '500',
  recipient: userAddress,
  destinationChain: 'ethereum'
})
```

## Token Properties

### Metadata
Each bridged token maintains:
- Original contract address
- Origin chain identifier
- Decimal precision
- Total supply cap
- Bridge contract reference

### Functionality
Bridged tokens support:
- Standard ERC-20 operations
- Bridge-specific mint/burn functions
- Metadata queries for origin information
- Upgrade mechanisms for contract improvements

## Integration Examples

### Wallet Integration
```javascript
// Add bridged token to wallet
await wallet.addToken({
  address: bridgedTokenAddress,
  symbol: 'eUSDC',
  decimals: 6,
  image: tokenLogoUrl
})

// Check if token is bridged
const isBridged = await bridge.isBridgedToken(tokenAddress)
```

### DApp Integration
```javascript
// Get bridged token info
const tokenInfo = await bridge.getBridgedTokenInfo('eUSDC')
// Returns: { originChain, originContract, totalSupply, bridge }

// Convert between bridged and native representations
const nativeAmount = await bridge.convertToNative('eUSDC', '1000')
```

## Security Features

### Mint/Burn Controls
- Only authorized bridge contracts can mint/burn
- Multi-signature validation for large amounts
- Rate limiting on mint operations
- Emergency pause functionality

### Audit Trail
All bridged token operations are logged:
```javascript
// Get token operation history
const history = await bridge.getTokenHistory('eUSDC', userAddress)
// Returns array of mint/burn events with proofs
```

## Liquidity Considerations

### Cross-Chain Arbitrage
Price differences between chains create arbitrage opportunities:
- Monitor price feeds across chains
- Execute profitable bridge transfers
- Contribute to price equilibrium

### Liquidity Incentives
- Bridge fee rebates for large transfers
- Liquidity mining rewards for bridged assets
- Reduced fees during low liquidity periods

## Governance and Upgrades

### Token Parameters
Governance can modify:
- Transfer limits per token
- Bridge fees and rebates
- Emergency pause states
- Upgrade implementations

### Upgrade Process
1. Propose upgrade through governance
2. Community review and testing period
3. Governance vote approval
4. Timelock execution
5. Contract upgrade deployment

## Troubleshooting

### Common Issues
- **Stuck Transfers**: Use bridge recovery tools
- **Balance Discrepancies**: Check cross-chain sync status
- **Failed Mints**: Verify proof validity and gas limits

### Recovery Mechanisms
```javascript
// Recover stuck bridge transfer
const recovery = await bridge.recoverTransfer({
  transferId: 'tx_hash_or_id',
  proof: validityProof,
  signatures: validatorSignatures
})
```