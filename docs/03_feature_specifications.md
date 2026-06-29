# Feature Specifications
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Document Type:** Product Feature Catalogue
**Version:** MVP 1.0

---

## Feature Index

| # | Feature | Surface | Priority |
|---|---|---|---|
| F-01 | Multi-Tenant Company Onboarding | Web | P0 |
| F-02 | Outlet & Branch Management | Web | P0 |
| F-03 | Device Registry & Token Management | Web | P0 |
| F-04 | Role-Based Access Control | Web + Mobile | P0 |
| F-05 | Offline POS — Sales & Refills | Mobile | P0 |
| F-06 | Offline POS — Cylinder Transactions | Mobile | P0 |
| F-07 | Shift Blind Close & Sync Bundle Upload | Mobile | P0 |
| F-08 | Anti-Tamper Clock Validation | Mobile | P0 |
| F-09 | Bulk Gas Inventory Reconciliation | Web + Mobile | P0 |
| F-10 | Cylinder Asset Matrix | Web + Mobile | P0 |
| F-11 | Inter-Branch Stock Transfers | Web + Mobile | P1 |
| F-12 | Shrinkage Categorization Engine | Backend | P0 |
| F-13 | Variance Escalation Workflow | Web + Mobile | P1 |
| F-14 | CVP Analytics Dashboard | Web | P1 |
| F-15 | Break-Even Progress Ring | Web | P1 |
| F-16 | Shift Variance Breakdown Report | Web | P1 |
| F-17 | Corporate Multi-Outlet Analytics | Web | P2 |
| F-18 | Barcode / Serial Cylinder Scanning | Mobile | P1 |
| F-19 | Expense Period Management | Web | P1 |
| F-20 | Sync Receipt Validation | Mobile + Backend | P0 |

---

## Detailed Feature Specifications

---

### F-01 — Multi-Tenant Company Onboarding

**Surface:** Web Dashboard
**Role:** System Owner (platform-level superadmin)

**Description:**
Provision a new company (tenant) on the platform. Each company operates in complete data isolation from all other tenants via PostgreSQL Row-Level Security.

**Acceptance Criteria:**
- System Owner can create a new company record with: `tenant_name`, `base_currency`, and `global_configurations` JSONB.
- On company creation, a root System Owner user account is provisioned and credentials are emailed to the registrant.
- A company cannot be deleted if it has active outlets with unsynced shift data.
- `global_configurations` accepts: default shrinkage tolerance, default variance threshold, allowed payment channels.

**Out of Scope (MVP):**
- Self-service registration flow
- Billing / subscription management

---

### F-02 — Outlet & Branch Management

**Surface:** Web Dashboard
**Role:** System Owner

**Description:**
Create and configure physically isolated branch locations (outlets) under a company tenant.

**Acceptance Criteria:**
- System Owner can create outlets with: `branch_name`, `base_price_per_kg`, `max_manager_variance_usd`, `shrinkage_tolerance_pct`.
- On outlet creation, an initial `inventory_snapshots` record must be created to anchor the opening balance (bulk mass kg, full/empty cylinder counts).
- Outlet Manager is scoped to a single outlet and cannot view data from sibling outlets.
- System Owner and Inventory Controller can view all outlets under their tenant.
- Outlet deletion is blocked if unresolved `variance_escalations` exist.

---

### F-03 — Device Registry & Token Management

**Surface:** Web Dashboard
**Role:** System Owner

**Description:**
Register mobile devices by IMEI before they are permitted to open shifts or submit sync bundles. Supports token revocation for lost or compromised devices.

**Acceptance Criteria:**
- System Owner inputs a device IMEI and associates it with an outlet. The system generates a `device_token`.
- Only registered, active devices may open shifts or submit bundles. Unregistered devices receive `DEVICE_001` error.
- System Owner can revoke a device token. Revoked devices receive `DEVICE_002` on next sync attempt.
- Device list view shows: IMEI, assigned outlet, registration date, last sync timestamp, active/revoked status.
- A single device may only be associated with one outlet at a time.

---

### F-04 — Role-Based Access Control

**Surface:** Web + Mobile
**Roles:** All

**Description:**
Enforce the permission matrix defined in the technical spec across all surfaces and API endpoints.

