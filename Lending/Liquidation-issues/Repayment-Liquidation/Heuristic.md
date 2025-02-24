# Function Grouping:

What to Look For:
Identify functions related to handling token interactions, such as repayment and liquidation.
Example:
Group together the repay() and liquidate() functions since both use token logic.


# Modifier & Permission Consistency:

What to Look For:
Check if both functions apply the same token approval logic.
Red Flag:
If repay() uses a modifier like onlyWhitelistedToken() but liquidate() does not, note the inconsistency.
Implication:
This can lead to scenarios where a previously approved token is disallowed for repayment but still allows liquidation.


# Existing Loan Exception Check:

What to Look For:
Verify that token disallowance only applies to new loans.
Check:
Look for logic that exempts existing positions from token whitelist checks.
Red Flag:
If no such exception exists, existing loans might be unfairly blocked from being repaid.