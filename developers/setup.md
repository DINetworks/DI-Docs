# Developer Setup

Welcome to DI Network development! This guide will help you set up your development environment and start building applications that integrate with the DI Network protocol.

## Prerequisites

Before you begin, ensure you have:

- [ ] **Node.js**: Version 16+ installed
- [ ] **Package Manager**: npm, yarn, or pnpm
- [ ] **Web3 Wallet**: MetaMask or similar for testing
- [ ] **Code Editor**: VS Code recommended with Solidity extensions
- [ ] **Git**: For version control and cloning repositories

## Quick Start

### 1. Install the DI Network SDK

{% tabs %}
{% tab title="npm" %}
```bash
npm install @dinetwork/sdk
```
{% endtab %}

{% tab title="yarn" %}
```bash
yarn add @dinetwork/sdk
```
{% endtab %}

{% tab title="pnpm" %}
```bash
pnpm add @dinetwork/sdk
```
{% endtab %}
{% endtabs %}

### 2. Basic Setup

```javascript
import { DINetwork } from '@dinetwork/sdk';

// Initialize the SDK
const di = new DINetwork({
  chainId: 1, // Ethereum mainnet
  provider: window.ethereum, // or your preferred provider
  apiKey: 'your-api-key' // Optional for enhanced features
});

// Check connection
const isConnected = await di.isConnected();
console.log('Connected to DI Network:', isConnected);
```

### 3. Your First Integration

```javascript
// Get user's DI token balance
const diBalance = await di.tokens.getDIBalance(userAddress);

// Get DUSD borrowing capacity
const borrowCapacity = await di.dusd.getBorrowingCapacity(userAddress);

// Mint DUSD
const tx = await di.dusd.mint({
  collateralAmount: ethers.parseEther("1000"), // 1000 DI tokens
  dusdAmount: ethers.parseEther("750") // 750 DUSD
});

await tx.wait();
console.log('DUSD minted successfully!');
```

## Development Environment Setup

### 1. Clone the Starter Template

```bash
git clone https://github.com/dinetwork/di-app-template.git
cd di-app-template
npm install
```

### 2. Environment Configuration

Create a `.env.local` file:

```bash
# Network Configuration
NEXT_PUBLIC_CHAIN_ID=1
NEXT_PUBLIC_RPC_URL=https://mainnet.infura.io/v3/YOUR_KEY

# DI Network Configuration
NEXT_PUBLIC_DI_API_URL=https://api.dinetwork.xyz
NEXT_PUBLIC_DI_API_KEY=your-api-key

# Contract Addresses (Ethereum Mainnet)
NEXT_PUBLIC_DI_TOKEN=0x...
NEXT_PUBLIC_DUSD_TOKEN=0x...
NEXT_PUBLIC_DI_GATEWAY=0x...
NEXT_PUBLIC_DSWAP=0x...
NEXT_PUBLIC_DPERP=0x...

# Optional: Analytics
NEXT_PUBLIC_ANALYTICS_ID=your-analytics-id
```

### 3. Development Server

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
```

Your app will be available at `http://localhost:3000`.

## SDK Reference

### Core Modules

The DI Network SDK is organized into several modules:

```javascript
const di = new DINetwork(config);

// Core token operations
di.tokens.getDIBalance(address)
di.tokens.getDUSDBalance(address)
di.tokens.approve(token, spender, amount)

// DUSD stablecoin operations
di.dusd.mint(params)
di.dusd.repay(params)
di.dusd.getBorrowingCapacity(address)
di.dusd.getPosition(address)

// Synthetic asset trading
di.dswap.mintSynthetic(assetId, dusdAmount)
di.dswap.burnSynthetic(assetId, synthAmount)
di.dswap.swapSynthetic(fromAssetId, toAssetId, amount)

// Perpetual trading
di.dperp.openPosition(params)
di.dperp.closePosition(params)
di.dperp.getPosition(address, assetId, isLong)

// Cross-chain operations
di.bridge.transferAsset(params)
di.bridge.executeGaslessTransaction(params)

// Governance
di.governance.delegate(delegatee)
di.governance.vote(proposalId, support)
di.governance.getVotingPower(address)

// Staking
di.staking.stake(amount, lockPeriod)
di.staking.unstake()
di.staking.claimRewards()
```

