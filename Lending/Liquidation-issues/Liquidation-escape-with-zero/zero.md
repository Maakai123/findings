``` solidity
 function decreaseLiquidity(DecreaseLiquidityParams calldata params)
        external
        payable
        override
        isAuthorizedForToken(params.tokenId)
        checkDeadline(params.deadline)
        returns (uint256 amount0, uint256 amount1)
    {   
        NOTE: Issues ?
       //This check ensures that the function is only called with a positive liquidity amount.
        require(params.liquidity > 0);

        Position storage position = _positions[params.tokenId];

        uint128 positionLiquidity = position.liquidity;
        require(positionLiquidity >= params.liquidity);

        PoolAddress.PoolKey memory poolKey = _poolIdToPoolKey[position.poolId];
        IUniswapV3Pool pool = IUniswapV3Pool(PoolAddress.computeAddress(factory, poolKey));
        (amount0, amount1) = pool.burn(position.tickLower, position.tickUpper, params.liquidity);

        require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');

        bytes32 positionKey = PositionKey.compute(address(this), position.tickLower, position.tickUpper);
        // this is now updated to the current transaction
        (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);

        position.tokensOwed0 +=
            uint128(amount0) +
            uint128(
                FullMath.mulDiv(
                    feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                    positionLiquidity,
                    FixedPoint128.Q128
                )
            );
        position.tokensOwed1 +=
            uint128(amount1) +
            uint128(
                FullMath.mulDiv(
                    feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                    positionLiquidity,
                    FixedPoint128.Q128
                )
            );

        position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
        position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;
        // subtraction is safe because we checked positionLiquidity is gte params.liquidity
        position.liquidity = positionLiquidity - params.liquidity;

        emit DecreaseLiquidity(params.tokenId, params.liquidity, amount0, amount1);
    }
```
## EXPLANATION

The Vulnerability:
A borrower can deposit a Uniswap V3 position with 0 liquidity, which causes the liquidation process to fail because decreaseLiquidity() will revert on a zero liquidity check.
Real-World Impact:
The borrower escapes liquidation even when undercollateralized, and the protocol is left with bad debt.


In this protocol, borrowers must deposit collateral (like NFTs or token positions) to secure loans

When a borrower becomes undercollateralized (their loan is at risk), a liquidator will call a function (e.g., liquidateAccount()) to seize the collateral.

Mitigattion
Before or during the process of removing liquidity, the system should check that the Uniswap V3 position actually has some liquidity—that is, the liquidity value is greater than 0. If it isn’t, the process should revert (i.e., stop and throw an error).

This line ensures that if someone tries to remove liquidity from a position with 0 liquidity, the function won’t proceed—it will throw an error and stop execution.

```require(params.liquidity > 0, "No liquidity available to remove");
```

 Vulnerability

A malicious borrower can deposit a Uniswap V3 position that has 0 liquidity (i.e., it exists but holds no real value).

When liquidation is attempted, the liquidator’s process calls decreaseLiquidity() with a liquidity value of 0.

The check require(params.liquidity > 0); fails (i.e., the function reverts), and the liquidation cannot proceed.


Normal Case:

Loan Details:
Borrower takes a loan of $1,000 by depositing collateral worth $1,500.
Uniswap V3 Position as Collateral:
Normally, this position might have liquidity = 100 (imagine it’s worth $500).
Liquidation Scenario:
If the collateral value drops (say to $800), a liquidator would try to remove some of the Uniswap liquidity to cover the shortfall. With liquidity > 0, the process works as expected.


## Real-World Analogy
Imagine you’re at a bank and you use a savings account as collateral for a loan.

Normal Situation:
Your savings account has a balance of $500. If you default on the loan, the bank can access this $500 to help cover the loss.
Problematic Scenario:
Now imagine someone tries to use an account that has a $0 balance as collateral.
What Happens:
If the bank later needs to seize the collateral (because you didn’t repay your loan), it finds that there’s nothing in the account—it can’t recover any funds.
