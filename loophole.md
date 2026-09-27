# Project Loophole Review

Reviewed against `ASSIGNMENT.md`, `INSTRUCTIONS.md`, `SPEC.md`, the FastAPI code, and the frontend.
No application code was changed for this review.

## Quick Summary

The application passes the required assignment tests, but it is not a real authenticated production system. The largest issue is that the frontend has a member selector, not authentication. Anyone who can reach the API can call the write endpoints directly and can identify themselves as any existing member ID.

The `b@m.com` example is accepted intentionally by the current specification. The required pattern is:

```text
^[^@\s]+@[^@\s]+\.[^@\s]+$
```

That pattern accepts `b@m.com`, so this is not a failure against the assigned task. It is only a product-validation improvement if the business wants stricter email addresses.

## Findings

### 1. Book edit is visible before sign-in

**What happens**

The catalog renders an `Edit` button for every book without checking whether a member is selected. The `edit-book` event directly calls `startBookEdit()`, and `saveBookEdit()` sends `PATCH /books/{id}`.

**Impact**

A visitor can edit book price and stock from the UI without signing in. This matches the behavior you found.

**Scope**

**Beyond the required assignment.** `SPEC.md` requires the PATCH endpoint and its validation, but it does not define authentication, an admin role, or a rule that only signed-in users may edit books. The tests therefore check that PATCH works, not who is allowed to use it.

**Simplest improvement**

- Short-term UI improvement: hide or disable `Edit` and `Add book` until a member/admin is selected.
- Correct security fix: add server-side authentication and an admin/ staff permission check to `POST /books` and `PATCH /books/{id}`. UI hiding alone is not security because anyone can call the API directly.

**Suggested priority**: High if this is a public production application; low for the take-home contract.

### 2. Add-book is also available before sign-in

**What happens**

The `Add book` button and create-book form are available on the catalog before a member is selected. The `POST /books` route has no authentication or authorization dependency.

**Impact**

An unauthenticated visitor can add catalog records, including restricted books, by using the UI or sending a direct HTTP request.

**Scope**

**Beyond the required assignment.** The assignment requires book creation but does not specify an admin workflow or authentication system.

**Simplest improvement**

Use the same solution as Finding 1: require a server-verified admin/staff identity on book creation. As a temporary UI-only measure, disable the form until sign-in, but do not treat that as a security control.

**Suggested priority**: High for production.

### 3. The member selector is not authentication

**What happens**

The frontend stores a member ID in `localStorage` and sends that ID in requests such as order creation and loan creation. The backend trusts the submitted `member_id`; there is no password, session, token, or authorization dependency in the routers.

**Impact**

Anyone can submit another member's ID and create an order or loan as that member. Anyone can also request that member's orders, loans, stats, and profile if they know the numeric ID.

**Scope**

**Beyond the required assignment.** `SPEC.md` describes acting as a member by ID and does not require login credentials. This is a missing production security layer, not a failed acceptance requirement.

**Simplest improvement**

Add a real authentication flow: login, a signed session or token, and a server-side dependency that loads the authenticated member. Do not accept the acting member identity solely from a client-supplied body field. For staff actions, add a separate role/permission check.

**Suggested priority**: Critical for production; not required for this assignment.

### 4. Failed member verification fails open

**What happens**

In the frontend sign-in handler, when the member lookup returns a server error, the code calls `setMember(id, null)` and displays that the user is “Acting as this ID anyway.”

**Impact**

If the API is temporarily unavailable or returns a 5xx response, the UI permits actions to continue with an unverified member ID. The backend may then reject the request, but the client-side identity state is still treated as usable. With the current backend's lack of authentication, this makes the trust problem worse.

**Scope**

**Beyond the required assignment.** The spec has no authentication failure behavior. This is still a real resilience/security design problem.

**Simplest improvement**

Fail closed: on any member lookup error other than a successful response, clear the member state and require a retry. Never treat an unverified ID as a signed-in member.

**Suggested priority**: High.

### 5. Stored member ID can be changed by the browser

**What happens**

The selected member ID is stored in `localStorage` under `sanctum.memberId`. A user can edit that value in browser developer tools. On reload, the frontend fetches that ID and uses it as the active member.

**Impact**

This lets a user switch the UI to another existing member without any credential check. The backend currently accepts that identity because it has no authentication layer.

**Scope**

**Beyond the required assignment.** Local storage is acceptable for a demo member selector, but not for authentication.

**Simplest improvement**

