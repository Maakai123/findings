```solidity
function lastRepaidTimestamp(Loan storage loan) internal view returns (uint32) {
    return
        // @audit if no repayments have yet been made, lastRepaidTimestamp()
        // will return acceptedTimestamp - time when loan was accepted
        loan.lastRepaidTimestamp == 0
            ? loan.acceptedTimestamp
            : loan.lastRepaidTimestamp;
}

function canLiquidateLoan(uint loanId) public returns (bool) {
    Loan storage loan = loans[loanId];

    // Make sure loan cannot be liquidated if it is not active
    if (loan.state != LoanState.ACCEPTED) return false;

    return (uint32(block.timestamp) - lastRepaidTimestamp(loan) > paymentDefaultDuration);
   
}

```

## Explanation
Imagine you take out a 30-year mortgage to buy a house, your first payment is due in 30days. But the bank has rule: If you dont make a payment within 7 days of signing the contract the house will be seized.

## Issues 
Day 1: Sign mortgage, Day8 Bank forecloses even though first payment is in Day 30.

# The system allows liquidation before the borrower's first payment deadline if the grace period `paymentDefaultDuration` is shorter than the payment cycle.


# Example => Premature Liquidation with small `paymentDefaultDuration` Loan Parameters.

`acceptedTimestamp`: February 5, 2025(loan start date)
`paymentCycleDuration`: 30days (first payment due March 7, 2025)
`paymentDefaultDuration`: 7 days (grace period)
`CURRENT DATE `: February 13, 2025 (8 days after loan start)


```solidity
uint32 defaultStart = loan.acceptedTimesstamp;// feb 5, 2025
bool liquidation = currentDate - acceptedTimestamp > paymentDefaultDuration
 (Feb 13 - Feb 5) > 7 days -> 8 days > 7 days 
 True, LIQUIDATE 

```
`Liquidator seizes $120k house for $100k debt (Borrower loses $20k equity).`


## SOLUTION
```solidity 
function canLiquidateLoan(uint loanId) public returns (bool) {
    Loan storage loan = loans[loanId];
    if (loan.state != LoanState.ACCEPTED) return false;

    // Calculate next payment due date
    uint32 dueDate = loan.acceptedTimestamp + paymentCycleDuration;
    
    // Only start default countdown AFTER due date
    if (block.timestamp <= dueDate) return false; 
    
    return (block.timestamp - dueDate) > paymentDefaultDuration;
}
```



# Heuristics Model for Liquidation Timing Vulnerability


#  Time Calculation Integrity:

"Does the liquidation logic use block.timestamp - lastRepaidTimestamp without considering when the next payment is actually due?"
"What is the default value for lastRepaidTimestamp when no repayments have occurred, and is it appropriate for triggering liquidation?"
Parameter Configuration and Relationships:

"How are paymentDefaultDuration and paymentCycleDuration set relative to each other? Could paymentDefaultDuration be smaller than paymentCycleDuration?"
"Is there an enforced relationship or safeguard ensuring that liquidation can only occur after the next scheduled repayment date plus the grace period?"
Edge Case and Scenario Testing:

"What happens if a borrower has never made a repayment? Does the system mistakenly calculate the overdue period starting from the loan acceptance date?"
"Have we simulated scenarios where a borrower’s first payment is due in the future, yet liquidation is triggered prematurely?"
Mitigation and Safeguards:

"Is there a mechanism to calculate the liquidation threshold as (next repayment due date + paymentDefaultDuration) rather than (block.timestamp - lastRepaidTimestamp)?"




# Audit Checklist
Parameter Validation:
 - Ensure paymentDefaultDuration ≥ paymentCycleDuration.
Payment Phase Tracking:
 -Confirm nextDueDate is calculated as acceptedTimestamp + N * paymentCycleDuration.
Liquidation Threshold:
 -Verify liquidation checks use (currentTime - dueDate) > paymentDefaultDuration.
Edge Case Tests:
 - Test loans with 0 repayments at dueDate - 1 day and dueDate + 1 day.








