# LiquidityProvider - User Documentation

## Overview

The **LiquidityProvider** is a smart contract that enables users to deposit bUSD and automatically provide liquidity for various loan tokens (bAssets like bSPY, bMSTR). The contract utilizes the AssetSystem forge system and Aerodrome DEX pools to implement a diversified liquidity strategy. The received LP tokens (lpBUSD) represent the user's share in the liquidity pool and act like "liquid staking" with its value in bUSD growing over time as commissions accumulate.

## Deployment Model

### Permissionless Deployment

The LiquidityProvider contract can be **deployed by anyone** to create their own custom liquidity strategy or offer it as a service to users. This permissionless design enables:

- **Personal Strategies:** Deploy your own instance with custom token weights and collateral ratios
- **Service Providers:** Offer managed liquidity provision as a service with your own parameters
- **Specialized Strategies:** Create niche strategies for specific asset combinations or risk profiles

### Official BasedAssets Deployment

**BasedAssets** will deploy and maintain a **"communityLP"** contract that:
- Is monitored and maintained by automated bots
- Provides a trusted, community-focused liquidity strategy
- Serves as the reference implementation for the broader ecosystem
- Ensures regular rebalancing and optimal performance through bot management

Users can choose to use the communityLP or deploy/use alternative instances based on their preferences and trust relationships.

## Main Functions

### 1. Deposits

**Function:** `deposit(uint256 amount)`

Users can deposit bUSD and receive LP tokens (lpBUSD) representing their share in the liquidity pool.

**Process:**
1. User approves the contract for bUSD transfer
2. Contract transfers bUSD from user
3. LP tokens are minted proportionally to current total value
4. bUSD is deposited as collateral in a forge
5. Automatic distribution to configured loan tokens occurs

**LP Token Calculation:**
- **First deposit:** 1:1 ratio (1 bUSD = 1 lpBUSD)
- **Subsequent deposits:** `lpTokens = (depositedAmount * currentSupply) / totalValue`

**Events:**
```solidity
event Deposited(address indexed user, uint256 amount, uint256 lpTokens);
```

### 2. Withdrawals

**Function:** `withdraw(uint256 lpAmount)`

Users can burn their LP tokens and receive bUSD proportionally.

**Process:**
1. LP tokens are burned from user
2. Proportional bUSD value is calculated: `bUSD = (lpTokens * totalValue) / totalSupply`
3. a fee (initially 1%) is deducted (to cover slippage and system fees, this also increasing value per LP token for remaining holders)
4. Liquidity is removed from pools if needed
5. Collateral is withdrawn from forge
6. bUSD is transferred to user

**Special Cases:**
- Last withdrawal (all LP tokens): Only possible by owner, receives all remaining funds
- Automatic liquidation dissolution on complete withdrawal

**Events:**
```solidity
event Withdrawn(address indexed user, uint256 amount, uint256 lpTokens);
```

### 3. Total Value Calculation

**Function:** `getTotalValue() returns (uint256)`

Calculates the current total value of all managed assets in bUSD.

**Components:**
1. **Forge Collateral:** Value of collateral deposited in AssetSystem
2. **Free bUSD:** Directly held bUSD balance
3. **Pool Liquidity:** Value of LP positions in Aerodrome pools
   - bUSD portion in pools
   - Net position of loan tokens (pool tokens - outstanding loans)

**Formula:**
```
TotalValue = CollateralValue + FreeBUSD + Σ(bUSD in Pools) + Σ((Tokens in Pools - Loans) * PoolPrice)
```

## Strategic Mechanisms

### Collateral Ratio Management

The contract maintains the collateral-to-loan ratio within defined bounds:

**Parameters:**
- `minCollRatio`: Minimum collateral ratio (e.g., 150% = 1.5e18)
- `maxCollRatio`: Maximum collateral ratio (e.g., 200% = 2.0e18)
- **Target Ratio:** Average between min and max

**Automatic Adjustments:**

#### Scenario A: Ratio too high (above maxCollRatio)
→ **Take more loans, add liquidity**

1. Withdraw bUSD from forge
2. For each loan token by weight:
   - Take loan (mint loan tokens)
   - Add liquidity to Aerodrome pool (loan tokens + bUSD)

#### Scenario B: Ratio too low (below minCollRatio)
→ **Reduce loans, remove liquidity**

1. For each loan token by weight:
   - Remove proportional LP tokens from pool
   - Receive loan tokens + bUSD back
   - Pay back loans with received loan tokens
2. Add remaining bUSD as collateral

**Manual Rebalancing:**

