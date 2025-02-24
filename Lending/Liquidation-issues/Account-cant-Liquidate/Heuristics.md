# Heuristic 1: Validate Input Extremes

Rule: Check how the function handles extreme values, such as a nearly zero price.
What to Look For:
Does the function gracefully handle a 99% price drop?
Are any divisions by price or multiplications with high scaling factors safe when the price is very low?

# Heuristic 2: Consistency Between Raw and Capped Values
Rule: Ensure that the same basis is used across calculations that depend on one another.
What to Look For:
Is the liquidation surcharge computed on the same value (or a related value) as the collateral being liquidated?
Check for a mismatch: e.g., raw collateral calculation vs. a capped value used later.

# Heuristic 3: Check for Underflow/Overflow Risks
Rule: Identify arithmetic operations where extreme inputs might lead to underflows or overflows.
What to Look For:
Multiplications or divisions that might generate abnormally high or low numbers.
Compare the computed surcharge to the available collateral.


# Heuristic 4: Simulation of Edge Cases
Rule: Simulate scenarios where key inputs hit extreme values.
What to Look For:
Test the function with a 99% price crash.
Ensure that the computed liquidation collateral and surcharge remain realistic (e.g., not negative).

# Heuristic 5: Verify Business Logic Alignment
Rule: Make sure that calculated fees and collateral amounts align with the intended business logic.
What to Look For:
Confirm that the surcharge is based on the appropriate collateral value (capped or minimum of raw and capped).
Ensure the logic doesn't penalize a borrower excessively when market conditions are extreme.


