# Sanctum Sanctorum Bookstore — Notes

## Live Deployment

Production URL: https://sanctum-sanctorum-gilt.vercel.app/

A few useful links:

- Frontend: https://sanctum-sanctorum-gilt.vercel.app/
- Health check: https://sanctum-sanctorum-gilt.vercel.app/health
- API docs: https://sanctum-sanctorum-gilt.vercel.app/docs

It's deployed on Vercel using their current FastAPI support, which picks up `app.main:app`
directly — no `api/index.py` or legacy routing config needed. Production data lives in
PostgreSQL, wired up through the private `SANCTUM_DATABASE_URL` env var. Locally I've kept
SQLite as the default for dev and testing.

Once the database is seeded, you can pick a demo member from the Members tab and try out the
different restricted-book behaviors across the four tiers (Supreme, Master, Adept, Apprentice).
Worth repeating: this member selector is just a stand-in for the take-home — it's not a real
auth system.

## What's Done

Here's what's actually built and passing:

**Books**
- Creation with full request/response validation
- Title/author trimming and length checks
- ISBN-13 normalization, shape checks, and checksum validation
- Duplicate ISBNs return a 409 instead of silently failing
- Retrieval and partial updates via PATCH
- Case-insensitive search on title/author
- Price filters (inclusive and restricted)
- Sorting by title/price with deterministic tie-breaking on ID
- Pagination, with totals computed before limit/offset are applied

**Members**
- Creation and retrieval
- Name trimming/validation
- Email trimming, lowercasing, format validation, duplicate handling
- Tier validation
- Order history per member
- Stats: paid orders, total spend, active loans, overdue loans, late fees

**Orders**
- Rejects empty item lists, bad quantities, duplicate books in one order
- Checks member/book existence and restricted-book access before anything else
- All-or-nothing stock validation — no partial mutation on failure
- Tier and bulk discounts calculated in integer cents (no float drift)
- Price snapshotting and stock reservation at creation time
- Full pending → paid → cancelled lifecycle
- Stock gets restored if a pending order is cancelled

**Loans**
- Tracks due date, return date, and late fee
- Enforces restricted-book access and per-tier loan limits
- Handles overdue rules and blocks duplicate active loans on the same book
- Stock decrements on borrow, restores on return
- Status (active/overdue/returned) is computed, not stored
- Strict boundary: a loan due exactly *now* still counts as active
- Late fees are day-based, capped at the book's price at return time

**Reports & platform**
- Top-books report, built only from paid orders, with quantity aggregation and validated limits
- `/health` and `/docs` endpoints
- Static frontend served from `/`
- PostgreSQL support via `SANCTUM_DATABASE_URL`, handling both `postgres://` and `postgresql://`
- No existing tests were touched

Full suite result:

```text
202 passed, 2 warnings
```

Both warnings are just deprecation notices from the FastAPI/Starlette test stack itself
nothing in the application is failing. As of this write-up, I don't have any tests actually
failing; everything in the supplied acceptance suite is green.

## Test Cases I'd Still Want to Add

These go beyond what the supplied test suite requires, but I think they'd matter a lot before
calling this production-ready. I'm splitting them into "things that are basically un-tested
gaps in coverage" versus "known, accepted limitations"  see the note at the bottom on the last
category.

**Auth and identity**
- Anonymous users shouldn't see or be able to use book create/edit controls
- `POST /books` and `PATCH /books/{id}` should reject requests without a real, server-verified
  staff role
- A regular member shouldn't be able to trigger staff-only catalog changes
- Editing the `localStorage` member ID shouldn't let someone hijack another identity
- A failed member lookup shouldn't leave the UI still "acting as" an unverified ID
- A member shouldn't be able to read another member's orders, loans, or stats

**Email policy**
- Decide (and document) whether something like `b@m.com` is meant to be valid
- If a stricter policy is wanted: test short local parts, short domains, malformed domain
  labels, consecutive dots, and internationalized addresses under whatever policy gets chosen
- Keep the existing trimming/lowercasing normalization tests either way

**Inventory and transactions**
- Two concurrent orders for the last copy shouldn't both succeed
- A DB error mid-order-creation should roll back every stock change and the order row, not leave
  things half-applied
- Same idea for loan creation a DB error should roll back both the loan and the stock
  decrement together
- Same for returns  roll back the return and the stock increment together
- Cancelling an order whose book was later deleted needs an explicit, defined policy rather than
  quietly skipping stock restoration

**Deployment and lifecycle**
- Two simultaneous cold starts shouldn't seed duplicate data
- A PostgreSQL connection failure should produce a clear health/startup error, not a silent hang
- Static assets and API routes should both work cleanly from a fresh browser session
- Preview and production deployments should each be pointed at the right `SANCTUM_DATABASE_URL`

A couple of these overlap with known, accepted limitations rather than bugs specifically the
concurrent last-copy race and the lack of a paginated `GET /members` endpoint. Both are
explicitly optional per `ASSIGNMENT.md`, so I didn't build them, but I've kept them here since
they're exactly the kind of thing the test list above would catch if priorities shift.

The app also doesn't do real user authentication or staff authorization anywhere. The assignment
works off a client-selected member ID and never defines passwords, sessions, tokens, or roles 
so for anything resembling a real product, book creation/editing and member actions would need
proper server-side auth rather than trusting whatever ID the client sends.

## Things I Noticed While Reviewing

These aren't required-feature gaps — they're product/security observations from poking at the
running frontend and API.

