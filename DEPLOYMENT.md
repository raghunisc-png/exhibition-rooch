# Deploying to exhibition.rooch.in (Hostinger VPS)

This deploys the existing `docker-compose.yml` stack (Postgres + backend + frontend)
behind an Nginx reverse proxy that terminates TLS for `exhibition.rooch.in`, using a
single domain with path-based routing (`/api/*`, `/files/*` → backend, everything else →
frontend). One domain means one certificate and no CORS to configure at all — the
frontend calls its own origin.

## 0. Repo layout on the server

`docker-compose.yml`'s `backend`/`frontend` services build from `./backend` and
`./frontend`, and both are separate git repos not included in this one (see the root
`.gitignore`). On the server you need this exact layout:

```
/opt/exhibition_rooch/
  docker-compose.yml        <- from the root repo
  .env                       <- created below
  backend/                   <- clone of the backend repo
  frontend/                  <- clone of the frontend repo
```

## 1. Point DNS at the server

In wherever `rooch.in`'s DNS is managed (Hostinger's DNS zone, typically), add:

```
Type: A
Name: exhibition
Value: 62.72.31.141
TTL:   default
```

Give it a few minutes to propagate before requesting a certificate in step 5.

## 2. Prepare the VPS

SSH in (swap `root` for whatever user Hostinger gave you, if different):

```bash
ssh root@62.72.31.141
```

Then:

```bash
apt update && apt upgrade -y

# Docker + the Compose plugin
curl -fsSL https://get.docker.com | sh

# Nginx + Let's Encrypt
apt install -y nginx certbot python3-certbot-nginx

# Firewall
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw --force enable
```

## 3. Clone the three repos

You'll need read access configured on the server for all three. If the repos are
private, either add a deploy key per repo, or clone over HTTPS with a PAT (e.g.
`https://<token>@github.com/raghunisc-png/exhibition-rooch.git`).

```bash
mkdir -p /opt/exhibition_rooch
git clone https://github.com/raghunisc-png/exhibition-rooch.git /opt/exhibition_rooch
cd /opt/exhibition_rooch
git clone https://github.com/raghunisc-png/exhibition-rooch-backend.git backend
git clone https://github.com/raghunisc-png/exhibition-rooch-frontend.git frontend
```

## 4. Configure environment variables

Only the **root** `.env` matters for the Docker deployment — `docker-compose.yml` passes
it to the backend container via `env_file:`, and to the frontend build via the
`VITE_API_BASE_URL` build arg. (`backend/.env` and `frontend/.env` are only read by
`uvicorn --reload` / `npm run dev` when running outside Docker — irrelevant here.)

```bash
cp .env.example .env
```

Edit `.env`:

| Variable | Set to | Why |
|---|---|---|
| `POSTGRES_PASSWORD` | a strong random password | the `.env.example` default (`expo_pass`) is a known dev placeholder |
| `VITE_API_BASE_URL` | `https://exhibition.rooch.in` | baked into the frontend at build time; **must** be the full URL — an empty value silently falls back to `http://localhost:8000` (see `frontend/src/api/client.ts`) |
| `JWT_SECRET_KEY` | output of `openssl rand -hex 32` | signs auth tokens; the default is a published dev placeholder |
| `CORS_ORIGINS` | `https://exhibition.rooch.in` | harmless with path-based routing (same-origin), but keep it correct in case that changes later |
| `COMPANY_NAME` / `COMPANY_ADDRESS` / `COMPANY_GSTIN` | your real invoice branding | printed on generated invoice PDFs |
| `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` | real credentials, if you want messages actually sent | note: `deliver_invoice()` in `backend/app/services/invoice_service.py` currently only generates the PDF — WhatsApp/SMS sending is disabled in code regardless of these being set, so this is a no-op until that's re-enabled |

**Also rewrite the seeded admin credential before first deploy.** `docker-compose.yml`'s
backend `command:` auto-creates `admin@rooch.in` / `Rooch@123` on every container start —
that exact password is sitting in this repo's git history, so anyone with repo access
knows it. Before deploying, edit the `command:` line in `docker-compose.yml`:

```yaml
(python -m app.create_admin 'Admin' admin@rooch.in 'Rooch@123' || true)
```

to a real name/email/password, or simply log in with the seeded account once and change
the password through the API/UI immediately after first deploy, then remove the
`create_admin` line from `command:` so it isn't re-asserted on every restart.

## 5. Nginx reverse proxy + TLS

`docker-compose.yml`'s ports are bound to `127.0.0.1` only (not the public interface) —
Nginx is the only thing that talks to the containers directly.

### 5.1 Confirm Nginx is running

