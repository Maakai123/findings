
```solidity
// load open markets for account being liquidated
ctx.amountOfOpenPositions = tradingAccount.activeMarketsIds.length();

// iterate through open markets
for (uint256 j = 0; j < ctx.amountOfOpenPositions; j++) {
    // load current active market id into working data
    // @audit assumes constant ordering of active markets
    ctx.marketId = tradingAccount.activeMarketsIds.at(j).toUint128();

    PerpMarket.Data storage perpMarket = PerpMarket.load(ctx.marketId);
    Position.Data storage position = Position.load(ctx.tradingAccountId, ctx.marketId);

    ctx.oldPositionSizeX18 = sd59x18(position.size);
    ctx.liquidationSizeX18 = unary(ctx.oldPositionSizeX18);

    ctx.markPriceX18 = perpMarket.getMarkPrice(ctx.liquidationSizeX18, perpMarket.getIndexPrice());

    ctx.fundingRateX18 = perpMarket.getCurrentFundingRate();
    ctx.fundingFeePerUnitX18 = perpMarket.getNextFundingFeePerUnit(ctx.fundingRateX18, ctx.markPriceX18);

    perpMarket.updateFunding(ctx.fundingRateX18, ctx.fundingFeePerUnitX18);
    position.clear();

    // @audit this calls `EnumerableSet::remove` which changes the order of `activeMarketIds`
    tradingAccount.updateActiveMarkets(ctx.marketId, ctx.oldPositionSizeX18, SD_ZERO);
```
# EXPLANATION

When a user's account is liquidated, the system must go through all the markets where that user has open positions (active markets) and close them. The code does this by looping through an array (or set) of active market IDs.

Key Assumption in the Code:
The code assumes that the order of the active market IDs stays the same while it loops through them.


# How the Liquidation Loop Works

```solidity
// Get the number of open markets for the account
ctx.amountOfOpenPositions = tradingAccount.activeMarketsIds.length();

// Loop through each market
for (uint256 j = 0; j < ctx.amountOfOpenPositions; j++) {
    // Assume the market order is constant:
    ctx.marketId = tradingAccount.activeMarketsIds.at(j).toUint128();

    // Do various operations for liquidation…
    // Then remove the active market from the set:
    tradingAccount.updateActiveMarkets(ctx.marketId, ctx.oldPositionSizeX18, SD_ZERO);
}
```

## What’s Happening?
The function retrieves all active market IDs.
It then loops over each market to process its liquidation.
During each loop, it removes the current market from the active list.

#  The Problem: Changing Order with "Swap-and-Pop"

The active market IDs are stored in an EnumerableSet. This data structure does not guarantee that the order of items remains constant. When you remove an element, it uses a “swap-and-pop” technique:

Swap: The element to remove is swapped with the last element.
Pop: The last element is removed.


## EXAMPLE

nitial List: [Task A, Task B, Task C]
What You Expect: If you remove Task A, you expect the list to simply become [Task B, Task C] in that order.
Swap-and-Pop Method: Instead, you take the last sticky note (Task C), stick it in the place of Task A, and then remove the last note. Now the list becomes [Task C, Task B].
The order has changed!
When the code later tries to access items by their original positions, the order is no longer what it expected. This can lead to accessing an index that doesn’t exist (an “array out-of-bounds” error) and causes the transaction to revert (fail).

## Impact
A user with positions in multiple markets can make it impossible to liquidate their account because the loop fails after the first removal.

# MITIGATIONS 

Preserve the Order in Storage:
Use a data structure that maintains a constant order (such as an array) instead of an EnumerableSet.

Remove After the Loop:
Instead of removing each market inside the loop, iterate over all markets first and then remove them afterward. This ensures that the loop order doesn’t change mid-iteration.



