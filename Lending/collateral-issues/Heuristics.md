
# Identify Liquidation Functions:
What to Look For:
Find functions that calculate the collateral share during liquidation (e.g., the liquidate() function).
Why It Matters:
These functions determine how much collateral a liquidator receives when they repay a debt.

# Examine the Collateral Share Calculation:

What to Look For:
Look for calculations that use a ratio like share / oldShare to decide the liquidator’s collateral.
Red Flag:
If the calculation only considers the debt from a single position (using variables like oldShare from one debt) instead of the total debt, then it may allow a small repayment to unlock a large share of the collateral.

# Check for Total Debt Comparison:

What to Look For:
Verify whether the function calculates the liquidator’s share based on the total debt (e.g., using a value like getDebtValue(positionId)).
Red Flag:
If the code uses only the debt related to one part of the borrower’s positions instead of comparing the repaid amount with the total debt, this indicates a flaw.
Simulate a Multi-Debt Scenario:

What to Look For:
Imagine or test a case where the borrower has multiple debts.
For instance, if the borrower has three loans but the liquidation calculation only uses one loan’s share.
Red Flag:
If the liquidator can repay a tiny amount on one loan and receive a disproportionate amount of collateral (as if it were based on total debt), this is a vulnerability.


