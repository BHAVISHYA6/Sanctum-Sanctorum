uv run pytest tests/test_reports.py
# Learnings: From Failing Tests to Working Code

This file explains the implementation work from the first test run through the final passing suite.
It is intended as a study reference. The acceptance tests were not modified.

## Big Picture

The project follows this flow:

```text
HTTP request
    -> router: parse and validate request
    -> schema: input shape and field validation
    -> service: business rules and database changes
    -> model: persisted data
```

The main rule used throughout the work was: keep routers thin, keep validation close to schemas,
and keep business rules and database mutations in services.

The first full test run showed:

```text
125 failed, 73 passed, 4 errors
```

Many failures were cascading failures from `NotImplementedError` placeholders. The final full run
passed:

```text
202 passed, 2 warnings
```

The warnings came from dependency deprecations in FastAPI/Starlette's test stack.

---

## 1. `tests/test_books.py`

### Issue A: Invalid ISBN check digits were accepted

**Previous code: `app/schemas.py`**

```python
def normalize_isbn13(raw: str) -> str:
    isbn = raw.replace("-", "").replace(" ", "")
    if len(isbn) != 13 or not isbn.isdigit():
        raise ValueError("isbn must contain exactly 13 digits")
    # TODO: verify the ISBN-13 check digit
    return isbn
```

**Problem**

The code checked only the ISBN length and numeric shape. A 13-digit ISBN with a wrong final check
digit was accepted, so the API returned `201` instead of `422`.

**Replacement code**

```python
check_sum = sum(
    int(digit) * (1 if index % 2 == 0 else 3)
    for index, digit in enumerate(isbn[:12])
)
expected_digit = (10 - check_sum % 10) % 10

if int(isbn[12]) != expected_digit:
    raise ValueError("isbn has an invalid checksum")
```

**Why this approach**

ISBN validation is input validation, so it belongs in the Pydantic schema before service code runs.
Raising `ValueError` lets FastAPI/Pydantic produce the expected `422` response consistently.

### Issue B: Duplicate ISBNs raised a database error

**Previous code: `app/services/books.py`**

```python
book = Book(**data.model_dump())
db.add(book)
db.commit()
```

**Problem**

The database correctly rejected duplicate ISBNs through its unique constraint, but the raw
`IntegrityError` escaped instead of becoming the API contract's `409` response.

**Replacement code**

```python
try:
    db.commit()
except IntegrityError:
    db.rollback()
    raise HTTPException(status_code=409, detail="ISBN already exists")
```

**Why this approach**

The unique constraint remains the final integrity guarantee. Catching the database exception in
the service translates persistence behavior into a stable HTTP response. The rollback is necessary
to make the SQLAlchemy session usable after the failed transaction.

### Issue C: PATCH was not available

**Previous code: `app/routers/books.py`**

```python
# TODO: expose PATCH /books/{book_id}
```

**Problem**

All PATCH requests returned `405 Method Not Allowed` because no route existed.

**Replacement code**

```python
@router.patch("/{book_id}", response_model=BookOut)
def update_book(book_id: int, data: BookUpdate, db: Session = Depends(get_db)):
    return service.update_book(db, book_id, data)
```

The service implementation applies only submitted fields:

```python
book = get_book(db, book_id)
for field, value in data.model_dump(exclude_unset=True).items():
    setattr(book, field, value)
db.commit()
db.refresh(book)
return book
```

**Why this approach**

The router only connects HTTP to the service. `exclude_unset=True` preserves partial-update
semantics, while `BookUpdate` intentionally has no ISBN field, so ISBN cannot be changed by PATCH.

### Issue D: Book search, filters, sorting, and totals were incomplete

**Previous code: `app/services/books.py`**

```python
if q:
    query = query.where(Book.title.icontains(q, autoescape=True))
# TODO: min_price / max_price filters
# TODO: apply ``sort``
books = db.scalars(query.order_by(Book.id.asc()).limit(limit).offset(offset)).all()
total = len(books)
```

**Problems**

- Author search did not work.
- Price filters were ignored.
- Requested sorting was ignored.
- `total` counted only the returned page instead of all matching rows.

**Replacement approach**

```python
if q:
    query = query.where(
        or_(
            Book.title.icontains(q, autoescape=True),
            Book.author.icontains(q, autoescape=True),
        )
    )
if min_price is not None:
    query = query.where(Book.price_cents >= min_price)
if max_price is not None:
    query = query.where(Book.price_cents <= max_price)

if sort == "title":
    query = query.order_by(Book.title.asc(), Book.id.asc())
elif sort == "-title":
    query = query.order_by(Book.title.desc(), Book.id.asc())
elif sort == "price":
    query = query.order_by(Book.price_cents.asc(), Book.id.asc())
elif sort == "-price":
    query = query.order_by(Book.price_cents.desc(), Book.id.asc())
else:
    query = query.order_by(Book.id.asc())

total = db.scalar(select(func.count()).select_from(query.subquery())) or 0
books = db.scalars(query.limit(limit).offset(offset)).all()
```

