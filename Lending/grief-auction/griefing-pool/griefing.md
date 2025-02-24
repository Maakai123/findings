
```solidity 

function buyLoan(uint256 loanId, bytes32 poolId) public {
        // get the loan info
        Loan memory loan = loans[loanId];
        // validate the loan
        if (loan.auctionStartTimestamp == type(uint256).max)
            revert AuctionNotStarted();
        if (block.timestamp > loan.auctionStartTimestamp + loan.auctionLength)
            revert AuctionEnded();
        // calculate the current interest rate
        uint256 timeElapsed = block.timestamp - loan.auctionStartTimestamp;
        uint256 currentAuctionRate = (MAX_INTEREST_RATE * timeElapsed) /
            loan.auctionLength;
        // validate the rate
        if (pools[poolId].interestRate > currentAuctionRate) revert RateTooHigh();
        // calculate the interest
        (uint256 lenderInterest, uint256 protocolInterest) = _calculateInterest(
            loan
        );

        // reject if the pool is not big enough
        uint256 totalDebt = loan.debt + lenderInterest + protocolInterest;
        if (pools[poolId].poolBalance < totalDebt) revert PoolTooSmall();

        // if they do have a big enough pool then transfer from their pool
        _updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
        pools[poolId].outstandingLoans += totalDebt;

        // now update the pool balance of the old lender
        bytes32 oldPoolId = getPoolId(
            loan.lender,
            loan.loanToken,
            loan.collateralToken
        );
        _updatePoolBalance(
            oldPoolId,
            pools[oldPoolId].poolBalance + loan.debt + lenderInterest
        );
        pools[oldPoolId].outstandingLoans -= loan.debt;

        // transfer the protocol fee to the governance
        IERC20(loan.loanToken).transfer(feeReceiver, protocolInterest);

        emit Repaid(
            loan.borrower,
            loan.lender,
            loanId,
            loan.debt + lenderInterest + protocolInterest,
            loan.collateral,
            loan.interestRate,
            loan.startTimestamp
        );

        // update the loan with the new info
        loans[loanId].lender = msg.sender;
        loans[loanId].interestRate = pools[poolId].interestRate;
        loans[loanId].startTimestamp = block.timestamp;
        loans[loanId].auctionStartTimestamp = type(uint256).max;
        loans[loanId].debt = totalDebt;

        emit Borrowed(
            loan.borrower,
            msg.sender,
            loanId,
            loans[loanId].debt,
            loans[loanId].collateral,
            pools[poolId].interestRate,
            block.timestamp
        );
        emit LoanBought(loanId);
    }


```


## Explanation

An attacker can call buyLoan() with a poolId that matches the selling pool, causing inconsistent updates. This results in a loss of pool balance and an increase in outstanding loans, which degrades the pool’s financial health.


## Impact:
The pool’s balance drops by the protocol interest amount, and outstanding loans are increased by extra fees. This “griefing” can negatively impact the pool’s stability.

## We have several pools of funds managed by protocol

Lend out funds to borrowers.
Hold a balance (pool balance) and track how much is currently loaned out (outstanding loans).

When a loan is auctioned (because, for example, a borrower is undercollateralized), someone can call buyLoan() to take over that loan. This function is supposed to:

Deduct the full debt (plus interest) from the buyer’s pool.
Increase that pool’s “outstanding loans” by the debt amount.
Then, credit the original lender’s pool with part of the debt (excluding some fees).

## The key expectation:
When buying a loan from an auction, the pool specified by the buyer should be different from the pool that originally auctioned (sold) the loan.

## What’s the Vulnerability?

The function buyLoan() does not check whether the pool ID passed as a parameter is the same as the pool that auctioned the loan. This means an attacker can:

Buy the auctioned loan back to the same pool that originally auctioned it.
This “re-purchase” (or reallocation) ends up harming that pool financially.

# First Update (Deduction):
The pool’s balance is reduced by the total debt, which is calculated as:

totalDebt=loan.debt+lenderInterest+protocolInterest
```solidity
_updatePoolBalance(poolId, pools[poolId].poolBalance - totalDebt);
pools[poolId].outstandingLoans += totalDebt;


```
# Second Update (Credit Back):

Then, using the old pool ID (which, in a proper scenario, should be different), the function credits back:
If the attacker calls buyLoan() with a poolId that is the same as the old pool ID, then the same pool is updated twice.

loan.debt = 100 tokens
lenderInterest = 10 tokens
protocolInterest = 5 tokens
Then:

totalDebt = 100 + 10 + 5 = 115 tokens
First Update (on the buying pool):

The pool’s balance is reduced by 115 tokens.
Outstanding loans in that pool are increased by 115 tokens.


# Second Update (on the old pool, which is the same pool in this attack):

The pool’s balance is increased by (loan.debt + lenderInterest) = 100 + 10 = 110 tokens.
Outstanding loans are decreased by 100 tokens.
Net Effect on the Pool:

Pool Balance Change:
Deduction of 115 tokens, then crediting 110 tokens results in a net loss of 5 tokens.
(That 5 tokens equals the protocolInterest.)
Outstanding Loans Change:
Increase of 115 tokens, then a reduction of 100 tokens gives a net increase of 15 tokens.

## Recommended Mitigation:

Modify buyLoan() to reject transactions where the provided poolId is the same as the selling pool’s ID (oldPoolId). For example:

```solidity 
require(poolId != getPoolId(loan.lender, loan.loanToken, loan.collateralToken), "Cannot buy loan back to same pool");


```




