## Heuristic 1: Verify Use of External Calls for Security Checks
Rule:
When a function performs a security check (like verifying the price source), confirm that any external calls it makes are properly constrained.

What to Look For:

External Call: The function calls an external router

## Heuristic 2: Test Edge Cases and Boundary Conditions
Rule:
Ensure that security-critical functions handle extreme or unexpected input values safely.

What to Look For:

Fixed Test Amount: The use of a constant (HUNDRED_TOKENS) ensures consistency.
Boundaries Check: The code calculates acceptable deviations from an oracle price

## Heuristic 3: Confirm Consistency Across All External Integrations
Rule:
When your contract interacts with external protocols, verify that your assumptions hold true across every variation of those protocols.

What to Look For:

Router Assumptions: The code assumes that router.getAmountOut will select the correct pool type.
Potential Inconsistencies: Different pools or protocol versions might return values differently or behave unexpectedly.


## Heuristic 5: Monitor for Unexpected Reverts or Failures in Critical Paths
Rule:
For functions that are critical to system security (like those that determine liquidation), monitor and log unexpected failures.

What to Look For:

Revert Conditions: The function includes checks that revert if the price is out of bounds

