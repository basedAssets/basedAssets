![](header.png)
# Decentralized RWA System on Base: Technical Whitepaper

## Abstract
This document presents a decentralized lending system on the Base network, pioneering the fully decentralized creation of Real-World Asset (RWA) loan tokens through a dual-forge architecture anchored by an overcollateralized stablecoin, bUSD. By integrating dynamic interest rates, an innovative Asset-Stability mechanism, and robust algorithmic controls, the system ensures stability while enabling users to mint bUSD and RWA tokens against diverse crypto collaterals. Designed for flexibility and resilience, it offers a sustainable framework for decentralized finance, balancing user accessibility with sophisticated risk management.

## 1. System Overview
The system consists of two primary components:
1. A decentralized stablecoin (bUSD) backed by crypto assets
2. Loan tokens (RWA) representing decentralized lending positions with oracle price references

### 1.1 Key Features
- Dual-forge system with different collateralization requirements
- Dynamic interest rate mechanism
- Asset-Stability feature for price alignment
- Multiple collateral options for both forge types
- Algorithmic bUSD management system
- Treasury-driven fee collection and liquidation mechanisms

![System overview](SystemOverview.png)

### 1.2 User Perspective
This section illustrates how users interact with the system through a practical example.

**Example: Alice’s Journey**
- **Step 1: Minting bUSD**  
  Alice deposits 300 USDC into the bUSD forge (minimum 120% collateralization). She mints 200 bUSD, valued at $1 each, incurring a 0.1% base interest rate that adjusts dynamically based on market conditions (e.g., rising if bUSD trades below $1).
- **Step 2: Creating an RWA Token**  
  Using her 200 bUSD, Alice opens an RWA forge (meeting the 150% minimum). She mints an RWA token representing a $100 loan tied to an oracle-priced asset, with a 1% default interest rate.
- **Step 3: Use RWA Token**  
  Alice can now sell the RWA token and bet on falling prices or put the RWA into Liquidity Mining earning commission from the system.
- **Step 4: Liquidation Risk**  
  If Alice’s RWA forge drops below 150% collateralization (e.g., RWA token value increases), a liquidator could repay part of her loan with their funds, taking 10% of her excess collateral as a reward, while the forge retains the rest, improving its health.

**Example: Bob's Journey**
- **Step 1: Buying RWA token**  
  Bob uses the RWA pools on the DEX (provided by users like Alice) to buy RWA tokens. He might do this to speculate on short term profits, or to diversify his portfolio into different asset classes.
- **Step 2: Liquidity Mining**  
  Bob can now also put his RWA tokens into Liquidity Mining to earn cash flow on top of his portfolio value
- **Step 3: Price stability**  
  Asset Stability ensures that the RWA-bUSD price is within the defined range (5% for medium volatile tokens) at least once a week. So Bob can be sure that his decentralized RWA portfolio keeps track with the real world prices.

These examples show how users can leverage the system for lending and stability while managing risks.

## 2. bUSD Stablecoin

### 2.1 Creation Methods
bUSD can be created through two distinct mechanisms:

1. **Forge Minting**
   - Supported collaterals: USDC, cbBTC, wETH
   - Minimum collateralization ratio: 120%
   - Users can mint bUSD against deposited collateral
   - bUSD is always valued at $1 in the forge

2. **Stability Module**
   - Direct USDC-to-bUSD conversion
   - 5% conversion fee (decreasing over time)
   - Bidirectional conversion (mint/burn)
   - Burn operation available when USDC exists in stability pool

### 2.2 Interest Rate Mechanism
- Base interest rate: 0.1%
- Dynamic interest rate component:
  - Negative when bUSD trades at premium (up to -200% at 20% premium)
  - Positive when bUSD trades at discount (up to 5000% at 100% discount)
  - Based on bUSD/USDC DEX pool price

### 2.3 Algorithmic bUSD
Algorithmic bUSD represents the portion of bUSD supply not backed by collateral but created through mechanisms like negative interest rates or token-to-bUSD conversions. It’s calculated as:

