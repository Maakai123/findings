
```solidity
 function priceLiquidity(uint256 _liquidity) external override view returns(uint256){
        uint256 token0Amount;
        uint256 token1Amount;
        (token0Amount, token1Amount) = viewQuoteRemoveLiquidity(_liquidity);
        //USDC route 
        uint256 value0;
        uint256 value1;
        if (token0 == USDC){
            //hardcode value of USDC at $1
            //check swap value of 100tokens to USDC to protect against flash loan attacks
            uint256 amountOut; //amount received by trade
            bool stablePool; //if the traded pool is stable or volatile.
            (amountOut, stablePool) = router.getAmountOut(HUNDRED_TOKENS, token1, USDC);
            require(stablePool == stable, "pricing occuring through wrong pool" );

            uint256 oraclePrice = getOraclePrice(priceFeed, tokenMaxPrice, tokenMinPrice);
            amountOut = (amountOut * oracleBase) / USDC_BASE / HUNDRED; //shift USDC amount to same scale as oracle

            //calculate acceptable deviations from oracle price
            uint256 lowerBound = (oraclePrice * (BASE - ALLOWED_DEVIATION)) / BASE;
            uint256 upperBound = (oraclePrice * (BASE + ALLOWED_DEVIATION)) / BASE;
            //because 1 USDC = $1 we can compare its amount directly to bounds
            require(lowerBound < amountOut, "Price shift low detected");
            require(upperBound > amountOut, "Price shift high detected");

            value0 = token0Amount * SCALE_SHIFT;
            
            value1 = (token1Amount * oraclePrice) / oracleBase;
        }
        //token1 must be USDC 
        else {
            //hardcode value of USDC at $1
            //check swap value of 100tokens to USDC to protect against flash loan attacks
            uint256 amountOut; //amount received by trade
            bool stablePool; //if the traded pool is stable or volatile.
            (amountOut, stablePool) = router.getAmountOut(HUNDRED_TOKENS, token0, USDC);
            require(stablePool == stable, "pricing occuring through wrong pool" );

            uint256 oraclePrice = getOraclePrice(priceFeed, tokenMaxPrice, tokenMinPrice);
            amountOut = (amountOut * oracleBase) / USDC_BASE / HUNDRED; //shift USDC amount to same scale as oracle

            //calculate acceptable deviations from oracle price
            uint256 lowerBound = (oraclePrice * (BASE - ALLOWED_DEVIATION)) / BASE;
            uint256 upperBound = (oraclePrice * (BASE + ALLOWED_DEVIATION)) / BASE;
            //because 1 USDC = $1 we can compare its amount directly to bounds
            require(lowerBound < amountOut, "Price shift low detected");
            require(upperBound > amountOut, "Price shift high detected");

            value1 = token1Amount * SCALE_SHIFT;
           
            value0 = (token0Amount * oraclePrice) / oracleBase;
        }
        //Invariant: both value0 and value1 are in ETH scale 18.d.p now
        //USDC has only 6 decimals so we bring it up to the same scale as other 18d.p ERC20s
        return(value0 + value1);
    }
```



Purpose: The system uses a test swap (of 100 tokens) to check how much money (USDC) it can get to determine a user’s deposit value.
Risk: If the system accidentally uses data from a manipulated pool (because it picked the “best” price rather than the right one), it can stop necessary actions (like selling tokens when a user is in trouble).
Impact: An attacker can avoid having their assets sold (liquidated) by deliberately causing the system to use the wrong price source.


## Explanation

Imagine you’re running a service where people deposit tokens (like digital money) into a vault. The system needs to know the value of each person’s deposit to decide if someone should be “liquidated” (forced to sell off assets to cover a debt) when their position gets risky.

To determine this value, the system “swaps” some tokens to USDC (a stablecoin pegged to $1) using a pricing function. This function checks how much USDC it would get in exchange for a fixed amount of tokens (called HUNDRED_TOKENS).



```solidity 

(amountOut, stablePool) = router.getAmountOut(HUNDRED_TOKENS, token, USDC);
require(stablePool == stable, "pricing occuring through wrong pool" );
```
Stable Pool: Think of this as a reliable, steady-price store (always around $1 per USDC).
Volatile Pool: This is like a market with prices that can change rapidly.

When pricing the deposit, the system calls a function on a router:

HUNDRED_TOKENS: A fixed quantity (say, 100 tokens) used to test the price.
getAmountOut: This function tells you how many USDC you’d get by swapping 100 tokens.
stablePool: A Boolean (true/false) indicating if the price came from the expected pool type (stable vs. volatile).


## The Problem:
The router’s getAmountOut function is designed to pick the best price between the two pools.
A crafty (malicious) user can manipulate one of these pools.

# A Numerical Example

HUNDRED_TOKENS = 100 tokens
Expected Rate in Stable Pool: 100 tokens = 100 USDC (i.e., 1 token = 1 USDC).
Manipulated Volatile Pool: The attacker manipulates it so that 100 tokens = 110 USDC (making it look “better”).

# Normal Scenario:
The router checks 100 tokens.
From the stable pool: it would get 100 USDC.
The function verifies the pool type and continues with liquidation if needed.


# Attack Scenario

The router now sees 110 USDC from the volatile pool because of the attacker's manipulation.
Even though 110 USDC is a “better” rate, the code expects the stable pool’s price.
The check fails: require(stablePool == stable, ...) because the router reported the volatile pool was used.
Outcome: The liquidation call is canceled, and the attacker avoids liquidation.

# FIX

The recommendation is to bypass the router’s automatic pool selection and directly query the correct (expected) pool. In code, this means:

```solidity
address pair = router.pairFor(token, USDC, stable);
amountOut = IPair(pair).getAmountOut(HUNDRED_TOKENS, token);

```











