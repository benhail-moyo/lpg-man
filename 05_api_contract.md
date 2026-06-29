# API Contract Reference
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Document Type:** Backend API Specification
**Base URL:** `/api/v1`
**Version:** MVP 1.0

---

## Authentication

All endpoints except `/auth/login` require a valid JWT access token in an httpOnly cookie (`access_token_cookie`). The JWT payload must contain:

```json
{
  "user_id": "uuid",
  "company_id": "uuid",
  "outlet_id": "uuid | null",
  "role": "SYSTEM_OWNER | INVENTORY_CONTROLLER | OUTLET_MANAGER | CASHIER"
}
```

---

## Response Envelope

All responses use this structure:

```json
{
  "status": "success | error",
  "data": {},
  "message": "Human-readable string",
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150
  }
}
```

---

## Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `AUTH_001` | 401 | Invalid credentials |
| `AUTH_002` | 401 | Token expired |
| `AUTH_003` | 403 | Insufficient permissions |
| `DEVICE_001` | 403 | Device not registered |
| `DEVICE_002` | 403 | Device revoked |
| `SYNC_001` | 409 | Duplicate bundle detected |
| `SYNC_002` | 422 | Clock integrity failure |
| `SYNC_003` | 422 | Bundle schema validation failed |
| `SYNC_004` | 202 | Chunk received, awaiting remaining chunks |
| `INV_001` | 423 | Stock lock active — unexplained loss pending review |
| `ESC_001` | 409 | Escalation already resolved |

---

## Endpoints

---

### Auth Module — `/auth`

#### `POST /auth/login`
**Auth:** None

**Request:**
```json
{
  "email": "string",
  "password": "string"
}
```

**Response `200`:**
```json
{
  "status": "success",
  "data": {
    "user_id": "uuid",
    "role": "OUTLET_MANAGER",
    "company_id": "uuid",
    "outlet_id": "uuid"
  },
  "message": "Login successful"
}
```
Access token and refresh token are set as httpOnly cookies.

---

#### `POST /auth/refresh`
**Auth:** Refresh token cookie

**Response `200`:** New access token set as cookie.

---

#### `POST /auth/logout`
**Auth:** Required

Blacklists the refresh token.

---

#### `POST /auth/device/register`
**Auth:** SYSTEM_OWNER only

**Request:**
```json
{
  "outlet_id": "uuid",
  "device_imei": "string"
}
```

**Response `201`:**
```json
{
  "status": "success",
  "data": {
    "device_id": "uuid",
    "device_token": "64-char-hex-string"
  }
}
```

---

#### `DELETE /auth/device/revoke/{device_id}`
**Auth:** SYSTEM_OWNER only

**Response `200`:** Device `is_active` set to `false`.

---

### Outlets Module — `/outlets`

#### `GET /outlets`
**Auth:** SYSTEM_OWNER, INVENTORY_CONTROLLER (all outlets); OUTLET_MANAGER (own outlet only)

**Response `200`:**
```json
{
  "status": "success",
  "data": [
    {
      "id": "uuid",
      "branch_name": "string",
      "base_price_per_kg": 1.85,
      "max_manager_variance_usd": 10.00,
      "shrinkage_tolerance_pct": 0.5
    }
  ]
}
```

---

#### `POST /outlets`
**Auth:** SYSTEM_OWNER only

**Request:**
```json
{
  "branch_name": "string",
  "base_price_per_kg": 1.85,
  "max_manager_variance_usd": 10.00,
  "shrinkage_tolerance_pct": 0.5,
  "opening_snapshot": {
    "bulk_mass_kg": 5000.0,
    "full_cylinders_count": 200,
    "empty_cylinders_count": 50,
    "third_party_cylinders_count": 0
  }
}
```

**Response `201`:** Created outlet object.

---

#### `PATCH /outlets/{outlet_id}`
**Auth:** SYSTEM_OWNER only

Partial update of outlet configuration fields.

---

### Time Module — `/time`

