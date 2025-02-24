
```solidity 
   function _checkReentrancyContext() internal override {
        uint256[2] memory minAmounts;
        ICurve2TokenPool(address(CURVE_POOL)).remove_liquidity(0, minAmounts);
    }
```


## Explanation

The _checkReentrancyContext() function is meant to prevent reentrancy attacks by using a safe call to the Curve pool.

# The Issue:
Some Curve pools (like the CRV/ETH pool) revert on the call remove_liquidity(0, [0,0]) due to an underflow error. This makes liquidations impossible for those pools.

# Impact:
If liquidation can’t occur, borrowers can remain undercollateralized, leading to bad debts in the protocol.

# Mitigation:
Instead of using remove_liquidity, use the claim_admin_fees function to check the pool’s state. This avoids the underflow issue and ensures that the liquidation process can continue as intended.


uint256[2] memory minAmounts;
This creates an array of two numbers that, by default, are both 0.
remove_liquidity(0, minAmounts);
The code calls the Curve pool’s remove_liquidity function, asking it to remove 0 liquidity (i.e., remove nothing) and passing in the minimum amounts required (which are both 0).


## The Vulnerability

Issue:
For some Curve pools—such as the CRV/ETH pool—the remove_liquidity(0, [0,0]) call always fails (reverts).
Why It Fails:
In these pools, the internal calculations in remove_liquidity perform an operation (like subtracting values) that results in an underflow when the liquidity amount is 0.

``` solidity 
function _checkReentrancyContext() internal override {
    uint256[2] memory minAmounts;
    ICurve2TokenPool(address(CURVE_POOL)).remove_liquidity(0, minAmounts);
}
It relies on calling remove_liquidity(0, [0,0]) as a “sanity check”. However, in certain Curve pools, this call always fails due to an underflow error.
```


## Illustrating with Real Numbers
Scenario A: A Normal Curve Pool

Operation: Call remove_liquidity(0, [0,0])
Expected Outcome: The function completes successfully because the internal math handles 0 correctly.
Liquidation: The liquidation process moves forward.
Scenario B: A Problematic Curve Pool (e.g., CRV/ETH)

Operation: Call remove_liquidity(0, [0,0])
Outcome:
Suppose the function internally tries to compute something like currentLiquidity - 0 but then later subtracts a fee or does another operation that, when combined with 0, causes an underflow.
Result: The call reverts with an error.
Liquidation: Because this check fails, the entire liquidation process reverts.
Impact: An undercollateralized borrower escapes liquidation, leaving the protocol with bad debt.

## Mitigation 

The recommendation is to change the way the system checks for reentrancy in the Curve vault. Instead of using the remove_liquidity function, the developers should use the claim_admin_fees function to verify the reentrancy context.