**Why this approach**

One SQL query keeps filtering and sorting consistent. Counting the filtered subquery before applying
pagination matches the API contract and avoids loading unnecessary rows into Python.

**Result**

```text
67 passed - tests/test_books.py
```

---

## 2. `tests/test_loans.py`

### Issue A: The loan model was missing required fields

**Previous code: `app/models.py`**

```python
borrowed_at: Mapped[datetime] = mapped_column(DateTime)
# TODO: due_at, returned_at, late_fee_cents
```

**Problem**

The API schema and tests required due dates, nullable return dates, and persisted late fees, but the
database model could not store them.

**Replacement code**

```python
borrowed_at: Mapped[datetime] = mapped_column(DateTime)
due_at: Mapped[datetime] = mapped_column(DateTime)
returned_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)
late_fee_cents: Mapped[int] = mapped_column(Integer, default=0)
```

**Why this approach**

The model must represent the complete domain state required by `LoanOut` and the specification.

### Issue B: Loan service operations were placeholders

**Previous code: `app/services/loans.py`**

```python
def create_loan(...):
    raise NotImplementedError("create_loan")

def get_loan(...):
    raise NotImplementedError("get_loan")

def return_loan(...):
    raise NotImplementedError("return_loan")
```

**Problem**

Borrow, fetch, return, member-loan listing, status, and late-fee tests received `501` responses.

**Replacement behavior**

The service now:

- Checks member and book existence.
- Checks restricted-book access.
- Blocks overdue members.
- Blocks duplicate unreturned books.
- Enforces tier loan limits.
- Checks stock before decrementing it.
- Creates a 14-day loan.
- Restores stock on return.
- Rejects returning an already returned loan.
- Lists loans by member and computed status.

Loan status is computed as:

```python
if loan.returned_at is not None:
    return "returned"
if now > loan.due_at:
    return "overdue"
return "active"
```

**Why this approach**

Status depends on the current clock, so it must be computed when the loan is read rather than stored
as a stale database value. The strict `now > due_at` condition preserves the required boundary where
a loan is still active exactly at its due time.

Late fees use started-day rounding:

```python
if returned_at <= due_at:
    return 0
days_late = ceil((returned_at - due_at).total_seconds() / 86400)
return min(days_late * 25, price_cents)
```

Stock and loan changes are committed together so a successful operation changes both consistently.

### Issue C: Master-tier access comparison was too strict

**Previous code: `app/services/members.py`**

```python
return TIER_ORDER.index(tier) > TIER_ORDER.index(minimum)
```

**Problem**

`master` was not considered high enough to access restricted books, even though the requirement says
master and supreme are allowed.

**Replacement code**

```python
return TIER_ORDER.index(tier) >= TIER_ORDER.index(minimum)
```

**Why this approach**

“At least master” includes the boundary tier itself, so `>=` expresses the rule directly.

**Result**

```text
45 passed - tests/test_loans.py
```

---

## 3. `tests/test_members.py`

### Issue A: Email normalization happened too late, or not at all

**Previous code: `app/schemas.py`**

```python
if not EMAIL_PATTERN.match(value):
    raise ValueError("email is not valid")
return value
```

**Problem**

Emails with surrounding spaces or uppercase characters were rejected or stored without the required
normal form. Case variants could also bypass the unique constraint.

**Replacement code**

```python
normalized = value.strip().lower()
if not EMAIL_PATTERN.fullmatch(normalized):
    raise ValueError("email is not valid")
return normalized
```

**Why this approach**

Normalize before validation and persistence. Every later layer sees one canonical value, which makes
case-insensitive uniqueness work naturally with the database unique constraint.

### Issue B: Duplicate emails were not translated to HTTP 409

**Previous behavior: `app/services/members.py`**

```python
db.add(member)
db.commit()
```

**Replacement behavior**

```python
try:
    db.commit()
except IntegrityError:
    db.rollback()
    raise HTTPException(status_code=409, detail="Email already exists")
```

**Why this approach**

The service owns persistence rules and is the right place to translate the database constraint into
the API response while rolling back the failed transaction.

### Issue C: Member statistics were unimplemented

**Previous code: `app/services/members.py`**

```python
def get_member_stats(...):
    raise NotImplementedError("get_member_stats")
```

**Replacement logic**