The function `updateDistribution()` can be called by **anyone** (not just the owner) to trigger a rebalancing of the forge. This ensures that:
- The collateral ratio stays within the safe range (`minCollRatio` to `maxCollRatio`)
- The system can be kept healthy by community members or automated keepers
- Protection against liquidation risk during market movements

This permissionless design allows external actors to help maintain the system's stability without requiring owner intervention.

### Token Weighting and Distribution

**Function:** `setParameters(address[] loanTokens, uint32[] weightsE4, uint256 minCollRatio, uint256 maxCollRatio)`

Only owner can configure the strategy.

**Parameters:**
- `loanTokens`: Array of loan token addresses to manage
- `weightsE4`: Target weights in basis points (e4 format)
  - Example: [6000, 4000] = 60% first token, 40% second token
  - Sum is automatically normalized to 10000 (100%)
- `minCollRatio` / `maxCollRatio`: Collateral ratio bounds

**Weighting Logic:**

Weights are applied to the **loan side**:
- 60% weight for bSPY means: 60% of loan value should be in bSPY
- Corresponding liquidity is provided in bSPY/bUSD pool
- If poolprice and oracle price differ significantly, the actual distribution will vary to maintain overall ratio

### Price Mechanisms

The contract uses two price sources:

1. **Oracle Price:** From Pyth Network via AssetSystem (for true value calculation)
2. **Pool Price:** From Aerodrome DEX reserves (for swap calculations)

**Pool Price Calculation:**
```
PoolPrice = reserveBUSD / reserveToken
```

**Weighted Price Factor:**

During distribution adjustment, a weighted factor is calculated that accounts for the difference between oracle and pool prices. This is the best estimate for how much bUSD is needed per loan token when adjusting positions.

```
weightedPriceFactor = Σ(usedTokenWeight * oraclePrice / poolPrice)
```

Since the usedTokenWeight changes based on the prices, this factor is refined in two iterations for precise collateral calculations.

## Cleanup Mechanism

**Function:** `cleanup(LoanToken[] memory usedTokens)`

Owner only. Cleans up positions for specified tokens (or currently defined tokens if empty). This is used when dust is accumulating in the contract.

**Process:**
1. **Remove liquidity:** For tokens with weight = 0
2. **Clean legacy loans:**
   - Check outstanding loans for unwanted tokens
   - Swap bUSD for missing tokens (if needed)
3. **Pay back loans:** With available tokens
4. **Swap excess:** Remaining tokens → bUSD
5. **Replenish collateral:** All bUSD back to forge

**Use Cases:**
- When removing tokens from strategy
- When reweighting the distribution
- Manual rebalancing by owner

## Security Mechanisms

### ReentrancyGuard
All main functions (`deposit`, `withdraw`, `cleanup`) are protected against reentrancy attacks.

### SafeERC20
All token transfers use OpenZeppelin's SafeERC20 for safe operations.

### Access Control
- **Owner:** Can set parameters and perform cleanup
- **Users:** Can only deposit and withdraw
- **Last withdrawal:** Only by owner (protects other liquidity providers)

## Gas Optimizations

1. **Batch Operations:** Multiple token operations in one transaction
2. **Weight Normalization:** Occurs once during `setParameters`
3. **Iterative Calculation:** Two-pass approach for price factor reduces rounding errors
4. **Minimal Swaps:** Only when necessary and above minimum amount

## Mathematical Foundations

### Collateral Ratio
```
Ratio = (CollateralValue * 1e18) / LoanValue
```

### LP Token Value
```
lpTokenValue = totalValue / lpTokenSupply
```

### Target Distribution
```
TargetLoanValue = (LoanValue * BASIS_POINTS) / tokenWeightE4
```

### Delta Calculation
For ratio adjustment:
```
ΔColl = |absCollDelta * PRECISION| / (PRECISION + targetRatio * weightedPriceFactor)
```

## Events

### Deposited
```solidity
event Deposited(address indexed user, uint256 amount, uint256 lpTokens);
```
Emitted on every successful deposit.

### Withdrawn
```solidity
event Withdrawn(address indexed user, uint256 amount, uint256 lpTokens);
```
Emitted on every successful withdrawal.

### ParametersSet
```solidity
event ParametersSet(address[] loanTokens, uint32[] weights, uint256 minCollRatio, uint256 maxCollRatio);
```
Emitted when setting new strategy parameters.

### DistributionUpdated
```solidity
event DistributionUpdated(uint256 collValue, uint256 loanValue, uint256 ratio);
```
Emitted after distribution update.

## Workflow Examples

