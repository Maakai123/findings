
```solidity
function liquidate(IERC20 collateralToken, address position) external override {
    // @audit collateralToken is never validated, could be empty object corresponding
    // to address(0) or a different address not linked to position's collateral
    (uint256 price,) = priceFeeds[collateralToken].fetchPrice();
    // @audit with empty/non-existent collateral, the value of the collateral will be 0
    // with another address, the value will be whatever that value is, not the value
    // of the Borrower's actual collateral. This allows Borrower to be Liquidated
    // before they are in default, since the value of Borrower's actual collateral is
    // never calculated.
    uint256 entirePositionCollateral = raftCollateralTokens[collateralToken].token.balanceOf(position);
    uint256 entirePositionDebt = raftDebtToken.balanceOf(position);
    uint256 icr = MathUtils._computeCR(entirePositionCollateral, entirePositionDebt, price);
}
```

## Explanation 
Imagine you use $500k house as collateral for a $400k bank loan, somebody tells the bank that your collateral was only $5k or give them a wrong property docs thats states your proper is worth 5k and not 500k dollars. then the bank will just sieze her property (liquidate her collateral) even though the house is worth more than the 5k.

## How it works 
-The function liquidate checks if a borrower's position is undercollaterized
-If the value of the collateral is too low compared to the debt, if true liquidate the position.

## Technical Flaw Breakdown

# 1.Missing Validation Check:
The code does not verify if the collateralToken input matches the borrower's actual collateral token.
eg=> A borrower deposits  ETH (value: $10k), but attacker feeds Usdc (value:$0) as collateral

# 2.Faulty Collateral Valuation:
 ``` solidity 
 uint256 entirePositionCollateral = raftCollateralTokens[collateralToken].token.balanceOf(position);
```
if `collateralToken` is invalid, this returns $0 or incorrect value

# 3. Debt Ratio Miscalculation:
Debt Ratio = (Collateral value / Debt)
=> Real collateral: $10k  (ETH)
=> Debt: $8k
# correct => 10k/8k = 125% 
# Attacker's Ratio: 0/8k = 0% => Triggers Liquidation

```diff
Total deposits: $100M
Average loan: $50k (collateralized at 125%)

Attack Scenario:

Attacker loans $40k using $50k ETH collateral.
Later, ETH drops to $45k (still safe at 112.5% ratio).
Attacker calls liquidate() with fake token (value $0):
System calculates ratio: $0 / $40k = 0%
Liquidates borrower’s $45k ETH unfairly.
Loss: Borrower loses $45k ETH despite being solvent. Attackers profit by buying ETH at a discount during liquidation.

```
##  Heuristic Questions for Auditing Similar Protocols

# Collateral Validation:
"Does the liquidation function verify that the provided collateral token matches the borrower's actual collateral?"

# Price Feed Security:
"How does the system ensure the price feed corresponds to the borrower's collateral token and not an arbitrary input?"

# Balance Checks:
"Are balance/ownership checks performed directly on the borrower's collateral contract, or can they be overridden by input parameters?"

# Liquidation Thresholds:
"Is there a secondary check (e.g., on-chain records) to validate collateral adequacy before executing liquidation?"
Input Sanitization:
"Are all user-provided inputs (like collateralToken) restricted to a pre-approved list of valid assets?"

