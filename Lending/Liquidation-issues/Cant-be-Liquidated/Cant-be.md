```solidity
// AddressSet from https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable
// a loan must have at least one collateral
// & only one amount per token is permitted
struct CollateralInfo {
    EnumerableSetUpgradeable.AddressSet collateralAddresses;
    // token => amount
    mapping(address => uint) collateralInfo;
}

// loanId -> validated collateral info
mapping(uint => CollateralInfo) internal _loanCollaterals;

function commitCollateral(uint loanId, address token, uint amount) external {
    CollateralInfo storage collateral = _loanCollaterals[loanId];

    // @audit doesn't check return value of AddressSet.add()
    // returns false if not added because already exists in set
    collateral.collateralAddresses.add(token);

    // @audit after loan offer has been created & validated, borrower can call
    // commitCollateral(loanId, token, 0) to overwrite collateral record 
    // with 0 amount for the same token. Any lender who accepts the loan offer
    // won't be protected if the borrower defaults since there's no collateral
    // to lose
    collateral.collateralInfo[token] = amount;
}
```


## EXPLANATION

# What’s Happening:
The borrower can manipulate the system by first committing a non-zero amount of collateral to secure a loan and then later calling the same function to overwrite that collateral amount with 0.

# Why It’s a Problem:
This means that if the borrower defaults, the lender cannot liquidate any collateral because it now shows as 0—even though a valid loan was originally offered with actual collateral.
Imagine a pawn shop system where:

Imagin
- Borrower pledges a $10,000 Rolex watch as collateral for a $5,000 loan

- Loan Agreement shows "Rolex - $10,000" as collateral

- Borrower later secretly changes the agreement to "Rolex - $0"

 - Pawn Shop accepts modified agreement without verifying collateral still exists

- Borrower defaults → Pawn Shop tries to seize collateral but finds $0 value


# Flaws 
``` solidity

function commitCollateral(uint loanId, address token, uint amount) external {
    CollateralInfo storage collateral = _loanCollaterals[loanId];
    
    // ❌ Doesn't check if token already exists, or was already added.
    collateral.collateralAddresses.add(token); 
    
    // ❌ This allows overwriting the collateral amount,
    //even to 0 after the loan is validated
    collateral.collateralInfo[token] = amount; 
   // ✅ No require(amount > previousAmount)
}
```
#  How the Code Is Meant to Work
```solidity
When a borrower “commits” collateral using the function commitCollateral, it does two things:

collateral.collateralAddresses.add(token);
This should add the token (like TokenA) to the list.

Set the Amount:
collateral.collateralInfo[token] = amount;
This records the amount pledged (e.g., 100 units).

```

# ISSUES
Issue 1: Unchecked Addition of the Token
What Happens:
The function add(token) returns false if the token is already in the set.

The code does not check this return value, so it doesn’t know if the token was added for the first time or if it was already there.

Issue 2: Overwriting Collateral Amount
What Happens:
After the token is added, the code sets:

collateral.collateralInfo[token] = amount;
Problem:
This means a borrower can call commitCollateral again with the same token but with an amount of 0. Even though the token is already in the set, the code still overwrites the recorded collateral amount with 0.

## Attacks 

Create loan with 10 ETH collateral ($20,000)

Wait for lender approval

Front-run loan acceptance with commitCollateral(loanId, ETH, 0)

Lender releases funds against $0 collateral

Default → No assets to seize

## Solution

Check the Return Value:
When adding a token to the collateral set, check if it was already added.
Disallow Overwriting:
If the token is already in the set, prevent the collateral amount from being changed—especially to 0.

```solidity
function commitCollateral(uint loanId, address token, uint amount) external {
    CollateralInfo storage collateral = _loanCollaterals[loanId];

    // If the token isn't already in the set, try to add it.
    if (!collateral.collateralAddresses.contains(token)) {
        bool added = collateral.collateralAddresses.add(token);
        require(added, "Failed to add collateral token");
    } else {
        // If token exists, ensure that we don't overwrite with 0.
        require(amount > 0, "Cannot update collateral to 0");
    }
    
    // Set or update the collateral amount.
    collateral.collateralInfo[token] = amount;
}

```