**Acceptance Criteria:**
- Four roles are supported: System Owner, Inventory Controller, Outlet Manager, Cashier.
- Role is embedded in the JWT payload and validated server-side on every API call.
- Frontend routes and UI elements are conditionally rendered based on role.
- Unauthorized API calls return a structured `403` with `AUTH_003` error code and the list of required roles.
- Role assignment is managed by System Owner only.

---

### F-05 — Offline POS — Sales & Refills

**Surface:** Mobile Client
**Role:** Cashier, Outlet Manager

**Description:**
The primary point-of-sale interface. Operates in complete network isolation. All transactions are committed to the local SQLite database only.

**Acceptance Criteria:**
- Cashier inputs: cylinder size (or custom mass in kg) and payment method (CASH, MOBILE_MONEY, CARD).
- The app calculates the total using the locally cached `base_price_per_kg`.
- Each transaction is assigned a `client_uuid` = `Device_ID + Cashier_ID + SequenceNumber`.
- Transactions are immediately persisted to `local_sales` SQLite table.
- The UI does not require network access at any point during a sale.
- Transaction count and running total are visible on the cashier home screen during the shift.

---

### F-06 — Offline POS — Cylinder Transactions

**Surface:** Mobile Client
**Role:** Cashier, Outlet Manager

**Description:**
Route all cylinder-related transactions through the three-type matrix (Refill, New Purchase, Cross-Fill). Track cylinder serial numbers via barcode scan.

**Acceptance Criteria:**
- On each sale, the cashier selects transaction type: REFILL, NEW_PURCHASE, or CROSS_FILL.
- For CROSS_FILL, cashier must input or scan the competitor brand name.
- For NEW_PURCHASE, a deposit liability amount is calculated and displayed.
- Cylinder serial number can be entered manually or captured via camera barcode scan.
- All cylinder transaction data is stored in `local_cylinder_txns` SQLite table linked to the parent sale's `client_uuid`.

---

### F-07 — Shift Blind Close & Sync Bundle Upload

**Surface:** Mobile Client
**Role:** Cashier + Outlet Manager (co-sign required)

**Description:**
The end-of-shift close workflow. Designed to prevent collusion by eliminating cashier visibility into expected ledger totals.

**Acceptance Criteria:**
- Cashier taps "Initiate Knock-Off." All POS inputs freeze immediately.
- The blind count UI presents denomination entry fields only. Expected totals are never displayed.
- Cashier enters physical cash by denomination, card receipt total, and mobile money token amounts per channel.
- Outlet Manager co-signs via PIN or biometric confirmation.
- App assembles, compresses, and AES-256-GCM encrypts the shift bundle.
- Bundle is uploaded to `POST /api/v1/sync/bundle`.
- On success receipt, app validates `accepted_count` and `receipt_hash` against local sequence.
- Local cache is cleared to read-only only after successful hash validation.
- If upload fails after 5 retry attempts, shift is marked `FAILED` and manager is alerted.

---

### F-08 — Anti-Tamper Clock Validation

**Surface:** Mobile Client
**Role:** All

**Description:**
Prevent back-dating of transactions by detecting device clock manipulation.

**Acceptance Criteria:**
- On shift open, the app fetches a server timestamp via a lightweight `GET /api/v1/time` endpoint (requires network; if offline, uses last-known network time cached at app launch).
- The app computes the delta between network time and uptime-derived time.
- If delta exceeds 5 minutes, the shift cannot be opened.
- A "Clock Integrity Failure" alert is displayed, requiring manager override with a logged justification note.
- The justification note is included in the sync bundle for server-side audit review.
- This check is also performed server-side during bundle ingestion (`SYNC_002` error on failure).

---

### F-09 — Bulk Gas Inventory Reconciliation

**Surface:** Web Dashboard + Mobile
**Role:** Outlet Manager, Inventory Controller, System Owner

**Description:**
Track theoretical vs physical bulk gas stock. Surface discrepancies and trigger the shrinkage categorization engine.

