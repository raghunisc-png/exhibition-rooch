# Exhibition Booth Invoicing

A small full-stack app for capturing customer purchases at trade-show/exhibition
booths and instantly emailing... no — instant-messaging the invoice to the
customer via **WhatsApp** (falling back to **SMS**).

A sales agent at the booth: photographs the product the customer bought,
enters the customer's name, phone number and product details, taps submit —
the app generates a PDF invoice and sends it straight to the customer's phone.
It works even when the booth wifi is unreliable: invoices are saved on the
device and sync automatically once a connection is available.

## Architecture

```
frontend/   React + TypeScript PWA (Vite, Tailwind CSS)
            - Offline-first: IndexedDB queue (Dexie) + background sync
            - Installable on phone/tablet home screen, works full-screen
backend/    FastAPI + PostgreSQL
            - JWT-authenticated agent accounts
            - Multipart invoice creation + batch offline-sync endpoint
            - Server-side PDF generation (ReportLab)
            - Twilio WhatsApp/SMS delivery with automatic SMS fallback
```

Data flow: agent fills the form and takes a photo → **online**: submitted
directly to the API, which stores it, renders a PDF, and sends WhatsApp (or
SMS if WhatsApp fails/isn't reachable) → **offline**: saved to IndexedDB on
the device, and a background sync loop (`online` event + 30s poll) pushes
the queue to `/api/sync/invoices` as soon as connectivity returns. Every
invoice carries a client-generated UUID so retries/duplicate syncs can never
create duplicate invoices or double-send messages.

## Quick start (Docker)

```bash
cp .env.example .env        # fill in Twilio credentials, JWT secret, etc.
docker compose up --build
```

- Backend API: http://localhost:8000 (docs at `/docs`)
- Frontend: http://localhost:5173

Create the first admin login:

```bash
docker compose exec backend python -m app.create_admin "Jane Doe" jane@example.com "StrongPass123"
```

Then open the frontend, sign in, and create an invoice. Add more booth-agent
accounts from `/docs` (`POST /api/auth/agents`, admin-only) or build a small
admin screen on top of that endpoint later.

## Running locally without Docker

See `backend/README.md` and the commands below for the frontend:

```bash
cd frontend
cp .env.example .env   # VITE_API_BASE_URL=http://localhost:8000
npm install
npm run dev
```

## Configuring WhatsApp/SMS delivery (Twilio)

1. Create a [Twilio](https://www.twilio.com) account.
2. For quick testing, use the Twilio **WhatsApp Sandbox** (Messaging →
   Try it out → Send a WhatsApp message) — set `TWILIO_WHATSAPP_FROM` to the
   sandbox number and have each test customer phone join the sandbox by
   sending the join code once.
3. For production WhatsApp sending to any customer without a join step,
   apply for a WhatsApp Business Sender in the Twilio console and get your
   message template(s) approved by Meta (required for the first message in
   a 24h window — see the comment at the top of
   `backend/app/services/messaging.py`).
4. Buy/port an SMS-capable number for `TWILIO_SMS_FROM` as the fallback
   channel for customers WhatsApp can't reach.
5. Set `TWILIO_ACCOUNT_SID` and `TWILIO_AUTH_TOKEN` from the Twilio console.
6. Set `PUBLIC_BASE_URL` to a **publicly reachable** HTTPS URL for the
   backend in production — Twilio fetches the invoice PDF from this URL to
   attach it to the WhatsApp message, so `localhost` will not work outside
   local dev.

If Twilio isn't configured yet, invoice creation and PDF generation still
work fully; message delivery is simply logged as failed and can be retried
later with the "Resend" button once credentials are set.

## Production deployment notes

- **Database**: run Postgres as a managed service (RDS, Cloud SQL, etc.)
  rather than the bundled `docker-compose` container; point `DATABASE_URL`
  at it and run `alembic upgrade head` on deploy.
- **File storage**: for more than one backend replica, switch
  `STORAGE_BACKEND=s3` and fill in the `S3_*` env vars (S3, R2, MinIO, or
  any S3-compatible provider) instead of local disk, since local files
  aren't shared across replicas.
- **HTTPS**: put the backend behind a reverse proxy / load balancer with
  TLS termination (required for Twilio to fetch invoice PDFs, and for
  camera access to work reliably in the PWA on iOS Safari).
- **Secrets**: set `JWT_SECRET_KEY` to a long random value and never commit
  `.env`.
- **Backups**: enable automated Postgres backups; invoices and message logs
  are the system of record for what was promised/sent to each customer.
- **Scaling agents**: create one login per booth staff member (or per booth)
  via `POST /api/auth/agents`; each invoice records which agent captured it.

## Tests

```bash
cd backend && pytest -q
cd frontend && npm run build   # runs the TypeScript compiler as a check
```
