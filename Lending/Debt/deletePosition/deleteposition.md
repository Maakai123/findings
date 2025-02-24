
```solidity 
function _deleteLienPosition(uint256 collateralId, uint256 position) public {
  uint256[] storage stack = liens[collateralId];
  require(position < stack.length, "index out of bounds");

  emit RemoveLien(
    stack[position],
    lienData[stack[position]].collateralId,
    lienData[stack[position]].position
  );
  for (uint256 i = position; i < stack.length - 1; i++) {
    stack[i] = stack[i + 1];
  }
  stack.pop();
}
```

Summary
Problem: The _deleteLienPosition function is public and lacks permission checks, allowing anyone to remove liens.

Impact: An attacker can remove lien records, leading to unauthorized collateral withdrawal and potential financial loss for lenders.

The function _deleteLienPosition is meant to remove a lien (a record of a lender’s security interest) from a collateral’s list. However, because it’s marked as public and has no check on who can call it, anyone can remove any lien—even if they shouldn’t be allowed to.

Why Is This a Problem?
If an attacker can remove a lien, they could bypass protections that keep a lender’s claim on collateral intact. This would let users withdraw collateral without repaying their debt, potentially “rugging” the lenders out of their security.


# Key Issue: 
The function is public. This means there’s no restriction on who can call it.

```solidity
function _deleteLienPosition(uint256 collateralId, uint256 position) public {
```

# fix
Fix the Issue:

Change the function’s visibility from public to internal or add strict access controls.
This ensures only authorized accounts (for example, the contract owner or a specific administrative role) can remove liens.







