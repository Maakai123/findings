
## Heuristic 1: Parameter Validation
Rule: Check that key parameters (like poolId) are validated against expected values.
What to Look For:
Ensure that the function rejects cases where the buying pool (poolId) is the same as the selling pool (calculated as oldPoolId).

## Heuristic 2: Consistency in Financial Updates
Rule: Verify that all financial updates (pool balances and outstanding loans) are consistent and use the same basis for calculations.
What to Look For:
Check that fees (like protocol interest) are applied consistently in both the debit and credit steps.


##  Heuristic 3: Isolation Between Parties
Rule: Confirm that operations intended for different entities (e.g., the seller pool vs. the buyer pool) are kept separate.
What to Look For:
Look for conditions that enforce that the auctioned loan cannot be re-bought into the same pool.

## Heuristic 4: Simulate Malicious Scenarios
Rule: Test edge cases where an attacker might intentionally use the same pool ID for both sides.
What to Look For:
Create tests where the provided poolId equals the originating pool’s ID and check if the function properly rejects the transaction.