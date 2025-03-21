# Contract reference

## ForgeToken

The ForgeToken is an ERC721 token representing the ownership of a forge. It provides functions to get information about the forge. The id of the forgeToken is always equal to the id of the represented forge.

### Views

#### `getForgeScheme`

parameters:
* uint256 tokenId : the ID of the forge token

returns the scheme ID of the forge.

#### `getForgeValues`

parameters:
* uint256 tokenId : the ID of the forge token

returns the collateral value, loan value, and collateral ratio of the forge.

#### `getForgeCollateral`

parameters:
* uint256 tokenId : the ID of the forge token
* bool withValue : whether to include the value of the collateral

returns an array of CollData representing the collateral in the forge. If `withValue` is true, it includes the $ values based on the oracle price. This fails if a valid oracle price is missing.

#### `getForgeLoan`

parameters:
* uint256 tokenId : the ID of the forge token
* bool withValue : whether to include the value of the loan

returns an array of LoanData representing the loans in the forge.
 If `withValue` is true, it includes the $ values based on the oracle price. This fails if a valid oracle price is missing.

## AssetSystem

The AssetSystem is the heart of basedAssets. It controls the forges and handles asset-splits and liquidations.

The following represent the relevant functions for users to interact with the system.

### Forge Interaction

#### `createForge`

parameters:
* uint256 loanScheme : 0 for USD forge, 1 for asset forge

creates a new forge with the defined scheme. It mints a new ForgeToken NFT representing the ownership of this forge.

#### `addCollateral`

parameters:
* uint256 forgeId : id of the forge
* ERC20 token : collateral token to use (must be allowed in the used scheme)
* uint256 amount : amount to add into the collateral

needs allowance to transfer the given token amount to the AssetSystem. Adding collateral does not need active oracle prices. Anyone can add collateral to any forge.

depositFee applies to the transfered amount.

#### `withdrawCollateral` 

parameters:
* uint256 forgeId : id of the forge
* ERC20 token : collateral token to use
* uint256 amount : amount to withdraw from the collateral

sender must be owner of the forge (in possesion of the corresponding forge NFT). amount of token is removed from the collateral and transfered to the sender. Fails if the removal would send the forge into liquidation. Only possible with active oracle prices.

withdrawalFee applies to the transfered amount.

#### `takeLoan` 

parameters:
* uint256 forgeId : id of the forge
* LoanToken token : loan token to use
* uint256 amount : amount to take as new loan

sender must be owner of the forge. mints new tokens into the owners account. Fails if it would send the forge into liquidation. Only possible with active oracle prices.

takeloanFee increases the effective loan amount.


#### `paybackLoan` 

parameters:
* uint256 forgeId : id of the forge
* LoanToken token : loan token to use
* uint256 amount : amount to use for payback

reduces the loan amount in the forge. Any user can pay back any loan in any forge. Does not need active oracle prices. need allowance to move the funds.
accumulated interest is always paid back first (send to treasury), remaining funds are burned reducing the principal. If amount is bigger than open loan + interest, only the necessary funds are used.

paybackFee applies to the specified amount

### Forge Liquidation

#### `liquidateForge`

parameters:
* uint256 forgeId : id of the forge
* LoanToken token : loan token to use
* uint256 amount : amount of the loan to liquidate
* IERC20Metadata preferredCollateral : preferred collateral token to receive

initiates the liquidation of a forge. The sender must specify the amount of the loan to liquidate and the preferred collateral token to receive. The liquidation process will transfer the corresponding collateral plus incentive to the liquidator and the treasury based on the liquidation ratio.

#### `liquidateForgeByTreasury`

parameters:
* uint256 forgeId : id of the forge
* LoanToken token : loan token to use
* uint256 amount : amount of the loan to liquidate
* IERC20Metadata preferredCollateral : preferred collateral token to receive

initiates the liquidation of a forge by the treasury. The sender must specify the amount of the loan to liquidate and the preferred collateral token to receive. The liquidation process will transfer the collateral to the treasury and the reward to the liquidator based on the liquidation ratio.

