
Identify all functions that manage loan closure and liquidation (e.g., closeLoan, callLiquidation).
Heuristic:
Check if these functions rely on a flag (like collateralValid) to determine if they should execute.

# State Flag Verification
Locate where the collateral is paused (e.g., in pauseCollateralType) and see how it sets flags:

```solidity 
collateralValid[_collateralAddress] = false;
collateralPaused[_collateralAddress] = true;
```

Heuristic:
Verify if the paused state (i.e., collateralValid being false) prevents both loan closure and liquidation. If yes, that’s a red flag.

# Contextual Checks:
What to Look For:
Determine whether the check for collateral validity is needed for all operations:
Should it apply to closing loans (where the user is recovering their collateral)?
Should it apply to liquidations (where the system should still allow debt resolution)?
Heuristic:
If the same collateral check blocks both operations even when closing a loan (which doesn’t require fresh collateral valuation), then it’s likely too strict.

# Simulation of State Changes:

What to Look For:
Simulate a scenario where governance pauses a collateral type.
Heuristic:
Verify if:
Existing loans using the paused collateral cannot be closed.
Liquidations revert because the collateral is flagged as invalid.
Red Flag:
If both operations are blocked, it indicates a potential for frozen user funds and unresolved bad debt.
