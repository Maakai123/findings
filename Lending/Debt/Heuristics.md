# Existence Check for Inputs

What to Look For:
Verify that functions retrieving data from mappings (like credits) explicitly check that the item exists.
Red Flag:
If the code directly retrieves an item (e.g., Credit memory credit = credits[id];) without verifying its existence, it may return default (empty) values.

# Validation of Critical Values
What to Look For:
Ensure that key properties (like credit.principal) are validated to be non-zero (or within expected ranges) before proceeding with state changes.
Red Flag:
Accepting an empty or zero-value credit as valid for closing can lead to unintended behavior.


# Proper Management of Counters

What to Look For:
Check how the system manages counters (e.g., count for open credits) when closing or updating items.
Red Flag:
If the counter is decremented without confirming that a valid credit has been closed, repeated invalid calls can wrongly mark the overall facility as repaid.
Access Control and Caller Verification

What to Look For:
Ensure that functions are restricted to authorized callers and that the caller’s identity is verified appropriately.
Red Flag:
If a function allows calls by any user without proper checks, attackers might exploit default values in storage.
Testing with Edge Cases

What to Look For:
Simulate calls with non-existent or default values to see how the contract handles them.
Red Flag:
If edge-case inputs lead to state changes (like decrementing counters or marking loans as repaid) without proper validation, the vulnerability is confirmed.