#### `GET /time`
**Auth:** Device token (lightweight header auth — no full JWT required)

**Response `200`:**
```json
{
  "status": "success",
  "data": {
    "server_utc_timestamp": 1735480800000
  }
}
```

Used by mobile client for anti-tamper clock validation at shift open.

---

### Sync Module — `/sync`

#### `POST /sync/bundle`
**Auth:** Device token (header: `X-Device-Token`)

**Request:** `multipart/form-data`
- `bundle_file`: encrypted, compressed binary payload
- `metadata`: JSON string

**Metadata schema:**
```json
{
  "device_id": "uuid",
  "outlet_id": "uuid",
  "cashier_id": "uuid",
  "shift_id": "uuid",
  "bundle_group_id": "uuid | null",
  "chunk_sequence": 1,
  "is_last_chunk": true,
  "transaction_count": 47
}
```

**Response `200` (final chunk accepted and processed):**
```json
{
  "status": "success",
  "data": {
    "upload_status": "SYNCED",
    "accepted_count": 47,
    "receipt_hash": "sha256-hex-string"
  },
  "message": "Bundle processed successfully"
}
```

**Response `202` (chunk received, not last):**
```json
{
  "status": "success",
  "data": { "upload_status": "CHUNK_RECEIVED", "chunk_sequence": 1 },
  "message": "Chunk received. Awaiting remaining chunks."
}
```

---

### Inventory Module — `/inventory`

#### `GET /inventory/{outlet_id}/snapshot`
**Auth:** OUTLET_MANAGER (own outlet), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Response `200`:**
```json
{
  "status": "success",
  "data": {
    "theoretical_stock_kg": 4820.5,
    "physical_stock_kg": 4815.0,
    "delta_kg": -5.5,
    "delta_pct": 0.114,
    "shrinkage_status": "WITHIN_TOLERANCE",
    "stock_lock_active": false,
    "last_gauge_reading": {
      "timestamp": "2026-06-29T08:00:00Z",
      "recorded_by": "Manager Name"
    }
  }
}
```

---

#### `POST /inventory/{outlet_id}/gauge`
**Auth:** OUTLET_MANAGER, INVENTORY_CONTROLLER, SYSTEM_OWNER

**Request:**
```json
{
  "physical_mass_kg": 4815.0,
  "reading_method": "DIPSTICK | LOAD_CELL | GAUGE",
  "notes": "string | null"
}
```

**Response `201`:** Inventory log record created. Shrinkage evaluation triggered.

---

#### `GET /inventory/{outlet_id}/logs`
**Auth:** OUTLET_MANAGER (own), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Query params:** `?page=1&per_page=20&reason=NATURAL_SHRINKAGE&from=2026-06-01&to=2026-06-30`

**Response `200`:** Paginated list of `inventory_mass_logs` records.

---

### Analytics Module — `/analytics`

#### `GET /analytics/{outlet_id}/cvp`
**Auth:** OUTLET_MANAGER (own), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Response `200`:**
```json
{
  "status": "success",
  "data": {
    "expense_period": "2026-06",
    "selling_price_per_kg": 1.85,
    "variable_cost_per_kg": 1.20,
    "contribution_margin_per_kg": 0.65,
    "total_fixed_costs_usd": 3200.00,
    "bep_units_kg": 4923.08,
    "accumulated_volume_kg": 3100.0,
    "remaining_bep_kg": 1823.08,
    "profit_surplus_kg": null,
    "expenses_configured": true
  }
}
```

---

#### `GET /analytics/{outlet_id}/shifts`
**Auth:** OUTLET_MANAGER (own), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Query params:** `?page=1&per_page=20&from=2026-06-01&to=2026-06-30&cashier_id=uuid&escalation_status=PENDING_REVIEW`

