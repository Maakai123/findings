## Immutable Oracle Parameter Verification:

Question: 
Is the Oracle parameter recorded at loan initiation immutable, or is there a mechanism that allows it to be updated after the loan becomes outstanding?
Check: Look for code that prevents modifications to the Oracle once the loan status changes from "requested" to "outstanding".


## Access Control and Authorization:

Question: Who is allowed to update the Oracle parameter in the loan parameters update function?
Check: Verify that only the borrower (or an authorized party) can change the Oracle before the loan starts and that the lender is restricted from making changes after the loan is active.


## Parameter Consistency in Update Functions:

Question: In functions like updateLoanParams(), does the code validate that critical parameters (including the Oracle) remain unchanged or meet strict conditions if updated?

Check: Confirm that the update function includes explicit checks to ensure that the Oracle parameter is either unaltered or updated only under conditions that do not disadvantage the borrower.
## Event Logging and Audit Trails:

Question: Are Oracle changes logged with events so that any modifications post-loan initiation are transparent and auditable?
Check: Review the emitted events to ensure they include the Oracle parameter, making it possible to track any unauthorized changes.

## Fallback and Redundancy Measures:

Question: Does the system have fallback or cross-validation for the Oracle’s reported values if the Oracle has been updated post-loan initiation?
Check: Assess whether the protocol implements secondary checks or reference data to validate the Oracle’s accuracy during liquidation or other critical operations.