**Acceptance Criteria:**
- Outlet Manager inputs physical gauge readings twice daily via a gauge log form.
- The system calculates Theoretical Stock using the delta-accumulation formula anchored to the latest `inventory_snapshots` record.
- The reconciliation view displays: Theoretical Stock, Physical Stock, Delta, Shrinkage Category.
- If `UNEXPLAINED_LOSS` is detected, a stock lock is applied to the outlet (`INV_001` error on further self-adjustments) until an administrator reviews.
- Stock lock can only be lifted by System Owner or Inventory Controller with a mandatory resolution note.

---

### F-10 — Cylinder Asset Matrix

**Surface:** Web Dashboard
**Role:** Outlet Manager, Inventory Controller, System Owner

**Description:**
Maintain real-time counts of full, empty, and third-party cylinders per outlet.

**Acceptance Criteria:**
- Cylinder counts are updated by synced `cylinder_transactions` records.
- Outlet view shows current count of: full (own-brand), empty (own-brand), third-party by brand.
- Cylinder deposit liabilities are surfaced as a balance sheet line item per outlet.
- Transaction history is filterable by type (REFILL, NEW_PURCHASE, CROSS_FILL) and date range.

---

### F-11 — Inter-Branch Stock Transfers

**Surface:** Web Dashboard + Mobile
**Role:** Outlet Manager (initiate), Outlet Manager at destination (acknowledge)

**Description:**
Move bulk gas inventory between outlets under the same company tenant. Enforces a state machine to prevent inventory duplication.

**Acceptance Criteria:**
- Source outlet initiates a transfer specifying destination outlet and mass (kg). Status: `DISPATCHED`.
- Transfer appears as `IN_TRANSIT` to both outlets.
- Destination outlet manager acknowledges receipt. Status: `ACKNOWLEDGED`. Inventory delta is applied to destination.
- If not acknowledged within 48 hours, transfer is flagged (`FLAGGED`) and escalated to System Owner.
- A dispatched transfer decrements the source outlet's theoretical stock immediately on server processing.
- Acknowledgment increments destination outlet's theoretical stock via a `STOCK_RECEIPT` inventory log.

---

### F-12 — Shrinkage Categorization Engine

**Surface:** Backend (automated)

**Description:**
Automatically classify inventory discrepancies as natural or unexplained based on configurable tolerance thresholds.

**Acceptance Criteria:**
- On each sync bundle ingestion, the engine compares theoretical vs physical stock.
- Discrepancy within `shrinkage_tolerance_pct` → `NATURAL_SHRINKAGE` write-off. No alert generated.
- Discrepancy exceeding tolerance → `UNEXPLAINED_LOSS` flag. Stock lock applied. Escalation created.
- If two bundles from the same outlet on the same calendar day both contain shrinkage logs, totals are summed before threshold evaluation.
- The tolerance percentage is outlet-level configurable by System Owner.

---

### F-13 — Variance Escalation Workflow

**Surface:** Web Dashboard + Mobile (alert)
**Role:** System Owner, Inventory Controller (resolve); Outlet Manager (view own)

**Description:**
Manage shift payment channel discrepancies that exceed the configured variance threshold.

**Acceptance Criteria:**
- Escalations are created automatically during sync bundle processing.
- Each escalation record contains: outlet, shift, payment channel, variance amount.
- System Owner and Inventory Controllers receive an in-app notification and see the escalation in a queue.
- Escalations can be resolved (`RESOLVED`) with mandatory notes, or written off (`WRITTEN_OFF`) with justification.
- An escalation in `PENDING_REVIEW` state does not block outlet operations — it only locks manual adjustment of that shift's ledger.
- Outlet Manager can view escalations for their outlet only.

---

### F-14 — CVP Analytics Dashboard

**Surface:** Web Dashboard
**Role:** Outlet Manager, Inventory Controller, System Owner

**Description:**
Display a monthly Cost-Volume-Profit analysis per outlet, helping owner-operators understand their break-even position in real time (post-sync).

**Acceptance Criteria:**
- Dashboard displays: CM per kg, BEP (monthly kg), Accumulated Volume Sold (month to date), Remaining BEP Units.
- Data reflects the most recent synced state (not live within a shift).
- Fixed and variable cost inputs are sourced from `outlet_expenses` filtered by current `expense_period`.
- If no expenses are entered for the current period, a prompt is shown to configure costs.
- Figures update within one sync cycle of a new expense being added.

