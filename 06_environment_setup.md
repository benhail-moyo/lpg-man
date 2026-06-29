# Environment Setup & Developer Onboarding Guide
## Project: Multi-Tenant Multi-Branch LPG Retail Management Platform
**Document Type:** Developer Operations Reference
**Version:** MVP 1.0

---

## Prerequisites

| Tool | Minimum Version | Install |
|---|---|---|
| Python | 3.12 | `pyenv` recommended |
| Node.js | 20 LTS | `nvm` recommended |
| PostgreSQL | 16 | Local or Docker |
| Redis | 7 | Local or Docker |
| Docker + Docker Compose | Latest | docker.com |
| Expo CLI | Latest | `npm install -g expo-cli` |

---

## Quickstart — Full Stack via Docker

```bash
# 1. Clone the repo
git clone https://github.com/your-org/lpg-platform.git
cd lpg-platform

# 2. Copy env files
cp backend/.env.example backend/.env
cp dashboard/.env.example dashboard/.env
cp mobile/.env.example mobile/.env

# 3. Edit backend/.env — set JWT_SECRET_KEY and TENANT_MASTER_KEY
#    Generate: python -c "import secrets; print(secrets.token_hex(32))"

# 4. Start infrastructure
docker-compose up -d postgres redis

# 5. Run migrations
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
flask db upgrade

# 6. Seed a test tenant and System Owner user
flask seed dev

# 7. Start backend
flask run --port 5000

# 8. Start dashboard (new terminal)
cd ../dashboard
npm install
npm run dev

# 9. Start mobile (new terminal)
cd ../mobile
npm install
npx expo start
```

---

## Backend Setup (Manual)

```bash
cd backend
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt

# Create and configure database
createdb lpg_db
cp .env.example .env
# Edit .env with your DATABASE_URL, JWT keys

# Run Alembic migrations
flask db upgrade

# Enable Row-Level Security (run once after migrations)
flask db run-rls-policies

# Seed dev data
flask seed dev

# Start development server
flask run --debug --port 5000

# Start Celery worker (separate terminal)
celery -A app.tasks worker --loglevel=info
```

### Flask CLI Commands

| Command | Description |
|---|---|
| `flask db upgrade` | Apply all pending Alembic migrations |
| `flask db downgrade` | Roll back last migration |
| `flask db run-rls-policies` | Apply PostgreSQL RLS policies (idempotent) |
| `flask seed dev` | Create test tenant, outlets, users, and devices |
| `flask seed clear` | Wipe all data (dev only) |

---

## Dashboard Setup (Manual)

```bash
cd dashboard
npm install
cp .env.example .env
# Set VITE_API_BASE_URL=http://localhost:5000/api/v1

npm run dev          # Dev server on http://localhost:5173
npm run build        # Production build to dist/
npm run preview      # Preview production build locally
```

---

## Mobile Setup (Manual)

```bash
cd mobile
npm install
cp .env.example .env
# Set EXPO_PUBLIC_API_BASE_URL=http://YOUR_LOCAL_IP:5000/api/v1
# Use your machine's LAN IP (not localhost) so the physical device can reach it

npx expo start              # Opens Expo Dev Tools
npx expo start --android    # Start with Android emulator
npx expo start --ios        # Start with iOS simulator (macOS only)
```

### Testing on Physical Device

1. Install the Expo Go app on your Android/iOS device.
2. Ensure your device is on the same Wi-Fi network as your dev machine.
3. Use your machine's LAN IP in `EXPO_PUBLIC_API_BASE_URL`.
4. Scan the QR code from `npx expo start` in Expo Go.

---

## Running Tests

### Backend
```bash
cd backend
source venv/bin/activate
pytest                        # All tests
pytest tests/test_sync.py     # Specific module
pytest -v --tb=short          # Verbose with short tracebacks
pytest --cov=app --cov-report=html  # Coverage report
```

### Dashboard
```bash
cd dashboard
npm run test         # Vitest unit tests
npm run test:ui      # Vitest UI mode
```

---

## Dev User Credentials (after `flask seed dev`)

| Role | Email | Password |
|---|---|---|
| System Owner | owner@lpg-dev.local | dev_password_1 |
| Inventory Controller | controller@lpg-dev.local | dev_password_1 |
| Outlet Manager | manager@lpg-dev.local | dev_password_1 |
| Cashier | cashier@lpg-dev.local | dev_password_1 |

Test device IMEI: `123456789012345`
Test device token: logged to console on `flask seed dev`

---

## Database Utilities

```bash
# Connect to local dev database
psql -U lpg_user -d lpg_db

# Check RLS policies are active
SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public';

# Check current migrations applied
flask db current

# Generate a new migration after model changes
flask db migrate -m "add_column_x_to_outlets"
flask db upgrade
```

---

## Generating Secrets

```python
# 256-bit random key for JWT_SECRET_KEY and TENANT_MASTER_KEY
import secrets
print(secrets.token_hex(32))
```

```bash
# Or via openssl
openssl rand -hex 32
```

---

## Common Issues

| Symptom | Fix |
|---|---|
| `RLS policy violation` on API calls | Ensure `flask db run-rls-policies` was run. Verify JWT contains `company_id`. |
| Mobile device cannot reach backend | Use LAN IP in `.env`, not `localhost`. Check firewall allows port 5000. |
| Celery tasks not processing | Ensure Redis is running. Check `CELERY_BROKER_URL` in `.env`. |
| `SYNC_002` clock error in testing | Disable clock check in dev via `SKIP_CLOCK_CHECK=true` in backend `.env`. |
| Expo build fails on SQLite | Run `npx expo install expo-sqlite` to align with the Expo SDK version. |
