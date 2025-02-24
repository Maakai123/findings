

```solidity 

     function _checkIfCollateralIsActive(bytes32 _currencyKey) internal view override {
            
             //Lyra LP tokens use their associated LiquidityPool to check if they're active
             ILiquidityPoolAvalon LiquidityPool = ILiquidityPoolAvalon(collateralBook.liquidityPoolOf(_currencyKey));
             bool isStale;
             uint circuitBreakerExpiry;
             //ignore first output as this is the token price and not needed yet.
             (, isStale, circuitBreakerExpiry) = LiquidityPool.getTokenPriceWithCheck();
             require( !(isStale), "Global Cache Stale, can't trade");
             require(circuitBreakerExpiry < block.timestamp, "Lyra Circuit Breakers active, can't trade");
    }
    

```

# Issue:
The system freezes important actions (closing or adding collateral) whenever the price is deemed stale or the circuit breaker is active. This could trap users in a harmful situation where their collateral is locked, interest accrues continuously, and they might be unfairly liquidated.


# The Problem:
When the system detects that the price is “stale” (old or unreliable) or that a “circuit breaker” (a safety mechanism) is tripped, it stops users from either closing their loan or adding more collateral to their Lyra vault positions. This may sound protective, but it can actually trap users in a bad situation:

# Frozen Collateral:
Your collateral (the asset you’ve put up to secure a loan) is frozen indefinitely.

# Interest Keeps Accumulating:
Even though you can’t take any action, interest on your loan keeps adding up.

# Unfair Liquidation Risk:
If you can’t add extra collateral, your position might eventually become underwater. And since liquidation (forcing the position to close) still works, you might end up being liquidated

isStale Check: If the price is outdated, it stops any action.
Circuit Breaker Check: If the safety mechanism is active (meaning market conditions might be too volatile or manipulated), it stops actions.

# EXAMPLE 

Normal Situation:
You decide to add $1,000 in extra collateral to secure your position further.

Outcome: Your risk of losing the car is reduced.
Stale Price / Circuit Breaker Activated:
Suddenly, the system flags the price as stale or trips the circuit breaker.

Impact:
You cannot add that $1,000.
You also can’t close out your loan even if you have the money.

# Key Code Flaws

The function _checkIfCollateralIsActive is applied even when a user is just closing out their loan or adding extra collateral, which in those cases doesn’t require a live price feed.

# Recommended Fix in closeLoan:
Only perform the price and circuit breaker check if the user still has outstanding debt. For example, after calculating the remaining debt:
This means if you’re closing the loan completely, you wouldn’t be blocked by a stale price

```solidity
if(outstandingisoUSD >= TENTH_OF_CENT){ // if significant debt remains
    // Only now check if the collateral is active
    _checkIfCollateralIsActive(currencyKey);
    // Continue with collateral and margin checks...
}
```

# Flaw 2: Unnecessary Liquidation Threshold Check on Adding Collateral

Problem:
In the function that allows users to add collateral, the contract checks if the total collateral meets a liquidation threshold. This check is unnecessary when you’re only adding collateral because doing so cannot harm your collateralization ratio.

Recommended Fix in increaseCollateralAmount:
Remove the check that compares total collateral in USD to the borrowed amount:

```solidity

// Remove these lines:
// uint256 totalCollat = collateralPosted[_collateralAddress][msg.sender] + _colAmount;
// uint256 colInUSD = priceCollateralToUSD(currencyKey, totalCollat);
// uint256 USDborrowed = (isoUSDLoanAndInterest[_collateralAddress][msg.sender] * virtualPrice) / LOAN_SCALE;
// uint256 borrowMargin = (USDborrowed * liquidatableMargin) / LOAN_SCALE;
// require(colInUSD >= borrowMargin, "Liquidation margin not met!");
```
This allows users to add collateral even when the price data might be stale, thereby improving their safety.








