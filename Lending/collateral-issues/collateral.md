```solidity
   function liquidate(uint256 positionId, address debtToken, uint256 amountCall)
    external override lock poke(debtToken) {
    // checks
    if (amountCall == 0) revert ZERO_AMOUNT();
    if (!isLiquidatable(positionId)) revert NOT_LIQUIDATABLE(positionId);

    // @audit get position to be re-paid by liquidator, however
    // borrower may have multiple debt positions
    Position storage pos = positions[positionId];
    Bank memory bank = banks[pos.underlyingToken];
    if (pos.collToken == address(0)) revert BAD_COLLATERAL(positionId);

    // @audit oldShare & share proportion of the one position being liquidated
    uint256 oldShare = pos.debtShareOf[debtToken];
    (uint256 amountPaid, uint256 share) = repayInternal(
        positionId,
        debtToken,
        amountCall
    );

    // @audit collateral shares to be given to liquidator calculated using
    // share / oldShare which only correspond to the one position being liquidated,
    // not to the total debt of the borrower (which can be in multiple positions)
    uint256 liqSize = (pos.collateralSize * share) / oldShare;
    uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
    uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;

    // @audit if the borrower has multiple debt positions, the liquidator
    // can take the whole collateral by paying off only the lowest value
    // debt position, since the shares are calculcated only from the one
    // position being liquidated, not from the total debt which can be
    // spread out across multiple positions

```




# EXPLANATION

`Imagine you borrow money from a bank, and to secure the loan, you give them your house as collateral. If you fail to repay, the bank can either:`

`Take the house instead of getting repaid.Allow someone else to pay your debt and take the house as a reward`

# The problem in the code is in the second case: A Liquidator can take the entire house by repaying only a small portion of your debt`


# Real-World Example with Numbers

# Scenario 1: Borrower’s Debt is Split Across Multiple Loans

You take 3 loans from a bank:
Loan A: $10,000
Loan B: $5,000
Loan C: $1,000
You put up your house (collateral) worth $20,000 to secure these loans.

# How Liquidation is Supposed to Work
If you fail to pay, someone (a Liquidator) can pay off your debts and receive a proportional part of the house.
If the Liquidator pays off $10,000, they should receive half the house.
If they pay off $5,000, they should get a quarter of the house.

#  How the Bug Works (Unfair Liquidation)

The Liquidator only pays Loan C ($1,000)
The smart contract calculates their share based on that single loan, not the total debt.
# The Liquidator gets ALL the house instead of just a small portion.


# Key Flaw in the Code

```solidity
uint256 oldShare = pos.debtShareOf[debtToken];
(uint256 amountPaid, uint256 share) = repayInternal(
    positionId,
    debtToken,
    amountCall
);
uint256 liqSize = (pos.collateralSize * share) / oldShare;
```
# Problem:

oldShare only considers one debt position, ignoring other debts.
This allows a Liquidator to pay the smallest debt but receive all collateral.


✅ What it should do:
Calculate the collateral share based on ALL of the Borrower’s debt, not just one small loan.

🚨 What it actually does:
Only looks at the single loan being repaid, allowing the Liquidator to take the whole collateral by paying off the smallest loan.

# Recommendation for Fixing the Issue

Modify the calculation to consider the total debt using getDebtValue(positionId)

```solidity
uint256 totalDebt = getDebtValue(positionId);  // Get total debt across all positions
(uint256 amountPaid, uint256 share) = repayInternal(
    positionId,
    debtToken,
    amountCall
);
uint256 liqSize = (pos.collateralSize * amountPaid) / totalDebt;

amountPaid / totalDebt ensures the Liquidator gets only a fair proportion of the collateral.
Prevents a small debt payment from unlocking the entire collateral.

```