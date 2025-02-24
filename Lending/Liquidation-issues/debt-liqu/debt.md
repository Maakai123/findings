 ```solidity

 //This function lets a user borrow funds by increasing their debt
 function _increaseDebt(
        address account,
        ERC721 asset,
        address mintTo,
        uint256 amount,
        ReservoirOracleUnderwriter.OracleInfo memory oracleInfo
    ) internal {
        uint256 cachedTarget = updateTarget();
         
        uint256 newDebt = _vaultInfo[account][asset].debt + amount;
        uint256 oraclePrice =
            underwritePriceForCollateral(asset, ReservoirOracleUnderwriter.PriceKind.LOWER, oracleInfo);

        uint256 max = _maxDebt(_vaultInfo[account][asset].count * oraclePrice, cachedTarget);

     Note: Issues
       @> if (newDebt > max) revert IPaprController.ExceedsMaxDebt(newDebt, max);

        if (newDebt >= 1 << 200) revert IPaprController.DebtAmountExceedsUint200();

        _vaultInfo[account][asset].debt = uint200(newDebt);
        PaprToken(address(papr)).mint(mintTo, amount);

        emit IncreaseDebt(account, asset, amount);
    }


//This function starts an auction to liquidate (sell) the collateral
//if user's debt becomes too high
function startLiquidationAuction(
        address account,
        IPaprController.Collateral calldata collateral,
        ReservoirOracleUnderwriter.OracleInfo calldata oracleInfo
    ) external override returns (INFTEDA.Auction memory auction) {
        if (liquidationsLocked) {
            revert LiquidationsLocked();
        }

        uint256 cachedTarget = updateTarget();

        IPaprController.VaultInfo storage info = _vaultInfo[account][collateral.addr];

        // check collateral belongs to account
        if (collateralOwner[collateral.addr][collateral.id] != account) {
            revert IPaprController.InvalidCollateralAccountPair();
        }
        
        //fecth the current collateral value
        uint256 oraclePrice =
            underwritePriceForCollateral(collateral.addr, ReservoirOracleUnderwriter.PriceKind.TWAP, oracleInfo);
            Note: Issue 
        //Checks if the users current debt is at or above the maximum allowes debt
       @> if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
            revert IPaprController.NotLiquidatable();
        }
       
       
        if (block.timestamp - info.latestAuctionStartTime < liquidationAuctionMinSpacing) {
            revert IPaprController.MinAuctionSpacing();
        }

        info.latestAuctionStartTime = uint40(block.timestamp);
        info.count -= 1;

        emit RemoveCollateral(account, collateral.addr, collateral.id);

        delete collateralOwner[collateral.addr][collateral.id];

      //if conditions are met, it starts an auction to sell part of the collateral
        _startAuction(
            auction = Auction({
                nftOwner: account,
                auctionAssetID: collateral.id,
                auctionAssetContract: collateral.addr,
                perPeriodDecayPercentWad: perPeriodAuctionDecayWAD,
                secondsInPeriod: auctionDecayPeriod,
                // start price is frozen price * auctionStartPriceMultiplier,
                // converted to papr value at the current contract price
                startPrice: (oraclePrice * auctionStartPriceMultiplier) * FixedPointMathLib.WAD / cachedTarget,
                paymentAsset: papr
            })
        );
    }     Since there’s no gap between the maximal LTV and the liquidation LTV, user positions may be liquidated as soon as maximal debt is taken, without leaving room for collateral and Papr token prices fluctuations. Users have no chance to add more collateral or reduce debt before being liquidated. This may eventually create more uncovered and bad debt for the protocol.

```

## EXPLANATION
This protocol allows liquidation the instant debt reaches maximum allowed levels.
You take a $100,000 mortgage on a $125,000 house (80%LVT). 
The bank says if the home value drops even 1%, the house will be seized without any warning

Whats wrong
Day1: Borrow $100k against $125k house
Day 2: Home value drops to $124k, the house is seized

## Scenerio 
-Bored Ape Value: 100 ETH
-Maximum Borrowing: 80 ETH (80% LVT)
- You borrow: 80ETH

What if Bored Ape prices drop 1% ?

- New Ape Value: 99ETH
- Maximum allowed debt at new value: 80% of 99 = 79.2 ETH
- Your actual debt: 80 ETH.

my debt > new maximum =>  80 > 79.2 

RESULT: LiquidationsLocked

``` solidity
// Borrowing check
if (newDebt > max) revert IPaprController.ExceedsMaxDebt(newDebt, max);

// Liquidation check
if (info.debt < _maxDebt(oraclePrice * info.count, cachedTarget)) {
    revert IPaprController.NotLiquidatable();
}
```


## Heuristic Questions for Auditing Similar Protocols

- Liquidation Buffer Check:
"Does the protocol enforce a safety buffer (e.g., 5-10%) between max borrowable debt and liquidation threshold?"

- User Protection and Protocol Health:

"What is the potential impact on users if their positions can be liquidated immediately after borrowing the maximum amount?"
"How might this design contribute to increased uncovered or bad debt in the protocol? Are there safeguards to mitigate these risks?"

-User Remediation Window:
"Can borrowers add collateral/repay debt after hitting max LTV but before liquidation?"

- Maximal Debt Calculation Consistency:

"Are the calculations for maximum allowable debt and liquidation thresholds identical? Could this lead to a situation where borrowing the maximum immediately enables liquidation?"

- Collateral Value Fluctuation Handling:
How does the protocol account for normal market fluctuations in collateral value? Is there any hysteresis or safety margin to prevent immediate liquidation from a minor price drop?"
"What happens if the collateral’s price drops slightly below the maximum threshold? Is liquidation triggered immediately?"

- Immediate Liquidation Risk:

"Can a borrower become liquidatable in the same transaction or block in which they take on the maximum allowable debt?"
"Is there any mechanism (like a grace period) that gives users time to add collateral or repay part of their debt before liquidation is triggered?"


