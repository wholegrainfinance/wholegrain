<h1 align="center">Deploying Wholegrain</h1>

A first deployment from nothing to a working, TLS-terminated, tenant-isolated
install. Roughly an hour, most of it waiting for images to build.

This document is opinionated about the order of steps, because several of them
are circular if taken in the wrong one, and about a handful of failure modes that
are silent — the kind where everything looks correct and something important is
switched off. Those are marked **⚠︎**.

---

## What you need first

| | |
|---|---|
| A host | 2 vCPU / 8 GB is comfortable. 1 vCPU / 2 GB works but is tight — see [Small hosts](#small-hosts). |
| Two DNS names | one for the API, one for the dashboard, both A-records to the host |
| Object storage | S3-compatible or Azure Blob |
| PostgreSQL | the `database` module, or a managed server with `pgvector`, `pg_trgm`, `btree_gin` |
| An LLM API key | only for the optional agent, labelling and embedding modules |

The two DNS names may be anything — `api.example.com` and `app.example.com`, or
`acme-api.example.com` and `acme-app.example.com`. They are independent
hostnames, not a shared base domain.

---

## Order of operations

Four of these steps depend on an earlier one in a way that is not obvious until
it fails:

```
1. DNS ─────────────────► must resolve before certbot will issue
2. certificates ────────► must exist before nginx will start
3. nginx ───────────────► creates wholegrain-edge, which the frontend joins
4. database ────────────► must be healthy before migrations
5. migrations ──────────► must run before the RLS roles can be granted
6. RLS login roles ─────► must exist before the API can connect as app_api
7. API, then frontend
```

### 1. Clone the modules

```sh
git clone https://github.com/wholegrainfinance/nginx.git
git clone https://github.com/wholegrainfinance/database.git
git clone https://github.com/wholegrainfinance/backend.git
git clone https://github.com/wholegrainfinance/browser-frontend.git
```

### 2. Point DNS at the host

Both names, A records, before going further. Certbot validates by fetching a
token over HTTP from the name you are asking it to certify; if the name does not
resolve to this host yet, issuance fails with a message about the challenge
rather than about DNS.

### 3. Issue certificates — with nothing on port 80

```sh
certbot certonly --standalone --cert-name "$API_DOMAIN" -d "$API_DOMAIN"
certbot certonly --standalone --cert-name "$APP_DOMAIN" -d "$APP_DOMAIN"
```

Two circular things are resolved here, both worth understanding:

- **`--standalone`, not `--webroot`.** nginx will not start without the
  certificate files its config references, so at this point there is no nginx to
  serve a webroot challenge. Certbot binds port 80 itself. Every *renewal*
  afterwards uses the webroot path the proxy serves and needs no downtime.
- **One certificate per host, separate runs.** A single
  `certbot -d api -d app` produces one SAN certificate in one directory named
  after the first host, but the vhost template expects a directory per domain.
  nginx then fails on a missing file and the error points nowhere near certbot.

### 4. nginx

```sh
cd nginx && cp .env.example .env    # API_DOMAIN, APP_DOMAIN, CERT_DIR, BLOB_CSP_ORIGIN
docker compose up -d
```

First, because it creates the `wholegrain-edge` network. That network is
`internal: true` — no route off the host — and the static frontends live on it
alone with nginx. A container that only ever serves files needs no outbound
access, and denying it turns a foothold into a dead end.

**⚠︎ `BLOB_CSP_ORIGIN`** is the scheme-and-host of your object storage, no bucket
and no path. Wrong or missing, document previews fail in the browser with a CSP
violation while working perfectly in `curl` — the two disagree because only the
browser enforces CSP.

### 5. Database

```sh
cd database && cp .env.example .env    # set POSTGRES_PASSWORD
docker compose up -d
```

Or point at a managed server. It needs `vector`, `pg_trgm` and `btree_gin`; the
module ships them.

### 6. Migrations

```sh
cd backend && cp .env.example .env
# DATABASE_URL / MIGRATION_DATABASE_URL -> the OWNER role
docker compose run --rm wholegrain-backend alembic upgrade head
```

Migrations run as the database **owner**. The application will not.

### 7. Create the RLS login roles

```sh
psql -v ON_ERROR_STOP=1 \
     -v api_password="'<api-secret>'" -v worker_password="'<worker-secret>'" \
     -f backend/db/sql/rls_roles_setup.sql
```

Then point the API at the scoped role, keeping the owner for migrations:

```ini
DATABASE_URL=postgresql://app_api:<api-secret>@wholegrain-db:5432/wholegrain
MIGRATION_DATABASE_URL=postgresql://<owner>:<pw>@wholegrain-db:5432/wholegrain
```

**⚠︎ This step is the one that matters most, and the one nothing forces you to
take.** See [Tenant isolation](#tenant-isolation-read-this-one).

### 8. API, then frontend

```sh
cd backend          && docker compose up -d
cd browser-frontend && docker compose up -d
```

The frontend takes `API_URL`, `APP_URL`, `BRAND` and `SUPPORT_EMAIL` from the
container environment at *start* time, not build time — its entrypoint writes
them into `config.js`, so one built image serves any install. Changing a domain
means recreating the container, not rebuilding it.

### 9. Create the first account

Self-registration is open by default; the first account is not special.

```sh
curl -X POST https://$API_DOMAIN/auth/register-user \
     -H 'Content-Type: application/json' \
     -d '{"email":"you@example.com","password":"..."}'
```

Then create a workspace, and you have a working install.

---

## Tenant isolation (read this one)

Isolation rests on 15 row-level-security policies. They are enforced by the
database, but **only for a role that is subject to them**, and it is entirely
possible to deploy a system where they are switched off without anything
appearing wrong.

A table owner bypasses RLS. No table here sets `FORCE ROW LEVEL SECURITY`. So an
API connected as the owner never evaluates a single policy: it starts, serves,
passes its tests, and lets every workspace read every other workspace's data. The
application-level permission checks still return 403 on the obvious paths, which
makes the gap look closed when it is not.

If you deploy the bundled `database` module, `POSTGRES_USER` is a superuser that
owns every table — precisely the role you must not give the API.

Confirm isolation is real, using **the API's own credentials**:

```sql
SELECT rolsuper, rolbypassrls FROM pg_roles WHERE rolname = current_user;
--  f | f     <- anything else means RLS is not running

SET ROLE app_api;
SELECT set_config('app.current_user_id', '', false);
SELECT count(*) FROM entities;   -- must be 0: no principal, no rows
RESET ROLE;
```

The last query is the one to keep. Fail-closed with no principal set is the
property that makes a forgotten `set_user()` a bug that returns nothing rather
than a bug that returns everything.

Three roles, by trust tier:

| Role | Used by | RLS |
|---|---|---|
| owner | migrations only | bypass (owner) |
| `app_api` | the API | **scoped** — per-request `app.current_user_id` GUC |
| `app_worker` | pipeline / extraction | bypass, via `app_trusted` membership |

`app_trusted` is a password-less group created by migration `0002`. It must
exist: `app.tenant_visible()` calls `pg_has_role(..., 'app_trusted', 'MEMBER')`,
and `pg_has_role` **raises** on an unknown role rather than returning false, so a
missing group means every query against a protected table errors out. This is
invisible while the API runs as the owner, and appears the moment you switch to
`app_api` — that is, when you start doing it right.

The policies also read tables in the `app` schema directly
(`app.dataroom_permissions`, `app.entity_members`), and a policy expression is
evaluated as the *querying* role. `app_api` therefore needs privileges in the
`app` schema as well as `public`; granting only on `public` produces a system
where listing a folder fails with `permission denied for table entity_members`.
`rls_roles_setup.sql` handles both.

---

## Object storage layout

`STORAGE_BUCKET_MODE` picks between two layouts:

| mode | container | object key |
|---|---|---|
| `per_workspace` (default) | `<S3_BUCKET_PREFIX><workspace_id>` | `<file_id>.<ext>` |
| `single` | `STORAGE_BUCKET_NAME` | `<workspace_id>/<file_id>.<ext>` |

**Per-workspace is the stronger isolation** — a credential scoped to one bucket
cannot reach another tenant's objects, and the storage provider enforces the
boundary. It needs `CreateBucket`, since buckets are made lazily on first upload.

**Single-bucket** exists for providers and credentials that cannot create
buckets. It needs no bucket rights, and in exchange tenant separation becomes a
key prefix that the *application* is responsible for honouring rather than the
provider. Choose it deliberately.

**⚠︎ The mode is fixed at install.** Object keys are persisted in
`files.original_blob_name` and `files.pdf_blob_name`. Switching modes on a
database that already holds files orphans every one of them — the rows keep the
old keys and the code looks for the new ones. Migrating means rewriting both the
objects and those columns.

**⚠︎ The S3 region must match the endpoint**, not the marketing name of the
region. SigV4 signs the region string, so a provider whose endpoint is
`fsn1.example.com` generally wants `fsn1`, and a plausible-looking `eu-central`
fails as `SignatureDoesNotMatch` — an error that reads like bad credentials.

---

## Configuration hazards

**Comments in `.env`.** Every comment in the shipped examples sits on its own
line. Compose's dotenv parser strips a trailing `# ...`, but the plain Docker CLI
parser (`docker run --env-file`) does not — it takes the rest of the line
verbatim, so `FOO=15   # minutes` becomes the string `"15   # minutes"` and the
first `int()` of it kills the process at import. Keep comments on their own line.

**`.env` must never enter an image.** Each module's `.dockerignore` excludes it.
If you add a module, do the same: `COPY . .` otherwise bakes your JWT secret,
database password, OAuth private key and storage credentials into a layer that
anyone who can pull or `docker save` the image can read. It is also a correctness
trap — the API declares `env_file='.env'`, so a baked build-time copy silently
shadows the values compose injects at run time, and a changed setting appears in
`printenv` while the application keeps using the old one.

**`cpus:` may not exceed the host's core count.** Docker rejects the container
outright rather than clamping: `range of CPUs is from 0.01 to 1.00, as there are
only 1 CPUs available`. The API and database modules take theirs from env
(`API_CPUS`, `PG_CPUS`) for this reason.

---

## Small hosts

1 vCPU / 2 GB runs the four services, with caveats:

- Give the host **swap** (4 GB). The frontend's Vite build is the peak, and it
  will not complete reliably without it.
- Retune Postgres — the defaults assume ~8 GB. See `database/.env.example`.
- Set `API_CPUS` to at most the host's core count.
- The extraction pipeline (Docling, LibreOffice, baked models) will not fit.
  Text extraction is a separate module for exactly this reason.

Build images one at a time. Two concurrent builds on a small host is a known way
to take the whole box down.

---

## Verifying a deployment

```sh
# TLS and routing
curl -sI  http://$APP_DOMAIN | head -1                       # 301 -> https
curl -s -o /dev/null -w '%{http_code}\n' https://$API_DOMAIN/docs
curl -s -o /dev/null -w '%{http_code}\n' http://<host-ip>/   # 000: unmatched host

# CORS: the app origin allowed, anything else refused
curl -sI -X OPTIONS https://$API_DOMAIN/auth/login \
     -H "Origin: https://$APP_DOMAIN" -H 'Access-Control-Request-Method: POST' \
     | grep -i access-control-allow-origin

# The frontends really have no egress
docker run --rm --network wholegrain-edge curlimages/curl \
     -s -m 8 -o /dev/null -w '%{http_code}\n' https://example.com   # 000

# Tenant isolation — see "Tenant isolation" above; the fail-closed count is the one
```

A second account, in no workspace, must receive 403 (not an empty list) when it
names another workspace's `workspace_id`, and must see nothing at the database
level under `app_api`. Test both layers: the first is the application's check,
the second is the one that still holds when the application's check has a bug.

---

## Hardening

The root [README](README.md#hardening) covers the infrastructure side — database
firewalling, WAF, region locality, egress-free frontends, admin access. The
deployment-side controls are: run the API as `app_api`, keep `.env` out of
images, keep `read_only: true` on every container that does not need to write,
and keep the frontends on `wholegrain-edge`.

See [SECURITY.md](SECURITY.md) for what the shipped configuration does and does
not defend against.
