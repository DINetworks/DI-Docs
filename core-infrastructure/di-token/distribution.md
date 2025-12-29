# Distribution

The DI token distribution is designed to ensure fair allocation across stakeholders while maintaining long-term protocol sustainability and community ownership.

## Distribution Overview

### Total Supply Allocation
**Total Supply**: 1,000,000,000 DI (1 billion tokens)

| Category | Allocation | Amount | Purpose |
|----------|------------|--------|---------|
| **Community Treasury** | 25% | 250M DI | Governance-controlled funds |
| **Ecosystem Incentives** | 30% | 300M DI | Liquidity mining & rewards |
| **Team & Advisors** | 20% | 200M DI | Core team compensation |
| **Private Sale** | 15% | 150M DI | Strategic investors |
| **Public Sale** | 5% | 50M DI | Public token sale |
| **Liquidity Bootstrap** | 5% | 50M DI | Initial DEX liquidity |

## Vesting Schedules

### Team & Advisors (200M DI)
```
Cliff Period: 12 months
Vesting Duration: 36 months linear
Monthly Release: 5.56M DI (1/36 of allocation)
Total Vesting: 4 years
```

### Private Sale (150M DI)
```
Cliff Period: 6 months
Vesting Duration: 18 months linear
Monthly Release: 8.33M DI (1/18 of allocation)
Total Vesting: 2 years
```

### Ecosystem Incentives (300M DI)
```
Year 1: 90M DI (30%)
Year 2: 75M DI (25%)
Year 3: 60M DI (20%)
Year 4: 45M DI (15%)
Year 5: 30M DI (10%)
```

## Release Schedule

### Monthly Token Releases
| Month | Team/Advisors | Private Sale | Ecosystem | Total Monthly |
|-------|---------------|--------------|-----------|---------------|
| 1-6 | 0 | 0 | 7.5M | 7.5M |
| 7-12 | 0 | 8.33M | 7.5M | 15.83M |
| 13-24 | 5.56M | 8.33M | 6.25M | 20.14M |
| 25-36 | 5.56M | 0 | 5M | 10.56M |
| 37-48 | 5.56M | 0 | 3.75M | 9.31M |
| 49-60 | 5.56M | 0 | 2.5M | 8.06M |

### Circulating Supply Projection
```javascript
const calculateCirculatingSupply = (monthsElapsed) => {
  let circulating = 50_000_000 // Public sale + liquidity bootstrap
  
  // Ecosystem incentives (starts immediately)
  if (monthsElapsed <= 12) {
    circulating += monthsElapsed * 7_500_000 // 7.5M per month
  } else if (monthsElapsed <= 24) {
    circulating += 90_000_000 + (monthsElapsed - 12) * 6_250_000
  } // ... continue for other years
  
  // Private sale (starts month 7)
  if (monthsElapsed >= 7) {
    const vestingMonths = Math.min(monthsElapsed - 6, 18)
    circulating += vestingMonths * 8_333_333
  }
  
  // Team & advisors (starts month 13)
  if (monthsElapsed >= 13) {
    const vestingMonths = Math.min(monthsElapsed - 12, 36)
    circulating += vestingMonths * 5_555_556
  }
  
  return circulating
}
```

## Community Treasury

### Treasury Management
The Community Treasury (250M DI) is controlled by governance and used for:
- **Development Grants**: Fund protocol improvements
- **Marketing & Partnerships**: Ecosystem growth initiatives
- **Security Audits**: Ongoing security assessments
- **Emergency Fund**: Crisis management reserves
- **Buyback Programs**: Token value support

### Treasury Allocation Guidelines
| Category | Allocation | Annual Budget |
|----------|------------|---------------|
| Development | 40% | 100M DI |
| Marketing | 25% | 62.5M DI |
| Security | 15% | 37.5M DI |
| Emergency Fund | 10% | 25M DI |
| Buybacks | 10% | 25M DI |

### Governance Approval Process
```javascript
// Propose treasury spending
const proposeTreasurySpending = async (recipient, amount, purpose) => {
  const targets = [TREASURY_ADDRESS]
  const values = [0]
  const calldatas = [
    treasury.interface.encodeFunctionData("transfer", [recipient, amount])
  ]
  const description = `Treasury allocation: ${purpose}`
  
  return await governor.propose(targets, values, calldatas, description)
}
```

## Ecosystem Incentives

### Liquidity Mining Distribution
| Pool | Year 1 | Year 2 | Year 3 | Year 4 | Year 5 |
|------|--------|--------|--------|--------|--------|
| **DI/ETH** | 36M | 30M | 24M | 18M | 12M |
| **DI/USDC** | 27M | 22.5M | 18M | 13.5M | 9M |
| **DI/DUSD** | 18M | 15M | 12M | 9M | 6M |
| **Trading Rewards** | 9M | 7.5M | 6M | 4.5M | 3M |