```
BackedValue = TotalUSDLoans + USDValueInStability
AlgorithmicUSD = TotalUSDSupply - BackedValue
AlgoRatio = AlgorithmicUSD / TotalUSDSupply
```
To manage high AlgoRatios, a dynamic fee applies to actions increasing it (e.g., loan paybacks, stability burns, LoanToken-to-bUSD conversions). These fees are burned to reduce AlgoRatio. 

The dynamic fee is calculated as:

`dynFee= ((algoRatio - 0.1)^2)/4`

See the table below for fee scaling in the Asset-Stability mechanism:

**Dynamic Fee Multiplier Table**  
| AlgoRatio Range | Dynamic Fee | AssetStability Fee Multiplier       | bUSD Generation Status |
|-----------------|-------------|-------------------------------------|-------------------------|
| 0-10%           | 0% | 1.0x (base fee)     | Enabled                 |
| 10-50%          | 0% - 4% | 1.0x to 2.0x (linear increase) | Enabled       |
| 50%-100%        | 4% - 20% | 2x                 | Disabled                |

*Example*: At 30% AlgoRatio, the Asset stability multiplier is 1.5x, scaling fees proportionally. 

## 3. RWA System

### 3.1 RWA Forge Specifications
- Supported collaterals: bUSD, cbBTC, wETH
- Minimum collateralization ratio: 150%
- Valuation mechanism:
  - bUSD valued at $1
  - Other assets valued at oracle price in $

### 3.2 RWA Token Interest
- Default interest rate: 1%
- Flexible rate structure allowing future adjustments based on:
  - Market demand
  - Price dynamics

## 4. Asset-Stability Mechanism

### 4.1 Operation
- Conversion once per week
- Bidirectional conversion between loan tokens and bUSD
- Pre-lock requirement before conversion block
- Post-conversion claim process
- May generate algorithmic bUSD
- Operation limitations:
  - Completely disabled for bUSD generation when AlgoRatio > 50%
  - Fee multiplier applied between 10-50% AlgoRatio

### 4.2 Fee Structure
The Asset-Stability mechanism incurs fees based on asset volatility and AlgoRatio. Base fees reflect asset risk, while dynamic multipliers adjust for system stability, as shown below:

**Base Conversion Fee Table**  
| Asset Volatility | Base Fee |
|-------------------|----------|
| Low              | 2%       |
| Medium           | 5%       |
| High             | 15%      |

**Dynamic Fee Multiplier**  
- 0-10% AlgoRatio: 1.0x (base fee)
- 10-50% AlgoRatio: Linear increase from 1.0x to 2.0x (e.g., 1.5x at 30%)  
- Above 50% AlgoRatio: bUSD generation disabled  

*Example*: Converting a high-volatility RWA token (15% base fee) at 30% AlgoRatio incurs a 1.5x multiplier, resulting in a 22.5% total fee.

### 4.3 Volume Limitations
- Maximum conversion volume per period: 10% of total bUSD supply
- Applies to net bUSD delta in each stability period

## 5. Treasury System

### 5.1 Treasury Inflow Streams
- All system fees directed to treasury (bUSD- and asset stability fees)
- Exception: Dynamic bUSD fees are burned directly
- Interest payments from both forge types
- 10% of overcollateral from liquidations with user funds
- Collateral + 15% of overcollateral from liquidations via treasury funds

### 5.2 Treasury Operations
- Can initiate liquidations
- Can incentivize liquidity mining for LoanToken-bUSD pools via treasury funds
- Might burn LoanTokens to reduce AlgoRatio

## 6. Liquidation Mechanism
When a forge falls below minimum collateral ratio, anyone can trigger a (partial) liquidation, either with their own funds or via treasury funds. Both methods pay part of the overcollateralization as an incentive/fee, with the remainder staying in the forge to increase its collateral ratio, improving forge health with each partial liquidation.

