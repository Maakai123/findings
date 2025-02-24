
## Access Control Verification

What to Check:
Look at each sensitive function (e.g., removeFeeder) to see if it uses proper access modifiers.
Heuristic:
If a function that should be limited to admins or owners does not use an admin-only modifier (like onlyRole(DEFAULT_ADMIN_ROLE)), flag it as a risk.

## Role Integrity
What to Check:
Ensure that roles (like the feeder’s UPDATER_ROLE) are only granted or revoked by authorized accounts.
Heuristic:
If any account (not just admins) can revoke a role through a function call, mark it as vulnerable.

## Function Behavior vs. Documentation

What to Check:
Compare what the comments say about a function’s behavior with what the code actually enforces.
Heuristic:
If comments say “owner only” but the code doesn’t enforce that, consider it a red flag.