Do not use local storage as proof of identity. Store only a server-issued session/token and verify it on the server. Until real auth exists, treat the selector honestly as “choose a demo member,” not “sign in.”

**Suggested priority**: High for production.

### 6. Email validation is intentionally permissive

**What happens**

`b@m.com` is accepted. The current validator strips and lowercases the value, then applies the exact simple regex required by `SPEC.md`. That regex accepts one-character local parts and one-character domain labels.

**Impact**

Some users may expect stricter validation, such as rejecting short or unusual domains. However, the current value is syntactically accepted by the assignment contract.

**Scope**

**Beyond the required assignment if stricter rules are desired.** Changing this behavior would change the stated contract and could reject addresses the assignment currently permits. It should not be changed casually.

**Simplest improvement**

First decide the business rule. If the goal is only a normal-looking email, document a stricter domain policy and add tests. Then use a carefully defined validator. Do not claim that every valid real-world email must pass a simplistic regex. The current implementation is correct for this assignment.

**Suggested priority**: Low for the assignment; medium only if the product needs stricter input rules.

### 7. Client-side cart data is mutable, but the server correctly recalculates

**What happens**

Cart contents, quantities, prices, and stock snapshots live in `localStorage` and browser state. A user can alter them in developer tools.

**Impact**

The UI estimate can be wrong or can request unusual quantities. This does not currently allow a cheaper order because the server recalculates prices, discounts, and stock during `POST /orders`.

**Scope**

**Mostly beyond the required assignment, and currently mitigated.** The spec requires server-side order pricing and stock rules, which the service implements.

**Simplest improvement**

Keep treating the cart as an untrusted draft. Re-fetch current book data before checkout for better user feedback, but continue relying on server-side price and stock validation as the authority.

**Suggested priority**: Low.

### 8. Concurrent last-copy orders can oversell stock

**What happens**

Order creation reads stock, checks it, decrements it, and commits. There is no row lock or atomic conditional update around the stock check.

**Impact**

Two concurrent requests can both observe the last available copy and both create successful orders. The assignment itself lists safe concurrent handling as optional.

**Scope**

**Explicitly beyond the required assignment.** `ASSIGNMENT.md` names concurrent last-copy protection as an optional extra.

**Simplest improvement**

For PostgreSQL, use a transaction with `SELECT ... FOR UPDATE` on the book row, or use an atomic update such as `UPDATE books SET stock = stock - :quantity WHERE id = :id AND stock >= :quantity` and verify the affected row count.

**Suggested priority**: Medium for production inventory.

### 9. Startup seeding can race in a scaled deployment

**What happens**

`seed_if_empty()` checks whether either the books or members table has data and then inserts all seed rows. Two simultaneous startup processes could both see an empty database and attempt to seed.

**Impact**

This can cause duplicate-key failures or partial startup behavior in a multi-instance deployment. It is not normally visible in the single-instance test fixture.

**Scope**

**Beyond the required assignment.** The spec only requires seeding an empty database during startup.

**Simplest improvement**

Run seeding as a one-time deployment task, or make it idempotent and protect it with a database lock/unique-conflict handling. A migration system should also replace `create_all()` for a mature deployment.

**Suggested priority**: Medium for scaled production.

## Things that look suspicious but are currently correct

- `b@m.com` is allowed by the exact email regex in `SPEC.md`.
- The frontend cart can be edited in browser storage, but the server recalculates order totals and validates stock.
- CORS allows all origins, but that is explicitly required by the fixed API contract.
- There is no `GET /members` pagination endpoint, but it is explicitly optional.
- Concurrent last-copy protection is absent, but it is explicitly optional.
- The API exposes book editing without auth, but authentication and roles are not part of the assignment.

## Recommended order of fixes

1. Add real server-side authentication and staff authorization for book creation/editing.
2. Remove the frontend fail-open behavior and never act as an unverified member.
3. Stop treating `localStorage` member IDs as identity; use a server-issued session/token.
4. Decide and document whether stricter email rules are actually wanted.
5. Add inventory concurrency protection before relying on this for real stock.
6. Move seeding and schema changes to controlled deployment/migration steps.

## Final assessment

The examples you found are valid observations. The edit access issue is a genuine security/product loophole, but it is outside the assigned contract because the assignment has no authentication or admin role requirement. The `b@m.com` behavior is not a loophole against the assignment: it is allowed by the exact required regex. The simplest immediate improvements are to hide edit/add controls for the demo UI, fail closed on member lookup errors, and plan real server-side authentication before calling the member selector a sign-in system.