```bash
systemctl status nginx --no-pager
```

If it's not active, `systemctl enable --now nginx`.

### 5.2 Write the site config

Write the file directly with a heredoc (no need to open an editor):

```bash
cat > /etc/nginx/sites-available/exhibition.rooch.in <<'EOF'
server {
    listen 80;
    server_name exhibition.rooch.in;

    # Invoice photo uploads go up to 8MB (see MAX_PHOTO_BYTES in
    # backend/app/routers/invoices.py) - default nginx limit is 1MB.
    client_max_body_size 20m;

    location /api/ {
        proxy_pass http://127.0.0.1:8126;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /files/ {
        proxy_pass http://127.0.0.1:8126;
        proxy_set_header Host $host;
    }

    location /health {
        proxy_pass http://127.0.0.1:8126;
    }

    location / {
        proxy_pass http://127.0.0.1:8125;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

(The quotes around `'EOF'` matter — they stop the shell from trying to expand `$host` /
`$remote_addr` itself; those need to reach the file literally, for Nginx to expand.)

### 5.3 Enable the site

```bash
ln -sf /etc/nginx/sites-available/exhibition.rooch.in /etc/nginx/sites-enabled/
```

Hostinger's default Ubuntu image sometimes ships a `default` site listening on `:80`
too, which can shadow this one if its `server_name` is `_` (catch-all). Check:

```bash
ls /etc/nginx/sites-enabled/
```

If `default` is listed and you don't need it, remove just the symlink (not the file in
`sites-available`, in case you want it back):

```bash
rm -f /etc/nginx/sites-enabled/default
```

### 5.4 Test and reload

```bash
nginx -t
```

`nginx -t` validates syntax without touching the running server — fix any errors it
reports before continuing. Once it prints `syntax is ok` / `test is successful`:

```bash
systemctl reload nginx
```

At this point `http://exhibition.rooch.in` should already reach the frontend (no TLS
yet) — worth a quick sanity check before requesting a certificate:

```bash
curl -sI http://exhibition.rooch.in
```

### 5.5 Get the certificate

```bash
certbot --nginx -d exhibition.rooch.in
```

This is interactive the first time: it asks for an email (for renewal/expiry notices)
and asks you to agree to Let's Encrypt's terms. To run it non-interactively instead
(e.g. scripted setup):

```bash
certbot --nginx -d exhibition.rooch.in --non-interactive --agree-tos -m you@rooch.in --redirect
```

`--redirect` makes Certbot add the `:80` → `:443` redirect to the config automatically.
Either way, it rewrites `/etc/nginx/sites-available/exhibition.rooch.in` in place with
the `listen 443 ssl` block and certificate paths.

### 5.6 Verify

```bash
curl -sI https://exhibition.rooch.in
systemctl status certbot.timer --no-pager   # confirms auto-renewal is scheduled
certbot renew --dry-run                      # simulates a renewal without changing anything
```

## 6. Bring the stack up

```bash
cd /opt/exhibition_rooch
docker compose up -d --build
docker compose ps
docker compose logs -f backend   # confirm migrations ran and the admin account was seeded
```

Verify:

```bash
curl -s https://exhibition.rooch.in/health
```

Then open `https://exhibition.rooch.in` in a browser and log in.

## Redeploying after changes

```bash
cd /opt/exhibition_rooch && git pull
cd backend && git pull
cd ../frontend && git pull
cd ..
docker compose up -d --build
```

Repos, for reference:

- Root (this compose stack + deployment docs): https://github.com/raghunisc-png/exhibition-rooch
- Backend: https://github.com/raghunisc-png/exhibition-rooch-backend
- Frontend: https://github.com/raghunisc-png/exhibition-rooch-frontend

## Backups

The Postgres data lives in the `db_data` named volume. Periodic dump:

```bash
docker compose exec -T db pg_dump -U expo_user expo_invoices > backup-$(date +%F).sql
```

Uploaded photos/PDFs live in the `uploads_data` volume — back that up too if you care
about historical invoices, e.g. `docker run --rm -v exhibition_rooch_uploads_data:/data -v $(pwd):/backup alpine tar czf /backup/uploads-$(date +%F).tar.gz /data`.

## Note on local development

`docker-compose.yml`'s ports were changed from `0.0.0.0` to `127.0.0.1` bindings as part
of this deployment hardening (steps above). If you also use `docker compose up` for
local development and were relying on reaching the containers from another device on
your LAN (e.g. testing on a phone), that will no longer work with this file as-is — the
non-Docker local workflow (`npm run dev` / `uvicorn --reload`, both bound to `0.0.0.0`
already) is unaffected and still the right path for that.
