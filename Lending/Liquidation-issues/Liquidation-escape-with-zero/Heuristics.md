
# Heuristic 1: Validate Collateral Quality
Rule: When collateral is deposited, its liquidity value should be positive.
What to Look For:
Check that the code verifies that liquidity > 0 when a Uniswap V3 position is recorded.
Real-World Analogy:
Like a bank checking that a savings account used as collateral has a balance above $0.


# Heuristic 2: Precondition Checks in Critical Functions

Rule: Before removing liquidity (during liquidation), the function must confirm that the liquidity parameter is greater than 0.
What to Look For:
Look for a line such as require(params.liquidity > 0, "No liquidity available to remove"); in functions like decreaseLiquidity().
Real-World Analogy:
Similar to a vending machine that verifies you’ve inserted enough money before dispensing an item.


# Heuristic 3: End-to-End Flow Verification
Rule: Trace the complete liquidation process to ensure no step allows a 0 liquidity position to pass through.
What to Look For:
Verify that all functions in the liquidation chain (_unwrapUniPosition(), removeLiquidity(), decreaseLiquidity()) include proper checks for liquidity.
Real-World Analogy:
Like checking every link in a chain to ensure none are weak or broken.

Mapping the Entire Process:

Identify every step in the lifecycle of a loan.
For this issue, the flow starts with a borrower depositing a Uniswap V3 position as collateral and ends with the liquidation process that attempts to remove liquidity from that position.

At Deposit:
Ensure that when collateral is deposited, the system checks that the position has a positive liquidity value (i.e., more than 0).
During Loan Lifecycle:
Verify that the collateral remains valid throughout the loan period, and that any changes (if permitted) are subject to checks that prevent malicious modifications.
At Liquidation:
Confirm that when a liquidator calls functions like liquidateAccount(), the process—through functions like _unwrapUniPosition(), uniV3Helper.removeLiquidity(), and finally positionManager.decreaseLiquidity()—includes checks to reject any attempt to remove liquidity from a position that has 0 liquidity.





# Heuristic 4: Edge Case Testing

Rule: Simulate scenarios where a position with 0 liquidity is used.
What to Look For:
In testing, if liquidation calls revert or fail due to 0 liquidity, it signals the system is catching the issue. If not, it’s a red flag.
Real-World Analogy:
Like stress-testing a system by trying extreme or unusual inputs to see if it fails safely.
