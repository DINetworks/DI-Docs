# Cross-Chain Interoperability Subsystem - DI Network

## Overview

The Cross-Chain Interoperability subsystem enables seamless asset transfers, contract interactions, and gasless transactions across multiple blockchain networks, forming the backbone of DI Network's multi-chain architecture.

## Core Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cross-Chain Infrastructure                    │
├─────────────────────────────────────────────────────────────────┤
│  DIGateway  │  BridgeHub  │  MetaTxGateway  │  GasCreditVault  │
├─────────────────────────────────────────────────────────────────┤
│                    Relayer Network                              │
│  Message Validation  │  Cross-Chain Execution  │  Gas Management │
├─────────────────────────────────────────────────────────────────┤
│                    Supported Chains                             │
│  Ethereum  │  BSC  │  Polygon  │  Arbitrum  │  Base  │  Crossfi │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. DIGateway (Main Bridge Contract)

**Contract**: `DIGateway.sol`

```solidity
contract DIGateway is IExecutable, AccessControl {
    bytes32 public constant RELAYER_ROLE = keccak256("RELAYER_ROLE");
    
    struct CrossChainCall {
        uint256 sourceChainId;
        uint256 destinationChainId;
        address sourceAddress;
        address destinationContract;
        bytes payload;
        uint256 gasLimit;
        address gasToken;
        uint256 gasAmount;
    }
    
    mapping(bytes32 => bool) public executedMessages;
    
    function callContract(
        uint256 destinationChainId,
        address destinationContract,
        bytes calldata payload,
        address gasToken,
        uint256 gasLimit
    ) external payable {
        bytes32 messageId = _generateMessageId(
            block.chainid,
            destinationChainId,
            msg.sender,
            destinationContract,
            payload
        );
        
        _collectGasFees(gasToken, gasLimit);
        
        emit CrossChainCallInitiated(
            messageId,
            block.chainid,
            destinationChainId,
            msg.sender,
            destinationContract,
            payload,
            gasLimit
        );
    }
    
    function callContractWithToken(
        uint256 destinationChainId,
        address destinationContract,
        bytes calldata payload,
        string memory symbol,
        uint256 amount,
        address gasToken,
        uint256 gasLimit
    ) external payable {
        // Lock tokens on source chain
        _lockToken(symbol, amount);
        
        bytes32 messageId = _generateMessageId(
            block.chainid,
            destinationChainId,
            msg.sender,
            destinationContract,
            abi.encode(payload, symbol, amount)
        );
        
        _collectGasFees(gasToken, gasLimit);
        
        emit CrossChainCallWithTokenInitiated(
            messageId,
            block.chainid,
            destinationChainId,
            msg.sender,
            destinationContract,
            symbol,
            amount,
            payload
        );
    }
    
    function execute(
        bytes32 messageId,
        uint256 sourceChainId,
        address sourceAddress,
        address destinationContract,
        bytes calldata payload
    ) external onlyRole(RELAYER_ROLE) {
        require(!executedMessages[messageId], "Already executed");
        executedMessages[messageId] = true;
        
        (bool success, bytes memory result) = destinationContract.call(payload);
        
        emit CrossChainCallExecuted(
            messageId,
            sourceChainId,
            sourceAddress,
            destinationContract,
            success,
            result
        );
    }
}
```

**Key Features**:
- **Contract Calls**: Execute functions on destination chains
- **Token Transfers**: Bridge tokens with contract calls
- **Gas Management**: Unified gas payment system
- **Message Validation**: Prevent replay attacks

### 2. BridgeHub (Token Management)

**Contract**: `BridgeHub.sol`

```solidity
contract BridgeHub is AccessControl {
    struct TokenConfig {
        address tokenAddress;
        uint256 decimals;
        uint256 minAmount;
        uint256 maxAmount;
        uint256 fee; // Basis points
        bool isActive;
    }
    
    mapping(string => TokenConfig) public supportedTokens;
    mapping(string => mapping(uint256 => uint256)) public chainLimits;
    
    function bridgeToken(
        string memory symbol,
        uint256 amount,
        uint256 destinationChainId,
        address recipient
    ) external {
        TokenConfig memory config = supportedTokens[symbol];
        require(config.isActive, "Token not supported");
        require(amount >= config.minAmount && amount <= config.maxAmount, "Invalid amount");
        
        uint256 fee = (amount * config.fee) / 10000;
        uint256 bridgeAmount = amount - fee;
        
        // Lock tokens on source chain
        IERC20(config.tokenAddress).transferFrom(msg.sender, address(this), amount);
        
        // Collect fees
        if (fee > 0) {
            IERC20(config.tokenAddress).transfer(feeCollector, fee);
        }
        
        emit TokenBridged(
            symbol,
            amount,
            bridgeAmount,
            block.chainid,
            destinationChainId,
            msg.sender,
            recipient
        );
    }
    
    function releaseToken(
        string memory symbol,
        uint256 amount,
        address recipient,
        bytes32 messageId
    ) external onlyRole(RELAYER_ROLE) {
        require(!processedMessages[messageId], "Already processed");
        processedMessages[messageId] = true;
        
        TokenConfig memory config = supportedTokens[symbol];
        
        // Mint or unlock tokens on destination chain
        if (config.tokenAddress == address(0)) {
            // Mint wrapped token
            IBridgedToken(wrappedTokens[symbol]).mint(recipient, amount);
        } else {
            // Unlock native token
            IERC20(config.tokenAddress).transfer(recipient, amount);
        }
        
        emit TokenReleased(symbol, amount, recipient, messageId);
    }
}
```

