

# 1. Input Validation Checks
Core Risk: Attacker provides fake/irrelevant collateral token to manipulate liquidation calculations.
Detection Questions:
Does the liquidation function cross-reference the provided collateralToken with the borrower's on-chain registered collateral?
Example: If Alice’s position uses ETH, does the code check collateralToken == ETH?
Are there safeguards against address(0) or non-whitelisted tokens being passed as collateral?
Real-World Exploit: Cream Finance allowed invalid tokens, leading to $130M losses35.
Does the code use an allowlist for valid collateral tokens?

# 2. Collateral Source Verification
Core Risk: Using attacker-provided token balance instead of the borrower’s actual collateral.
Detection Questions:
Does the code fetch collateral balances directly from the protocol’s storage (e.g., position.collateral) instead of relying on input parameters?
Vulnerable Code:
text
// BAD: Uses input token instead of stored collateral
raftCollateralTokens[collateralToken].token.balanceOf(position)  
Is there a position registry mapping borrowers to their collateral?
Analogy: Like a bank verifying your house is the collateral, not a random shed.

# 3. Oracle Manipulation Risks
Core Risk: Price feed tied to unvalidated collateral token.
Detection Questions:
Does the price feed (priceFeeds[collateralToken]) validate that collateralToken matches the position’s actual asset?
Exploit Scenario: Attacker passes a token with manipulated price feed (e.g., low-liquidity asset).
Are price feeds restricted to pre-approved tokens?
Best Practice: AAVE uses curated oracles for approved assets1.
#  4. Liquidation Threshold Logic
Core Risk: Incorrect collateral/debt ratio calculation.
Detection Questions:
Does the liquidation logic recalculate the borrower’s actual collateral value independently of inputs?
Critical Check:
text
// SAFER: Fetch from stored position data
uint256 realCollateral = positions[position].collateralBalance;  
Are there secondary checks (e.g., health factor) beyond the immediate icr calculation?
# 5. Protocol-Wide Consistency
Core Risk: Mismatched collateral handling across integrated systems.
Detection Questions:
If the protocol integrates with external platforms (e.g., Iron Bank in Alpha Homora1), do they share the same collateral registry?
Are liquidation parameters (LTV, thresholds) synchronized across all modules?
💥 Real-World Exploit Examples
Protocol	Flaw	Impact
Alpha Homora	Mismatched collateral logic with Iron Bank	$37M stolen1
Cream Finance	Allowed fake collateral tokens	$130M exploited3
Compound	Self-liquidation bug	Collateral stolen1


## SOLUTION

Actionable Audit Checklist
Input Sanitization:
 Validate collateralToken against a protocol-managed allowlist.
 Revert if collateralToken is address(0) or unregistered.
Collateral Verification:
 Replace raftCollateralTokens[collateralToken] with positions[position].collateralToken.
Oracle Security:
 Use time-weighted (TWAP) prices for low-liquidity tokens15.
 Implement multi-oracle fallback (e.g., Chainlink + Uniswap).
Liquidation Circuit Breakers:
 Add delay for large liquidations to prevent cascading risks

# Authorization and Caller Restrictions

Heuristic: Consider if the function’s access control might allow attackers to manipulate the parameters.
Questions:
"Who is allowed to call the liquidate function, and are there checks to ensure that only authorized entities can trigger liquidation?"
"Could an attacker, by being an authorized caller, inject a malicious collateral token value into the liquidation process?"