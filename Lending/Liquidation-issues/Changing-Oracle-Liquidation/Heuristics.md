

##  Heuristic 1: Lock Critical Parameters After Activation

Rule: Once a loan is active (e.g., status is LOAN_OUTSTANDING), key parameters such as the Oracle should not be changeable.
What to Look For:
Check if update functions allow the Oracle address to be changed after the loan is live.

## Heuristic 2: Verify Invariant Checks in Update Functions
Rule: Update functions must include checks (using require statements) that prevent altering critical values.
What to Look For:
Look for code in functions like updateLoanParams() that should enforce params.oracle == cur.oracle but doesn’t.
Analogy:
Think of it as a security seal on a package—if the seal is missing, anyone could tamper with the contents.


## Heuristic 3: Role-Based Update Permissions
Rule: Only the party allowed to update certain parameters should have that ability, and only in a way that doesn’t harm the other party.
What to Look For:
Determine if lenders (or any party with a conflict of interest) are permitted to update parameters that affect the valuation of collateral.


## Heuristic 4: Check for Oracle-Dependent Value Calculations
Rule: Identify where the contract uses the Oracle’s reported values to make decisions (like seizing collateral).
What to Look For:
Look at sections of the code (like in removeCollateral()) where the Oracle value is used in checks. Confirm that this value can’t be manipulated by changing the Oracle.

##  Heuristic 5: Simulate the Loan Lifecycle
Rule: Walk through the loan’s process from request to activation to updating parameters.
What to Look For:
See if there’s a point where an update (by a lender, for example) can change the Oracle after the loan is live.


