Does the update function for loan parameters validate that the Oracle remains unchanged after a loan becomes outstanding?"
"Can a lender change the Oracle to one under their control after the loan is active?"
Parameter Integrity:

"Are all critical parameters (duration, valuation, interest, LTV) checked to ensure they are not worsened for the borrower, including the Oracle address or mechanism?"
"Is there a safeguard preventing a lender from lowering the reported collateral value via Oracle manipulation?"
Collateral Valuation During Liquidation:

"During the collateral removal process, is the Oracle’s value cross-checked with an independent or fallback source to avoid manipulation?"
"What happens if the Oracle returns a value significantly lower than expected? Is there a mechanism to detect or dispute such a result?"
Access Control:

"Who is authorized to update the loan parameters, and are there any checks to ensure malicious updates cannot occur?"
"Is it clearly enforced that only the borrower should be able to update certain parameters before the loan is funded, and only benign changes are allowed post-funding?"


