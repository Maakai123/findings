
```solidity
function liquidate(uint256 positionId, address debtToken, uint256 amountCall) 
    external override lock poke(debtToken) {
    if (!isRepayAllowed()) revert Errors.REPAY_NOT_ALLOWED();

function repay(address token, uint256 amountCall) 
    external override inExec poke(token) onlyWhitelistedToken(token) {
    if (!isRepayAllowed()) revert Errors.REPAY_NOT_ALLOWED();
```
`Governance can disallow tokens for new loans, but if the same restriction applies to existing loans, borrowers may be prevented from repaying. Yet, liquidation can still occur—trapping borrowers in an open position and risking forced loss of collateral.`

# Recommendation:
Governance should only affect new loans. Existing loans using previously allowed tokens must continue to be repaid and liquidated normally to prevent a critical loss of funds


# Borrower’s Position:
When a borrower takes a loan, they use a specific token (say, TokenX) as collateral and/or for repayment.

# Allowed vs. Disallowed Tokens:
Initially, TokenX is allowed. But later, governance may decide TokenX is no longer acceptable for new loans.

# Key Issue:
If the system disallows usdc for new loans—and mistakenly applies that restriction to existing loans—then borrowers cannot use usdc to repay their loans.

# Flaw in repay() Function:
```solidity
function repay(address token, uint256 amountCall)
    external override inExec poke(token) onlyWhitelistedToken(token) {
    if (!isRepayAllowed()) revert Errors.REPAY_NOT_ALLOWED();
    // ... (rest of the repayment logic)
}
```


Critical Check:
The use of onlyWhitelistedToken(token) prevents repayment with a token that is no longer approved.
Problem:
This check should not apply to existing loans; it should only affect new loans.

# law in liquidate() Function:

```solidity
function liquidate(uint256 positionId, address debtToken, uint256 amountCall)
    external override lock poke(debtToken) {
    if (!isRepayAllowed()) revert Errors.REPAY_NOT_ALLOWED();
    // ... (rest of the liquidation logic)
}

```
Missing Check:
There is no onlyWhitelistedToken() modifier in the liquidate() function.
Problem:
This inconsistency means that while repayments are blocked for a disallowed token, liquidations can still occur. This imbalance creates a risk for borrowers.

#  Borrower “Alice” Takes a Loan:

Loan Details:
Collateral/Repayment Token: TokenX
Loan Amount: $10,000
Collateral Value: $15,000
Governance Change:
Later, the platform’s governance disallows TokenX for new loans.

Impact on Alice:

Repayment Blocked:
Because of the onlyWhitelistedToken() modifier in the repay() function, Alice can no longer use TokenX to repay her $10,000 loan.

Liquidation Still Possible:
However, if market conditions worsen (say, her collateral drops in value), the system still allows liquidation via the liquidate() function.






