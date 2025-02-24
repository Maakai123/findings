
Collateral:
The user deposits 2 NFTs
Oracle Price per NFT: $100
Total Collateral Value: 2* $100 = $200

Max LVT(Loan to Value): 80%
Max Loan Amount(Max Debt)

maxDebt = (Total Collateral Value * maxLTV)/ cachedTarget
assume cachedTarget = 1

maxDebt = $200 * 0.8 = $160
80 % of 200

```solidity 

uint256 newDebt = _vaultInfo[account][asset].debt + amount;
uint256 max = _maxDebt(_vaultInfo[account][asset].count * oraclePrice, cachedTarget);

if (newDebt > max) revert IPaprController.ExceedsMaxDebt(newDebt, max);

```
## Borrowing Debt(_increaseDebt)

What Happens:
The borrower is allowed to borrow as long as their newDebt is ≤ $160.
Scenario:
The borrower takes on exactly $160 of debt.
This is permitted because the check only reverts if newDebt is greater than $160.


## Liquidation Trigger (startLiquidationAuction):

```solidity

if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
    revert IPaprController.NotLiquidatable();
}

```
What Happens:
The same maximum debt calculation is used here.
The function checks if the user's current debt is less than the maxDebt (which is $160).
If the debt equals $160 (i.e., the maximum), the condition is not met (since $160 is not less than $160), and therefore the function does not revert.
Result: The borrower is immediately considered liquidatable.

Suppose the price of each NFT drops slightly to $95.
New Total Collateral Value: 2 × $95 = $190

New Max Debt Calculation:

maxDebt=$190×0.8=$152

The borrower still owes $160, which is now above the allowed maximum of $152.
# Outcome:
The protocol will trigger liquidation immediately—even though the price drop is minor.


## Key Issues Highlighted in the Code
Borrowing Check Allows Maximum Debt:
```solidity 
if (newDebt > max) revert IPaprController.ExceedsMaxDebt(newDebt, max);

```
## Flaw:
The check uses > instead of >=, allowing a borrower to reach exactly the maximum debt limit. This leaves no room for fluctuations.

Liquidation Check Uses the Same Maximum Debt:

```solidity
if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
    revert IPaprController.NotLiquidatable();
}

```

Because the same calculation is used, if a borrower’s debt is exactly at the maximum, the protocol deems them liquidatable immediately. There is no buffer (or gap) between the borrowing limit and the liquidation trigger.

I want study this audit report break down every single thing explain it to a non technical person and also use real world examples with figures for illustration


Heuristics Model for detecting  and 

illustrate how this liquidation would work with real numbers, and highlight the key codes with flaws