### 6.1 User-Funded Liquidations
1. Liquidator pays back portion/full loan amount
2. Liquidator receives corresponding collateral plus 10% of overcollateral as liquidation incentive
3. 10% of overcollateral goes to treasury
4. remaining overcollateral stays in the forge, improving forge health

![User-Funded Liquidation](LiquidationUser.png)

#### Example: User-Funded Liquidation of a bUSD-Forge

bUSD-Forge A has a minimum collateral ratio of 120%. It currently holds cbBTC collateral worth $119 (based on the oracle price of cbBTC) and a 100 bUSD loan (including accrued interest). This gives a collateral ratio of 119% (119 / 100), meaning the forge is undercollateralized and eligible for liquidation.

Bob decides to partially liquidate the forge using 10 of his own bUSD, which is 10% of the total loan. At a collateral ratio of 119%, the 10 bUSD (valued at $10 in the forge) corresponds to $11.9 worth of collateral: $10 to back the repaid loan portion and $1.9 as overcollateral. The liquidation proceeds as follows:

- *Bob’s Actions and Rewards*:
  - Bob pays 10 bUSD to the forge.
  - The 10 bUSD loan is repaid (accrued interest moves to the treasury, remaining principal is burned).
  - Bob receives cbBTC worth $10 as the collateral backing the repaid loan.
  - Bob also receives 10% of the overcollateral ($1.9 × 10% = $0.19) as a liquidation incentive.
- *Treasury’s Share*:
  - The treasury receives 10% of the overcollateral ($1.9 × 10% = $0.19) as system fee.
- *Forge’s New State*:
  - Loan: 90 bUSD remaining (100 - 10).
  - Collateral: $119 - $10.38 = $118,62 worth of cbBTC.
  - New collateral ratio: $118,62 / 90 = 131,8% (back in strong overcollateralization).

***Summary of Transfers***:
- *Bob*: Pays 10 bUSD, receives cbBTC worth $10.19 ($10 + $0.19), netting a 1.9% profit on his funds.
- *Treasury*: Gains cbBTC worth $0.19.
- *Forge Owner*: Loses $0.38 of overcollateral ($0.19 to Bob + $0.19 to treasury), or 0.32% of their original $119 collateral.

Bob’s 1.9% profit incentivizes liquidators to maintain system health, while the forge owner loses only a small portion of collateral to restore stability. This partial liquidation (10% of the loan) nudges the forge back into a healthy state.



### 6.2 Treasury-Funded Liquidations
1. Liquidator triggers the liquidation
2. Treasury funds are used to pay back portion/full loan amount
3. Treasury receives corresponding collateral plus 15% of overcollateral
4. Liquidator receives 5% of overcollateral as compensation for triggering the liquidation
5. remaining overcollateral stays in the forge, improving forge health

![Treasury-Funded Liquidation](LiquidationTreasury.png)

#### Example: Treasury-Funded Liquidation of an RWA-Forge

Assume a SPY price in the oracle of $1000. RWA-Forge B has a minimum collateral ratio of 150%. It currently holds 145 bUSD collateral (worth $145 in the forge) and a 0,1 bSPY loan  (valued at $100 via oracle price, including accrued interest). This gives a collateral ratio of 145% (145 / 100), meaning the forge is undercollateralized and eligible for liquidation.

Alice triggers a treasury-funded liquidation for 20% of the forge’s loan, using the treasury’s funds. The treasury uses 0,02 bSPY to repay 20% of the RWA loan. At a collateral ratio of 145%, the 0.02 bSPY (valued at $20 in the forge) corresponds to $29 worth of bUSD collateral: 20 to back the repaid loan portion and 9 as overcollateral. The liquidation proceeds as follows:

- *Treasury’s Actions and Rewards*:
  - The treasury uses 0.02 bSPY to reduce the loan in the forge (accrued interest is retained by the treasury, and the principal is burned).
  - The treasury receives 20 bUSD collateral as the backing for the repaid loan.
  - The treasury also receives 15% of the overcollateral (9 × 15% = 1.35 bUSD).