### Configuration Options

```javascript
const config = {
  // Required
  chainId: 1, // Network chain ID
  provider: window.ethereum, // Web3 provider
  
  // Optional
  apiKey: 'your-api-key', // For enhanced features
  apiUrl: 'https://api.dinetwork.xyz', // Custom API endpoint
  gasPrice: 'fast', // 'slow', 'standard', 'fast', or custom
  gasLimit: 500000, // Default gas limit
  slippage: 0.5, // Default slippage tolerance (%)
  
  // Contract addresses (auto-detected if not provided)
  contracts: {
    diToken: '0x...',
    dusdToken: '0x...',
    diGateway: '0x...',
    dswap: '0x...',
    dperp: '0x...'
  }
};
```

## Smart Contract Integration

### Direct Contract Interaction

If you prefer to interact with contracts directly:

```javascript
import { ethers } from 'ethers';
import { DI_TOKEN_ABI, DUSD_PROVIDER_ABI } from '@dinetwork/contracts';

// Connect to contracts
const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

const diToken = new ethers.Contract(DI_TOKEN_ADDRESS, DI_TOKEN_ABI, signer);
const dusdProvider = new ethers.Contract(DUSD_PROVIDER_ADDRESS, DUSD_PROVIDER_ABI, signer);

// Deposit collateral and mint DUSD
const collateralAmount = ethers.parseEther("1000");
const dusdAmount = ethers.parseEther("750");

// Approve DI tokens
await diToken.approve(dusdProvider.address, collateralAmount);

// Deposit collateral
await dusdProvider.depositCollateral(collateralAmount);

// Mint DUSD
await dusdProvider.borrowDUSD(dusdAmount);
```

### Contract ABIs and Addresses

Install the contracts package for ABIs and addresses:

```bash
npm install @dinetwork/contracts
```

```javascript
import {
  // Contract ABIs
  DI_TOKEN_ABI,
  DUSD_ABI,
  DSWAP_ABI,
  DPERP_ABI,
  
  // Contract addresses by network
  getContractAddress,
  NETWORK_CONFIGS
} from '@dinetwork/contracts';

// Get contract address for current network
const diTokenAddress = getContractAddress('DI_TOKEN', chainId);
```

## API Integration

### GraphQL API

Query protocol data using our GraphQL API:

```javascript
import { ApolloClient, InMemoryCache, gql } from '@apollo/client';

const client = new ApolloClient({
  uri: 'https://api.dinetwork.xyz/graphql',
  cache: new InMemoryCache()
});

// Query user positions
const GET_USER_POSITIONS = gql`
  query GetUserPositions($userAddress: String!) {
    user(address: $userAddress) {
      dusdPosition {
        collateralAmount
        borrowedAmount
        interestAccrued
        healthFactor
      }
      perpPositions {
        id
        asset
        size
        collateral
        entryPrice
        isLong
        pnl
        liquidationPrice
      }
      synthBalances {
        asset
        balance
        value
      }
    }
  }
`;

const { data } = await client.query({
  query: GET_USER_POSITIONS,
  variables: { userAddress }
});
```

### REST API

For simpler integrations, use our REST API:

```javascript
// Get protocol statistics
const stats = await fetch('https://api.dinetwork.xyz/v1/stats')
  .then(res => res.json());

// Get asset prices
const prices = await fetch('https://api.dinetwork.xyz/v1/prices')
  .then(res => res.json());

// Get user data
const userData = await fetch(`https://api.dinetwork.xyz/v1/users/${userAddress}`)
  .then(res => res.json());
```

### WebSocket API

Real-time data streaming:

```javascript
import { io } from 'socket.io-client';

const socket = io('wss://ws.dinetwork.xyz');

// Subscribe to price updates
socket.emit('subscribe', { channel: 'prices' });

socket.on('price_update', (data) => {
  console.log('Price update:', data);
  // { asset: 'BTC', price: 50000, timestamp: 1234567890 }
});

// Subscribe to user position updates
socket.emit('subscribe', { 
  channel: 'positions', 
  userAddress: userAddress 
});

