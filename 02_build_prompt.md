# Master Build Prompt
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Stack:** React (Vite) + Flask + PostgreSQL
**Prompt Version:** 1.0 — MVP

---

## Context Block

You are building a production-grade, multi-tenant LPG (Liquefied Petroleum Gas) retail management platform for downstream gas retailers operating in emerging markets with unreliable network and power infrastructure. The system must function under complete offline conditions and synchronize in bulk when connectivity is restored.

The platform has two surfaces:
1. A **React web dashboard** (Vite + TailwindCSS) for corporate managers and administrators.
2. A **React Native mobile client** (Expo) for cashiers and outlet managers — the primary point of sale terminal.

The backend is a **Flask REST API** with a **PostgreSQL** database, deployed on a cloud VPS. Multi-tenant row-level security is enforced at the PostgreSQL layer.

All specification details, schema definitions, and business rules are contained in `01_technical_spec.md`. This build prompt references that document as ground truth. Do not invent business logic; derive it from the spec.

---

## Tech Stack

### Backend
| Layer | Technology |
|---|---|
| API Framework | Flask 3.x (Python 3.12) |
| ORM | SQLAlchemy 2.x with Alembic migrations |
| Authentication | Flask-JWT-Extended (access + refresh token pair) |
| Database | PostgreSQL 16 with Row-Level Security |
| Task Queue | Celery + Redis (async sync bundle processing) |
| Encryption | Python `cryptography` library (AES-256-GCM) |
| Bundle Upload | Flask-Uploads or direct multipart stream handler |
| Testing | pytest + pytest-flask |
| Env Config | python-dotenv + Pydantic Settings |

### Frontend Web Dashboard
| Layer | Technology |
|---|---|
| Framework | React 18 + Vite |
| Styling | TailwindCSS 3.x |
| State Management | Zustand |
| Server State | TanStack Query (React Query) v5 |
| Charts | Recharts |
| Forms | React Hook Form + Zod |
| Auth | JWT stored in httpOnly cookies |
| HTTP Client | Axios with interceptor-based token refresh |

### Mobile Client
| Layer | Technology |
|---|---|
| Framework | React Native via Expo SDK 51 |
| Local DB | Expo SQLite (WAL mode enabled) |
| State | Zustand with MMKV persistence |
| Camera/Scan | Expo Camera + expo-barcode-scanner |
| Encryption | expo-crypto + react-native-aes-crypto |
| Secure Storage | expo-secure-store (key storage) |
| Sync Engine | Custom background task via expo-background-fetch |

---

## Backend Build Instructions

### Step 1 — Project Initialization

```bash
python -m venv venv && source venv/bin/activate
pip install flask flask-jwt-extended flask-sqlalchemy alembic psycopg2-binary \
            celery redis cryptography python-dotenv pydantic-settings pytest pytest-flask
```

Create the Flask application using the Application Factory pattern (`create_app()`). Register all blueprints inside the factory. Never instantiate Flask at module level.

### Step 2 — Database Setup

Apply the full normalized schema from `01_technical_spec.md`, Section 6 via an Alembic initial migration. After applying the schema:

1. Enable Row-Level Security on all tables that carry `company_id`:
```sql
ALTER TABLE outlets ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON outlets
  USING (company_id = current_setting('app.current_tenant')::uuid);
```
Apply equivalent policies to: `devices`, `inventory_snapshots`, `outlet_expenses`, `staff_shifts`, `sales_ledger`, `cylinder_transactions`, `inventory_mass_logs`, `stock_transfers`, `variance_escalations`.

2. Create a database session middleware that sets `app.current_tenant` from the authenticated JWT claim at the start of every request.

### Step 3 — Authentication Module (`/api/v1/auth`)

| Endpoint | Method | Description |
|---|---|---|
| `/auth/login` | POST | Returns access token (15 min TTL) + refresh token (7 day TTL) in httpOnly cookies |
| `/auth/refresh` | POST | Rotates access token using refresh token |
| `/auth/logout` | POST | Blacklists refresh token |
| `/auth/device/register` | POST | System Owner only — registers device IMEI, returns `device_token` |
| `/auth/device/revoke/{id}` | DELETE | System Owner only — sets `is_active = false` |

JWT payload must include: `user_id`, `company_id`, `outlet_id`, `role`.

### Step 4 — Sync Bundle Ingestion Endpoint (`/api/v1/sync`)

This is the most critical backend endpoint.

