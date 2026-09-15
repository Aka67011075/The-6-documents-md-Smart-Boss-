# implementation-plan.md — Milestones

This plan is derived from `PRD.md`, `Architecture.md`, and `Agents.md`.

Each milestone is small, independently testable, and should leave the application in a working state. Do not start milestone N+1 until milestone N meets the Definition of Done in `Agents.md`. Update `progress.md` at the end of each milestone.

---

## M0 — Project Skeleton + Database

**Goal:** Create an empty but running local application with the SQLite database connected.

### Tasks
- Scaffold Next.js 14 with App Router, TypeScript strict mode, and Tailwind CSS.
- Connect `better-sqlite3` through `db/client.ts`.
- Create the local database at `data/stock.db`.
- Add forward-only migrations under `db/migrations/`.
- Create the initial schema defined in `schema.md`.
- Create the basic application layout and a placeholder page.
- Add the basic testing setup with Vitest and Testing Library.
- Confirm Thai and English UI structure can be supported.

### Database
Initial migration must create the tables defined by `schema.md`, including:
- `users`
- `sku_types`
- `sku_units`
- `skus`
- `stock_events`
- `customers`
- `sales`
- `sale_items`
- `audit_logs`
- `schema_migrations`

### Test
- Fresh checkout can install dependencies and start the application.
- Migration runs successfully on a new database.
- `npm run build` succeeds.
- No negative stock can be inserted through the database constraints.

**Shippable as:** a running local application with the database structure in place.

---

## M1 — Authentication + Sessions

**Goal:** Users can log in securely and protected routes require authentication.

### Tasks
- Seed one Admin account for development.
- Create `/login`.
- Validate username and password on the server.
- Use bcrypt password hashes.
- Create a secure HTTP-only session cookie.
- Protect application routes from unauthenticated access.
- Add logout.
- Keep authentication local; do not add third-party authentication.

### Test
- Correct credentials create a session.
- Wrong credentials do not create a session.
- Unauthenticated access to protected routes redirects to `/login`.
- Logout removes the active session.

**Shippable as:** a locked-down application with Admin authentication.

---

## M2 — Admin User Management + Permissions

**Goal:** Admin can manage staff accounts while Operators only access permitted functions.

### Tasks
- Create `/admin/users`.
- Allow Admin to create Operator accounts.
- Store password hashes only.
- Allow Admin to deactivate users instead of deleting them.
- Define the Operator responsibility used by the application, such as:
  - `salesperson`
  - `stock`
  - `other`
- Enforce permission checks on the server at both route and action level.
- Keep Admin as the full-access role.

### Test
- Admin can create an Operator.
- Operator can log in.
- Operator cannot open Admin-only user management.
- Deactivated users cannot log in.
- Unauthorized server actions are rejected even if the UI is bypassed.

**Shippable as:** Admin can onboard and control staff accounts securely.

---

## M3 — SKU Management

**Goal:** Build the product/SKU catalog required by stock and sales workflows.

### Tasks
- Seed/manage `sku_types` and `sku_units`.
- Create SKU form with:
  - SKU code
  - Name
  - Description
  - Type
  - Unit
  - Low-stock threshold
  - Stale-check threshold
  - Initial quantity
- Generate a unique QR code reference for each SKU.
- Create `/skus` list page.
- Create `/skus/new`.
- Create `/skus/[id]` detail page.
- Support search.
- Support edit and deactivate operations according to permissions.
- Validate data with Zod on the client and server.

### Test
- Duplicate SKU codes are rejected.
- Negative thresholds are rejected.
- A valid SKU appears in the catalog.
- Initial quantity cannot be negative.
- QR code reference is unique.
- Deactivated SKUs are not treated as active stock items.

**Shippable as:** the system has a usable SKU catalog.

---

## M4 — Stock Update / Weekly Count

**Goal:** Operators can replace the recorded quantity with a verified physical count while preserving history.

### Tasks
- Add an "Update quantity" action to the SKU detail page.
- Server action validates the new quantity.
- Use a database transaction.
- Update `skus.current_quantity`.
- Insert an append-only `stock_events` row with:
  - `type = 'update'`
  - previous quantity
  - new quantity
  - user
  - timestamp
- Update `last_updated_at`.
- Create an `audit_logs` entry.
- Show recent stock history on the SKU detail page.

### Test
- Quantity updates to the entered value.
- Previous and new quantities are correct.
- Negative quantity is rejected.
- Stock event is not editable/deletable.
- Audit log identifies the user and action.

**Shippable as:** weekly stock counting is fully traceable.

---

## M5 — Add Stock / Restock

**Goal:** Staff can record incoming stock without overwriting the existing quantity.

### Tasks
- Add a separate "Add stock" action.
- Validate that the added quantity is greater than zero.
- Calculate:
  `new quantity = current quantity + added quantity`