### 3. MetaTxGateway (Gasless Transactions)

**Contract**: `MetaTxGateway.sol`

```solidity
contract MetaTxGateway is AccessControl {
    struct MetaTransaction {
        address from;
        address to;
        uint256 value;
        bytes data;
        uint256 nonce;
        uint256 gasPrice;
        address gasToken; // DUSD for gasless txs
        uint256 gasLimit;
        uint256 deadline;
    }
    
    mapping(address => uint256) public nonces;
    mapping(address => bool) public approvedTokens;
    
    function executeMetaTransaction(
        MetaTransaction memory metaTx,
        bytes memory signature
    ) external {
        require(block.timestamp <= metaTx.deadline, "Transaction expired");
        require(metaTx.nonce == nonces[metaTx.from], "Invalid nonce");
        
        bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(
            keccak256("MetaTransaction(address from,address to,uint256 value,bytes data,uint256 nonce,uint256 gasPrice,address gasToken,uint256 gasLimit,uint256 deadline)"),
            metaTx.from,
            metaTx.to,
            metaTx.value,
            keccak256(metaTx.data),
            metaTx.nonce,
            metaTx.gasPrice,
            metaTx.gasToken,
            metaTx.gasLimit,
            metaTx.deadline
        )));
        
        address signer = ECDSA.recover(digest, signature);
        require(signer == metaTx.from, "Invalid signature");
        
        nonces[metaTx.from]++;
        
        // Collect gas fees in DUSD
        uint256 gasCost = metaTx.gasPrice * metaTx.gasLimit;
        IERC20(metaTx.gasToken).transferFrom(metaTx.from, msg.sender, gasCost);
        
        // Execute transaction
        (bool success, bytes memory result) = metaTx.to.call{value: metaTx.value}(metaTx.data);
        
        emit MetaTransactionExecuted(
            metaTx.from,
            metaTx.to,
            metaTx.nonce,
            success,
            result
        );
    }
    
    function batchExecuteMetaTransactions(
        MetaTransaction[] memory metaTxs,
        bytes[] memory signatures
    ) external {
        require(metaTxs.length == signatures.length, "Length mismatch");
        
        for (uint256 i = 0; i < metaTxs.length; i++) {
            executeMetaTransaction(metaTxs[i], signatures[i]);
        }
    }
}
```

### 4. GasCreditVault (Gas Management)

**Contract**: `GasCreditVault.sol`

```solidity
contract GasCreditVault {
    IERC20 public dusd;
    
    struct GasCredit {
        uint256 balance;
        uint256 lastUpdated;
    }
    
    mapping(address => GasCredit) public gasCredits;
    mapping(uint256 => uint256) public chainGasRates; // DUSD per gas unit
    
    function depositGasCredits(uint256 amount) external {
        dusd.transferFrom(msg.sender, address(this), amount);
        gasCredits[msg.sender].balance += amount;
        gasCredits[msg.sender].lastUpdated = block.timestamp;
        
        emit GasCreditsDeposited(msg.sender, amount);
    }
    
    function withdrawGasCredits(uint256 amount) external {
        require(gasCredits[msg.sender].balance >= amount, "Insufficient credits");
        
        gasCredits[msg.sender].balance -= amount;
        dusd.transfer(msg.sender, amount);
        
        emit GasCreditsWithdrawn(msg.sender, amount);
    }
    
    function consumeGasCredits(
        address user,
        uint256 chainId,
        uint256 gasUsed
    ) external onlyRole(RELAYER_ROLE) {
        uint256 gasCost = gasUsed * chainGasRates[chainId];
        require(gasCredits[user].balance >= gasCost, "Insufficient gas credits");
        
        gasCredits[user].balance -= gasCost;
        
        emit GasCreditsConsumed(user, chainId, gasUsed, gasCost);
    }
    
    function getGasCreditsBalance(address user) external view returns (uint256) {
        return gasCredits[user].balance;
    }
}
```

## Bridged Token System

