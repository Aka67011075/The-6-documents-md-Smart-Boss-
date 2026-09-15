# schema.md — Data Model

This document is the source of truth for the database shape.

If the database schema changes in code, this file must be updated in the same change. AI agents must not invent tables, columns, or relationships beyond what is defined here; propose schema changes here first.

**Database engine:** SQLite via `better-sqlite3`  
**Database file:** `data/stock.db` (gitignored)  
**Timestamp format:** `TEXT`, ISO-8601 UTC, for example `2026-09-16T12:30:00Z`  
**Naming:** snake_case for tables and columns  
**Boolean values:** SQLite `INTEGER`, using `0` / `1`

For quantities, use `INTEGER` for whole-unit stock. `REAL` should only be introduced if fractional SKU quantities are explicitly confirmed as necessary.

---

## 1. Entity-Relationship Summary

```text
users
  │
  ├──────────────< stock_events >────────────── skus
  │                                             │
  ├──────────────< audit_logs                   ├── sku_types
  │                                             └── sku_units
  │
  ├──────────────< sales >────────────── customers
  │                     │
  │                     └──────────────< sale_items >──────────── skus
  │
  └──────────────< audit_logs
```

### Main relationships

- One `user` can create many `skus`.
- One `user` can create many `stock_events`.
- One `sku` can have many `stock_events`.
- One `customer` can have many `sales`.
- One `user` can be the salesperson for many `sales`.
- One `sale` contains one or more `sale_items`.
- One `sku` can appear in many `sale_items`.
- One `user` can create many `audit_logs`.
- `sku_types` and `sku_units` are lookup tables referenced by `skus`.

---

## 2. Tables

### 2.1 `users`

Stores all accounts that can access the system.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `name` | TEXT | NOT NULL | Display name |
| `username` | TEXT | NOT NULL, UNIQUE | Login identifier |
| `password_hash` | TEXT | NOT NULL | bcrypt hash; never store plaintext |
| `role` | TEXT | NOT NULL, CHECK (`role` IN ('admin','operator')) | Admin = CEO/Manager |
| `responsibility` | TEXT | NULL | Operator responsibility, e.g. `salesperson`, `stock`, `other` |
| `is_active` | INTEGER | NOT NULL DEFAULT 1 | 0/1; deactivate instead of delete |
| `created_at` | TEXT | NOT NULL | ISO-8601 UTC |

**Indexes**
- `UNIQUE INDEX idx_users_username ON users(username)`
- `INDEX idx_users_role ON users(role)`
- `INDEX idx_users_responsibility ON users(responsibility)`

---

### 2.2 `sku_types`

Lookup table for configurable SKU types.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `name` | TEXT | NOT NULL, UNIQUE | Example: Equipment, Machine, Material |

---

### 2.3 `sku_units`

Lookup table for configurable SKU units.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `name` | TEXT | NOT NULL, UNIQUE | Example: unit, meter, set |

---

### 2.4 `skus`

Stores products/items managed by the inventory system.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `sku_code` | TEXT | NOT NULL, UNIQUE | Human/business SKU code |
| `name` | TEXT | NOT NULL | SKU/product name |
| `description` | TEXT | NULL | Product description |
| `type_id` | INTEGER | NOT NULL, FOREIGN KEY → `sku_types(id)` | |
| `unit_id` | INTEGER | NOT NULL, FOREIGN KEY → `sku_units(id)` | |
| `low_stock_threshold` | INTEGER | NOT NULL, CHECK (`low_stock_threshold` >= 0) | Low-stock warning threshold |
| `stale_days_threshold` | INTEGER | NOT NULL, CHECK (`stale_days_threshold` > 0) | Days before stale warning |
| `current_quantity` | INTEGER | NOT NULL DEFAULT 0, CHECK (`current_quantity` >= 0) | Current inventory quantity |
| `qr_code_ref` | TEXT | NOT NULL, UNIQUE | QR path/identifier |
| `last_updated_at` | TEXT | NULL | Last stock-changing event; NULL means never updated |
| `is_active` | INTEGER | NOT NULL DEFAULT 1 | 0/1; deactivate instead of delete |
| `created_at` | TEXT | NOT NULL | ISO-8601 UTC |
| `created_by` | INTEGER | NOT NULL, FOREIGN KEY → `users(id)` | User who created the SKU |

**Indexes**
- `UNIQUE INDEX idx_skus_code ON skus(sku_code)`
- `INDEX idx_skus_type ON skus(type_id)`
- `INDEX idx_skus_low_stock ON skus(current_quantity, low_stock_threshold)`
- `INDEX idx_skus_last_updated ON skus(last_updated_at)`
- `INDEX idx_skus_active ON skus(is_active)`

---

### 2.5 `stock_events`

Append-only history of every inventory quantity change.

