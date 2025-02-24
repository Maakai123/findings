Imagine you run a car valuation service that tells you the current value of used cars. To keep prices accurate, you rely on trusted appraisers (feeders) who periodically update each car’s value.

If the appraisers stop reporting, your system’s information becomes outdated, and people can’t make accurate decisions (like selling or buying a car).
In the smart contract, feeders update the NFT price. If the price isn’t updated often enough (i.e., becomes “stale”), the system won’t use it.

## NOTE: How Liquidation Uses the Price Oracle

In this protocol, if a user’s loan is at risk, their NFT collateral can be liquidated. The liquidation process needs a current price of the NFT to work correctly.

The function getPrice() is used to fetch this price.
However, it checks that the price is fresh by ensuring the price was updated within a certain number of blocks (or time).


# Check for freshness  
```solidity

function getPrice(address _asset)
    external
    view
    override
    returns (uint256 price)
{
    uint256 updatedAt = assetPriceMap[_asset].updatedAt;
    require(
        (block.number - updatedAt) <= config.expirationPeriod,
        "NFTOracle: asset price expired"
    );
    return assetPriceMap[_asset].twap;
}


```

## 3. The Vulnerability: Unrestricted Removal of Feeders
Only the owner (or an admin) should be able to remove a feeder, ensuring that trusted sources continue updating the price.
What’s Actually Happening:
The function removeFeeder() is supposed to let the owner remove a feeder.
However, due to a flaw in the access control, anyone can call it—even if they are not the owner.

```solidity
function removeFeeder(address _feeder)
    external
    onlyWhenFeederExisted(_feeder)
{
    _removeFeeder(_feeder);
}

Note:

function _removeFeeder(address _feeder)
    internal
    onlyWhenFeederExisted(_feeder)
{
    // Removes the feeder from the list...
    revokeRole(UPDATER_ROLE, _feeder);
    emit FeederRemoved(_feeder);
}


```
Notice that it uses the modifier onlyWhenFeederExisted(_feeder). This modifier simply checks that the feeder exists; it does not restrict who can call the function.

The feeder’s role (UPDATER_ROLE) is revoked, which means that this feeder can no longer update NFT prices.


## Why Is This a Problem?

Liquidation relies on an up-to-date NFT price.
If the price is stale (because no feeder is updating it), the getPrice() function reverts with an error ("NFTOracle: asset price expired").


# Normal Scenario:
A feeder updates the NFT price at block 1000.
The expiration period is 1,800 blocks.
At block 2000, the price is still fresh (2000 - 1000 = 1000 < 1800), so liquidation can proceed if needed.

# Attack Scenario

A malicious user calls removeFeeder() and removes the only feeder.
No one updates the price. The last update was at block 1000.
At block 3000, when liquidation is attempted, the difference is 3000 - 1000 = 2000 blocks, which exceeds the 1,800-block expiration period.
The getPrice() function then reverts, blocking any liquidation process.

A user near liquidation risk can remove the feeder(s) so that the NFT price becomes stale.
As a result, any attempt to liquidate their collateral fails, and their risky position remains open.

## The Recommended Fix

The report recommends adding an access control modifier to restrict who can call the removeFeeder() function. Instead of using onlyWhenFeederExisted, which merely checks for the feeder’s existence, the function should also require that the caller has the DEFAULT_ADMIN_ROLE.

```solidity
function removeFeeder(address _feeder)
    external
    onlyRole(DEFAULT_ADMIN_ROLE) // Only admins can remove feeders
    onlyWhenFeederExisted(_feeder)
{
    _removeFeeder(_feeder);
}

```
Feeders are essential: They keep NFT prices updated so the protocol knows when to liquidate a position.
If the price data becomes stale (because feeders were removed), the price query reverts.
This prevents liquidations, allowing risky positions to linger and potentially