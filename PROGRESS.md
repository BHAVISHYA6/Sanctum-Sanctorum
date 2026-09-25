# Development Progress Reference

This file is a personal reference for the work completed so far. `NOTES.md` is the concise submission document; this file keeps the fuller working history.

## 1. Initial Test Setup

The project uses `uv` and defines the main test command as:

```powershell
uv run pytest
```

On Windows, `make test` was unavailable because GNU Make was not installed. The direct pytest command is the correct replacement.

`uv` was initially unavailable in PowerShell, but the project already had a local `.venv` with the required dependencies. The reliable command used for the focused tests was:

```powershell
.\\.venv\\Scripts\\python.exe -m pytest tests/test_books.py
```

The terminal working directory also mattered: commands had to run from:

```text
C:\Merkle_Science\Sanctum-Sanctorum-main
```

## 2. Initial Test Results

The first full test run showed many failures because several application areas contained TODOs or `NotImplementedError` placeholders.

Important examples from the original output:

- `create_loan` returned HTTP 501.
- `create_order` returned HTTP 501.
- Book PATCH requests returned HTTP 405 because the route did not exist.
- Duplicate ISBNs raised an unhandled SQLite `IntegrityError`.
- ISBNs with invalid check digits were accepted with HTTP 201.
- Book filters, sorting, and pagination totals were incomplete.

The original full run reported:

```text
125 failed, 73 passed, 4 errors
```

Those failures were not all caused by one defect. Many were cascading failures from the unfinished loans and orders services.

## 3. Books Investigation

The relevant acceptance tests were in `tests/test_books.py`.

The owning implementation was split across:

- `app/schemas.py` for request validation and ISBN normalization
- `app/routers/books.py` for HTTP routes
- `app/services/books.py` for database operations and listing behavior

The tests showed four main missing areas:

1. ISBN checksum validation and duplicate handling
2. PATCH route and update behavior
3. Search and price filters
4. Sorting and pre-pagination totals

## 4. Changes Made

### ISBN validation

In `app/schemas.py`, `normalize_isbn13()` now:

- Removes hyphens and spaces.
- Requires exactly 13 numeric characters.
- Calculates the ISBN-13 check digit using alternating weights of 1 and 3.
- Raises `ValueError` when the supplied check digit is invalid.

Pydantic converts that validation error into the expected HTTP 422 response.

### Duplicate ISBN handling

In `app/services/books.py`, book creation now catches SQLAlchemy `IntegrityError` exceptions caused by the unique ISBN constraint.

The service rolls back the failed transaction and raises HTTP 409 with a stable error message. Rolling back is important so the database session can continue to be used safely after the failed insert.

### Book PATCH support

In `app/routers/books.py`, a `PATCH /books/{book_id}` route was added.

The router:

- Accepts the validated `BookUpdate` schema.
- Delegates the operation to the service.
- Returns the `BookOut` response model.

In `app/services/books.py`, `update_book()` now:

- Returns 404 when the book does not exist.
- Applies only fields supplied in the request.
- Leaves ISBN unchanged because ISBN is not part of `BookUpdate`.
- Commits and refreshes the updated model.

### Book listing

In `app/services/books.py`, listing now supports:

- Case-insensitive search across both title and author.
- Exact restricted filtering.
- Inclusive minimum and maximum price filters.
- Title ascending and descending sorting.
- Price ascending and descending sorting.
- ID ascending tie-breaking for deterministic results.
- Total count calculated before limit and offset are applied.

## 5. Validation After Changes

The focused Books test suite was run with:

```powershell
Push-Location C:\Merkle_Science\Sanctum-Sanctorum-main
.\\.venv\\Scripts\\python.exe -m pytest tests/test_books.py
Pop-Location
```

Result:

```text
67 passed, 2 warnings in 1.92s
```

The two warnings were dependency deprecation warnings from the installed FastAPI/Starlette test stack. They did not cause test failures.

## 6. Files Changed So Far

Application files changed for the Books work:

- `app/schemas.py`
- `app/routers/books.py`
- `app/services/books.py`

Documentation files added:

- `NOTES.md` for submission-oriented decisions and status
- `PROGRESS.md` for this detailed personal reference

No test files were modified.

## 7. Loan Work Completed

The loan implementation was completed after the Books work:

- Added `due_at`, nullable `returned_at`, and `late_fee_cents` to the `Loan` model.
- Implemented computed loan status with the strict `now > due_at` overdue rule.
- Implemented borrowing, including member/book existence checks, restricted-book access,
	overdue blocking, duplicate-book blocking, tier limits, stock checks, and stock decrement.