### Example 1: First Deposit

```
1. User has 10,000 bUSD
2. Calls deposit(10000e18)
3. Receives 10,000 lpBUSD
4. Contract:
   - Adds 10,000 bUSD as collateral
   - Takes loans (e.g., 5,000 bSPY, 3,333 bMSTR)
   - Adds liquidity to DEX pools
   - Keeps rest as collateral in forge
```

### Example 2: Second Deposit

```
1. Current total value: 12,000 bUSD (from commission)
2. Current lpBUSD supply: 10,000
3. User deposits 6,000 bUSD
4. Receives: (6,000 * 10,000) / 12,000 = 5,000 lpBUSD
5. New supply: 15,000 lpBUSD
6. New total value: 18,000 bUSD
```

### Example 3: Withdrawal

```
1. User has 5,000 lpBUSD
2. Total value: 18,000 bUSD, Supply: 15,000 lpBUSD
3. Share: (5,000 * 18,000) / 15,000 = 6,000 bUSD
4. After 1% fee: 5,940 bUSD
5. Contract:
   - Removes liquidity from pools
   - Pays back loans
   - Withdraws bUSD from forge
   - Transfers 5,940 bUSD to user
```

## Best Practices for Users

### Deposits
✅ **Recommended:**
- Deposit larger amounts (reduces relative gas costs)
- During times of low network congestion
- After checking current total value

❌ **Avoid:**
- Very small amounts (high gas relative to deposit)
- During extreme market volatility

### Withdrawals
✅ **Recommended:**
- Only withdraw when truly needed (1% fee!)
- Check total value before withdrawal
- Consider gas costs

❌ **Avoid:**
- Frequent small withdrawals
- Withdrawal as last user (unless owner)

### Monitoring
📊 **To Monitor:**
- `getTotalValue()`: Current total value
- LP token balance: Your share
- Events: Deposited/Withdrawn for transparency
- Forge's collateral ratio

## Risks and Notices

### Market Risks
- **Price Volatility:** Loan tokens can fluctuate in value
- **Impermanent Loss:** From providing liquidity in DEX pools
- **Oracle Deviations:** Differences between oracle and pool prices

### Smart Contract Risks
- **Dependencies:** AssetSystem, Aerodrome Router, Pyth Oracle
- **Complexity:** Automatic rebalancing logic
- **Liquidation Risk:** During strong market movements

### Operational Risks
- **Owner Control:** Owner can change parameters
- **Last Withdrawal:** Only owner can withdraw all funds

### Fees
- **1% Withdrawal Fee:** Fixed on every withdrawal
- **DEX Fees:** Aerodrome swap/liquidity fees
- **AssetSystem Fees:** Loan interest and system fees

## Technical Details

### Token Standard
- **ERC20:** lpBUSD is fully ERC20 compliant
- **Name:** "Liquidity Provider Token"
- **Symbol:** "lpBUSD"
- **Decimals:** 18 (OpenZeppelin ERC20 standard)

### Forge Management
The contract creates a forge in AssetSystem during deployment. 
This forge is used for all collateral and loan operations.

### Pool Integration
- **DEX:** Aerodrome (volatile pools)
- **Pairs:** Loan-Token / bUSD
- **Factory:** Standard Aerodrome Factory
- **LP Token:** ERC20 from Aerodrome pools

## FAQ

**Q: What happens to my LP tokens when the strategy changes?**
A: LP tokens remain valid. The contract adjusts the distribution internally, your share of total value remains the same.

**Q: Why a 1% withdrawal fee?**
A: Covers slippage, DEX fees, and transaction costs when unwinding positions. Also protects remaining liquidity providers.

**Q: Can I transfer my LP tokens?**
A: Yes, lpBUSD is a standard ERC20 token and can be transferred.

**Q: What happens during a bank run?**
A: Last withdrawal is only possible by owner. This protects against unfair distribution of remaining assets.

**Q: How often is rebalancing performed?**
A: Automatically on every deposit/withdraw. Anyone can also manually call `updateDistribution()`.

## Summary

The LiquidityProvider is an automated liquidity manager that:
- ✅ Converts user deposits into diversified bAsset liquidity
- ✅ Automatic rebalancing within defined collateral ratios
- ✅ ERC20 LP tokens for proportional share management
- ✅ Integration with AssetSystem forge and Aerodrome DEX
- ✅ Multiple security mechanisms and safeguards
- ✅ The strategy is delta neutral and adapts to market conditions

**Ideal for:** Users who want to passively provide liquidity and participate in the bAsset ecosystem's returns without worrying about manual management.
