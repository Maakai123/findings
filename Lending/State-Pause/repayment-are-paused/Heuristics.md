# What to Look For:
Identify functions handling repayments and liquidations (e.g., repay() and liquidate()).
Heuristic:
Verify that both functions check the same flag (e.g., isRepayAllowed()). If they immediately allow liquidation when repayments resume, that’s a warning sign.


# What to Look For:
Examine how the system behaves when the repayment flag switches from paused to active.
Heuristic:
Check if there’s any logic delaying liquidations after repayments resume. If the state change allows instant liquidation, flag it.

