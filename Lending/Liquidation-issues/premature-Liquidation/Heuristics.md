Static vs. Dynamic Collateral Amount:

Heuristic: Verify how the collateral amount is stored.
Question: Is the underlying deposit amount updated over time (to reflect accrued interest) or is it only increased on deposit?
Red Flag: A static variable (e.g., underlyingAmount) that only tracks deposits and never adjusts for interest.
Collateral Valuation Calculation:

Heuristic: Review functions (like getUnderlyingValue()) that compute collateral value.
Question: Does the valuation function incorporate any adjustments for interest earned, or does it simply multiply the static deposit amount by the current price?
Red Flag: Valuation based solely on an unadjusted deposit amount.
Risk Assessment Formula:

Heuristic: Analyze the risk calculation logic (e.g., in getPositionRisk()).
Question: Does the risk formula use the up-to-date (dynamic) value of the collateral, or does it rely on the unadjusted underlyingAmount?
Red Flag: Liquidation thresholds triggered by a ratio computed from an understated collateral value.
Interest Accrual Mechanism:

Heuristic: Check for functions or mechanisms that update or account for accrued interest on collateral.
Question: Is there a separate mechanism that tracks interest and adjusts the effective collateral value over time?
Red Flag: Absence of any interest accrual or compounding logic for collateral amounts.
Consistency Across Functions:

Heuristic: Ensure that all parts of the protocol that reference collateral value (lending, risk evaluation, liquidation) use a consistent and updated measure.
Question: Are the same valuation methods applied across deposit, risk, and liquidation functions, and do they reflect the true current value (including interest)?
Red Flag: Discrepancies where one part of the protocol (e.g., lending) uses a dynamic value but another (e.g., liquidation) uses the static deposit amount.


Check if one part of the protocol (like the deposit or lending function) updates and stores a dynamic collateral value (reflecting interest) while another part (like the risk or liquidation function) uses the original, static deposit amount.
Potential Issue:
This inconsistency can lead to a situation where a user’s position looks riskier than it actually is.
 For example, if the risk function underestimates the collateral value, it might trigger liquidation unnecessarily, 
 even though the user’s real collateral (including earned interest) is sufficient to cover the debt.