---

### F-15 — Break-Even Progress Ring

**Surface:** Web Dashboard
**Role:** Outlet Manager, Inventory Controller, System Owner

**Description:**
A high-impact visual indicator of monthly BEP progress. The primary at-a-glance KPI for outlet managers.

**Acceptance Criteria:**
- SVG radial arc fills from 0% (month start) toward 100% (BEP reached) and beyond (profit zone).
- Color zones: Red (0% to BEP), Green (BEP onward).
- Center label alternates between "X kg to break-even" and "+X kg profit surplus" depending on position.
- Animated fill on page load (300 ms ease-in-out transition).
- Responsive — renders correctly on tablet and desktop viewport widths.

---

### F-16 — Shift Variance Breakdown Report

**Surface:** Web Dashboard
**Role:** Outlet Manager, Inventory Controller, System Owner

**Description:**
Per-shift tabular breakdown of payment channel variances. The primary fraud detection view.

**Acceptance Criteria:**
- Table shows one row per shift with columns: Date, Cashier Name, CASH variance, MOBILE_MONEY variance, CARD variance, Total Gross Revenue, Escalation Status.
- Zero-variance channels are shown in green. Exceeding-tolerance channels are shown in red with escalation badge.
- Rows are filterable by date range, cashier, and escalation status.
- CSV export button produces a downloadable report of the filtered view.
- Outlet Manager sees shifts for their outlet only. Inventory Controller and System Owner can switch between outlets.

---

### F-17 — Corporate Multi-Outlet Analytics

**Surface:** Web Dashboard
**Role:** System Owner, Inventory Controller

**Description:**
Aggregated performance view across all outlets under the tenant for corporate decision-making.

**Acceptance Criteria:**
- Dashboard shows: total revenue (all outlets), total kg sold, top-performing outlet by CM, lowest-stock outlet.
- Revenue trend chart (Recharts LineChart) showing last 30 days across outlets.
- Filterable by date range and outlet subset.
- Data is read-only and derived from synced records only.

---

### F-18 — Barcode / Serial Cylinder Scanning

**Surface:** Mobile Client
**Role:** Cashier, Outlet Manager

**Description:**
Use the device camera to scan cylinder barcodes or QR codes to auto-populate serial numbers during transactions.

**Acceptance Criteria:**
- Scan button is accessible from the cylinder transaction input screen.
- Camera opens within 500 ms of button tap.
- Successfully scanned serial number is auto-populated into the `cylinder_serial` field.
- If no barcode is detected after 10 seconds, a manual entry fallback prompt is shown.
- Supports Code 128, QR Code, and EAN-13 formats.

---

### F-19 — Expense Period Management

**Surface:** Web Dashboard
**Role:** Outlet Manager, System Owner

**Description:**
Manage monthly fixed and variable cost inputs for CVP calculations, scoped to calendar month periods.

**Acceptance Criteria:**
- Expense form requires selection of `expense_period` (month picker, default: current month).
- Fixed cost categories available: Rent, Payroll, Generator Fuel, Site Utilities, Other.
- Variable cost categories available: Wholesale Gas Cost per KG, Delivery Surcharge, Mobile Processor Fee, Other.
- "Clone from last month" button copies prior period's expense set as the starting point for the new period.
- Editing a prior period's expense triggers a recalculation note: "Historical CVP data for [period] will be updated on next refresh."
- Expenses cannot be deleted once a shift has been synced against that period (audit integrity).

---

### F-20 — Sync Receipt Validation

**Surface:** Mobile Client + Backend
**Role:** System (automated)

**Description:**
Cryptographic handshake between mobile client and backend to confirm zero data loss after bundle ingestion.

**Acceptance Criteria:**
- Backend returns `{ accepted_count: N, receipt_hash: "SHA-256" }` where hash is computed over sorted `client_uuid` list.
- Mobile client independently computes the same hash over its local sequence for the shift.
- If `accepted_count` and `receipt_hash` both match: local cache is cleared and shift status is set to `SYNCED`.
- If either value mismatches: local cache is preserved in `PENDING` state, manager is alerted, and the bundle is queued for re-upload on next network connection.
- This validation is logged server-side for audit purposes.
