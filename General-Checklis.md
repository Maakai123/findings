The function should check that the _rewardProportion value is not greater than 1e18. This ensures that the keeper can only set a value between 0% and 100%

- What happens if i pass in a rewardProportion value that is greater than specified?

- What happens if i pass in a credit id that does not exist? 
Simulate calls with non-existent or default values to see how the contract handles them.

- What happens  when refinancing a loan, and the contract checks whether the new pool is different from the original (old) ?

- What happens when the buying pool (poolId) is the same as the selling pool (calculated as oldPoolId).

- What happens when the ratio check is done inside the loop when totals are still zero ? . 
-The check should run after the loop, once all amounts are accumulated.

- are  (e.g., the seller pool vs. the buyer pool) all kept separate.

-Check how the function handles extreme values, such as a nearly zero price like 99% price drop?

- Is the liquidation surcharge computed on the same value (or a related value) as the collateral being liquidated?
Check for a mismatch: e.g., raw collateral calculation vs. a capped value used later.

- code relies on the order of elements in a data structure ? e.g., an EnumerableSet

- Does the  Removal methods (used  “swap-and-pop”) that change the collection’s order during iteration eg [a,b,c] => [c,b]? 
- The function includes checks that revert if the price is out of bounds ?
- When a function performs a security check (like verifying the price source), confirm that any external calls it makes are properly constrained.
- "Does the liquidation logic use block.timestamp - lastRepaidTimestamp without considering when the next payment is actually due?"
-  Liquidation logic that doesn't reference payment cycle duration


- Heuristic: Run tests where inputs are at or above the limit (e.g., repay more than the debt).
Example: If the total debt is 1 ETH, test with a repayment of 2 ETH to see if the contract correctly blocks the 

- Look for arithmetic (like subtraction) inside unchecked blocks.because it does not handle underflow and overflow

- When a collateral token is added, the system should verify if it already exists
- If it exist can the collateral be overwritten to zero ?
- Once a loan is active (e.g., status is LOAN_OUTSTANDING), can key parameters such as the Oracle ?
- Verify that the function includes a check to ensure no open positions exist before disabling the market.

- Verify that token disallowance only applies to new loans.


- Heuristic:
Verify if the paused state (i.e., collateralValid being false) prevents both loan closure and liquidation. If yes, that’s a red flag.

- What happens if you Test the system in a state where the collateral token is paused or frozen.

- is There a  mechanism to delay liquidations for a short "grace period" after repayments resume.