```
POST /api/v1/sync/bundle
Content-Type: multipart/form-data
Body: { bundle_file: <encrypted_compressed_payload>, metadata: <json> }
```

**Processing pipeline (must be atomic via Celery task):**

1. Decrypt bundle using device's derived key.
2. Decompress payload.
3. Validate bundle schema (jsonschema).
4. Check all `client_uuid` values against 72-hour deduplication window. Reject any duplicates.
5. Validate `clock_integrity`: compare `shift_opened_at` against `device_uptime_seconds`. Reject if delta > 5 minutes.
6. Write all records within a single database transaction: `staff_shifts`, `sales_ledger`, `cylinder_transactions`, `inventory_mass_logs`.
7. Run shrinkage evaluation: compute theoretical vs physical stock. Categorize as `NATURAL_SHRINKAGE` or `UNEXPLAINED_LOSS`. Create `variance_escalations` records as needed.
8. Generate SHA-256 receipt hash over sorted list of accepted `client_uuid` values.
9. Return success receipt: `{ status: "SYNCED", accepted_count: N, receipt_hash: "..." }`.

Chunked bundles: collect all chunks with shared `bundle_group_id`. Assemble and process only when `is_last_chunk: true` is received.

### Step 5 — Core API Modules

Build the following blueprint modules, each in its own file under `app/blueprints/`:

**`outlets.py`** — CRUD for outlet management. System Owner only for create/delete.

**`inventory.py`**
- `GET /inventory/{outlet_id}/snapshot` — Current theoretical vs physical stock
- `POST /inventory/{outlet_id}/gauge` — Log manual physical gauge reading
- `GET /inventory/{outlet_id}/logs` — Paginated `inventory_mass_logs`

**`transfers.py`**
- `POST /transfers` — Initiate inter-branch transfer (status: `DISPATCHED`)
- `PATCH /transfers/{id}/acknowledge` — Destination branch acknowledges receipt (status: `ACKNOWLEDGED`)
- `GET /transfers` — List transfers for company with status filter

**`analytics.py`**
- `GET /analytics/{outlet_id}/cvp` — Returns BEP, CM, remaining units for current month
- `GET /analytics/{outlet_id}/shifts` — Paginated shift history with variance summary
- `GET /analytics/corporate` — Aggregated cross-outlet performance (System Owner / Inventory Controller only)

**`escalations.py`**
- `GET /escalations` — List pending escalations for authorized roles
- `PATCH /escalations/{id}/resolve` — Resolve with mandatory `resolution_notes`

### Step 6 — Role Enforcement

Implement a `@require_role(*roles)` decorator using Flask-JWT-Extended's `get_jwt()`. Apply to all endpoints per the matrix in `01_technical_spec.md`, Section 2. The decorator must raise `403 Forbidden` with a structured error body `{ "error": "insufficient_permissions", "required": [...] }`.

---

## Frontend Web Dashboard Build Instructions

### Step 1 — Project Initialization

```bash
npm create vite@latest lpg-dashboard -- --template react-ts
cd lpg-dashboard && npm install tailwindcss @tanstack/react-query zustand axios react-hook-form zod recharts
```

### Step 2 — Auth Shell

Implement a top-level `AuthProvider` using Zustand. On mount, attempt token refresh via `POST /auth/refresh`. If refresh fails, redirect to `/login`. Store `user`, `role`, and `company_id` in Zustand auth store. Protect all routes with a `<ProtectedRoute>` wrapper that checks role.

### Step 3 — Route Structure

```
/login                          — Public
/dashboard                      — System Owner, Inventory Controller
/dashboard/outlets              — Outlet list
/dashboard/outlets/:id          — Outlet detail (inventory, shifts, CVP)
/dashboard/outlets/:id/cvp      — CVP Analytics with progress ring
/dashboard/transfers            — Inter-branch transfer management
/dashboard/escalations          — Variance escalation queue
/dashboard/devices              — Device registry (System Owner only)
/outlet/:id                     — Outlet Manager view (single-outlet scope)
```

### Step 4 — CVP Progress Ring Component

Build a `BEPProgressRing` React component using an SVG arc:
- Red zone: 0% to break-even
- Green zone: break-even to 100%+
- Animated fill on data load
- Center label: remaining kg to break-even or profit surplus kg

Use Recharts `RadialBarChart` as an alternative if SVG arc complexity is undesirable.

### Step 5 — Shift Variance Dashboard

