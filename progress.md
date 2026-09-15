# progress.md — Project Memory

This file is the shared memory between sessions and between the user and any AI agent working on this repository.

Always read this file at the start of a session. Always append a new session entry at the end; do not delete or rewrite previous entries.

Newest entries are at the top.

---

## How to use this file

Each session should use this format:

```text
## [YYYY-MM-DD] Session N — M<milestone> <short title>

**Status:** in progress | blocked | done

**Done:**
- ...

**Broke / had to fix:**
- ...

**Changed from the plan (and why):**
- ...

**Left for next session:**
- ...
```

Keep entries factual and specific. The next human or AI agent should be able to continue the project using this file together with `PRD.md`, `Architecture.md`, `Agents.md`, `implementation-plan.md`, and `schema.md`.

---

## Current State

- **Milestone:** M0 — Project Skeleton + Database
- **Status:** not started — planning documents are prepared; application code has not started.
- **Blocking issues:** none.
- **Next action:** start M0 by scaffolding the Next.js application and implementing the initial SQLite migration from `schema.md`.

---

## [2026-09-16] Session 0 — Planning Alignment

**Status:** done

**Done:**
- Reviewed the project requirements in `PRD.md`.
- Reviewed the application structure and data flows in `Architecture.md`.
- Reviewed development rules and Definition of Done in `Agents.md`.
- Rebuilt the implementation plan around the actual Stock + Sales Management scope.
- Defined the database model for users, SKU data, stock history, customers, sales, sale items, audit logs, and migration tracking.
- Added milestone dependencies so database, authentication, SKU, stock, sales, dashboard, audit, export, QR, and language work are implemented in a safe order.

**Broke / had to fix:**
- The previous planning draft contained forecasting-oriented concepts that were not part of the project's source requirements.
- The previous schema concept did not fully cover the sales, customer, and audit requirements in `PRD.md`.
- The corrected plan removes unsupported forecasting entities and aligns the data model with the actual project scope.

**Changed from the plan (and why):**
- The project is treated as a Stock + Sales Management System, not a sales forecasting system.
- Sales and stock deduction are planned as one database transaction because this is a core project rule.
- Audit history is append-only and must not be silently deleted.
- SKU code, description, sales records, sale items, customer information, and audit records are included because they are required by the project documents.

**Left for next session:**
- Start M0 — Project Skeleton + Database.
- Resolve the remaining schema open questions before implementation:
  1. Whether any SKU units require fractional quantities.
  2. Whether large decreases during a stock `update` require a reason/note.
  3. Confirm the exact Operator responsibility categories and whether Operators may create SKUs.
- Keep `schema.md`, `implementation-plan.md`, and `progress.md` synchronized with approved project changes.
