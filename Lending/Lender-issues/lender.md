```solidity

function removeCollateral(uint256 tokenId, address to) public {
        TokenLoan memory loan = tokenLoan[tokenId];
        if (loan.status == LOAN_REQUESTED) {
            // We are withdrawing collateral that is not in use:
            require(msg.sender == loan.borrower, "NFTPair: not the borrower");
        } else if (loan.status == LOAN_OUTSTANDING) {
            // We are seizing collateral towards the lender. The loan has to be
            // expired and not paid off, or underwater and not paid off:
            require(to == loan.lender, "NFTPair: not the lender");

            if (uint256(loan.startTime) + tokenLoanParams[tokenId].duration > block.timestamp) {
                TokenLoanParams memory loanParams = tokenLoanParams[tokenId];
                // No underflow: loan.startTime is only ever set to a block timestamp
                // Cast is safe: if this overflows, then all loans have expired anyway
                uint256 interest = calculateInterest(
                    loanParams.valuation,
                    uint64(block.timestamp - loan.startTime),
                    loanParams.annualInterestBPS
                ).to128();
                uint256 amount = loanParams.valuation + interest;
                (, uint256 rate) = loanParams.oracle.get(address(this), tokenId);
                require(rate.mul(loanParams.ltvBPS) / BPS < amount, "NFT is still valued");
            }
        }
   
   function updateLoanParams(uint256 tokenId, TokenLoanParams memory params) public {
        TokenLoan memory loan = tokenLoan[tokenId];
        if (loan.status == LOAN_OUTSTANDING) {
            // The lender can change terms so long as the changes are strictly
            // the same or better for the borrower:
            require(msg.sender == loan.lender, "NFTPair: not the lender");
            TokenLoanParams memory cur = tokenLoanParams[tokenId];
            require(
                params.duration >= cur.duration &&
                    params.valuation <= cur.valuation &&
                    params.annualInterestBPS <= cur.annualInterestBPS &&
                    params.ltvBPS <= cur.ltvBPS,
                "NFTPair: worse params"
            );
        } else if (loan.status == LOAN_REQUESTED) {
            // The borrower has already deposited the collateral and can
            // change whatever they like
            require(msg.sender == loan.borrower, "NFTPair: not the borrower");
        } else {
            // The loan has not been taken out yet; the borrower needs to
            // provide collateral.
            revert("NFTPair: no collateral");
        }
        tokenLoanParams[tokenId] = params;
        emit LogUpdateLoanParams(tokenId, params.valuation, params.duration, params.annualInterestBPS, params.ltvBPS);
    }
```



## Explanation
Purpose:
- An Oracle in this context is meant to report the current market value of the NFT collateral.

- It protects both parties:
Lender: Can seize the collateral if its value drops, ensuring the loan is covered.
Borrower: Should not lose their asset if its value has gone up, meaning they shouldn’t be liquidated when their collateral is worth more than what they owe.

The Intended Flow
Loan Request:
The borrower calls requestLoan() and sets up the initial parameters, including choosing an Oracle that reports the NFT’s value.
Loan Agreement:
A lender agrees to the terms and calls lend(), changing the loan’s status to LOAN_OUTSTANDING. At this point, both parties expect the Oracle to remain the one agreed upon.
Collateral Seizure (Liquidation):
If the borrower fails to repay, the protocol can liquidate the collateral. During liquidation, the current Oracle is used to get the NFT’s value.

## The Critical Flaw: Oracle Manipulation

The Critical Flaw: Oracle Manipulation
What Goes Wrong:
After the loan is active, the lender can call updateLoanParams() to change the loan parameters. The critical mistake is that the code does not check if the Oracle is the same as the one originally agreed upon.
Impact:
A malicious lender could change the Oracle to one that reports a lower value for the NFT collateral. This manipulated value makes it easier to trigger liquidation—even when the actual market value of the NFT is much higher—allowing the lender to seize the collateral at the borrower’s expense.


Borrower sets up the loan with an Oracle that shows the NFT value as $1,000.
After Loan is Outstanding:
The lender calls updateLoanParams() and changes the parameters, including swapping the Oracle with one under their control.
Manipulated Oracle Outcome:
The malicious Oracle now reports the NFT value as $500 instead of $1,000.
Liquidation Trigger:
The system calculates the maximum allowed debt based on the manipulated $500 value.
For instance, if the max LTV is 60%, the new maximum loan would be $300 (i.e., 60% of $500).
Since the borrower still owes $600, the system sees the position as underwater and triggers liquidation immediately.

## FLAWS

The updateLoanParams function allows the lender to change various parameters once the loan is outstanding. However, it does not verify whether the Oracle parameter (params.oracle) remains unchanged. This omission lets a lender substitute a malicious Oracle that under-reports the NFT's true value.

During collateral removal, the contract calls the Oracle via loanParams.oracle.get(...) to fetch the current value. If this Oracle has been swapped for a malicious one, it might return a lower value, triggering the liquidation process even if the real market value of the NFT is much higher.


