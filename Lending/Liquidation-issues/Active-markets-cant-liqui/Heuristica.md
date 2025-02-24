

## Heuristics Model Checklist
Check Data Structure Guarantees

What to Look For:
Examine if the code relies on the order of elements in a data structure (e.g., an EnumerableSet).
Red Flag:
The data structure used does not guarantee order (e.g., OpenZeppelin’s EnumerableSet) but the code assumes a constant order.

## Iterating While Modifying the Collection

What to Look For:
Identify loops that iterate over a collection while also removing or modifying elements in that same collection.
Red Flag:
Removal methods (like “swap-and-pop”) that change the collection’s order during iteration.


## Simulated Testing

What to Look For:
Write tests or simulations where multiple elements are removed during iteration.
Red Flag:
If the iteration fails (e.g., accessing an index that no longer exists), it confirms that removal during iteration is problematic.


