
#  Visibility Check

Rule: Identify functions that are declared as public.
Why: Public functions can be called by anyone. For sensitive operations (like modifying lien records), they should be restricted.
Example: A function that removes an element from a lien array should not be publicly accessible without extra checks.


#  Access Control Modifier Check
Rule: Look for access control modifiers (e.g., onlyOwner, onlyAdmin) or explicit authorization checks.
Why: These modifiers ensure that only trusted or pre-approved users can execute the function.
Example: If a function that alters lien data lacks any modifiers or a check like require(msg.sender == owner), it should be flagged

# Caller Verification Heuristic
Rule: Verify that the function includes a check confirming the caller’s identity or role before proceeding with sensitive changes.
Why: Without verifying msg.sender, any user could potentially perform unauthorized actions.
Example: Adding a statement like require(isAuthorized(msg.sender), "Not authorized") before modifying the state is a good practice

# Critical Data Modification Check

Rule: Identify functions that perform critical state changes (such as modifying arrays that hold financial records).
Why: Operations that shift or remove data from critical arrays (like lien lists) can have significant financial impacts if misused.
Example: A function that “pops” an element from a lien array must be secured because it directly affects collateral security.

