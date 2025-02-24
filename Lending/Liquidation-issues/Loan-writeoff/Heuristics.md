# Heuristic 1: State Consistency Check
Rule: Every time a debt is modified, all related variables (e.g., principal and lastRepay) must be updated.
Check: Look at functions that change the borrower's debt. If the principal is reset but lastRepay remains unchanged, that’s a red flag.
# Heuristic 2: Full Debt Write-Off Reset
Rule: When the entire debt is cleared (i.e., fully written off), the system should reset or update any timestamp markers like lastRepay.
Check: Verify that after a full write-off, lastRepay is either reset (e.g., set to 0) or updated to reflect the current state.
# Heuristic 3: Timing Validation
Rule: The system’s timing checks (e.g., overdue calculations) should always use the most recent action time.
Check: Compare the lastRepay value with the current block/time. If the lastRepay is outdated (like still showing block 1000 when a new loan happens at block 2000), the timing check could mistakenly trigger a write-off.
# Heuristic 4: Sequential Operation Simulation
Rule: Simulate the normal flow of operations.
Check:
Borrow a loan (expect lastRepay to update).
Write off the debt (verify if lastRepay resets on a full write-off).
Borrow again (ensure lastRepay updates with the new block/time).
Outcome: Any break in this flow signals a potential flaw.


Real-World Analogy
Imagine a library system:

Book Borrowing: When you check out a book, the system records the checkout date.
Book Return (Write-Off): When you return the book, the system should update your record to show the book is returned (reset the due date).
New Borrow: If you borrow a new book, the checkout date should be updated to today.
Heuristic Check: If the checkout date from your previous loan is still there, the system might wrongly assume you’re overdue on the new book. That’s similar to the stale lastRepay issue.
