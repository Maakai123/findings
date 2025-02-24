

## Spot Unchecked Arithmetic:

Heuristic: Look for arithmetic (like subtraction) inside unchecked blocks.because it does not handle underflow and overflow
Example: An unchecked subtraction of repayment from the principal can hide underflow errors.


Check Boundary Conditions:
Heuristic: Verify that the operation won’t remove more than what exists.
Example: Before subtracting, check that the repayment amount isn’t larger than the sum of the principal and interest.

## Ensure Proper Input Validation:

Heuristic: Look for require statements that validate inputs.
Example: A statement like
```solidity
require(amount <= credit.principal + credit.interestAccrued, "Repayment exceeds total debt");
```
helps prevent overpayments that could cause underflow.


## Test with Extreme Values:

Heuristic: Run tests where inputs are at or above the limit (e.g., repay more than the debt).
Example: If the total debt is 1 ETH, test with a repayment of 2 ETH to see if the contract correctly blocks the operation.