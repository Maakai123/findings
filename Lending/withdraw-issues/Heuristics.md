# Heuristic 1: 
Ensure that any ratio or invariant checks occur after all related state updates (e.g., summing totals).
Example: In the flawed code, the ratio check is done inside the loop when totals are still zero. The check should run after the loop once all amounts are accumulated.

# Edge Case Testing:
Heuristic: Look at what happens when key values (like payments or withdrawals) are zero.
Example: An input of 0 USDC with a full collateral withdrawal passes the check because multiplying by zero always gives zero.

# Loop Behavior Analysis:

Heuristic: When using loops to process multiple entries, check if per-iteration checks are appropriate or if a final check should validate the aggregated result.
Example: Instead of checking inside the loop, accumulate totals for asset payments and collateral withdrawals, then perform one overall ratio check.

In the code example, performing the ratio check inside the loop allowed an attacker to take advantage of the fact that the accumulated totals were still zero during the first iteration. By moving the check outside the loop (after all entries are processed), the system correctly verifies that the final ratio of debt to collateral is maintained, preventing exploitation.