- Implemented loan lookup and member loan listing with optional computed-status filtering.
- Implemented returns, stock restoration, returned-state protection, and capped late fees.
- Corrected the shared tier comparison so the minimum tier itself is allowed; `master` can access
	restricted books as required by the specification.

The focused loan suite now passes:

```text
45 passed, 2 warnings in 3.07s
```

## 8. Member And Order Work Completed

The initial `tests/test_members.py` run had these failures:

- `test_email_is_stripped_and_lowercased`: `MemberCreate.normalize_email()` validated and returned
	the raw value, so whitespace and uppercase letters were not normalized.
- `test_duplicate_email_returns_409`: `create_member()` did not handle the database uniqueness
	error, so the `IntegrityError` escaped instead of becoming HTTP 409.
- `test_duplicate_email_is_case_insensitive`: raw email storage allowed differently cased values
	to bypass the unique constraint.
- The member statistics tests failed because `get_member_stats()` raised `NotImplementedError`.
- `test_order_stats_count_only_paid_orders` also depended on `create_order()`, which returned 501.

The previous email code in `app/schemas.py` was:

```python
if not EMAIL_PATTERN.match(value):
		raise ValueError("email is not valid")
return value
```

It was replaced with normalization before validation:

```python
normalized = value.strip().lower()
if not EMAIL_PATTERN.fullmatch(normalized):
		raise ValueError("email is not valid")
return normalized
```

The previous member creation code in `app/services/members.py` committed directly. It now catches
`IntegrityError`, rolls back the failed transaction, and returns HTTP 409 for duplicate emails.
`get_member_stats()` now verifies the member, counts only paid orders, sums paid totals, counts
unreturned and overdue loans using the strict `now > due_at` rule, and sums fees from returned loans.

The order dependency was fixed in `app/schemas.py` and `app/services/orders.py`. Empty or duplicate
items now return 422; member and book checks happen before mutation; restricted access is checked
before stock; all stock is checked before any decrement; discounts and price snapshots are stored;
and cancellation restores reserved stock.

Focused validation now passes:

```text
46 passed, 2 warnings in 1.89s  # tests/test_orders.py
32 passed, 2 warnings in 0.90s  # tests/test_members.py
```

## 9. Remaining Work

The full assignment is not complete yet. Based on the earlier test output, the next implementation areas are:

- Reports: paid-order aggregation and top-book reporting
- Cross-feature member statistics

After those changes, the full suite should be rerun with:

```powershell
.\\.venv\\Scripts\\python.exe -m pytest
```

The assignment also expects:

- Incremental Git commits with meaningful messages
- A deployed public URL
- No committed `.venv`, database files, caches, or secrets
- A final update to `NOTES.md` with deployment and final test results

## 10. Reports Work Completed

The report test file was `tests/test_reports.py`. The report router already had the correct
endpoint and validation:

```python
@router.get("/top-books", response_model=List[TopBook])
def top_books(limit: int = Query(5, ge=1, le=50), db: Session = Depends(get_db)):
	return service.top_books(db, limit)
```

Therefore, the `limit=0` and `limit=51` tests were not code failures in the service; FastAPI's
`Query(5, ge=1, le=50)` correctly returns 422 for those values.

The failing report cases were caused by the previous code in `app/services/reports.py`:

```python
def top_books(db: Session, limit: int = 5) -> List[TopBook]:
	raise NotImplementedError("top_books")
```

This caused HTTP 501 for empty reports, paid-order aggregation, unpaid-order exclusion, sorting,
and limit tests.

The replacement code now builds a SQLAlchemy aggregation query:

```python
copies_sold = func.sum(OrderItem.quantity).label("copies_sold")
rows = db.execute(
	select(Book.id, Book.title, copies_sold)
	.join(OrderItem, OrderItem.book_id == Book.id)
	.join(Order, Order.id == OrderItem.order_id)
	.where(Order.status == OrderStatus.PAID.value)
	.group_by(Book.id, Book.title)
	.order_by(copies_sold.desc(), Book.title.asc())
	.limit(limit)
).all()
```

The result rows are converted into `TopBook` schema objects. This approach:

- Aggregates quantities across paid orders only.
- Omits books with no paid sales naturally because the query starts from order items.
- Uses the book's current title from the `books` table.
- Sorts by copies sold descending and title ascending.
- Applies the requested limit in the database query.

Validation result:

```text
11 passed, 2 warnings in 0.52s  # tests/test_reports.py
```