**Response `200`:**
```json
{
  "status": "success",
  "data": [
    {
      "shift_id": "uuid",
      "cashier_name": "string",
      "opened_at": "2026-06-29T07:00:00Z",
      "closed_at": "2026-06-29T18:00:00Z",
      "gross_revenue": 520.00,
      "total_kg_sold": 281.08,
      "variances": {
        "CASH": -2.50,
        "MOBILE_MONEY": 0.00,
        "CARD": 0.00
      },
      "escalation_status": "PENDING_REVIEW | RESOLVED | null"
    }
  ],
  "pagination": { "page": 1, "per_page": 20, "total": 45 }
}
```

---

#### `GET /analytics/corporate`
**Auth:** SYSTEM_OWNER, INVENTORY_CONTROLLER

**Query params:** `?from=2026-06-01&to=2026-06-30&outlets[]=uuid1&outlets[]=uuid2`

**Response `200`:**
```json
{
  "status": "success",
  "data": {
    "total_revenue_usd": 14200.00,
    "total_kg_sold": 7676.0,
    "outlet_breakdown": [
      {
        "outlet_id": "uuid",
        "branch_name": "string",
        "revenue_usd": 5200.00,
        "kg_sold": 2810.0,
        "cm_per_kg": 0.65
      }
    ],
    "revenue_trend": [
      { "date": "2026-06-01", "revenue_usd": 480.00 }
    ]
  }
}
```

---

### Transfers Module — `/transfers`

#### `POST /transfers`
**Auth:** OUTLET_MANAGER, INVENTORY_CONTROLLER, SYSTEM_OWNER

**Request:**
```json
{
  "destination_outlet_id": "uuid",
  "mass_kg": 500.0
}
```

**Response `201`:**
```json
{
  "status": "success",
  "data": {
    "transfer_id": "uuid",
    "status": "DISPATCHED",
    "dispatched_at": "2026-06-29T10:00:00Z"
  }
}
```

---

#### `PATCH /transfers/{transfer_id}/acknowledge`
**Auth:** OUTLET_MANAGER (destination outlet), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Response `200`:** Transfer status updated to `ACKNOWLEDGED`. Destination inventory incremented.

---

#### `GET /transfers`
**Auth:** OUTLET_MANAGER (own outlet transfers), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Query params:** `?status=DISPATCHED&page=1`

**Response `200`:** Paginated transfer list.

---

### Escalations Module — `/escalations`

#### `GET /escalations`
**Auth:** OUTLET_MANAGER (own outlet), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Query params:** `?status=PENDING_REVIEW&outlet_id=uuid&page=1`

**Response `200`:** Paginated escalation records.

---

#### `PATCH /escalations/{escalation_id}/resolve`
**Auth:** INVENTORY_CONTROLLER, SYSTEM_OWNER

**Request:**
```json
{
  "escalation_status": "RESOLVED | WRITTEN_OFF",
  "resolution_notes": "string (required)"
}
```

**Response `200`:** Escalation record updated. Shift manual-adjustment lock lifted.

---

### Expenses Module — `/expenses`

#### `GET /expenses/{outlet_id}`
**Auth:** OUTLET_MANAGER (own), INVENTORY_CONTROLLER, SYSTEM_OWNER

**Query params:** `?period=2026-06`

**Response `200`:** List of expense records for the period.

---

#### `POST /expenses/{outlet_id}`
**Auth:** OUTLET_MANAGER, SYSTEM_OWNER

**Request:**
```json
{
  "expense_type": "FIXED | VARIABLE",
  "cost_category": "Rent | Payroll | Generator Fuel | Wholesale Gas Cost per KG | ...",
  "financial_amount": 1200.00,
  "expense_period": "2026-06",
  "effective_date": "2026-06-01"
}
```

**Response `201`:** Created expense record.

---

#### `POST /expenses/{outlet_id}/clone`
**Auth:** OUTLET_MANAGER, SYSTEM_OWNER

**Request:**
```json
{
  "source_period": "2026-05",
  "target_period": "2026-06"
}
```

Clones all expense records from `source_period` into `target_period`.

**Response `201`:** List of cloned expense records.