**Edit / Add book controls show up before sign-in.** The catalog renders these without checking
for a selected member, and the API has no auth or role dependency behind them either. So right
now, anyone can edit price/stock or add books straight from the UI, or by hitting the endpoints
directly.
*Scope:* beyond the assignment — `SPEC.md` defines the book endpoints but says nothing about
login or staff roles.
*Fix:* for the demo, just hide/disable these controls until a member is selected. For anything
real, add server-side auth and require a staff role on both `POST /books` and
`PATCH /books/{id}`  hiding UI elements alone isn't a security boundary.

**Member selection isn't real authentication.** The frontend stores a member ID in
`localStorage` and sends it along with orders/loans. The API trusts whatever ID shows up, so
anyone can act as another member just by changing it.
*Scope:* beyond the assignment — the contract is explicitly ID-based with no session/token
concept.
*Fix:* add a server-issued session or token, resolve the member from that instead of a raw
client-supplied ID.

**Member verification fails open on server errors.** If the member lookup call errors out, the
frontend can end up continuing on as if that ID were valid anyway not great, even though the
backend would likely reject the request downstream.
*Scope:* resilience/hardening issue, beyond the assignment.
*Fix:* fail closed clear member state and require a successful lookup before allowing any
member action.

**`b@m.com` passes validation, and that's intentional.** The validator follows `SPEC.md`'s exact
regex (`^[^@\s]+@[^@\s]+\.[^@\s]+$`), which does let this through. Not a bug against the spec as
written just worth flagging if the product wants tighter rules later.
*Scope:* out of scope unless stricter validation gets explicitly requested.
*Fix:* nail down the desired policy first, write tests for it, then change the validator on
purpose rather than quietly tightening it and risking a contract break.

**Last-copy race condition.** Order creation checks stock and decrements it in separate ORM
calls with no row locking, so two simultaneous requests could both think they got the last copy.
*Scope:* explicitly optional per `ASSIGNMENT.md`.
*Fix:* PostgreSQL row locking, or an atomic conditional update that checks the affected row
count.

## Architecture and Design Choices

Kept to the layered structure the assignment asked for:

- Routers handle HTTP concerns input parsing, dependencies, response models, status codes
- Pydantic schemas own shape, normalization, and field-level validation
- Services hold the business rules, queries, transaction boundaries, and domain errors
- SQLAlchemy models represent persisted state and relationships
- The frontend calls the API with same-origin relative paths, so the static UI and the API ship
  together without needing a separate frontend service

A few decisions worth explaining:

1. **Validation lives in schemas, integrity handling lives in services.** ISBN and email
   normalization happen up front. The database's uniqueness constraints are still the final
   safety net expected `IntegrityError`s get caught, rolled back, and turned into stable 409s.

2. **Orders are all-or-nothing.** Every member/book/access/stock check happens before any stock
   or order data actually changes. Stock reservation and order-item creation commit together, so
   a multi-book order can't fail halfway through and leave inventory in a weird state.

3. **Order prices are snapshotted at purchase time.** Later catalog price changes don't
   retroactively change what an existing order shows as its total.

4. **Loan status is computed, not stored.** Whether a loan is active/overdue/returned depends on
   the current time, so it's calculated on read. Keeps things from going stale and preserves the
   strict `now > due_at` rule.

5. **Reports use SQL aggregation.** The top-books report groups paid order items directly in the
   database and joins in the current title — keeps the query lean and naturally leaves out books
   with zero paid sales.

6. **Startup init is intentionally simple.** `Base.metadata.create_all()` plus seed data runs on
   startup since the assignment doesn't call for migrations. A real production version would
   want migrations and an idempotent seed process instead.

7. **Synchronous SQLAlchemy, on purpose.** The spec calls for sync SQLAlchemy 2.x, so I stuck
   with that rather than pulling in an async stack for no real benefit here.

## Spec Decisions and Trade-offs

- Email validation follows `SPEC.md`'s regex exactly, which is why something like `b@m.com`
  passes. Tightening this would be a deliberate contract change with its own tests, not a
  silent tweak.
- CORS is wide open (`*`) because that's part of the fixed API contract.
- No `api/index.py` or legacy Vercel routing — current Vercel FastAPI discovery works fine with
  `app/main.py` and a plain `app` instance.
- SQLite stays for local tests; production uses PostgreSQL since serverless local disk isn't
  persistent or shared.
- Concurrent last-copy protection remains a known, optional gap row locking or an atomic
  conditional update would be the next step if it's prioritized.
- There's no authentication contract in the assignment at all. The member selector works fine
  for the demo but shouldn't be mistaken for real login in an actual product.

## Git and Submission

Commits are broken up by feature area — book validation, loan lifecycle, members/orders,
reports, PostgreSQL support, deployment. Generated files, secrets, the local DB, and the venv
are all gitignored.

Checked the public deployment at `/`, `/health`, and `/docs`. Ran the full local suite from the
repo's own virtualenv — 202 passed.

## AI Usage

Used GitHub Copilot to help read through the assignment, spec, and tests; spot incomplete
behavior; suggest fixes; make sense of test failures; sanity-check the architecture; and help
draft deployment/submission notes.

Everything it suggested got checked against the actual source and tests rather than taken at
face value. One place it wasn't helpful: it initially assumed commands could be run from the
parent workspace directory and that `uv` was available in the active PowerShell session neither
was true, so I just used the repo's existing `.venv` and the correct working directory directly.
Final calls like the all-or-nothing order updates and the strict loan due-date boundary were
checked against the spec myself rather than taken from the AI's suggestions as-is.