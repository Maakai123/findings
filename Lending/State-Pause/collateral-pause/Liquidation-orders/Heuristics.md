# Identify and Categorize Functions:
# What to Look For:
Group functions by their action type:
Token Transfer Functions: Functions that actually move tokens (e.g., closing a position or adding collateral).
Internal Accounting Functions: Functions (like liquidation orders) that only update internal numbers (using increment/applyDelta) without transferring tokens.
# Red Flag:
If liquidation functions update state without performing a token transfer, that’s a potential vulnerability.

# Check for Token Transfer Dependencies:
Verify whether each function performs a token transfer. For example:
Does the function call methods like transfer(), safeTransferFrom(), etc.?
Or does it simply adjust balances internally?
Red Flag:
Liquidation functions that lack token transfer calls can operate even when token transfers would revert (e.g., when the token is paused).

Simulate a Paused Token Scenario:

What to Look For:
Test the system in a state where the collateral token is paused or frozen.
Expected Behavior:
Functions that depend on token transfers (like closing positions or adding collateral) should revert.
Liquidation functions that only update accounting should still execute.
Red Flag:
This discrepancy means that users can be liquidated even if they’re unable to act to protect their position.

# Compare Consistency Across Similar Operations:

What to Look For:
Check if similar operations (e.g., closing a position versus liquidating one) use consistent logic regarding token transfers.
Red Flag:
Inconsistent behavior (actual transfers vs. internal updates) indicates that liquidations can bypass restrictions imposed on user actions.

# Simulate a Paused Token Scenario:

What to Look For:
Test the system in a state where the collateral token is paused or frozen.
Expected Behavior:
Functions that depend on token transfers (like closing positions or adding collateral) should revert.
Liquidation functions that only update accounting should still execute.
Red Flag:
This discrepancy means that users can be liquidated even if they’re unable to act to protect their position.

# What to Look For:
Evaluate the overall impact:
If liquidations bypass token transfers, users might be forced into liquidation without any means to close or safeguard their positions.

Red Flag:
High risk is flagged if users are exposed to liquidation even when they could potentially avoid it by acting (but cannot due to paused token transfers).
Final Recommendation Check:

# What to Look For:
Verify if the protocol holds collateral in a separate address from the pool.