### Reward Calculation
```solidity
contract LiquidityMining {
    mapping(address => uint256) public rewardRate;
    mapping(address => uint256) public lastUpdateTime;
    mapping(address => uint256) public rewardPerTokenStored;
    
    function earned(address account) public view returns (uint256) {
        return balanceOf(account) * 
               (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18 + 
               rewards[account];
    }
    
    function rewardPerToken() public view returns (uint256) {
        if (totalSupply() == 0) return rewardPerTokenStored;
        
        return rewardPerTokenStored + 
               (lastTimeRewardApplicable() - lastUpdateTime) * 
               rewardRate * 1e18 / totalSupply();
    }
}
```

## Private Sale Details

### Sale Structure
- **Total Raise**: $15M USD
- **Token Price**: $0.10 per DI
- **Tokens Sold**: 150M DI
- **Valuation**: $100M FDV

### Investor Categories
| Category | Allocation | Vesting | Lock-up |
|----------|------------|---------|---------|
| **Strategic Partners** | 60% | 2 years | 6 months |
| **VCs & Funds** | 30% | 2 years | 6 months |
| **Angel Investors** | 10% | 2 years | 6 months |

### Investor Rights
- Governance voting rights (after vesting)
- Anti-dilution protection
- Information rights
- Board observer rights (major investors)

## Public Sale

### Sale Mechanics
- **Platform**: Decentralized launch pad
- **Price**: $0.20 per DI (2x private sale)
- **Allocation**: 50M DI
- **Raise**: $10M USD
- **No Vesting**: Immediate liquidity

### Fair Launch Features
- **Whitelist**: Community-based allocation
- **Max Purchase**: $5,000 per wallet
- **Anti-Bot**: Captcha and KYC verification
- **Refund Mechanism**: Oversubscription handling

## Liquidity Bootstrap

### Initial DEX Liquidity
- **DI Allocation**: 50M DI
- **Paired Assets**: ETH, USDC, DUSD
- **Initial Price**: $0.20 per DI
- **Lock Period**: 12 months

### Liquidity Provision
```javascript
// Bootstrap liquidity pools
const bootstrapLiquidity = async () => {
  // DI/ETH pool (60% of allocation)
  await addLiquidity(
    DI_ADDRESS,
    WETH_ADDRESS,
    parseEther("30000000"), // 30M DI
    parseEther("3000")      // 3000 ETH at $2000/ETH
  )
  
  // DI/USDC pool (40% of allocation)
  await addLiquidity(
    DI_ADDRESS,
    USDC_ADDRESS,
    parseEther("20000000"), // 20M DI
    parseUnits("4000000", 6) // 4M USDC
  )
}
```

## Distribution Monitoring

### Real-time Tracking
```javascript
const useTokenDistribution = () => {
  return useQuery(['tokenDistribution'], async () => {
    const [
      totalSupply,
      circulatingSupply,
      treasuryBalance,
      teamVested,
      privateVested
    ] = await Promise.all([
      diToken.totalSupply(),
      getCirculatingSupply(),
      diToken.balanceOf(TREASURY_ADDRESS),
      getTeamVestedAmount(),
      getPrivateVestedAmount()
    ])
    
    return {
      totalSupply: formatEther(totalSupply),
      circulatingSupply: formatEther(circulatingSupply),
      treasuryBalance: formatEther(treasuryBalance),
      teamVested: formatEther(teamVested),
      privateVested: formatEther(privateVested),
      circulationRatio: Number(circulatingSupply) / Number(totalSupply)
    }
  })
}
```

### Distribution Dashboard
```javascript
const DistributionDashboard = () => {
  const { data: distribution } = useTokenDistribution()
  
  return (
    <div className="distribution-dashboard">
      <div className="supply-metrics">
        <div>Total Supply: {formatNumber(distribution.totalSupply)} DI</div>
        <div>Circulating: {formatNumber(distribution.circulatingSupply)} DI</div>
        <div>Circulation Ratio: {(distribution.circulationRatio * 100).toFixed(1)}%</div>
      </div>
      
      <div className="vesting-progress">
        <VestingChart category="team" />
        <VestingChart category="private" />
        <VestingChart category="ecosystem" />
      </div>
    </div>
  )
}
```

## Compliance & Transparency

### Vesting Contracts
All vesting schedules are enforced by smart contracts:
```solidity
contract TokenVesting {
    struct VestingSchedule {
        uint256 cliff;
        uint256 duration;
        uint256 slicePeriodSeconds;
        bool revocable;
        uint256 amountTotal;
        uint256 released;
        bool revoked;
    }
    
    function release(bytes32 vestingScheduleId, uint256 amount) external {
        VestingSchedule storage vestingSchedule = vestingSchedules[vestingScheduleId];
        require(getReleasableAmount(vestingScheduleId) >= amount, "Insufficient vested amount");
        
        vestingSchedule.released += amount;
        token.transfer(vestingSchedule.beneficiary, amount);
    }
}
```

### Transparency Reports
- Monthly distribution reports
- Vesting schedule updates
- Treasury spending summaries
- Governance proposal impacts

## Future Considerations

### Distribution Adjustments
- Governance can modify future emissions
- Emergency distribution mechanisms
- Cross-chain distribution expansion
- Deflationary mechanisms implementation