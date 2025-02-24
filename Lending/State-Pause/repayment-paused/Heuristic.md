
# List Similar Functions:

Identify functions that perform related actions—such as debt repayment, collateral adjustments, or liquidation.

Example: Compare the repay() function (which uses a flag check) with the liquidate() function (which does not).
Compare Common Logic:
Analyze if functions that invoke similar internal logic (like calling repayInternal()) enforce the same state or permission checks.

Identify Essential Flags:
Determine the critical flags or state variables (e.g., isRepayAllowed()) that should gate sensitive operations.

Consistency Across Functions:
Ensure every function that adjusts debt or collateral checks these flags.

Example: Notice that repay() checks isRepayAllowed(), while liquidate() does not. This discrepancy is a red flag.
