Real-World Example Walkthrough
Let's say Alice borrows against her Bored Ape NFT:

Alice's Bored Ape is worth 100 ETH
She borrows 70 ETH from Bob (70% LTV)
They agree to use TrustedOracle
Later, the Bored Ape price increases to 200 ETH
Alice's loan is still only 70 ETH (plus some interest)

The Attack:

Bob creates FakeOracle that always returns 50 ETH for Alice's Bored Ape
Bob calls updateLoanParams() with FakeOracle
Bob calls removeCollateral() to seize the NFT
Bob now owns a 200 ETH Bored Ape after lending only 70 ETH!


Initial Setup

NFT Value (per TrustedOracle): 100 ETH
Loan Amount: 70 ETH
LTV: 70%
Duration: 30 days
NFT actual market value increases to 200 ETH

Attack Path

Bob deploys FakeOracle that reports value as 50 ETH
Bob calls updateLoanParams() with:

Same duration (30 days)
Same valuation (70 ETH)
Same interest (5%)
Same LTV (70%)
But different oracle: FakeOracle


Contract checks:

✅ duration >= old duration
✅ valuation <= old valuation
✅ interest <= old interest
✅ LTV <= old LTV
❌ Doesn't check if oracle changed!


Bob calls removeCollateral():

FakeOracle reports: 50 ETH
50 ETH × 70% LTV = 35 ETH (reported "value")
Loan amount: 70 ETH + interest
Since 35 ETH < 70 ETH, NFT appears "underwater"
Seizure succeeds!

The Real Consequence

Alice loses her 200 ETH Bored Ape
Bob profits 130 ETH (200 ETH - 70 ETH loan)
Alice still loses her 70 ETH deposit
This breaks the fundamental trust in the system


##  How to Fix It

The simple fix is to add one more check in the updateLoanParams() function:
```solidity
Copyrequire(
    params.duration >= cur.duration &&
    params.valuation <= cur.valuation &&
    params.annualInterestBPS <= cur.annualInterestBPS &&
    params.ltvBPS <= cur.ltvBPS &&
    params.oracle == cur.oracle,  // ADD THIS LINE!
    "NFTPair: worse params"
);
```