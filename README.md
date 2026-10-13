# VIKMO Smart Spare Parts Management

VIKMO combines the existing Django assignment API with a versioned FastAPI application over the **same PostgreSQL database**. Django remains available on port 8000 for the supplied API contract; the role-protected application API runs on port 8001. The React/Vite client is configured to proxy `/api` to FastAPI.

## Requirements

- Python 3.12 (Python 3.11+ supported)
- PostgreSQL 14+
- Node.js 20+ and npm (React client)

## Local Setup

1. Create a PostgreSQL database and copy `.env.example` to `.env`. Set `DATABASE_URL`, a strong `DJANGO_SECRET_KEY`, and a `JWT_SECRET_KEY` with at least 32 random characters. Keep `.env` private.
2. Install and migrate the database:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python manage.py migrate --noinput
python manage.py seed_data
alembic upgrade head
python -m api.cli seed-categories
```

3. Create the first administrator through the secure prompt (no public admin registration exists):

```powershell
python -m api.cli create-admin
```

4. Start both APIs in separate terminals:

```powershell
python manage.py runserver 0.0.0.0:8000
```

```powershell
$env:JWT_SECRET_KEY = "your-local-32-character-or-longer-secret"
uvicorn api.main:app --host 0.0.0.0 --port 8001 --reload
```

5. FastAPI OpenAPI is at `http://localhost:8001/docs`; liveness and database readiness are `/api/health` and `/api/ready`.

The React screen source and responsive stylesheet are in `frontend/src`. The package manifest was skipped in this workspace, so the browser client is not currently buildable. The FastAPI backend can be run and tested independently using the commands above.

The old assignment API remains at `http://localhost:8000/api/`. Run its supplied contract smoke check with:

```powershell
python smoke_test.py --base-url http://localhost:8000 --track django
```

## Application Features

- Argon2-hashed users, expiring JWT sessions in HTTP-only cookies, role checks, admin-only account provisioning, and dealer ownership enforced server-side.
- Product search/filtering, low-stock review, audited stock adjustments, persistent dealer carts, draft orders, confirmation-time price/stock validation, idempotency, status history, and safe reservation release on cancellation.
- Dashboard and CSV reports read current PostgreSQL records. Order values are explicitly unpaid until a payment integration exists.
- The assistant is **deterministic demo mode** unless a provider adapter is configured. It uses restricted database lookups and can only prepare drafts; it never claims to be a live LLM or commits an order.
- Forecasts use an eight-week moving-average baseline over confirmed/dispatched/delivered order lines. Missing history and fallback supplier lead times are surfaced as warnings. Forecasts recommend replenishment only; they do not create supplier orders.

## Validation

```powershell
python -m pytest tests -q
python manage.py check
python manage.py makemigrations --check --dry-run
alembic upgrade head
```

The FastAPI tables and additive columns are managed by Alembic; the legacy tables remain managed by Django migrations. Do not run `Base.metadata.create_all()` in production.

## Deployment Notes

- Use PostgreSQL as the shared source of truth; configure `DATABASE_URL`, `DJANGO_SECRET_KEY`, `JWT_SECRET_KEY`, explicit `CORS_ORIGINS`, and `APP_ENV=production` in the deployment secret manager.
- Serve behind HTTPS and a reverse proxy. Production sets secure cookies. Do not expose Django's development server to the public internet; use a production WSGI server for the compatibility API and Uvicorn workers for FastAPI.
- Once the frontend package manifest is supplied, build it with `npm run build` and serve `frontend/dist` from the reverse proxy. Route `/api/v1/` to FastAPI and the legacy `/api/` routes to Django.
- Run migrations once per release: `python manage.py migrate --noinput` followed by `alembic upgrade head`. Seed only when intended; the Django seed command is idempotent.
- A scheduled forecast can call the protected forecast-run operation under an admin service identity. Avoid running duplicate schedules. Forecasts do not place purchase orders.

- ## Known Scope

The assistant currently ships with the labeled deterministic fallback; provider-specific LLM tool calling is not enabled. The React source has not been built because its package manifest is missing. The legacy API is retained for compatibility and is intentionally unauthenticated per its checked-in assignment contract; use `/api/v1` for authenticated application workflows.
