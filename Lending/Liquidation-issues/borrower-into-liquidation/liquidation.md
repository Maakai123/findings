
```Solidity

function repay(

  ILineOfCredit.Credit memory credit,

  bytes32 id,

  uint256 amount

)

  external

  returns (ILineOfCredit.Credit memory)

{ unchecked {
   
    if (amount <= credit.interestAccrued) {

        credit.interestAccrued -= amount;

        credit.interestRepaid += amount;

        emit RepayInterest(id, amount);

        return credit;

    } else {

        uint256 interest = credit.interestAccrued;

        uint256 principalPayment = amount - interest;


        // update individual credit line denominated in token

        credit.principal -= principalPayment; // @audit-info potential underflow without an error due to the unchecked block

        credit.interestRepaid += interest;

        credit.interestAccrued = 0;


        emit RepayInterest(id, interest);

        emit RepayPrincipal(id, principalPayment);


        return credit;

    }

} }

```

## EXPLANATION

If someone repays more than what is owed (interest + principal), the contract subtracts too much from the principal without any error-checking due to an unchecked block. This causes an underflow, making the principal a huge number.

## ISSUES
`Principal`: The original amount you borrowed.
`Interest`: Extra money you owe for borrowing that money.

When you make a repayment, you’re supposed to pay back both the interest and some (or all) of the principal. The smart contract function in question is designed to deduct your payment from the interest first and then from the principal.

## The bug:
If someone sends in more money than what’s actually owed (i.e., more than the sum of the interest and the principal), the contract subtracts too much from the principal. Because it does this math without proper checks, the subtraction “underflows” – which means instead of throwing an error, it wraps around to a gigantic number. This huge number makes it look like you owe an almost impossible-to-pay amount, forcing your account into liquidation.

```solidity 
 credit.principal -= principalPayment;
 happens inside an unchecked block. In Solidity, this means no automatic error is thrown if the subtraction goes below zero (an “underflow”).
 ```
You borrow $100 (this is your principal).
Over time, you owe $0 in interest for simplicity.
Now, imagine you accidentally pay $150 instead of $100.
What should happen:
The extra $50 might be refunded or flagged as an error.

What happens with this bug:
The system takes your $150, subtracts $0 for interest, and then tries to subtract $150 from your $100 principal. Since $150 is more than $100, instead of stopping or refunding you, the math “wraps around” and records your new principal as a huge number (like $2^256 - 50). Now, it looks like you owe an astronomical amount instead of nothing.
## FIX
the contract should check that the repayment amount does not exceed the total owed. This can be done with a simple line like:
```solidity
require(amount <= credit.principal + credit.interestAccrued, "Repayment exceeds total debt");
```
credit.principal: 1 ETH
credit.interestAccrued: 0 ETH
Total Debt: 1ETH + 0ETH= 1ETH
Repayment Amount (amount): 2 ETH

2ETH  ≤  1ETH

Since 2 ETH is not less than or equal to 1 ETH, the check fails. This failure causes the function to revert with the error message:




