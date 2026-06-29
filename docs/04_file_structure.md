# Project File Structure Scaffolding
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Document Type:** Architecture Reference — Directory & File Layout
**Version:** MVP 1.0

---

## Repository Layout (Monorepo)

```
lpg-man/
├── backend/                    # Flask REST API
├── dashboard/                  # React + Vite web dashboard
├── mobile/                     # React Native Expo mobile client
├── docs/                       # All specification documents
│   ├── 01_technical_spec.md
│   ├── 02_build_prompt.md
│   ├── 03_feature_specifications.md
│   ├── 04_file_structure.md
│   ├── 05_api_contract.md
│   └── 06_environment_setup.md
├── .github/
│   └── workflows/
│       ├── backend-ci.yml
│       ├── dashboard-ci.yml
│       └── mobile-ci.yml
├── docker-compose.yml          # Local dev: postgres + redis + backend
└── README.md
```

---

## Backend — Flask API

```
backend/
├── app/
│   ├── __init__.py             # Application factory: create_app()
│   ├── extensions.py           # db, jwt, celery, redis — initialized here, bound in factory
│   ├── config.py               # Pydantic Settings — loads from .env
│   │
│   ├── blueprints/
│   │   ├── __init__.py
│   │   ├── auth.py             # /api/v1/auth — login, refresh, logout, device register/revoke
│   │   ├── outlets.py          # /api/v1/outlets — CRUD, scoped by role
│   │   ├── inventory.py        # /api/v1/inventory — snapshots, gauge logs, mass logs
│   │   ├── sync.py             # /api/v1/sync — bundle ingestion endpoint
│   │   ├── transfers.py        # /api/v1/transfers — inter-branch transfer state machine
│   │   ├── analytics.py        # /api/v1/analytics — CVP, shift reports, corporate view
│   │   ├── escalations.py      # /api/v1/escalations — variance escalation queue
│   │   ├── expenses.py         # /api/v1/expenses — outlet expense period management
│   │   └── time.py             # /api/v1/time — lightweight NTP-sync endpoint for mobile
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── company.py          # Company (Tenant)
│   │   ├── outlet.py           # Outlet (Branch)
│   │   ├── device.py           # Device Registry
│   │   ├── user.py             # User + Role
│   │   ├── shift.py            # StaffShift
│   │   ├── sales.py            # SalesLedger
│   │   ├── cylinder.py         # CylinderTransaction
│   │   ├── inventory.py        # InventoryMassLog + InventorySnapshot
│   │   ├── transfer.py         # StockTransfer
│   │   ├── expense.py          # OutletExpense
│   │   └── escalation.py       # VarianceEscalation
│   │
│   ├── schemas/                # Marshmallow / Pydantic request+response schemas
│   │   ├── auth_schema.py
│   │   ├── outlet_schema.py
│   │   ├── sync_bundle_schema.py  # JSON Schema for bundle validation
│   │   ├── inventory_schema.py
│   │   ├── analytics_schema.py
│   │   └── escalation_schema.py
│   │
│   ├── services/               # Business logic layer — no Flask imports
│   │   ├── sync_service.py         # Bundle decrypt → validate → ingest pipeline
│   │   ├── shrinkage_service.py    # Shrinkage categorization engine
│   │   ├── cvp_service.py          # CVP / BEP calculation engine
│   │   ├── transfer_service.py     # Transfer state machine logic
│   │   ├── escalation_service.py   # Escalation creation and resolution
│   │   └── receipt_service.py      # SHA-256 receipt hash generation
│   │
│   ├── tasks/                  # Celery async tasks
│   │   ├── __init__.py
│   │   ├── sync_tasks.py       # Async bundle processing task
│   │   └── alert_tasks.py      # Push notification dispatch
│   │
│   ├── middleware/
│   │   ├── rls_middleware.py   # Sets app.current_tenant on each request from JWT
│   │   └── error_handler.py    # Global error → structured JSON response
│   │
│   └── utils/
│       ├── crypto.py           # AES-256-GCM encrypt/decrypt helpers
│       ├── pagination.py       # Shared paginate() utility
│       ├── role_guard.py       # @require_role decorator
│       └── response.py         # Envelope builder: success_response(), error_response()
│
├── migrations/                 # Alembic migrations
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       └── 0001_initial_schema.py
│
├── tests/
│   ├── conftest.py             # Pytest fixtures: test app, test db, test client
│   ├── test_auth.py
│   ├── test_sync.py            # Sync integrity tests (the 200-transaction test)
│   ├── test_shrinkage.py
│   ├── test_cvp.py
│   ├── test_transfers.py
│   └── test_escalations.py
│
├── .env.example
├── requirements.txt
├── Dockerfile
└── gunicorn.conf.py
```

---

## Web Dashboard — React + Vite

```
dashboard/
├── public/
│   └── favicon.ico
│
├── src/
│   ├── main.tsx                # Vite entry — mounts <App />
│   ├── App.tsx                 # Router + QueryClientProvider + AuthProvider
│   │
│   ├── api/                    # Axios instances + API call functions
│   │   ├── client.ts           # Axios base instance, interceptors, token refresh
│   │   ├── auth.api.ts
│   │   ├── outlets.api.ts
│   │   ├── inventory.api.ts
│   │   ├── sync.api.ts
│   │   ├── analytics.api.ts
│   │   ├── escalations.api.ts
│   │   ├── expenses.api.ts
│   │   └── transfers.api.ts
│   │
│   ├── stores/                 # Zustand stores
│   │   ├── auth.store.ts       # user, role, company_id, token status
│   │   └── ui.store.ts         # sidebar state, active outlet selection
│   │
│   ├── hooks/                  # TanStack Query hooks (data fetching)
│   │   ├── useOutlets.ts
│   │   ├── useInventory.ts
│   │   ├── useCVP.ts
│   │   ├── useShifts.ts
│   │   ├── useEscalations.ts
│   │   └── useTransfers.ts
│   │
│   ├── pages/
│   │   ├── Login/
│   │   │   └── LoginPage.tsx
│   │   ├── Dashboard/
│   │   │   └── DashboardPage.tsx       # Corporate overview — S.Owner + Inv.Controller
│   │   ├── Outlets/
│   │   │   ├── OutletListPage.tsx
│   │   │   └── OutletDetailPage.tsx
│   │   ├── Inventory/
│   │   │   └── InventoryPage.tsx
│   │   ├── CVP/
│   │   │   └── CVPPage.tsx
│   │   ├── Shifts/
│   │   │   └── ShiftsPage.tsx
│   │   ├── Escalations/
│   │   │   └── EscalationsPage.tsx
│   │   ├── Transfers/
│   │   │   └── TransfersPage.tsx
│   │   ├── Expenses/
│   │   │   └── ExpensesPage.tsx
│   │   └── Devices/
│   │       └── DevicesPage.tsx         # System Owner only
│   │
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── TopBar.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   │
│   │   ├── charts/
│   │   │   ├── BEPProgressRing.tsx     # SVG arc BEP ring (F-15)
│   │   │   ├── RevenueTrendChart.tsx   # Recharts LineChart (F-17)
│   │   │   └── VarianceHeatMap.tsx
│   │   │
│   │   ├── tables/
│   │   │   ├── ShiftVarianceTable.tsx  # F-16
│   │   │   ├── CylinderTable.tsx
│   │   │   └── EscalationQueue.tsx
│   │   │
│   │   ├── forms/
│   │   │   ├── GaugeReadingForm.tsx
│   │   │   ├── ExpenseForm.tsx
│   │   │   ├── TransferForm.tsx
│   │   │   └── OutletForm.tsx
│   │   │
│   │   └── ui/                         # Shared primitives
│   │       ├── Badge.tsx
│   │       ├── Card.tsx
│   │       ├── Modal.tsx
│   │       ├── Button.tsx
│   │       └── Spinner.tsx
│   │
│   ├── types/                          # TypeScript interfaces mirroring API schemas
│   │   ├── auth.types.ts
│   │   ├── outlet.types.ts
│   │   ├── inventory.types.ts
│   │   ├── cvp.types.ts
│   │   ├── shift.types.ts
│   │   └── escalation.types.ts
│   │
│   └── utils/
│       ├── formatters.ts               # Currency, weight, date formatters
│       └── csvExport.ts               # F-16 CSV export utility
│
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── .env.example
└── package.json
```

---

## Mobile Client — React Native (Expo)

```
mobile/
├── app/                        # Expo Router file-based routing
│   ├── _layout.tsx             # Root layout — AuthGate + SQLite init
│   ├── index.tsx               # Entry redirect → /login or /pos
│   ├── (auth)/
│   │   └── login.tsx
│   ├── (cashier)/
│   │   ├── _layout.tsx         # Cashier tab navigator
│   │   ├── pos.tsx             # Primary POS screen (F-05, F-06)
│   │   ├── scan.tsx            # Camera barcode scan screen (F-18)
│   │   └── knock-off/
│   │       ├── _layout.tsx     # Blind close modal stack
│   │       ├── freeze.tsx      # Step 1 — Shift freeze confirmation
│   │       ├── cash.tsx        # Step 2 — Cash denomination entry
│   │       ├── card.tsx        # Step 3 — Card receipts
│   │       ├── mobile-money.tsx # Step 4 — Mobile money tokens
│   │       ├── cosign.tsx      # Step 5 — Manager co-sign
│   │       ├── upload.tsx      # Step 6 — Bundle upload + progress
│   │       └── receipt.tsx     # Step 7 — Sync receipt confirmation
│   └── (manager)/
│       ├── _layout.tsx         # Manager tab navigator
│       ├── inventory.tsx       # Gauge log entry + reconciliation view
│       ├── cvp.tsx             # CVP dashboard (post-sync, read-only)
│       ├── shifts.tsx          # Shift history list
│       └── transfers.tsx       # Initiate / acknowledge transfers
│
├── src/
│   ├── db/
│   │   ├── database.ts         # SQLite init, WAL mode, schema creation
│   │   ├── migrations.ts       # Local SQLite migration runner
│   │   └── queries/
│   │       ├── sales.queries.ts
│   │       ├── cylinders.queries.ts
│   │       └── shifts.queries.ts
│   │
│   ├── sync/
│   │   ├── SyncEngine.ts       # Bundle build → encrypt → upload → validate
│   │   ├── BundleBuilder.ts    # Assembles SyncBundle from local SQLite
│   │   ├── BundleEncryptor.ts  # AES-256-GCM encrypt using expo-secure-store key
│   │   ├── ReceiptValidator.ts # SHA-256 hash validation against local sequence
│   │   └── RetryQueue.ts       # Exponential backoff retry manager
│   │
│   ├── stores/
│   │   ├── auth.store.ts       # JWT, user, role, outlet context
│   │   ├── shift.store.ts      # Active shift state, transaction count, running total
│   │   └── sync.store.ts       # Upload status, retry count, last receipt
│   │
│   ├── api/
│   │   ├── client.ts           # Axios mobile instance
│   │   ├── auth.api.ts
│   │   ├── sync.api.ts
│   │   └── time.api.ts         # GET /api/v1/time for clock validation
│   │
│   ├── utils/
│   │   ├── clockGuard.ts       # Anti-tamper clock validation logic (F-08)
│   │   ├── uuidFactory.ts      # Client UUID generation: Device+Cashier+Sequence
│   │   └── formatters.ts
│   │
│   └── types/
│       ├── sync.types.ts       # SyncBundle, EncryptedPayload, SyncReceipt
│       ├── sale.types.ts
│       └── cylinder.types.ts
│
├── assets/
│   └── icons/
│
├── app.json
├── babel.config.js
├── tsconfig.json
├── .env.example
└── package.json
```

---

## Infrastructure — Docker Compose (Local Dev)

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: lpg_db
      POSTGRES_USER: lpg_user
      POSTGRES_PASSWORD: lpg_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build: ./backend
    depends_on: [postgres, redis]
    env_file: ./backend/.env
    ports:
      - "5000:5000"
    volumes:
      - ./backend:/app

  celery:
    build: ./backend
    command: celery -A app.tasks worker --loglevel=info
    depends_on: [postgres, redis]
    env_file: ./backend/.env
    volumes:
      - ./backend:/app

volumes:
  postgres_data:
```

---

## Environment Files

### `backend/.env.example`
```env
DATABASE_URL=postgresql://lpg_user:lpg_pass@localhost:5432/lpg_db
REDIS_URL=redis://localhost:6379/0
CELERY_BROKER_URL=redis://localhost:6379/1
JWT_SECRET_KEY=replace-with-256-bit-random
TENANT_MASTER_KEY=replace-with-256-bit-random
FLASK_ENV=development
FLASK_DEBUG=1
```

### `dashboard/.env.example`
```env
VITE_API_BASE_URL=http://localhost:5000/api/v1
```

### `mobile/.env.example`
```env
EXPO_PUBLIC_API_BASE_URL=http://localhost:5000/api/v1
```
