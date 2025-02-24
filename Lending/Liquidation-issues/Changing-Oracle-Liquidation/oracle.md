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
        // If there somehow is collateral but no accompanying loan, then anyone
        // can claim it by first requesting a loan with `skim` set to true, and
        // then withdrawing. So we might as well allow it here..
        delete tokenLoan[tokenId];
        collateral.transferFrom(address(this), to, tokenId);
        emit LogRemoveCollateral(tokenId, to);
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

            // **Flaw:** There is no check to ensure that the Oracle remains the same.

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

Intended Functionality:
The Oracle is supposed to provide a fair, current value of the NFT collateral to protect both parties.
Flaw:
A lender can change the Oracle after a loan is active, potentially using a manipulated Oracle that undervalues the collateral.
Real-World Impact:
The borrower risks losing a valuable asset because the lender can seize collateral based on a false, lower valuation.

## MITIGATION

```solidity 
require(params.oracle == cur.oracle, "Oracle cannot be changed");
```

## Scenario:
Imagine you pledge a painting as collateral for a loan.
Initial Situation:
An independent appraiser (OracleA) values your painting at $100, and you receive a $50 loan.
Market Change:
Over time, the painting becomes more valuable, now worth $150.
Lender’s Trick:
The lender, wanting to recover more money, convinces the bank to use a different appraiser (OracleB) that reports the painting’s value as only $60.
Outcome:
Because the bank now “believes” the painting is worth less, it allows the lender to seize the painting—even though in reality it’s worth much more.
Impact:
The borrower loses a valuable asset due to the manipulated valuation.



