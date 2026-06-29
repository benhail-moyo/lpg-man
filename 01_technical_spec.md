# Technical Product Feature Specifications (MVP)
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Target Audience:** Engineering, Architecture, and QA Teams
**Document Status:** Approved for Development — Rev 2 (Gaps Resolved)
**Last Revised:** 2026-06-29

---

## 1. System Architecture & Philosophy

The platform is architected as a mobile-first, cloud-based ERP and Retail Management platform optimized for downstream Liquefied Petroleum Gas (LPG) retailers.

To mitigate infrastructure costs and maintain stability in regions suffering from erratic network coverage and power grids, the MVP abandons real-time WebSocket synchronization in favour of an **Offline-First Batch-Processing Architecture**.

---

### 1.1 The Connected Edge Batch Engine

- **Zero-Connectivity Run-time:** The mobile client executes all business logic, local data validation, and transaction tracking in complete isolation from the network. Data is committed locally to an encapsulated SQLite database on the mobile device via the Expo SQLite API.
- **The "Knock-Off" Sync Bundle:** When a cashier triggers the shift closing sequence, the application packages the entire shift ledger (sales, inventory adjustments, cylinder transactions, and the cashier's blind count) into a single compressed, AES-256-GCM encrypted JSON payload.
- **Idempotency & Handshake Protocol:** Every transaction is tagged with a client-generated UUID composed of `Device_ID + Cashier_ID + Sequence_Number`. The cloud server processes the bundle atomically. The local mobile cache is only cleared or marked read-only once a cryptographically verified success receipt — containing a server-confirmed count of accepted transaction UUIDs — is returned and validated against the local sequence log.

#### 1.1.1 Encryption Key Management

| Concern | Implementation |
|---|---|
| Algorithm | AES-256-GCM with a 96-bit IV per bundle |
| Key Derivation | PBKDF2-HMAC-SHA256, 100,000 iterations, salted per device |
| Key Storage | Device key stored in Android Keystore / iOS Secure Enclave |
| Key Scope | Per-device key; tenant-level wrapping key held server-side |
| Rotation | Device key rotates on re-registration; wrapping key rotates quarterly |

#### 1.1.2 Bundle Transmission & Retry Policy

| Parameter | Value |
|---|---|
| Max Bundle Size | 5 MB (compressed). Bundles exceeding this are chunked into 2 MB parts with a shared `bundle_group_id` |
| Chunk Order | Enforced server-side via `chunk_sequence` field; final assembly triggered by `is_last_chunk: true` flag |
| Retry Strategy | Exponential backoff: 5 s → 30 s → 2 min → 10 min. Max 5 attempts. After failure, status set to `FAILED` and a manager alert is raised |
| Idempotency Window | Server deduplicates on `Device_ID + Cashier_ID + Sequence_Number` for 72 hours |

#### 1.1.3 Conflict Resolution Policy

Inter-branch inventory conflicts (e.g., a stock transfer acknowledged by the sender before the destination has synced) are resolved by the following server-side rules:

1. **Inventory deltas are additive, not absolute.** The server never accepts a full stock count from a bundle; it accepts signed delta values only. This eliminates overwrite conflicts.
2. **Transfer state machine:** Inter-branch transfers exist in three server states — `DISPATCHED`, `IN_TRANSIT`, `ACKNOWLEDGED`. The destination branch's acknowledgment bundle must reference the originating `transfer_id`. Orphaned dispatches older than 48 hours are flagged for administrator review.
3. **Shrinkage write-offs:** If two bundles from the same outlet on the same calendar day both contain a `NATURAL_SHRINKAGE` log, the server sums them and evaluates the combined total against the tolerance threshold. It does not reject either bundle.

---

### 1.2 Enterprise Tenant Hierarchy

```
Company (Tenant Domain)
│
├── Corporate Account Dashboard (Global Administrative Layer)
└── Outlets (Physically Isolated Branches)
    ├── Device Registry (Authorized Device Tokens + IMEI Bindings)
    ├── Inventory Lots (Bulk LPG Mass Tally & Cylinder Vaults)
    ├── Localized Pricing Profiles
    └── Daily Shift Vaults (Localized Payment Channel Sub-Ledgers)
```

**Multi-Tenant Isolation Strategy:** Row-Level Security (RLS) enforced at the PostgreSQL layer. Every table carrying operational data includes a `company_id` column. All server-side queries are wrapped in an RLS policy that validates the authenticated JWT's `company_id` claim against the row. Cross-tenant data access is structurally impossible at the database layer.

---

## 2. Granular Role & Permission Matrix

The authorization layer enforces a decentralized operational control model backed by hard financial guardrails.

| Feature / Action | System Owner | Inventory Controller | Outlet Manager | Cashier |
|:---|:---:|:---:|:---:|:---:|
| Create Outlets & Global Architecture | ✓ | ✗ | ✗ | ✗ |
| Set System Base Price per KG | ✓ | ✓ | ✗ | ✗ |
| Apply Local Promo / Weight Discounts | ✓ | ✓ | ✓ (Max Threshold) | ✗ |
| Initiate Inter-Branch Stock Transfer | ✓ | ✓ | ✓ | ✗ |
| Acknowledge / Receive Incoming Mass | ✓ | ✓ | ✓ | ✗ |
| Log Daily Physical Tank Gauge Audits | ✓ | ✓ | ✓ | ✗ |
| Input Shift Closing Blind Counts | ✗ | ✗ | ✓ | ✓ |
| Review Own Shift Transaction History | ✗ | ✗ | ✓ | ✓ (Current Shift Only) |
| Review Shift Variances & Force Sync | ✓ | ✓ | ✓ (Escalates if > limit) | ✗ |
| Execute Retail Refills & Sales Actions | ✗ | ✗ | ✓ | ✓ |
| Execute Invoice Voids / Cash Refunds | ✓ | ✗ | ✓ (Mandatory Log Notes) | ✗ |
| Register / Revoke Device Tokens | ✓ | ✗ | ✗ | ✗ |

**Escalation Definition (Outlet Manager — Variance Review):**
- Threshold: Variance exceeding the owner-configured `max_manager_variance_usd` (default `$10.00`).
- Escalation target: System Owner and all Inventory Controllers at the tenant level.
- Mechanism: In-app push notification + database `variance_escalation` record flagged `PENDING_REVIEW`. Branch sync is not blocked, but the shift is locked from manual adjustment until an escalation record is resolved by a qualifying role.

**Cashier Read Access Clarification:**
Cashiers may view their own current open shift's transaction list for dispute resolution purposes. The UI surfaces a read-only transaction log for the active shift only. Closed-shift history is not accessible at the cashier role level.

---

## 3. Dual-Inventory Management Module

LPG retail requires tracking two fundamentally distinct asset classes in parallel within the same branch structure.

### 3.1 Volatile Bulk Gas Accounting (Mass Tally)

The platform tracks bulk liquid product strictly by weight (kg). Volume entries via pump meters (litres) are translated automatically through an internal temperature-density conversion layer using standard LPG tables (ASTM D-1250).

#### 3.1.1 Opening Balance Anchoring

Each outlet requires an initial `inventory_snapshots` record at branch activation, establishing the physical opening mass and cylinder counts. This record serves as the base for all subsequent theoretical stock calculations. Snapshot records are also written automatically at the beginning of each calendar month to provide a clean period boundary for reconciliation.

**Theoretical Stock Formula:**

```
Theoretical Stock = Opening Snapshot Mass
                  + SUM(delta_mass_kg WHERE log_reason = 'STOCK_RECEIPT')
                  + SUM(delta_mass_kg WHERE log_reason = 'INTER_BRANCH_TRANSFER')
                  - SUM(total_kg_sold FROM sales_ledger)
                  - ABS(SUM(delta_mass_kg WHERE log_reason = 'NATURAL_SHRINKAGE'))
```

**Physical Stock:** Entered manually by the manager twice daily (`08:00` and `18:00`) via tank gauges, load cells, or dipsticks.

#### 3.1.2 Shrinkage Categorization Engine

Discrepancies between Theoretical and Physical measurements are processed via a threshold matrix:

- **Natural Shrinkage:** Losses within the owner-defined tolerance (e.g., < 0.5%) are automatically written off.
- **Unexplained Stock Loss:** Losses exceeding tolerance trigger a critical flag on the next synchronization, locking further self-adjustments at that branch until reviewed by System Owner or Inventory Controller.

---

### 3.2 Cylinder Asset Matrix & Financial Liabilities

Cylinders are returnable containers representing distinct balance sheet liabilities. The system isolates gas mass value from the physical container asset.

All incoming POS transactions are routed through the following structural matrix and recorded in the dedicated `cylinder_transactions` table:

| Transaction Type | Gas Mass | Full Cylinders | Empty Cylinders | Financial Effect |
|---|---|---|---|---|
| **Refill Only** | Decrements | Decrements | Increments | Revenue: gas mass only |
| **New Purchase (Gas + Carcass)** | Decrements | Decrements | No change | Revenue: gas + cylinder deposit liability or asset sale recorded |
| **Cross-Fill / Competitor Exchange** | Decrements | No change | Increments (3rd-party) | Revenue: gas mass; competitor cylinder logged for reverse logistics |

---

## 4. Fraud-Proof Shift & Payment Ledger

### 4.1 The Mobile Blind Close Workflow

Cashiers have zero visibility into expected ledger balances at shift close.

1. **Shift Freeze:** Cashier taps "Initiate Knock-Off." Terminal UI freezes. No further sales or weight logs can be modified.
2. **Blind Count Processing:** Cashier inputs physical cash by individual denomination, card terminal receipts, and mobile money payment tokens. Expected balances are not displayed.
3. **Manager Verification Drop:** Outlet Manager verifies counted assets, audits high-value bills, and co-signs the device screen via PIN or biometric confirmation.
4. **Batch Analysis on Sync:** Post-upload, the system calculates shift variances per payment channel:

```
Variance(Channel) = Actual Inputted Count - Expected Ledger Total
```

Each payment channel is an independent sub-ledger. Cash shortages cannot be masked by mobile money overages. Discrepancies exceeding the configured tolerance trigger the escalation flow defined in Section 2.

---

## 5. Mobile-First User Experience Framework

### 5.1 The Cashier Interface

- **Thumb-Optimized Layout:** All checkout workflows use touch zones of minimum 48 × 48 px for rapid outdoor station operation.
- **Rapid Fill Calculator:** Cashier inputs cylinder size or custom mass; the app computes totals inline without modal overlays.
- **Integrated Scanning:** Built-in smartphone camera for zero-latency barcode and serial number tracking of cylinder units.
- **Current Shift Visibility:** A read-only transaction log of the current open shift is accessible from the cashier home screen for dispute handling.

### 5.2 The Managerial CVP Analytics Dashboard

Because data updates occur in daily batches, managers interact with a historical, daily-updating business analysis engine.

**Cost Engine Setup:** Managers input local fixed costs monthly (rent, payroll, generator fuel, utilities). The system layers these against variable costs captured during sales (wholesale gas cost per kg, delivery surcharges, mobile processor fees).

**Break-Even Point Matrix:**

```
Contribution Margin per KG (CM) = Selling Price per KG - Variable Cost per KG

BEP (Monthly Volume, kg) = Total Monthly Fixed Costs / CM

Remaining BEP Units = BEP(units) - Accumulated Monthly Volume Sold (to Yesterday)
```

**`outlet_expenses` Period Boundary:**
Each expense record carries an `expense_period` column (format: `YYYY-MM`). The CVP engine queries expenses filtered by `expense_period = current_month`. Fixed costs entered for prior periods are not re-evaluated in current-month calculations. Managers may clone a prior month's expense set as a starting point for the new month.

**The Visual Progress Wheel:** A progress ring maps current operational position from the red deficit zone through the break-even zero point into the green profit contribution zone.

---

## 6. Data Structure Design — Normalized MVP Schema

```sql
-- Tenant isolation
CREATE TABLE companies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_name VARCHAR(255) NOT NULL,
    base_currency VARCHAR(3) DEFAULT 'USD',
    global_configurations JSONB NOT NULL DEFAULT '{}'::jsonb
);

-- Branch structure
CREATE TABLE outlets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    branch_name VARCHAR(255) NOT NULL,
    base_price_per_kg NUMERIC(10, 2) NOT NULL,
    max_manager_variance_usd NUMERIC(8, 2) NOT NULL DEFAULT 10.00,
    shrinkage_tolerance_pct NUMERIC(5, 3) NOT NULL DEFAULT 0.500,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Device registry for idempotency and anti-tamper
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    device_imei VARCHAR(20) UNIQUE NOT NULL,
    device_token VARCHAR(64) UNIQUE NOT NULL,
    registered_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    revoked_at TIMESTAMP WITH TIME ZONE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

-- Inventory opening balances and monthly snapshots
CREATE TABLE inventory_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    snapshot_type VARCHAR(20) CHECK (snapshot_type IN ('OPENING', 'MONTHLY_CLOSE')),
    bulk_mass_kg NUMERIC(12, 3) NOT NULL,
    full_cylinders_count INTEGER NOT NULL DEFAULT 0,
    empty_cylinders_count INTEGER NOT NULL DEFAULT 0,
    third_party_cylinders_count INTEGER NOT NULL DEFAULT 0,
    snapshot_date DATE NOT NULL,
    recorded_by UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Expenses with period boundary
CREATE TABLE outlet_expenses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    expense_type VARCHAR(20) CHECK (expense_type IN ('FIXED', 'VARIABLE')),
    cost_category VARCHAR(50) NOT NULL,
    financial_amount NUMERIC(12, 2) NOT NULL,
    expense_period VARCHAR(7) NOT NULL, -- Format: YYYY-MM
    effective_date DATE NOT NULL
);

-- Staff shifts
CREATE TABLE staff_shifts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    user_id UUID NOT NULL,
    device_id UUID REFERENCES devices(id),
    shift_opened_at TIMESTAMP WITH TIME ZONE NOT NULL,
    shift_closed_at TIMESTAMP WITH TIME ZONE,
    blind_count_payload JSONB,
    upload_status VARCHAR(20) DEFAULT 'PENDING'
        CHECK (upload_status IN ('PENDING', 'SYNCED', 'FAILED')),
    sync_receipt_hash VARCHAR(64), -- Server-returned SHA-256 of accepted UUID list
    accepted_transaction_count INTEGER -- Validated against local sequence on device
);

-- Sales ledger (per transaction, not per shift aggregate)
CREATE TABLE sales_ledger (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_uuid VARCHAR(128) UNIQUE NOT NULL, -- Device_ID+Cashier_ID+Sequence_Number
    shift_id UUID REFERENCES staff_shifts(id) ON DELETE CASCADE,
    total_kg_sold NUMERIC(10, 3) NOT NULL,
    gross_revenue NUMERIC(12, 2) NOT NULL,
    cost_of_goods_sold_kg NUMERIC(10, 2) NOT NULL,
    payment_channel_enum VARCHAR(30)
        CHECK (payment_channel_enum IN ('CASH', 'MOBILE_MONEY', 'CARD')),
    transaction_timestamp TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Cylinder asset transactions (separate from gas mass ledger)
CREATE TABLE cylinder_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sales_ledger_id UUID REFERENCES sales_ledger(id) ON DELETE CASCADE,
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    transaction_type VARCHAR(30)
        CHECK (transaction_type IN ('REFILL', 'NEW_PURCHASE', 'CROSS_FILL')),
    cylinder_size_kg NUMERIC(6, 2) NOT NULL,
    cylinder_brand VARCHAR(100),  -- NULL for own-brand; populated for competitor cylinders
    cylinder_serial VARCHAR(50),
    deposit_liability_usd NUMERIC(10, 2), -- Populated for NEW_PURCHASE only
    transaction_timestamp TIMESTAMP WITH TIME ZONE NOT NULL
);

-- Bulk gas inventory audit log (delta-based)
CREATE TABLE inventory_mass_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outlet_id UUID REFERENCES outlets(id) ON DELETE CASCADE,
    order_reference_id UUID,
    delta_mass_kg NUMERIC(12, 3) NOT NULL,
    log_reason_enum VARCHAR(30)
        CHECK (log_reason_enum IN (
            'SALE', 'NATURAL_SHRINKAGE', 'UNEXPLAINED_LOSS',
            'STOCK_RECEIPT', 'INTER_BRANCH_TRANSFER'
        )),
    evaluation_timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    recorded_by UUID NOT NULL
);

-- Inter-branch transfer state machine
CREATE TABLE stock_transfers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id UUID REFERENCES companies(id) ON DELETE CASCADE,
    source_outlet_id UUID REFERENCES outlets(id),
    destination_outlet_id UUID REFERENCES outlets(id),
    mass_kg NUMERIC(12, 3) NOT NULL,
    status VARCHAR(20) DEFAULT 'DISPATCHED'
        CHECK (status IN ('DISPATCHED', 'IN_TRANSIT', 'ACKNOWLEDGED', 'FLAGGED')),
    dispatched_at TIMESTAMP WITH TIME ZONE NOT NULL,
    acknowledged_at TIMESTAMP WITH TIME ZONE,
    flagged_at TIMESTAMP WITH TIME ZONE
);

-- Variance escalation records
CREATE TABLE variance_escalations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shift_id UUID REFERENCES staff_shifts(id) ON DELETE CASCADE,
    outlet_id UUID REFERENCES outlets(id),
    payment_channel VARCHAR(30) NOT NULL,
    variance_amount_usd NUMERIC(10, 2) NOT NULL,
    escalation_status VARCHAR(20) DEFAULT 'PENDING_REVIEW'
        CHECK (escalation_status IN ('PENDING_REVIEW', 'RESOLVED', 'WRITTEN_OFF')),
    resolved_by UUID,
    resolution_notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 7. MVP Technical Verification Criteria

### Test 1 — Intermittent Sync Integrity Check

The mobile app must log 200 consecutive sales transactions under total hardware network isolation. Mid-operation, the device is power-cycled via battery-pull simulation. On reboot, the app must read from its local write-ahead log and execute a batch upload recovery routine.

**Pass Condition:** The server's success receipt must contain a SHA-256 hash of all accepted transaction `client_uuid` values. The device validates this hash against its local sequence log. Zero missing UUIDs and zero duplicate mutations permitted. Any mismatch causes the local cache to remain in `PENDING` state and the test fails.

### Test 2 — Anti-Tamper Time Validation

The client must cross-verify device hardware uptime counters against network-synchronized timestamps captured at shift initialization. Attempts to roll back the device clock to overwrite backdated ledger sequences must be programmatically rejected.

**Pass Condition:** Any detected clock delta exceeding 5 minutes between hardware uptime-derived time and last-known network time locks the shift from opening and surfaces a "Clock Integrity Failure" alert requiring manager override with a logged justification.

### Test 3 — CVP Impact Test

An unexpected fixed expense (e.g., a $400 emergency repair charge) is pushed to an outlet's ledger for the current `expense_period`.

**Pass Condition (post-sync only):** After the manager's app performs its next sync, the recalculated `Break-Even Units` target must be reflected in the CVP dashboard. The test explicitly validates the post-sync state, acknowledging that offline clients will not see the update until the next connection. The pass criterion requires correct recalculation within one sync cycle, not instant pre-sync display.