This table must never be updated or deleted during normal operation.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `sku_id` | INTEGER | NOT NULL, FOREIGN KEY → `skus(id)` | Affected SKU |
| `user_id` | INTEGER | NOT NULL, FOREIGN KEY → `users(id)` | User who caused the change |
| `type` | TEXT | NOT NULL, CHECK (`type` IN ('update','add','sale')) | `update` = count/correction, `add` = restock, `sale` = automatic deduction |
| `quantity_before` | INTEGER | NOT NULL, CHECK (`quantity_before` >= 0) | Quantity before change |
| `quantity_after` | INTEGER | NOT NULL, CHECK (`quantity_after` >= 0) | Quantity after change |
| `created_at` | TEXT | NOT NULL | ISO-8601 UTC |
| `sale_id` | INTEGER | NULL, FOREIGN KEY → `sales(id)` | Set when event was caused by a sale |

**Indexes**
- `INDEX idx_stock_events_sku ON stock_events(sku_id, created_at)`
- `INDEX idx_stock_events_user ON stock_events(user_id, created_at)`
- `INDEX idx_stock_events_sale ON stock_events(sale_id)`

**Rules**
- Append-only.
- `quantity_after` can never be negative.
- Every stock-changing operation must use a transaction.
- `sale` events are created in the same transaction as the related sale.

---

### 2.6 `customers`

Stores customer information used by sales transactions.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `name` | TEXT | NOT NULL | Customer name |
| `created_at` | TEXT | NOT NULL | ISO-8601 UTC |
| `is_active` | INTEGER | NOT NULL DEFAULT 1 | 0/1; preserve historical sales |

---

### 2.7 `sales`

Stores the header of each completed sales transaction.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `document_number` | TEXT | NOT NULL, UNIQUE | Invoice/document number |
| `customer_id` | INTEGER | NULL, FOREIGN KEY → `customers(id)` | Customer reference |
| `customer_name` | TEXT | NOT NULL | Snapshot of customer name at sale time |
| `salesperson_id` | INTEGER | NOT NULL, FOREIGN KEY → `users(id)` | Operator responsible for the sale |
| `total_amount` | REAL | NOT NULL, CHECK (`total_amount` >= 0) | Calculated by server |
| `payment_status` | TEXT | NOT NULL | Payment state; exact allowed values can be finalized before implementation |
| `payment_information` | TEXT | NULL | Additional payment information |
| `created_at` | TEXT | NOT NULL | System-generated sale timestamp |
| `created_by` | INTEGER | NOT NULL, FOREIGN KEY → `users(id)` | User who entered the sale |

**Indexes**
- `UNIQUE INDEX idx_sales_document_number ON sales(document_number)`
- `INDEX idx_sales_customer ON sales(customer_id)`
- `INDEX idx_sales_salesperson_date ON sales(salesperson_id, created_at)`
- `INDEX idx_sales_created_at ON sales(created_at)`

**Rules**
- Sale date/time is generated by the system.
- `total_amount` is calculated from `sale_items`; users must not manually override the calculated total.
- A sale must contain at least one `sale_item`.
- Creating a sale and deducting stock must occur in one database transaction.

---

### 2.8 `sale_items`

Stores individual products/items within a sale.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `sale_id` | INTEGER | NOT NULL, FOREIGN KEY → `sales(id)` | Parent sale |
| `sku_id` | INTEGER | NOT NULL, FOREIGN KEY → `skus(id)` | Sold SKU |
| `product_description` | TEXT | NOT NULL | Snapshot of product description/name |
| `quantity` | INTEGER | NOT NULL, CHECK (`quantity` > 0) | Quantity sold |
| `unit` | TEXT | NOT NULL | Snapshot of SKU unit |
| `unit_price` | REAL | NOT NULL, CHECK (`unit_price` >= 0) | Price per unit |
| `item_total` | REAL | NOT NULL, CHECK (`item_total` >= 0) | Calculated as quantity × unit_price |

**Indexes**
- `INDEX idx_sale_items_sale ON sale_items(sale_id)`
- `INDEX idx_sale_items_sku ON sale_items(sku_id)`

**Rules**
- `item_total = quantity × unit_price`.
- The server calculates and validates the value.
- A sale item cannot request more stock than the SKU currently has.

---

### 2.9 `audit_logs`

Append-only accountability history for important system changes.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY AUTOINCREMENT | |
| `user_id` | INTEGER | NOT NULL, FOREIGN KEY → `users(id)` | User who performed the action |
| `action` | TEXT | NOT NULL | Example: `create_sku`, `update_stock`, `add_stock`, `create_sale` |
| `entity_type` | TEXT | NOT NULL | Example: `sku`, `sale`, `user` |
| `entity_id` | INTEGER | NOT NULL | Affected record |
| `previous_value` | TEXT | NULL | Serialized previous value when applicable |
| `new_value` | TEXT | NULL | Serialized new value when applicable |
| `created_at` | TEXT | NOT NULL | ISO-8601 UTC |

