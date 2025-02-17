
## Heuristic 1: Check for Duplicate Entries

Rule: When a collateral token is added, the system should verify if it already exists.
What to Look For:
Does the code check if the token is already in the list?

If not, a duplicate may allow an overwrite.

## Heuristic 2: Prevent Unauthorized Value Overwrites
Rule: Once collateral is committed with a nonzero amount, the value should not be allowed to be overwritten (especially not to 0) without proper authorization or state changes.
What to Look For:
Does the function allow setting the collateral amount to 0 after it’s been recorded?


## Heuristic 3: Verify Return Value Handling
Rule: When using helper functions (like adding an address to a set), always check the return value to know if the operation was new or a duplicate.
What to Look For:
Is the return value from the token addition function (e.g., add()) being checked?
If not, that may hide duplicate additions that could be exploited.

## Heuristic 4: Enforce State Transitions
Rule: Collateral should be modifiable only during allowed phases of the loan (for example, before the loan is finalized).
What to Look For:
Does the code allow changes to collateral after the loan is validated or accepted?