- *Alice’s Reward*:
  - Alice, as the liquidator who triggered the process, receives 5% of the overcollateral (9 × 5% = 0.45 bUSD) as a reward.
- *Forge’s New State*:
  - Loan: 0.08 bSPY remaining worth $80.
  - Collateral: 145 - (20+1.35+0.45) = 123.2 bUSD valued $123.2
  - New collateral ratio: $123.2 / 80 = 154% (back in overcollateralization).

***Summary of Transfers***:
- *Alice*: Receives 0.45 bUSD as a reward for triggering the liquidation.
- *Treasury*: Pays up to 0.02 bSPY worth $20, receives 21.35 bUSD, netting at least a $1.35 gain.
- *Forge Owner*: Loses $1.80 of overcollateral ($1.35 to treasury + $0.45 to Alice), or 1.24% of their original $145 collateral.

Alice earns a small reward for her role in maintaining system health, while the treasury benefits from the liquidation by acquiring collateral at a favorable rate. The forge owner loses a modest portion of their overcollateral to restore stability, and the forge’s health is incrementally improved.

Note that the accrued interest on the loan is reduced from the forge balance, but stays in the treasury. This is because the treasury receives all paid interests, so if the treasury pays back a loan, the interest is kept there. 

## 7. Risk Management

### 7.1 Collateral Risk
- Multiple high-quality collateral options
- Conservative collateralization ratios
- Oracle-based price feeds for valuation
- Liquidation mechanisms for undercollateralized positions

### 7.2 Stability Risks
- Volume limitations on Asset-Stability
- Fee structure discouraging excessive arbitrage
- Dynamic interest rates for market alignment
- Algorithmic bUSD monitoring and fee system

## 8. Stock Split Handling
RWAs tied to equities require split handling. The system is prepared to handle both forward and reverse stock splits.

### 8.1 Split Initialization Process
1. **Oracle Deactivation**
   - Oracle for affected token is deactivated
   - Prevents creation of new loans
   - Existing loans and paybacks remain functional
2. **Split Registration**
   - Stock split info recorded in the asset system
   - Includes split ratio and new token specs
   - Automatically:
     - Creates new LoanToken with same interest as old token
     - Deactivates old token for new loans
     - Deactivates asset-stability for old token
     - Activates asset-stability for new token with identical fee structure
3. **Oracle Transition**
   - After new price data is available:
     - Old token oracle receives multiplier to match new pricing and is reactivated
     - New token oracle is activated
     - System maintains price consistency across transition

### 8.2 Token Conversion Process
1. **User Token Conversion**
   - Users convert old tokens to new via the asset system
   - Conversion ratio based on split definition
   - Old tokens burned; new tokens minted per split ratio
   - No fee on split conversion
2. **Forge Loan Conversion**
   - Forge owners convert loans to new token denomination
   - Whole loan converted at once
   - Maintains loan-to-collateral ratios and forge health
   - Optional but recommended for active forges
3. **Liquidation Protection**
   - Anyone can trigger split conversion for liquidatable forges
   - Ensures liquidation remains possible if old tokens become unavailable
   - Preserves system security and stability during transitions

### 8.3 Split Impact Management
1. **System Parameters**
   - Collateralization ratios unchanged
   - Interest rates carry over to new tokens
   - Asset-stability fees maintain consistency
2. **Risk Management**
   - Continuous oracle price monitoring during transition
   - Automatic validation of conversion transactions
   - Protection against split-related arbitrage

## 9. Conclusion
This system provides a framework for decentralized RWAs based on loan tokens while maintaining stability through multiple mechanisms. The dual-forge architecture, combined with stability modules, dynamic interest rates, and sophisticated liquidation mechanisms, creates a sustainable ecosystem for both stablecoin and loan token operations. The treasury system and algorithmic bUSD management provide additional layers of security and sustainability.

## 10. Benefits for the Ecosystem

The decentralized Asset system introduces a transformative approach to Real-World Asset (RWA) tokenization and lending, delivering tangible advantages to users, protocols, and the broader DeFi ecosystem. By leveraging a dual-forge architecture, dynamic pricing mechanisms, and user-driven minting, the system creates opportunities for participation, innovation, and financial empowerment. Below are the key benefits:

