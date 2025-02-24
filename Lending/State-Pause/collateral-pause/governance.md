
# Issue: Pausing a collateral type sets a flag that blocks both closing and liquidating loans.
Impact: Users can’t recover their collateral and the protocol may be left with unresolved, bad debt

```solidity
function pauseCollateralType(
    address _collateralAddress,
    bytes32 _currencyKey
    ) external collateralExists(_collateralAddress) onlyAdmin {
    require(_collateralAddress != address(0)); //this should get caught by the collateralExists check but just to be careful
    //checks two inputs to help prevent input mistakes
    require( _currencyKey == collateralProps[_collateralAddress].currencyKey, "Mismatched data");
    collateralValid[_collateralAddress] = false;
    collateralPaused[_collateralAddress] = true;
}
```

When a governance admin “pauses” a collateral type (for example, a token used as collateral), the system sets a flag (collateralValid) to false. This pause is intended to prevent new actions on that collateral type. However, this design flaw has two major consequences:

User Lock-In:
Users with existing loans that use this collateral can no longer close their loans to recover their collateral.

Bad Debt Accumulation:
Because liquidations (which normally force the closure of risky positions) are also blocked, the protocol may be left with many loans that can never be closed or liquidated—leading to “bad debt.”

```solidity 
function pauseCollateralType(
    address _collateralAddress,
    bytes32 _currencyKey
) external collateralExists(_collateralAddress) onlyAdmin {
    require(_collateralAddress != address(0)); // safeguard: address should not be zero
    // Ensure that the currency key matches the collateral's expected currency
    require(_currencyKey == collateralProps[_collateralAddress].currencyKey, "Mismatched data");
    
    // Mark this collateral as invalid and paused
    collateralValid[_collateralAddress] = false;
    collateralPaused[_collateralAddress] = true;
}
```

```solidity
collateralValid[_collateralAddress] = false;
collateralPaused[_collateralAddress] = true;
```
These lines mark the collateral as “paused,” which later prevents any loan closure or liquidation operations that use this collateral.

Loan Closure Blocked:
Functions such as closeLoan will check collateralValid and, finding it false, will revert (fail) the operation.

# Imagine Bob has a car loan where his car is the collateral. If the bank suddenly “pauses” the car as acceptable collateral, Bob cannot repay his loan to reclaim his car—even if he has saved enough money to do so.

Liquidation Blocked:
Similarly, the liquidation process (normally used to force-close risky positions) will also be blocked.
Real-World Example:
Loan Amount: $10,000
Collateral (Car) Value: $15,000
Situation: Bob takes a loan using his car. Later, due to market conditions or a governance decision, the car collateral is paused.
Outcome:
Bob cannot close his loan to recover his car because the system blocks the action.
If the car value drops further, Bob’s position becomes underwater, but liquidation is blocked—leaving the bank with an outstanding $10,000 loan that can never be resolved.

Borrowers are unable to reclaim their collateral or repay their loans, effectively trapping their funds.

Protocol Risk:
The protocol may accumulate “bad debt” because loans that cannot be closed or liquidated linger indefinitely, potentially harming the overall financial health of the platform.


# Recommendation
Allow liquidations and loan closures even when a collateral is paused.

Why?
This change ensures that existing loans can be resolved—either by repaying or by liquidating the collateral—preventing users from being trapped and reducing the risk of bad debt.


