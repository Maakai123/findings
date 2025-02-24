
## Heuristic 1: Verify Use of External Calls for Security Checks
Rule: When a function is used as a security check (like preventing reentrancy), check if it calls external functions that might behave unexpectedly.
What to Look For:
Does the check call an external function (e.g., remove_liquidity(0, [0,0]))?
Is that external function known to have edge-case issues (like underflow) for some target contracts?


## Heuristic 2: Test Edge Cases and Boundary Conditions
Rule: Ensure that functions used in security checks are tested with extreme or boundary input values (such as zero).
What to Look For:
Look for calls where parameters are set to zero.
Check whether such calls are safe or if they revert in some cases.

# Heuristic 3: Confirm Consistency Across All External Integrations
Rule: When your contract relies on external protocols (like Curve pools), verify that your assumptions hold for every variation of that protocol.
What to Look For:
Check if the same external function behaves differently in various Curve pools.
Look for documented behavior differences (e.g., some Curve pools revert when removing 0 liquidity).


# Heuristic 5: Monitor for Unexpected Reverts or Failures in Critical Paths
Rule: In functions that must succeed for the system to work (like liquidations), add tests or logging to catch unexpected reverts.
What to Look For:
Look for unexpected failures in the reentrancy check path.
Check if these failures are caused by calling functions with 0 or boundary values.