- Perform the update in a transaction.
- Insert `stock_events` with `type = 'add'`.
- Update `last_updated_at`.
- Create an audit log entry.
- Display update and add events distinctly in stock history.

### Test
- Adding 20 to 50 produces exactly 70.
- The original quantity is preserved in the event.
- Zero/negative additions are rejected.
- Stock can never become negative.

**Shippable as:** the complete stock count + restock workflow is available.

---

## M6 — Sales + Automatic Stock Deduction

**Goal:** Staff can record multi-item sales and stock is deducted automatically and atomically.

### Tasks
- Create customer selection/entry flow.
- Create sales transaction form.
- Support multiple sale items.
- Record:
  - invoice/document number
  - customer
  - salesperson
  - sale timestamp
  - payment information/status
  - sale items
- Calculate item totals:
  `item total = quantity × unit price`
- Calculate the complete sale total on the server.
- Validate that every SKU has sufficient stock.
- Save `sales` and `sale_items`.
- Deduct stock for every sold SKU.
- Create stock events for the deductions.
- Create audit log entries.
- Perform sale creation and all stock deductions in **one database transaction**.

### Test
- A sale can contain multiple items.
- Totals are calculated by the server.
- A sale with insufficient stock is rejected.
- No SKU becomes negative.
- If any part of the transaction fails, the sale and stock changes are rolled back together.
- Successful sales reduce stock by the exact sold quantity.

**Shippable as:** the core sales workflow is live and safely connected to inventory.

---

## M7 — Dashboard + Salesperson Performance

**Goal:** Give CEO/Manager a clear overview of inventory and sales performance.

### Dashboard information
- Total number of SKUs.
- Current inventory status.
- Low-stock items.
- Stale-stock items.
- Total sales.
- Sales by salesperson.
- Sales trends over time.
- Top-selling products.
- Other useful company-level summaries.

### Filters
- Today
- Last 7 days
- This month
- Last month
- Custom date range

### Tasks
- Build `/dashboard`.
- Query real database data.
- Calculate low stock:
  `current_quantity <= low_stock_threshold`
- Calculate stale stock:
  `last_updated_at IS NULL` OR older than `stale_days_threshold`.
- Add charts/graphs appropriate for management review.
- Restrict dashboard information according to permissions.

### Test
- Low-stock items appear at or below their threshold.
- Never-updated SKUs are marked stale.
- Time filters change sales results correctly.
- Salesperson totals match the underlying sales data.
- Operators do not receive unauthorized management information.

**Shippable as:** CEO/Manager can monitor inventory and sales from one dashboard.

---

## M8 — Audit Log + Export + QR + Language

**Goal:** Complete accountability, reporting, and usability requirements.

### Tasks
- Build an Audit Log view for authorized users.
- Record important changes, including:
  - create/edit/deactivate SKU
  - stock update
  - add stock
  - create/modify sale
  - corrections
  - user management
- Never silently delete audit history.
- Add CSV export.
- Add management dashboard/report export when technically feasible.
- Generate and display QR codes for SKUs.
- Allow QR scanning to identify/open the SKU.
- Support Thai and English UI text.

### Test
- Important changes appear in the audit history.
- Audit records contain user, action, affected data, previous value, new value, and timestamp where applicable.
- CSV output contains the expected fields.
- QR code identifies the correct SKU.
- Both Thai and English interfaces render correctly.

**Shippable as:** the system satisfies the main reporting, accountability, QR, and language requirements.

---

## M9 — Polish, Security Review + Release

**Goal:** Prepare the local application for real staff use.

### Tasks
- Add loading and empty states.
- Add clear error handling for failed server actions.
- Review all route-level and action-level permission checks.
- Confirm no stock-changing operation bypasses a transaction.
- Confirm stock can never become negative.
- Confirm audit history is append-only.
- Run a performance check with approximately 500–1,000 seeded SKUs.
- Write/update README setup and local deployment instructions.
- Run the complete user flows from `PRD.md`.
- Run the full test suite and production build.

### Final Test
A fresh machine following only the README can:
1. install dependencies,
2. initialize the database,
3. start the application,
4. log in,
5. create/manage a SKU,
6. update stock,
7. add stock,
8. record a sale,
9. see automatic stock deduction,
10. view dashboard information,
11. review audit history,
12. export data.

**Shippable as:** Smart Boss v1 for local deployment.

---

## Sequencing Notes

- M0 must be completed before any feature milestone.
- M1 must precede protected application workflows.
- M2 must be completed before relying on Operator responsibility permissions.
- M3 must precede stock and sales operations because SKUs must exist first.
- M4 and M5 create the stock history used by the dashboard.
- M6 depends on M3 and the stock transaction rules.
- M7 depends on real stock and sales data; dashboard queries must not use mocked data.
- M8 completes audit, export, QR, and language requirements.
- M9 is hardening and release preparation, not a place to add unrelated features.
- Any new requirement should be proposed in `PRD.md` first and then reflected in the relevant architecture/schema/plan documents.
