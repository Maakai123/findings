The operator can disable a market with open positions. This traps traders (like Naruto) who then cannot close their positions, yet they still face liquidation.

Solutions: Either prevent market disablement when there are open positions or allow traders to close positions in disabled mark

`GlobalConfigurationBranch::updatePerpMarketStatus—lets the protocol operator disable a trading market (like turning it “off”) without checking whether there are still active, open positions in that market. This creates a dangerous situation for traders.`

# Real-World Example:
Imagine Naruto, a trader, deposits $100,000 and opens a leveraged position in the BTC market.
Figures:
Starting Balance: $100,000
Position Size: 10 units (for simplicity)
Expectation: Naruto can later close (or reverse) his position if he needs to protect his funds or lock in profits.

# Market Gets Disabled
The protocol operator decides to disable the BTC market by calling updatePerpMarketStatus. This is like the operator turning off the trading floor for that market.
Key Issue:
There’s no check in this function to see if there are any open positions—like Naruto’s leveraged trade—that still exist. This means the market can be disabled even if traders are “in play.”

# Trader Can’t Close the Position

After the market is disabled, Naruto tries to close his position (to either take profit or cut his losses). However, the system now prevents him from doing so, showing an error (e.g., “PerpMarketDisabled”).

Although Naruto cannot voluntarily close his position, the liquidation process (a safety mechanism) still works. This means if the market price moves against him, his position can be forcibly closed (liquidated) by the system.

Liquidation Still Works: After forcing a price drop (halving the price), the test shows that Naruto’s account becomes liquidatable and is eventually liquidated—even though he could never close his position himself.



