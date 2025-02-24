```solidity
function repay(address token, uint256 amountCall)
    external
    override
    inExec
    poke(token)
    onlyWhitelistedToken(token)
{
    if (!isRepayAllowed()) revert REPAY_NOT_ALLOWED();
    (uint256 amount, uint256 share) = repayInternal(
        POSITION_ID,
        token,
        amountCall
    );
    emit Repay(POSITION_ID, msg.sender, token, amount, share);
}

function liquidate(
    uint256 positionId,
    address debtToken,
    uint256 amountCall
) external override lock poke(debtToken) {
    if (amountCall == 0) revert ZERO_AMOUNT();
    if (!isLiquidatable(positionId)) revert NOT_LIQUIDATABLE(positionId);
    Position storage pos = positions[positionId];
    Bank memory bank = banks[pos.underlyingToken];
    if (pos.collToken == address(0)) revert BAD_COLLATERAL(positionId);

    uint256 oldShare = pos.debtShareOf[debtToken];
    (uint256 amountPaid, uint256 share) = repayInternal(
        positionId,
        debtToken,
        amountCall
    );

    uint256 liqSize = (pos.collateralSize * share) / oldShare;
    uint256 uTokenSize = (pos.underlyingAmount * share) / oldShare;
    uint256 uVaultShare = (pos.underlyingVaultShare * share) / oldShare;

    pos.collateralSize -= liqSize;
    pos.underlyingAmount -= uTokenSize;
    pos.underlyingVaultShare -= uVaultShare;

    // Transfer position (Wrapped LP Tokens) to liquidator
    IERC1155Upgradeable(pos.collToken).safeTransferFrom(
        address(this),
        msg.sender,
        pos.collId,
        liqSize,
        ""
    );
    // Transfer underlying collaterals (vault share tokens) to liquidator
    if (
        address(ISoftVault(bank.softVault).uToken()) == pos.underlyingToken
    ) {
        IERC20Upgradeable(bank.softVault).safeTransfer(
            msg.sender,
            uVaultShare
        );
    } else {
        IERC1155Upgradeable(bank.hardVault).safeTransferFrom(
            address(this),
            msg.sender,
            uint256(uint160(pos.underlyingToken)),
            uVaultShare,
            ""
        );
    }

    emit Liquidate(
        positionId,
        msg.sender,
        debtToken,
        amountPaid,
        share,
        liqSize,
        uTokenSize
    );
}

```

# EXPLANATION
Repayment: Borrowers repay their debt to avoid losing collateral.

Liquidation: When a borrower’s position is in trouble (for example, if the value of their collateral drops or they become undercollateralized), a liquidator (an external actor) can step in, repay part of the borrower’s debt, and seize some of their collateral.

`repay()`
This function lets a borrower repay some of their debt by transferring tokens back to the bank.

#

`liquidate()`
# there is no check to see if repayments are allowed.
`This means that even if the system has disabled repayments (closing the “counter”), the liquidate function will still proceed.`


`isRepayAllowed()`
This function checks a status flag (bankStatus) to see if repayments are currently allowed.

```solidity
function isRepayAllowed() public view returns (bool) {
    return (bankStatus & 0x02) > 0;
}
```

# FLAWS

`The system is designed so that repayments can be paused by the contract owner (using the status flag checked by isRepayAllowed()). This is supposed to be a safety or administrative feature. However, while the repay() function respects this flag (it won’t allow repayments if the flag isn’t set), the liquidate() function does not check whether repayments are allowed.`

```solidity 
 if (!isRepayAllowed()) revert REPAY_NOT_ALLOWED();

```

# Why Is This a Problem?

`If repayments are paused, borrowers lose the ability to reduce their debt. Yet, because liquidations remain enabled, a liquidator can still force the liquidation of a borrower’s position. This can force borrowers to lose their collateral without any chance to remedy their situation by repaying—even if market conditions might improve shortly.`


# Scenario 1: Normal Operations
Loan amount (Debt): $1000
Collateral : $1500

You decide to make a payment of $250.

# Scenario 2: Repayments Disabled
The bank (or contract owner) disables repayments by turning off the repayment flag 
You can no longer repay your $1,000 loan.

However, because the liquidate() function doesn’t check the repayment flag, a liquidator can step in.


# Recommendations and Conclusion
Option 1: Disallow liquidations when repayments are paused. This would ensure that borrowers are not punished (via forced liquidation) if they cannot repay due to an administrative pause.
Option 2: Never disable repayments, so that borrowers can always try to keep their positions healthy.