### DIBridgedToken (Wrapped Tokens)

**Contract**: `DIBridgedToken.sol`

```solidity
contract DIBridgedToken is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    
    uint256 public sourceChainId;
    address public sourceToken;
    
    constructor(
        string memory name,
        string memory symbol,
        uint256 _sourceChainId,
        address _sourceToken
    ) ERC20(name, symbol) {
        sourceChainId = _sourceChainId;
        sourceToken = _sourceToken;
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
    
    function burn(address from, uint256 amount) external onlyRole(BURNER_ROLE) {
        _burn(from, amount);
    }
    
    function burnFrom(address account, uint256 amount) external onlyRole(BURNER_ROLE) {
        _spendAllowance(account, msg.sender, amount);
        _burn(account, amount);
    }
}
```

### Token Registry

**Contract**: `DIBridgedTokenRegistry.sol`

```solidity
contract DIBridgedTokenRegistry is AccessControl {
    struct TokenMapping {
        address sourceToken;
        address bridgedToken;
        uint256 sourceChainId;
        bool isActive;
    }
    
    mapping(bytes32 => TokenMapping) public tokenMappings;
    mapping(uint256 => address[]) public chainTokens;
    
    function registerToken(
        address sourceToken,
        uint256 sourceChainId,
        string memory name,
        string memory symbol
    ) external onlyRole(ADMIN_ROLE) returns (address bridgedToken) {
        bytes32 key = keccak256(abi.encodePacked(sourceToken, sourceChainId));
        require(tokenMappings[key].bridgedToken == address(0), "Token already registered");
        
        bridgedToken = address(new DIBridgedToken(name, symbol, sourceChainId, sourceToken));
        
        tokenMappings[key] = TokenMapping({
            sourceToken: sourceToken,
            bridgedToken: bridgedToken,
            sourceChainId: sourceChainId,
            isActive: true
        });
        
        chainTokens[sourceChainId].push(bridgedToken);
        
        emit TokenRegistered(sourceToken, bridgedToken, sourceChainId);
    }
    
    function getBridgedToken(
        address sourceToken,
        uint256 sourceChainId
    ) external view returns (address) {
        bytes32 key = keccak256(abi.encodePacked(sourceToken, sourceChainId));
        return tokenMappings[key].bridgedToken;
    }
}
```

## Cross-Chain Message Flow

### 1. Message Initiation

```solidity
// User initiates cross-chain call
diGateway.callContract(
    destinationChainId,
    destinationContract,
    payload,
    gasToken,
    gasLimit
);
```

### 2. Relayer Processing

```javascript
// Off-chain relayer monitors events
const filter = diGateway.filters.CrossChainCallInitiated();
diGateway.on(filter, async (messageId, sourceChainId, destinationChainId, sourceAddress, destinationContract, payload, gasLimit) => {
    // Validate message
    const isValid = await validateMessage(messageId, sourceChainId, payload);
    if (!isValid) return;
    
    // Execute on destination chain
    const destinationGateway = getGatewayContract(destinationChainId);
    await destinationGateway.execute(
        messageId,
        sourceChainId,
        sourceAddress,
        destinationContract,
        payload
    );
});
```

### 3. Message Execution

```solidity
// Relayer executes on destination chain
function execute(
    bytes32 messageId,
    uint256 sourceChainId,
    address sourceAddress,
    address destinationContract,
    bytes calldata payload
) external onlyRole(RELAYER_ROLE) {
    require(!executedMessages[messageId], "Already executed");
    executedMessages[messageId] = true;
    
    (bool success, bytes memory result) = destinationContract.call(payload);
    
    emit CrossChainCallExecuted(messageId, sourceChainId, sourceAddress, destinationContract, success, result);
}
```

## Gasless Transaction Flow

### 1. Meta-Transaction Creation

```javascript
// User creates meta-transaction
const metaTx = {
    from: userAddress,
    to: targetContract,
    value: 0,
    data: encodedFunctionCall,
    nonce: await metaTxGateway.nonces(userAddress),
    gasPrice: ethers.parseUnits("20", "gwei"),
    gasToken: dusdAddress,
    gasLimit: 200000,
    deadline: Math.floor(Date.now() / 1000) + 3600
};

// User signs meta-transaction
const signature = await user.signTypedData(domain, types, metaTx);
```

### 2. Relayer Execution

```javascript
// Relayer executes meta-transaction
await metaTxGateway.executeMetaTransaction(metaTx, signature);
```

### 3. Gas Credit Management

```javascript
// User deposits gas credits
await dusd.approve(gasCreditVault.address, ethers.parseEther("100"));
await gasCreditVault.depositGasCredits(ethers.parseEther("100"));

// Check gas credits balance
const balance = await gasCreditVault.getGasCreditsBalance(userAddress);
```

