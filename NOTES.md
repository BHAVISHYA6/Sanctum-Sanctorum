deploy link : https://sanctum-sanctorum-m0d8.onrender.com/
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

The Members feature is complete for the behavior covered by `tests/test_members.py`:

- 32 member tests pass.
- Emails are stripped, lowercased, validated, and kept unique.
- Duplicate emails return HTTP 409 with a rolled-back transaction.
- Statistics count paid orders, active loans, overdue loans, and returned-loan fees.

The Orders feature is complete for the behavior covered by `tests/test_orders.py`:

- 46 order tests pass.
- Validation, pricing, tier and bulk discounts, price snapshots, stock reservation, payment,
  cancellation, and all-or-nothing stock checks are implemented.

The Reports feature is complete for the behavior covered by `tests/test_reports.py`:

- 11 report tests pass.
- Top-book results aggregate quantities from paid orders only.
- Books with only pending or cancelled orders are excluded.
- Results use the current book title, sort by copies sold descending and title ascending, and
    respect the validated limit range of 1 to 50.

The remaining work is final full-suite verification and deployment.

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
.\\.venv\\Scripts\\python.exe -m pytest tests/test_members.py
.\\.venv\\Scripts\\python.exe -m pytest tests/test_orders.py
.\\.venv\\Scripts\\python.exe -m pytest tests/test_reports.py
```

Result:

```text
67 passed, 2 warnings
45 passed, 2 warnings
32 passed, 2 warnings
46 passed, 2 warnings
11 passed, 2 warnings
```

The warnings came from dependency deprecations in the installed FastAPI/Starlette test stack.
The complete suite has not yet been rerun after the Books fixes.

## Remaining Work

1. Complete reports and verify cross-feature statistics.
2. Run the complete test suite and address all remaining failures.
3. Deploy the application and add the public URL at the top of this file.
4. Commit each logical feature with a specific commit message and verify that generated files,
   local databases, virtual environments, and secrets are not committed.

## AI Usage

GitHub Copilot was used to inspect the assignment, tests, and nearby implementation code; identify
which book behaviors were incomplete; suggest focused changes; and help interpret test failures.
The changes were reviewed against the tests and the local project structure rather than accepted
blindly. Copilot also initially suggested commands that could not run in the active terminal
because of working-directory and environment differences; the repository virtual environment was
then located and used directly for validation.
