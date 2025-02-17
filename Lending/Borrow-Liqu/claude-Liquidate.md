# Heuristics Model for Detecting Premature Liquidation Vulnerabilities

## 1. Core Time Variables Analysis

### Critical Variables to Identify
- Loan acceptance timestamp
- Last repayment timestamp
- Payment cycle duration
- Default/grace period duration
- Next payment due timestamp
- Liquidation threshold timestamp

### Time Variable Relationships
```
nextPaymentDue = max(acceptedTimestamp + paymentCycleDuration, lastRepaidTimestamp + paymentCycleDuration)
liquidationAllowed = block.timestamp > (nextPaymentDue + defaultDuration)
```

## 2. Risk Patterns

### Pattern A: Base Time Miscalculation
- **Red Flag**: Using loan acceptance time or last repayment time directly for liquidation calculations
- **Detection**: Search for direct comparisons between `block.timestamp` and `acceptedTimestamp` or `lastRepaidTimestamp`
- **Risk**: Can allow liquidation before first payment is due

### Pattern B: Missing Payment Cycle Check
- **Red Flag**: Liquidation logic that doesn't reference payment cycle duration
- **Detection**: Absence of `paymentCycleDuration` in liquidation calculations
- **Risk**: Disconnection between payment schedule and liquidation timing

### Pattern C: Duration Comparison Vulnerability
- **Red Flag**: `defaultDuration < paymentCycleDuration`
- **Detection**: Compare configured durations in initialization or configuration functions
- **Risk**: Could allow liquidation during valid payment period

### Pattern D: Zero-State Handling
- **Red Flag**: Incorrect handling of initial state (no payments made yet)
- **Detection**: Check handling of `lastRepaidTimestamp == 0`
- **Risk**: Incorrect liquidation timing for new loans

## 3. Code Analysis Checklist

### Function Level
- [ ] Verify base timestamp selection logic
- [ ] Check payment cycle integration
- [ ] Validate grace period application
- [ ] Review zero-state handling
- [ ] Examine timestamp comparison operations

### State Variable Level
- [ ] Track timestamp update points
- [ ] Verify timestamp initialization
- [ ] Check duration constant relationships
- [ ] Review timestamp access patterns

### System Level
- [ ] Analyze timestamp manipulation vectors
- [ ] Review cross-function timestamp consistency
- [ ] Check liquidation trigger points
- [ ] Validate state transition impacts

## 4. Test Scenarios

### Essential Test Cases
1. New Loan Liquidation Attempt
   ```solidity
   time = acceptedTimestamp + defaultDuration
   assert(!canLiquidate) // Should fail if vulnerable
   ```

2. First Payment Cycle
   ```solidity
   time = acceptedTimestamp + paymentCycleDuration + defaultDuration
   assert(canLiquidate) // Should pass if payment missed
   ```

3. Mid-cycle Liquidation Attempt
   ```solidity
   makePayment()
   time = lastRepaidTimestamp + defaultDuration
   assert(!canLiquidate) // Should fail if vulnerable
   ```

## 5. Mitigation Strategies

### Design Level
1. Always calculate liquidation threshold relative to next payment due date
2. Enforce `defaultDuration < paymentCycleDuration` in configuration
3. Implement explicit payment schedule tracking
4. Use safe timestamp comparison patterns

### Implementation Level
1. Separate timestamp calculation logic from liquidation checks
2. Implement robust zero-state handling
3. Add explicit payment schedule validation
4. Use time-based state machines for loan lifecycle

## 6. Common Pitfalls

1. Direct Timestamp Comparisons
   ```solidity
   // Vulnerable
   block.timestamp - lastRepaidTimestamp > defaultDuration
   
   // Safe
   block.timestamp > getNextPaymentDue() + defaultDuration
   ```

2. Missing State Validation
   ```solidity
   // Vulnerable
   function canLiquidate() returns (bool) {
       return isDefaulted();
   }
   
   // Safe
   function canLiquidate() returns (bool) {
       return isActive() && isDefaulted() && isAfterGracePeriod();
   }
   ```

3. Implicit Time Assumptions
   ```solidity
   // Vulnerable
   lastRepaidTimestamp == 0 ? acceptedTimestamp : lastRepaidTimestamp
   
   // Safe
   lastRepaidTimestamp == 0 ? acceptedTimestamp + paymentCycleDuration : lastRepaidTimestamp + paymentCycleDuration
   ```

## 7. Validation Framework

### Static Analysis Rules
1. Flag direct timestamp comparisons in liquidation logic
2. Detect missing payment cycle references
3. Identify unsafe timestamp initialization
4. Check duration constant relationships

### Dynamic Analysis Points
1. Time-based state transition testing
2. Payment schedule simulation
3. Edge case timestamp testing
4. Multi-cycle payment testing