### Asset Split

#### `splitToken`

parameters:
* LoanToken oldToken : the original loan token to be split
* uint256 amount : the amount of the original token to be split

splits the specified amount of the original token into the new token. The original tokens are burned and the new tokens are minted into the sender's account.

#### `splitTokenInForge`

parameters:
* uint256 forgeId : id of the forge
* LoanToken token : the original loan token to be split

splits the loan amount of the specified token in the forge into the new token. The original loan amount is converted to the new token based on the split ratio. The new loan amount and interest are updated in the forge.
If forge is healthy, this can only be triggered by the owner. If forge can be liquidated, anyone can trigger the conversion.

## SystemHelper

The SystemHelper contract provides various helper functions for the AssetSystem, including token splits, scheme access, and liquidation checks.

### Scheme Access

#### `getCollateralsInScheme`

parameters:
* uint256 schemeId : the ID of the loan scheme

returns an array of IERC20Metadata representing the collateral tokens allowed in the specified loan scheme.

#### `getLoansInScheme`

parameters:
* uint256 schemeId : the ID of the loan scheme

returns an array of LoanToken representing the loan tokens allowed in the specified loan scheme.

### Forge Helpers

#### `forgeHasActivePrices`

parameters:
* uint256 forgeId : the ID of the forge

returns true if the forge has active oracle prices for all collateral and loan tokens.

#### `getForgesInLiquidation`

parameters:
* address collateralFilter : the collateral token filter
* address loanFilter : the loan token filter

returns an array of ForgeData representing the forges that are in liquidation based on the specified filters.

### overall system information

#### `algoRatio`

parameters:
* LoanToken token : the loan token

returns the algorithmic ratio of the specified loan token.

#### `projectedAlgoRatio`

parameters:
* LoanToken token : the loan token

returns the projected algorithmic ratio of the specified loan token. The projection includes currently locked in conversions in the Asset Stability.

#### `calcUSDInterestRate`

returns the current USD interest rate based on the average price of the USD token in the stability pool.

## AssetStability

The AssetStability contract handles the stability of the bUSD token and allows for scheduled conversions between bUSD and bAssets.

### USD Stability

#### `mintUSDFromStab`

parameters:
* uint256 amountOfStabilityToken : the amount of stability tokens to use for minting bUSD

mints new bUSD tokens from the specified amount of stability tokens. The stability tokens are transferred to the contract and the corresponding amount of bUSD tokens is minted to the sender's account. A fee is applied to the stability tokens.

#### `burnUSDForStab`

parameters:
* uint256 usdAmount : the amount of bUSD tokens to burn for stability tokens

burns the specified amount of bUSD tokens and returns the corresponding amount of stability tokens to the sender's account. A fee is applied to the bUSD tokens.

### Scheduled bAsset Conversions

#### `scheduleConversion`

parameters:
* LoanToken input : the input token for the conversion
* uint256 amount : the amount of the input token to convert
* LoanToken output : the output token for the conversion

schedules a conversion of the specified amount of the input token to the output token. The conversion will be executed in the next period. A fee is applied to the conversion.

#### `claim`

parameters:
* uint256 period : the period in which the conversion was scheduled
* uint256 scId : the ID of the scheduled conversion

claims the output tokens of a scheduled conversion. The conversion must have been scheduled in a previous period and the prices must be locked in.

#### `refund`

parameters:
* uint256 period : the period in which the conversion was scheduled
* uint256 scId : the ID of the scheduled conversion

refunds the input tokens of a scheduled conversion. The conversion must have not yet been claimed.

#### `lockPrices`

parameters:
* uint256 offset : the offset to start locking prices from
* uint256 limit : the maximum number of prices to lock

locks the prices of the tokens for the previous period. This function can be called multiple times with different offsets and limits to lock all prices.

#### `lockPrice`

parameters:
* LoanToken token : the token to lock the price for

locks the price of the specified token for the previous period. This function can be used to lock the price of a single token if it failed to lock before.