### 10.1 Universal Access to Rising Asset Values
- **Decentralized and Uncensorable**: Anyone with an internet connection can participate, free from intermediaries or geographic restrictions. The system operates 24/7 on the Base network, ensuring constant availability without downtime or censorship.
- **Profit from Growth**: Users can mint, trade, or hold RWA tokens tied to real-world assets, benefiting from potential price increases in a transparent, decentralized marketplace.

### 10.2 Flexible Trading Opportunities
- **Long and Short Positions**: Users can speculate on RWA price movements by going long (buying tokens to profit from rising prices) or shorting (selling tokens to capitalize on declines). This flexibility empowers users to adapt to market conditions, whether bullish or bearish.
- **Portfolio Diversification**: With RWAs representing various asset classes, users can build diversified portfolios, reducing risk and tapping into multiple markets—all within a single decentralized system.

### 10.3 Loose Oracle Coupling for Regulatory Flexibility
- **Innovative Price Connection**: RWA tokens are loosely tied to oracle prices without a fixed peg, offering a unique balance of real-world relevance and regulatory independence. This makes them ideal for protocols needing stock or asset price exposure without directly referencing regulated markets.
- **Cross-Protocol Utility**: Other DeFi platforms can integrate RWAs as collateral, price feeds, or yield-bearing assets, fostering interoperability while sidestepping compliance hurdles tied to traditional financial instruments.

### 10.4 Be Your Own Broker with Real Yield
- **Self-Directed Finance**: Users can mint RWAs via forges and manage their own lending positions, effectively acting as their own brokers. This eliminates reliance on centralized institutions and puts control in the hands of individuals.
- **Earn Passive Income**: By staking RWA tokens in Liquidity Mining pools, users earn commissions from system fees and trading activity. This provides a real, sustainable yield on diversified basedAsset portfolios, rewarding active participation.

### 10.5 Enhanced Stability and Resilience
- **Dynamic Stability Mechanisms**: The Asset-Stability mechanism and dynamic interest rates ensure RWA prices align with market trends over time, while bUSD remains a reliable anchor. This stability protects users and supports long-term ecosystem health.
- **Robust Risk Management**: Overcollateralization, liquidation incentives, and treasury operations safeguard the system against volatility, ensuring it remains solvent and trustworthy even in turbulent markets.

### 10.6 Empowerment Through Ownership
- **User-Driven Minting**: The forge system democratizes asset creation—anyone can mint RWAs or bUSD with supported collateral, lowering barriers to entry and encouraging widespread adoption.
- **Community Governance Potential**: While not yet implemented, the treasury’s role in fee allocation and liquidity incentives lays the groundwork for future community-driven decision-making, aligning the system with DeFi’s ethos of decentralization.

### 10.7 Economic Efficiency and Innovation
- **Low-Cost Operations**: By running on Base, the system minimizes transaction costs, making it affordable for users to mint, trade, and manage RWAs. This efficiency drives higher participation and liquidity.
- **Catalyst for New Use Cases**: The availability of decentralized RWAs unlocks opportunities for developers to build novel applications—such as synthetic derivatives, insurance products, or tokenized real estate—expanding the boundaries of DeFi.

### 10.8 A Sustainable Financial Future
- **Scalable Framework**: The dual-forge design and algorithmic bUSD management provide a scalable foundation that can accommodate growing demand and new asset types without compromising stability.
- **Incentivized Participation**: Liquidators, liquidity providers, and forge operators all benefit from clear incentives (e.g., liquidation rewards, mining yields), creating a self-sustaining ecosystem where every role contributes to overall success.

By combining accessibility, flexibility, and resilience, this system redefines how users interact with real-world assets in a decentralized context. It not only empowers individuals to take charge of their financial strategies but also strengthens the DeFi landscape by offering a versatile, stable, and innovative platform for asset tokenization and lending.