socket.on('position_update', (data) => {
  console.log('Position update:', data);
});
```

## Testing

### Unit Testing

Set up Jest for testing your integration:

```javascript
// __tests__/di-integration.test.js
import { DINetwork } from '@dinetwork/sdk';
import { ethers } from 'ethers';

describe('DI Network Integration', () => {
  let di;
  
  beforeEach(() => {
    // Use a test provider
    const provider = new ethers.JsonRpcProvider('http://localhost:8545');
    di = new DINetwork({
      chainId: 31337, // Hardhat local network
      provider
    });
  });
  
  test('should mint DUSD successfully', async () => {
    const tx = await di.dusd.mint({
      collateralAmount: ethers.parseEther("1000"),
      dusdAmount: ethers.parseEther("750")
    });
    
    expect(tx.hash).toBeDefined();
  });
});
```

### Integration Testing

Test against a local fork:

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: "https://mainnet.infura.io/v3/YOUR_KEY",
        blockNumber: 18500000 // Optional: pin to specific block
      }
    }
  }
};
```

```bash
# Start local fork
npx hardhat node

# Run tests against fork
npm test
```

## Example Applications

### 1. Simple Trading Interface

```javascript
import React, { useState, useEffect } from 'react';
import { DINetwork } from '@dinetwork/sdk';

function TradingInterface() {
  const [di, setDI] = useState(null);
  const [positions, setPositions] = useState([]);
  
  useEffect(() => {
    const initDI = async () => {
      const diInstance = new DINetwork({
        chainId: 1,
        provider: window.ethereum
      });
      setDI(diInstance);
      
      // Load user positions
      const userPositions = await diInstance.dperp.getPositions(userAddress);
      setPositions(userPositions);
    };
    
    initDI();
  }, []);
  
  const openPosition = async (asset, size, isLong) => {
    try {
      const tx = await di.dperp.openPosition({
        asset,
        size: ethers.parseEther(size),
        isLong,
        leverage: 10
      });
      
      await tx.wait();
      alert('Position opened successfully!');
    } catch (error) {
      console.error('Error opening position:', error);
    }
  };
  
  return (
    <div>
      <h2>DI Network Trading</h2>
      
      {/* Position opening form */}
      <div>
        <button onClick={() => openPosition('BTC', '1000', true)}>
          Long BTC
        </button>
        <button onClick={() => openPosition('BTC', '1000', false)}>
          Short BTC
        </button>
      </div>
      
      {/* Positions list */}
      <div>
        <h3>Your Positions</h3>
        {positions.map(position => (
          <div key={position.id}>
            {position.asset} - {position.isLong ? 'Long' : 'Short'} - 
            Size: ${position.size} - PnL: ${position.pnl}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 2. Portfolio Dashboard

```javascript
import React, { useState, useEffect } from 'react';
import { DINetwork } from '@dinetwork/sdk';

