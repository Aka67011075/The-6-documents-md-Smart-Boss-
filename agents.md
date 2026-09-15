# agents.md — Rules for AI Coding Agents

This file defines the rules for AI agents and contributors working in this
repository.

Related docs: `prd.md`, `architecture.md`, `schema.md`,
`implementation-plan.md`, `progress.md`.

agents.md — Rules for developing the system

Agents defines the rules that developers or AI coding agents must follow when working on the project.

For example:

Use Next.js, TypeScript, Tailwind CSS, and SQLite.
Stock must never become negative.
Sales must automatically deduct stock.
Sale recording and stock deduction must happen safely together.
Important changes must be recorded in the Audit Log.
User permissions must always be checked.
Do not change the technology or database without approval.
Code must be tested and the project must build successfully.

In short: agents.md = "What rules must we follow when writing the code?"

---

## 1. Standing Conventions

- Use Next.js 14 with App Router.
- Use TypeScript with strict mode.
- Use Tailwind CSS for styling.
- Use `better-sqlite3` with a single local SQLite database.
- Use functional React components.
- Use Server Components by default.
- Use `"use client"` only when necessary.
- Use Server Actions or Route Handlers for data mutations.
- Store dates and timestamps in UTC ISO-8601 format.
- Use kebab-case for files and snake_case for database tables/columns.
- Every stock-changing database operation must use a transaction.
- The application must support Thai and English.

---

## 2. Forbidden Actions

The agent must not:

- Add cloud hosting or an external database.
- Replace SQLite or `better-sqlite3`.
- Add third-party authentication.
- Add email, SMS, or push notifications.
- Allow stock quantity to become negative.
- Delete or overwrite audit history.
- Change requirements in `prd.md` without approval.
- Add database tables or fields without updating `schema.md`.
- Ignore user permission checks.

---

## 3. Stock and Sales Rules

- Stock quantity must never be negative.
- Stock updates must record the previous and new quantity.
- Adding stock increases the current quantity.
- Recording a sale automatically decreases stock.
- Sale and stock deduction must be handled in one database transaction.
- Important stock and sales changes must be recorded in the audit log.
- Audit logs must record the user, action, previous value, new value, and time.

---

## 4. Roles and Permissions

- There are two main access levels: Admin and Operator.
- Admin represents the CEO/Manager and can access all system information.
- Operators can only access functions related to their assigned responsibility.
- Permission checks must be performed on the server.

---

## 5. Definition of Done

A task is complete when:

1. The application builds successfully.
2. Relevant functionality has been tested.
3. The implementation matches `prd.md`.
4. The feature works through the actual UI.
5. Database changes are reflected in `schema.md`.
6. `progress.md` is updated.
7. No new errors are introduced.

---

## 6. Session Rules

At the start of a session:

- Read `progress.md`.
- Check the current milestone in `implementation-plan.md`.
- Review relevant requirements in `prd.md`.

At the end of a session:

- Update `progress.md`.
- Record completed work and remaining work.
- Leave the repository in a working state.

=======================================================================================================================================================================

