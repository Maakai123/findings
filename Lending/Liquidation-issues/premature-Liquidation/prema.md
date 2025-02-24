
```solidity 

function getPositionRisk(uint256 positionId)
    public
    view
    returns (uint256 risk)
{
    Position storage pos = positions[positionId];
    uint256 pv = getPositionValue(positionId);
    uint256 ov = getDebtValue(positionId);
    uint256 cv = oracle.getUnderlyingValue(
        pos.underlyingToken,
        pos.underlyingAmount   // <-- THIS IS THE PROBLEM!
    );
    if (cv == 0) risk = 0;
    else if (pv >= ov) risk = 0;
    else {
        risk = ((ov - pv) * DENOMINATOR) / cv;
    }
}


function lend(address token, uint256 amount)
    external
    override
    // ...
{
    // ...
    pos.underlyingAmount += amount;  // <-- Only updated on new deposits
    // ...
}

function getUnderlyingValue(address token, uint256 amount)
    external
    view
    returns (uint256 collateralValue)
{
    uint256 decimals = IERC20MetadataUpgradeable(token).decimals();
    collateralValue = (_getPrice(token) * amount) / 10**decimals;
}
```

## Explanation

When a user deposits assets (like tokens) as collateral, they may earn interest on that deposit over time. However, the system only tracks the original deposit amount (the "underlyingAmount") and does not update it to include any interest earned. As a result, when the protocol calculates the “value” of the collateral, it underestimates how much the collateral is really worth. This underestimation can make a healthy (or even overcollateralized) position look risky, causing the user to be liquidated (have their collateral sold) prematurely.

More context 
Imagine you deposit $1,000 in a savings account that earns 5% interest per year. After one year, your account has $1,050. But what if the bank still thinks you only have $1,000 when deciding if you've fallen behind on your credit card payments?
This lending protocol has exactly this problem - it ignores the interest you've earned when calculating if your position is at risk.




## Lending Process – Depositing Collateral

Function: lend()
What Happens:
When a user deposits tokens as collateral, the system records:
The token type (e.g., an ERC20 token)
The underlyingAmount – which is simply increased by the amount deposited.

```solidity
 pos.underlyingAmount += amount;

```
The code only adds the new deposit amount. It never increases underlyingAmount when interest is earned on the deposited tokens.

## Calculating Collateral Value
Function: getUnderlyingValue()

What Happens:
This function computes the collateral value by taking the underlying token’s price (from an Oracle) and multiplying it by the underlyingAmount (the total deposited amount).

```solidity
collateralValue = (_getPrice(token) * amount) / 10**decimals;

```
Flaw:
Since amount is just the sum of deposits (without interest), the computed collateral value is lower than it should be if interest was considered.


## Assessing Risk and Liquidation
Function: getPositionRisk()
What Happens:
The protocol calculates a “risk” metric using:
The value of the collateral (cv), which is derived from underlyingAmount (and thus is undervalued).
The borrower's debt and other factors.
The liquidation condition is based on:

``` solidity 
((borrowsValue - collateralValue) / underlyingValue) >= underlyingLiqThreshold

```
Because the collateral’s value is understated (missing accrued interest), even a well-collateralized user might appear to have a high risk level and be liquidated too early.

## Technical 

The problem is with pos.underlyingAmount. This value is only set when you first deposit funds, and it's never updated to include earned interest:
Initial Setup

User deposits 1,000 USDC
Interest rate: 10% per year
Liquidation threshold: 80%
User borrows: 800 USDC

After One Year

Actual underlying value: 1,200 USDC (original 1,000 + 200 interest)
Recorded underlying value: Still 1,000 USDC (never updated!)
Debt: 800 USDC, borrowed amount
Collateral: 1,200 USDC

Risk Calculation

Correct risk: (800 - 1,200) / 1,200 = -0.33 (negative risk, very safe!)

Since the protocol only sees the deposit amount ($1,000) and ignores the $200 interest:

Protocol's flawed risk: (800 - 1,000) / 1,000 = -0.2 => 20%



Imagine a minor market drop causes the token’s price to dip slightly. The system now sees the collateral value as, say, $950 instead of $1,000 (and still ignoring interest). Now the risk might cross the liquidation threshold—even though the true value (with interest) is still strong.


When `lend()` is called, it records the initial deposit amount in `pos.underlyingAmount`
The interest earned is reflected in pos.underlyingVaultShare, but this value isn't used for risk calculations
`getPositionRisk()` calls oracle.getUnderlyingValue() with the original amount
`getUnderlyingValue()` calculates value based on the token price and amount, ignoring earned interest


Heuristic Questions for Future Audits
Interest Adjustment:
"Does the protocol adjust the collateral’s recorded value to reflect interest accrued, or does it only use the initial deposit amount?"
Collateral Valuation:
"In functions that calculate collateral value (e.g., getUnderlyingValue()), is the current value computed based on dynamic parameters (including interest), or is it static?"
Risk Calculation:
"Does the risk calculation in getPositionRisk() accurately reflect the true current value of the collateral, including any interest earned?"
Edge Case Testing:
"Have we simulated scenarios where interest accrues significantly over time? Does the system correctly account for that increased value in its liquidation conditions?"
Consistency Across Functions:
"Are all functions that rely on the collateral’s value (borrowing, risk assessment, liquidation) using a consistent and up-to-date valuation method?"