Display per-shift variance breakdown as a table with:
- Column per payment channel (CASH, MOBILE_MONEY, CARD)
- Green cell for zero-variance channels
- Red cell + escalation badge for channels exceeding tolerance
- Export to CSV button

---

## Mobile Client Build Instructions

### Step 1 — Expo Initialization

```bash
npx create-expo-app lpg-mobile --template blank-typescript
cd lpg-mobile && npx expo install expo-sqlite expo-camera expo-barcode-scanner \
    expo-secure-store expo-background-fetch expo-crypto zustand
```

### Step 2 — Local SQLite Schema

Initialize the following tables in the local SQLite database on first app launch:

```sql
CREATE TABLE IF NOT EXISTS local_sales (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    client_uuid TEXT UNIQUE NOT NULL,
    shift_id TEXT NOT NULL,
    kg_sold REAL NOT NULL,
    gross_revenue REAL NOT NULL,
    cogs_per_kg REAL NOT NULL,
    payment_channel TEXT NOT NULL,
    transaction_timestamp TEXT NOT NULL,
    synced INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS local_cylinder_txns (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    client_uuid TEXT NOT NULL,
    transaction_type TEXT NOT NULL,
    cylinder_size_kg REAL NOT NULL,
    cylinder_brand TEXT,
    cylinder_serial TEXT,
    deposit_liability REAL,
    transaction_timestamp TEXT NOT NULL,
    synced INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS local_shift (
    id TEXT PRIMARY KEY,
    opened_at TEXT NOT NULL,
    closed_at TEXT,
    status TEXT DEFAULT 'OPEN'
);
```

Enable WAL mode: `PRAGMA journal_mode=WAL;`

### Step 3 — Sync Engine

Implement a `SyncEngine` class with the following interface:

```typescript
class SyncEngine {
  buildBundle(shiftId: string): Promise<SyncBundle>
  encryptBundle(bundle: SyncBundle): Promise<EncryptedPayload>
  uploadBundle(payload: EncryptedPayload): Promise<SyncReceipt>
  validateReceipt(receipt: SyncReceipt, localSequence: string[]): boolean
  clearLocalCache(shiftId: string): void  // Only called after validateReceipt returns true
}
```

### Step 4 — Blind Close UI Flow

Implement the blind close sequence as a multi-step modal stack (not a new screen):

1. Step 1 — Freeze confirmation
2. Step 2 — Cash denomination input (per ZAR/USD denomination)
3. Step 3 — Card receipts total input
4. Step 4 — Mobile money token input per channel (EcoCash, OneMoney, etc.)
5. Step 5 — Manager PIN or biometric co-sign
6. Step 6 — Bundle assembly + upload with progress indicator
7. Step 7 — Receipt confirmation screen (show accepted count, hash prefix)

**At no point in steps 1–7 should the expected ledger total be visible in the UI.**

### Step 5 — Anti-Tamper Clock Check

On shift open:
```typescript
const networkTime = await fetchNetworkTime(); // NTP or server timestamp
const uptimeDerivedTime = Date.now() - DeviceInfo.getUptimeSync();
const delta = Math.abs(networkTime - uptimeDerivedTime);

if (delta > 5 * 60 * 1000) {
  lockShiftOpen();
  showClockIntegrityAlert();
}
```

---

## Shared Conventions

### API Response Envelope

All API responses must use this envelope:
```json
{
  "status": "success" | "error",
  "data": { ... },
  "message": "Human-readable string",
  "pagination": { "page": 1, "per_page": 20, "total": 150 }
}
```

### Error Codes

| Code | Meaning |
|---|---|
| `AUTH_001` | Invalid credentials |
| `AUTH_002` | Token expired |
| `AUTH_003` | Insufficient permissions |
| `DEVICE_001` | Device not registered |
| `DEVICE_002` | Device revoked |
| `SYNC_001` | Duplicate bundle detected |
| `SYNC_002` | Clock integrity failure |
| `SYNC_003` | Bundle schema validation failed |
| `SYNC_004` | Chunk assembly incomplete |
| `INV_001` | Stock lock active — unexplained loss pending review |
| `ESC_001` | Escalation already resolved |

### Environment Variables (Backend)

```env
DATABASE_URL=postgresql://user:pass@host:5432/lpg_db
REDIS_URL=redis://localhost:6379/0
JWT_SECRET_KEY=<256-bit-random>
TENANT_MASTER_KEY=<256-bit-random>   # Wrapping key for device key encryption
CELERY_BROKER_URL=redis://localhost:6379/1
FLASK_ENV=production
```
