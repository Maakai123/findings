
```solidity 
// File: gmx-synthetics/contracts/position/DecreasePositionCollateralUtils.sol : DecreasePositionCollateralUtils.getLiquidationValues()   #1

346            } else {
347 @>             values.pnlAmountForPool = (params.position.collateralAmount() - fees.funding.fundingFeeAmount).toInt256();
348            }
349    
350            PositionPricingUtils.PositionFees memory _fees;
351    
352            PositionUtils.DecreasePositionCollateralValues memory _values = PositionUtils.DecreasePositionCollateralValues(
353                values.pnlTokenForPool,
354 @>             values.executionPrice, // executionPrice
355                0, // remainingCollateralAmount
356                values.positionPnlUsd, // positionPnlUsd
357                values.pnlAmountForPool, // pnlAmountForPool
358                0, // pnlAmountForUser
359                values.sizeDeltaInTokens, // sizeDeltaInTokens
360                values.priceImpactAmount, // priceImpactAmount
361                0, // priceImpactDiffUsd
362                0, // priceImpactDiffAmount
363                PositionUtils.DecreasePositionCollateralValuesOutput(
364:                   address(0),
```


Issue: Liquidation orders update internal records without moving tokens, while closing or adding collateral involves actual token transfers.
Consequence: If a collateral token is paused, users can’t act to protect their positions (because transfers fail), yet liquidation still goes through via internal accounting.

# Users cannot close or add collateral to protect their position (because transfers are blocked).
# Liquidation orders still work because they only update internal numbers and don’t require a token transfer.

```solidity
values.pnlAmountForPool = (params.position.collateralAmount() - fees.funding.fundingFeeAmount).toInt256();
```

# Recommendation:
Require that liquidations perform an actual token transfer (by storing user collateral separately) so that if the token is paused, the liquidation will revert instead of proceeding unchecked.


# The Liquidation Calculation

The function getLiquidationValues computes values used to update a liquidated position. Here’s a key part of the code:

```solidity
if (fees.funding.fundingFeeAmount > params.position.collateralAmount()) {
    values.pnlAmountForPool = 0;
    // Emit an event to signal insufficient collateral to pay fees
    PositionEventUtils.emitInsufficientFundingFeePayment(...);
} else {
    // Key Line: Subtract fees from collateral to determine value for the pool
    values.pnlAmountForPool = (params.position.collateralAmount() - fees.funding.fundingFeeAmount).toInt256();
}
```

Imagine you have $1,000 as collateral and your fee is $100. The system computes $1,000 - $100 = $900, which represents the “net” collateral value used in the liquidation accounting.

Internal Accounting:
The system just changes numbers (like “you now have $900 allocated to the pool”) without actually moving money.

Token Transfers in Other Functions:
By contrast, when you close a position or add collateral, the system must physically move tokens between addresses. If the collateral token is paused (for example, USDC is frozen), then the transfer will fail.

#  Real‑World Analogy and Figures

Imagine you have a bank account with $1,000.

Normal Action (Token Transfer):
When you want to withdraw money, the bank must physically hand you cash. But if the bank’s vault is locked (token paused), you can’t withdraw your money.
Internal Accounting (Liquidation):
However, the bank’s computer system might still update your balance to zero internally—even though you never received your money—because it just changes numbers without handing you cash.

Initial Position:

Collateral: $1,000
Funding Fee: $100
Liquidation Calculation:

Net value for liquidation: $1,000 - $100 = $900
What Happens If Collateral Is Paused:

Close Position / Add Collateral:
These actions try to move tokens (money) and fail because the token is paused.
Liquidation:
The system simply updates its records to say your position is worth $900 and liquidates you, even though you couldn’t act to avoid liquidation.

# Flaw in Liquidation Function:
```solidity 
values.pnlAmountForPool = (params.position.collateralAmount() - fees.funding.fundingFeeAmount).toInt256();
```
The liquidation process uses internal accounting (e.g., the line below) without performing an actual token transfer:

This means liquidation does not depend on whether a token transfer can succeed.

Lack of Token Transfer Check:
There is no step that would cause the liquidation to revert if the token transfer would fail. Thus, even when token transfers are blocked (e.g., collateral token paused), liquidation updates go through.

# Recommendation

The recommendation is to keep user collateral in a separate address from the pool. This way, liquidations would have to perform an actual token transfer:

Why It Matters:
If the transfer fails (e.g., because the token is paused), the liquidation process would also revert. This would align the behavior of liquidations with other functions (like closing positions) and prevent unexpected liquidations.





