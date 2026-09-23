# Sanctum Sanctorum Bookstore Notes

## Status

No public deployment is available yet. The application is currently being completed and
validated locally.

The Books feature is complete for the behavior covered by `tests/test_books.py`:

- 67 book tests pass.
- ISBN-13 input is normalized by removing hyphens and spaces.
- ISBN-13 check digits are validated before persistence.
- Duplicate ISBNs return HTTP 409 instead of leaking a database integrity error.
- Book updates support partial PATCH requests while ignoring ISBN changes.
- Book updates validate fields, trim text, persist changes, and return 404 for missing books.
- Book listing supports title/author search, restricted filtering, inclusive price ranges,
  sorting with ID tie-breakers, and pagination totals calculated before pagination.

The Loans feature is complete for the behavior covered by `tests/test_loans.py`:

- 45 loan tests pass.
- Loan fields include due dates, return timestamps, and persisted late fees.
- Borrowing enforces access, overdue, duplicate-book, tier-limit, and stock rules.
- Borrowing and returning update stock with the loan change in one transaction.
- Loan status is computed at read time with the strict due-date boundary from the spec.
- Returns calculate started-day late fees capped at the book price.
- Member loan lists support computed-status filtering and ID ordering.

The remaining application areas are not complete yet. The full suite still has failures in
members, orders, member statistics, and reports. Those areas contain unfinished service
and validation logic and should be implemented against `SPEC.md` before submission.

## Approach And Decisions

The existing project separates HTTP transport from business logic:

- Routers define endpoints, request parameters, response models, and status codes.
- Services perform database access, filtering, sorting, updates, and business rules.
- Pydantic schemas perform request validation and normalization before service code runs.
- SQLAlchemy models remain the persistence representation.

For the Books work, the router was kept thin. The PATCH route only accepts the validated
`BookUpdate` schema and delegates to the book service. ISBN validation remains in the schema,
while duplicate detection and database updates remain in the service layer.

Book listing builds one filtered SQL query, applies sorting before pagination, and calculates
`total` from the filtered query before applying `limit` and `offset`. This preserves the API
contract for searches and pagination and avoids counting only the returned page.

Duplicate ISBN persistence errors are caught in the service, the transaction is rolled back,
and the client receives a stable HTTP 409 response. The rollback is important because failed
writes must not leave the SQLAlchemy session in a broken transaction state.

## Validation

The focused Books and Loans suites were run with the repository virtual environment:

```powershell
.\\.venv\\Scripts\\python.exe -m pytest tests/test_books.py
.\\.venv\\Scripts\\python.exe -m pytest tests/test_loans.py
```

Result:

```text
67 passed, 2 warnings
45 passed, 2 warnings
```

The warnings came from dependency deprecations in the installed FastAPI/Starlette test stack.
The complete suite has not yet been rerun after the Books fixes.

## Remaining Work

1. Complete member validation, duplicate handling, access rules, and statistics.
2. Complete order creation, pricing, discounts, stock reservation, payment, and cancellation.
3. Complete reports and verify cross-feature statistics.
4. Run the complete test suite and address all remaining failures.
5. Deploy the application and add the public URL at the top of this file.
6. Commit each logical feature with a specific commit message and verify that generated files,
   local databases, virtual environments, and secrets are not committed.

## AI Usage

GitHub Copilot was used to inspect the assignment, tests, and nearby implementation code; identify
which book behaviors were incomplete; suggest focused changes; and help interpret test failures.
The changes were reviewed against the tests and the local project structure rather than accepted
blindly. Copilot also initially suggested commands that could not run in the active terminal
because of working-directory and environment differences; the repository virtual environment was
then located and used directly for validation.