```python
orders = db.scalars(
    select(Order).where(
        Order.member_id == member_id,
        Order.status == OrderStatus.PAID.value,
    )
).all()
loans = db.scalars(select(Loan).where(Loan.member_id == member_id)).all()

active_loans = [loan for loan in loans if loan.returned_at is None]
overdue_loans = [loan for loan in active_loans if now > loan.due_at]
late_fees_cents = sum(
    loan.late_fee_cents for loan in loans if loan.returned_at is not None
)
```

**Why this approach**

The specification defines statistics by status, not by all records. Filtering paid orders and using the
same strict loan boundary as the loan service keeps member statistics consistent with other endpoints.

**Result**

```text
32 passed - tests/test_members.py
```

---

## 4. `tests/test_orders.py`

### Issue A: Order validation and pricing were unimplemented

**Previous code: `app/schemas.py`**

```python
items: List[OrderItemIn]
```

**Problem**

An empty item list and duplicate book IDs were accepted even though the API requires `422`.

**Replacement code**

```python
items: List[OrderItemIn] = Field(min_length=1)

@model_validator(mode="after")
def reject_duplicate_books(self) -> OrderCreate:
    book_ids = [item.book_id for item in self.items]
    if len(book_ids) != len(set(book_ids)):
        raise ValueError("each book may appear only once")
    return self
```

The quantity field already used `Field(ge=1)`, so zero and negative quantities are rejected by the
schema.

### Issue B: Order creation was a placeholder

**Previous code: `app/services/orders.py`**

```python
def create_order(...):
    raise NotImplementedError("create_order")
```

**Replacement approach**

```python
member = get_member(db, data.member_id)
books = []
for item in data.items:
    book = db.get(Book, item.book_id)
    if book is None:
        raise HTTPException(status_code=404, detail="Book not found")
    books.append((item, book))

if any(book.restricted for _, book in books):
    ensure_can_access_restricted(member)

if any(book.stock < item.quantity for item, book in books):
    raise HTTPException(status_code=409, detail="Insufficient stock")
```

Only after all checks succeed does the service decrement stock and create order items. Pricing then
uses current book prices, calculates tier plus bulk discount, floors discount cents, and stores the
unit-price snapshot.

**Why this approach**

The all-checks-before-mutations order is essential. If one item is unavailable, no stock can be
changed and no partial order can be created. This directly satisfies the assignment's data-integrity
requirement.

### Issue C: Cancellation did not restore reserved stock

**Previous behavior: `cancel_order()` changed only the status.**

**Replacement code**

```python
for item in order.items:
    book = db.get(Book, item.book_id)
    if book is not None:
        book.stock += item.quantity
order.status = OrderStatus.CANCELLED.value
db.commit()
```

**Why this approach**

Stock is reserved during order creation, so cancelling a pending order must return every reserved
quantity. Payment does not change stock because it was already reserved.

**Result**

```text
46 passed - tests/test_orders.py
```

---

## 5. `tests/test_reports.py`

### Issue: The report service was unimplemented

**Previous code: `app/services/reports.py`**

```python
def top_books(db: Session, limit: int = 5) -> List[TopBook]:
    raise NotImplementedError("top_books")
```

**Problem**

Every report behavior returned `501`: empty reports, paid aggregation, unpaid exclusion, sorting,
and limits.

**Replacement code**

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

Rows are converted into `TopBook` schema objects.

**Why this approach**

Aggregation belongs in SQL because the database can group and sum efficiently. Starting from paid
order items naturally excludes books with no paid sales, while joining `Book` provides the current
title. The router already validates `limit` using `Query(5, ge=1, le=50)`.

**Result**

```text
11 passed - tests/test_reports.py
```

---

## 6. Test Commands And Final Result

## Health Test

`tests/test_health.py` already passed from the beginning. It checks that `GET /health` returns
HTTP 200 and `{"status": "ok"}`. No application change was needed for this test.

Run a focused file:

```powershell
uv run pytest tests/test_books.py
uv run pytest tests/test_loans.py
uv run pytest tests/test_members.py
uv run pytest tests/test_orders.py
uv run pytest tests/test_reports.py
```

Run the complete suite:

```powershell
uv run pytest tests
```

Final result after all feature fixes:

```text
202 passed, 2 warnings
```

The warnings are third-party dependency deprecation warnings, not application test failures.

## 7. What To Remember

- Validate request shape and normalization in schemas.
- Keep routers thin and delegate behavior to services.
- Use database constraints for integrity, but translate expected constraint failures into API errors.
- Roll back failed transactions before reusing a SQLAlchemy session.
- Check every condition before changing stock or creating an order.
- Compute time-sensitive statuses from the injected clock at read time.
- Use SQL aggregation for reporting queries.
- Preserve deterministic ordering with explicit tie-breakers.
- Run focused tests after each feature, then run the complete suite.