## Multi-Chain Deployment

### Supported Networks

| Network | Chain ID | Gateway Address | Status |
|---------|----------|-----------------|--------|
| Ethereum | 1 | `0x...` | ✅ Live |
| BSC | 56 | `0x...` | ✅ Live |
| Polygon | 137 | `0x...` | ✅ Live |
| Arbitrum | 42161 | `0x...` | ✅ Live |
| Base | 8453 | `0x...` | ✅ Live |
| Crossfi | 4157 | `0x...` | ✅ Live |

### Chain Configuration

```solidity
struct ChainConfig {
    uint256 chainId;
    string name;
    address gateway;
    address bridgeHub;
    address metaTxGateway;
    uint256 gasRate; // DUSD per gas unit
    uint256 blockTime; // Average block time
    bool isActive;
}
```

## Security Features

### Message Validation

```solidity
function _validateMessage(
    bytes32 messageId,
    uint256 sourceChainId,
    bytes calldata payload
) private view returns (bool) {
    // Check message hasn't been executed
    if (executedMessages[messageId]) return false;
    
    // Validate source chain
    if (!supportedChains[sourceChainId]) return false;
    
    // Validate payload size
    if (payload.length > MAX_PAYLOAD_SIZE) return false;
    
    return true;
}
```

### Rate Limiting

```solidity
mapping(address => uint256) public lastTransactionTime;
mapping(address => uint256) public transactionCount;
uint256 public constant RATE_LIMIT_WINDOW = 1 hours;
uint256 public constant MAX_TRANSACTIONS_PER_HOUR = 10;

modifier rateLimited() {
    if (block.timestamp - lastTransactionTime[msg.sender] > RATE_LIMIT_WINDOW) {
        transactionCount[msg.sender] = 0;
        lastTransactionTime[msg.sender] = block.timestamp;
    }
    
    require(transactionCount[msg.sender] < MAX_TRANSACTIONS_PER_HOUR, "Rate limit exceeded");
    transactionCount[msg.sender]++;
    _;
}
```

### Emergency Controls

```solidity
bool public emergencyPaused;

modifier whenNotPaused() {
    require(!emergencyPaused, "Emergency pause active");
    _;
}

function emergencyPause() external onlyRole(ADMIN_ROLE) {
    emergencyPaused = true;
    emit EmergencyPaused();
}
```

## Integration Examples

### Basic Cross-Chain Call

```javascript
// Call contract on another chain
const payload = targetContract.interface.encodeFunctionData("someFunction", [param1, param2]);

await diGateway.callContract(
    destinationChainId,
    targetContract.address,
    payload,
    dusdAddress, // Gas token
    200000 // Gas limit
);
```

### Cross-Chain Token Transfer

```javascript
// Bridge tokens to another chain
await bridgeHub.bridgeToken(
    "DUSD",
    ethers.parseEther("1000"),
    destinationChainId,
    recipientAddress
);
```

### Gasless Transaction

```javascript
// Execute gasless transaction
const metaTx = {
    from: userAddress,
    to: targetContract.address,
    value: 0,
    data: encodedCall,
    nonce: await metaTxGateway.nonces(userAddress),
    gasPrice: ethers.parseUnits("20", "gwei"),
    gasToken: dusdAddress,
    gasLimit: 200000,
    deadline: Math.floor(Date.now() / 1000) + 3600
};

const signature = await user.signTypedData(domain, types, metaTx);
await metaTxGateway.executeMetaTransaction(metaTx, signature);
```

## Monitoring & Analytics

### Key Metrics

- **Cross-Chain Volume**: Total value transferred across chains
- **Message Success Rate**: Percentage of successful cross-chain calls
- **Gas Credits Usage**: DUSD consumed for gasless transactions
- **Bridge Utilization**: Usage per chain and token
- **Relayer Performance**: Message processing times

### Events for Tracking

```solidity
event CrossChainCallInitiated(bytes32 indexed messageId, uint256 sourceChainId, uint256 destinationChainId, address sourceAddress, address destinationContract, bytes payload, uint256 gasLimit);
event CrossChainCallExecuted(bytes32 indexed messageId, uint256 sourceChainId, address sourceAddress, address destinationContract, bool success, bytes result);
event TokenBridged(string symbol, uint256 amount, uint256 bridgeAmount, uint256 sourceChainId, uint256 destinationChainId, address sender, address recipient);
event MetaTransactionExecuted(address indexed from, address indexed to, uint256 nonce, bool success, bytes result);
event GasCreditsDeposited(address indexed user, uint256 amount);
event GasCreditsConsumed(address indexed user, uint256 chainId, uint256 gasUsed, uint256 gasCost);
```

---

*Cross-Chain Interoperability Subsystem - Connecting the Multi-Chain Future*