**Indexes**
- `INDEX idx_audit_logs_user ON audit_logs(user_id, created_at)`
- `INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id, created_at)`
- `INDEX idx_audit_logs_created_at ON audit_logs(created_at)`

**Rules**
- Audit records are append-only.
- Do not silently delete or overwrite audit history.
- Important stock, sales, SKU, and user-management changes must be recorded.

---

### 2.10 `schema_migrations`

Tracks which forward-only database migrations have already been applied.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | INTEGER | PRIMARY KEY | Migration order |
| `name` | TEXT | NOT NULL, UNIQUE | Migration filename |
| `applied_at` | TEXT | NOT NULL | ISO-8601 UTC |

---

## 3. Relationships and Delete Rules

| From | To | Type | On delete |
|---|---|---|---|
| `skus.type_id` | `sku_types.id` | many-to-one | RESTRICT |
| `skus.unit_id` | `sku_units.id` | many-to-one | RESTRICT |
| `skus.created_by` | `users.id` | many-to-one | RESTRICT |
| `stock_events.sku_id` | `skus.id` | many-to-one | RESTRICT/CASCADE only for exceptional hard-delete |
| `stock_events.user_id` | `users.id` | many-to-one | RESTRICT |
| `stock_events.sale_id` | `sales.id` | many-to-one | RESTRICT |
| `sales.customer_id` | `customers.id` | many-to-one | SET NULL |
| `sales.salesperson_id` | `users.id` | many-to-one | RESTRICT |
| `sales.created_by` | `users.id` | many-to-one | RESTRICT |
| `sale_items.sale_id` | `sales.id` | many-to-one | CASCADE |
| `sale_items.sku_id` | `skus.id` | many-to-one | RESTRICT |
| `audit_logs.user_id` | `users.id` | many-to-one | RESTRICT |

### General rule

Prefer soft deletion using `is_active = 0` for users, SKUs, and customers. Do not hard-delete records that are required to preserve historical stock, sales, or audit information.

---

## 4. Derived / Computed Values

These values are calculated when needed and are not stored as separate columns.

### Low stock

```text
is_low_stock =
  current_quantity <= low_stock_threshold
```

### Stale stock

```text
is_stale =
  last_updated_at IS NULL
  OR
  (current_time - last_updated_at) > stale_days_threshold days
```

A SKU with `last_updated_at IS NULL` is considered stale.

### Sale item total

```text
item_total = quantity × unit_price
```

### Sale total

```text
total_amount = SUM(sale_items.item_total)
```

### Stock after sale

```text
new_quantity = current_quantity - sold_quantity
```

The transaction must reject the sale if the result would be negative.

---

## 5. Transaction Rules

### Stock update

```text
BEGIN
  read current quantity
  validate new quantity >= 0
  update skus.current_quantity
  insert stock_events
  insert audit_logs
  update skus.last_updated_at
COMMIT
```

### Add stock

```text
BEGIN
  read current quantity
  validate added quantity > 0
  calculate new quantity
  update skus.current_quantity
  insert stock_events
  insert audit_logs
  update skus.last_updated_at
COMMIT
```

### Record sale

```text
BEGIN
  validate sale and all sale items
  read stock for every SKU
  reject if any SKU has insufficient stock
  calculate item totals and sale total
  insert sales
  insert sale_items
  deduct stock for every SKU
  insert stock_events with type = 'sale'
  insert audit_logs
  update last_updated_at for affected SKUs
COMMIT
```

If any step fails, the whole transaction must roll back.

---

## 6. Migrations

Use numbered, forward-only SQL files in `db/migrations/`.

Suggested initial migrations:

```text
001_init.sql
  -- schema_migrations
  -- users
  -- sku_types
  -- sku_units
  -- skus
  -- stock_events
  -- customers
  -- sales
  -- sale_items
  -- audit_logs
  -- indexes

002_seed_lookups.sql
  -- starter sku_types
  -- starter sku_units
```

Each migration is applied once and recorded in `schema_migrations`.

---

## 7. Open Questions

These items should be resolved before the affected implementation is finalized.

1. **Fractional quantities:** Do any SKU units require values such as `2.5 meters`? If yes, quantity columns will need to change from `INTEGER` to an appropriate fractional representation.
2. **Large stock corrections:** Should a large decrease during a normal `update` require a reason/note?
3. **Operator SKU creation:** Can Operators create SKUs, or is SKU creation Admin-only?
4. **Operator responsibilities:** Confirm the exact responsibility categories and whether one Operator can have more than one responsibility.
5. **Payment status values:** Confirm the allowed payment states, such as `pending`, `paid`, `partial`, or `cancelled`.

No new table or column should be added solely to answer these questions until the project requirements are approved and this file is updated.
