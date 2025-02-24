
# Identify Collateral-Related Functions:

What to Look For:
List all functions that call the collateral check (e.g., _checkIfCollateralIsActive) when users try to close a loan or add collateral.

# Context Analysis of the Check:
What to Look For:
Verify if the price/stale or circuit breaker check is needed in every case.
Red Flag:
If a function blocks actions like closing a fully repaid loan or adding extra collateral (which only improves safety) solely because of stale prices or active circuit breakers, that’s overly restrictive.

# Examine the Code Flow:

What to Look For:
Check where and how the collateral check is applied within each function.
Red Flag:
A blanket check in all scenarios without conditionally allowing safe actions.

# Conditional Checks Verification:

What to Look For:
Look for conditional logic that only triggers the collateral check when there is outstanding debt.
Example:
```solidity
if(outstandingDebt > minimalThreshold) {
    _checkIfCollateralIsActive(currencyKey);
}
```
Red Flag:
A missing condition or always-on check indicates over-restriction.

# Contextual Appropriateness:

What to Look For:
Check if the function’s purpose truly requires a live price feed:
For Closing Loans: If a user is closing the loan completely (no remaining debt), the price check should be skipped.
For Adding Collateral: If the user is simply adding collateral (which improves safety), ensure there isn’t an unnecessary liquidation threshold check.