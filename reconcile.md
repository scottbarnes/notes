Reconcile
=========

# Refactoring for PostgreSQL tasks
- The `fetch` function needs support for the `jsonl` input.

# Queries
- I should check for the OL dump for duplicate tiles or ISBNs or both. Maybe convert all ISBNs to 13 before checking. That way there is one number to check against? But how to check, because I can't do that within the DB -- and some books have multiple ISBNs; do I need to convert them all to ISBN 13? But what if only one matches? Think of all the different permutations and see if any give a good way to find duplicates.
