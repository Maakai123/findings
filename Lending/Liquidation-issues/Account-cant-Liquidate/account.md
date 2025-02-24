
``` solidity
 function _calcLiquidation(
        uint256 _accountCollateral,
        uint256 _accountDebt,
        uint256 _debtToLiquidate
    ) internal view returns (uint256 collateralToLiquidate, uint256 liquidationSurcharge) {
        (uint256 price, uint8 decimals) = _getCollPrice();
        uint256 _collRatio = TauMath._computeCR(_accountCollateral, _accountDebt, price, decimals);

        uint256 totalLiquidationDiscount = _calcLiquidationDiscount(_collRatio);

        uint256 collateralToLiquidateWithoutDiscount = (_debtToLiquidate * (10 ** decimals)) / price;
        collateralToLiquidate = (collateralToLiquidateWithoutDiscount * totalLiquidationDiscount) / Constants.PRECISION;
        if (collateralToLiquidate > _accountCollateral) {
            collateralToLiquidate = _accountCollateral;
        }

        // Revert if requested liquidation amount is greater than allowed
        if (
            _debtToLiquidate >
            _getMaxLiquidation(_accountCollateral, _accountDebt, price, decimals, totalLiquidationDiscount)
        ) revert wrongLiquidationAmount();

        return (
            collateralToLiquidate,
            (collateralToLiquidateWithoutDiscount * LIQUIDATION_SURCHARGE) / Constants.PRECISION
        );
    }
```

## EXPLANATION

When prices are normal, the function calculates the right amount of collateral to liquidate and a reasonable surcharge.

Imagine you borrow money using ETH as collateral. If ETH’s value crashes by 99%, the system should automatically liquidate your ETH to repay the debt. But due to a coding flaw, the system miscalculates fees during extreme crashes, causing it to freeze. This is like a bank trying to charge a fee larger than your collateral, which is impossible.

 This function, _calcLiquidation, is used to determine:

How much collateral should be liquidated
How much extra fee (called a liquidation surcharge) should be applied to the liquidator for doing the work

_accountCollateral: The total collateral the borrower has.
_accountDebt: The total debt the borrower owes.
_debtToLiquidate: The portion of the debt that’s being liquidated.






## Compute Raw Collateral Required (Without Discount)
```solidity 
uint256 collateralToLiquidateWithoutDiscount = (_debtToLiquidate * (10 ** decimals)) / price;

```

debtToLiquidate = 1 ETH
Price of ETH = $3000 (with 2 decimals, so think of 300000 when working with cents)
For simplicity, assume decimals = 2.
Calculation:


=> `collateralToLiquidateWithoutDiscount= 1*10^2 = 100 / 3000   ≈0.0333 ETH`

​This means, under normal conditions, liquidating 1 ETH of debt would require roughly 0.0333 ETH of collateral 

# Step 4: Apply the Liquidation Discount

```solidity 
collateralToLiquidate = (collateralToLiquidateWithoutDiscount * totalLiquidationDiscount) / Constants.PRECISION;
```
If the discount factor (totalLiquidationDiscount) is 0.9 (meaning a 10% discount), then:
collateralToLiquidate≈0.0333×0.9≈0.03 ETH
Cap Check:
If the calculated collateral exceeds the borrower’s total ETH collateral, it’s capped at the total amount.

## Step 5: Calculate the Liquidation Surcharge
```solidity
uint256 liquidationSurcharge = (collateralToLiquidateWithoutDiscount * LIQUIDATION_SURCHARGE) / Constants.PRECISION;
```
Normal Price: If LIQUIDATION_SURCHARGE is 10% (0.10 scaled appropriately), 
liquidationSurcharge≈0.0333×0.10≈0.00333 ETH


# Step 6: What Happens When ETH Price Crashes by 99%

Now, consider the edge case where ETH’s price falls by 99% (from $3000 to $30).

New Price: $30

Calculation of Raw Collateral Needed:


`collateralToLiquidateWithoutDiscount = 1*10^2 / 30 = 100/30 = 3.33 ETH`

With a price crash, the math now suggests you’d need 3.33 ETH to cover 1 ETH of debt if computed in this raw way.

Cap Check:
Suppose the borrower only has 2 ETH as collateral. The function will cap the collateralToLiquidate at 2 ETH:


```solidity
if (collateralToLiquidate > _accountCollateral) {
    collateralToLiquidate = _accountCollateral;
}

```
Problem with Surcharge Calculation:
However, the liquidation surcharge is still calculated on the raw value (3.33 ETH):

liquidationSurcharge=3.33×0.10≈0.333 ETH
Then, later in the process, the protocol might try to compute:

collateralToLiquidator=collateralToLiquidate−liquidationSurcharge=2−0.333=1.667 ETH

``` solidity
uint256 collateralToLiquidateWithoutDiscount = (_debtToLiquidate * (10 ** decimals)) / price;


```
# Issue:
 With a very low price (e.g., $30 instead of $3000), this value becomes huge (e.g., 3.33 ETH instead of 0.0333 ETH).

```solidity 
 uint256 liquidationSurcharge = (collateralToLiquidateWithoutDiscount * LIQUIDATION_SURCHARGE) / Constants.PRECISION;

```
# Issue: 
The surcharge is based on the raw (inflated) collateral, not the capped collateral. If the raw value is too high, the surcharge can exceed the actual collateral available, causing an underflow when subtracting.

# Recommended Fix
Calculate the Surcharge on the Capped Collateral: Instead of using the raw collateral value, base the surcharge on the capped collateral:

```solidity 
uint256 liquidationSurcharge = (collateralToLiquidate * LIQUIDATION_SURCHARGE) / Constants.PRECISION;
```

2
Or, Use the Minimum Value: Calculate the surcharge on the minimum of the raw and capped collateral values:

```solidity
uint256 collateralToTakeSurchargeOn = Math.min(collateralToLiquidate, collateralToLiquidateWithoutDiscount);
uint256 liquidationSurcharge = (collateralToTakeSurchargeOn * LIQUIDATION_SURCHARGE) / Constants.PRECISION;

```