function PortfolioDashboard({ userAddress }) {
  const [portfolio, setPortfolio] = useState(null);
  
  useEffect(() => {
    const loadPortfolio = async () => {
      const di = new DINetwork({
        chainId: 1,
        provider: window.ethereum
      });
      
      const [
        diBalance,
        dusdBalance,
        dusdPosition,
        perpPositions,
        synthBalances
      ] = await Promise.all([
        di.tokens.getDIBalance(userAddress),
        di.tokens.getDUSDBalance(userAddress),
        di.dusd.getPosition(userAddress),
        di.dperp.getPositions(userAddress),
        di.dswap.getSynthBalances(userAddress)
      ]);
      
      setPortfolio({
        diBalance,
        dusdBalance,
        dusdPosition,
        perpPositions,
        synthBalances
      });
    };
    
    loadPortfolio();
  }, [userAddress]);
  
  if (!portfolio) return <div>Loading...</div>;
  
  return (
    <div>
      <h2>Portfolio Overview</h2>
      
      <div>
        <h3>Token Balances</h3>
        <p>DI: {ethers.formatEther(portfolio.diBalance)}</p>
        <p>DUSD: {ethers.formatEther(portfolio.dusdBalance)}</p>
      </div>
      
      <div>
        <h3>DUSD Position</h3>
        <p>Collateral: {ethers.formatEther(portfolio.dusdPosition.collateralAmount)}</p>
        <p>Borrowed: {ethers.formatEther(portfolio.dusdPosition.borrowedAmount)}</p>
        <p>Health Factor: {portfolio.dusdPosition.healthFactor}</p>
      </div>
      
      <div>
        <h3>Perpetual Positions</h3>
        {portfolio.perpPositions.map(pos => (
          <div key={pos.id}>
            {pos.asset} - {pos.isLong ? 'Long' : 'Short'} - 
            PnL: ${pos.pnl}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## Best Practices

### 1. Error Handling

```javascript
import { DINetworkError, TransactionError } from '@dinetwork/sdk';

try {
  const tx = await di.dusd.mint(params);
  await tx.wait();
} catch (error) {
  if (error instanceof DINetworkError) {
    // Handle DI Network specific errors
    console.error('DI Network Error:', error.message);
  } else if (error instanceof TransactionError) {
    // Handle transaction errors
    console.error('Transaction failed:', error.reason);
  } else {
    // Handle other errors
    console.error('Unexpected error:', error);
  }
}
```

### 2. Gas Optimization

```javascript
// Batch multiple operations
const batch = di.batch()
  .approve('DI', 'DUSD_PROVIDER', amount)
  .dusd.depositCollateral(amount)
  .dusd.mint(dusdAmount);

const tx = await batch.execute();
await tx.wait();

// Use gasless transactions when possible
const metaTx = await di.bridge.createMetaTransaction({
  to: targetContract,
  data: encodedData,
  gasToken: 'DUSD'
});

await di.bridge.executeMetaTransaction(metaTx);
```

### 3. State Management

```javascript
// Use React Context for DI Network state
import React, { createContext, useContext, useState } from 'react';

const DINetworkContext = createContext();

export function DINetworkProvider({ children }) {
  const [di, setDI] = useState(null);
  const [userAddress, setUserAddress] = useState(null);
  const [chainId, setChainId] = useState(1);
  
  const connect = async () => {
    const diInstance = new DINetwork({ chainId, provider: window.ethereum });
    setDI(diInstance);
    
    const address = await diInstance.getAddress();
    setUserAddress(address);
  };
  
  return (
    <DINetworkContext.Provider value={{
      di, userAddress, chainId, connect
    }}>
      {children}
    </DINetworkContext.Provider>
  );
}

export const useDINetwork = () => useContext(DINetworkContext);
```

## Deployment

### Environment Variables

Set up environment variables for different environments:

```bash
# .env.production
NEXT_PUBLIC_CHAIN_ID=1
NEXT_PUBLIC_RPC_URL=https://mainnet.infura.io/v3/YOUR_KEY
NEXT_PUBLIC_DI_API_URL=https://api.dinetwork.xyz

# .env.staging
NEXT_PUBLIC_CHAIN_ID=5
NEXT_PUBLIC_RPC_URL=https://goerli.infura.io/v3/YOUR_KEY
NEXT_PUBLIC_DI_API_URL=https://api-staging.dinetwork.xyz
```

### Build and Deploy

```bash
# Build for production
npm run build

# Deploy to Vercel
vercel --prod

# Deploy to Netlify
netlify deploy --prod --dir=dist
```

## Resources

### Documentation
- [Core Concepts](../core-concepts/di-token.md)
- [Subsystems](../subsystems/Tokens-Subsystem.md)
- [User Guides](../user-guides/getting-started.md)

### Tools
- [DI Network Explorer](https://explorer.dinetwork.xyz)
- [Contract Addresses](../resources/addresses.md)
- [Testnet Faucet](https://faucet.dinetwork.xyz)

### Community
- [Discord Developer Channel](https://discord.gg/dinetwork)
- [GitHub Repository](https://github.com/dinetwork)
- [Developer Forum](https://forum.dinetwork.xyz)

## Next Steps

{% content-ref url="../core-concepts/di-token.md" %}
[di-token.md](../core-concepts/di-token.md)
{% endcontent-ref %}

{% content-ref url="../subsystems/Tokens-Subsystem.md" %}
[Tokens-Subsystem.md](../subsystems/Tokens-Subsystem.md)
{% endcontent-ref %}

---

{% hint style="info" %}
**Need Help?** Join our [Discord developer channel](https://discord.gg/dinetwork) for real-time support and community discussions.
{% endhint %}