<h1 align="center">Deploying Wholegrain</h1>

A first deployment from nothing to a working, TLS-terminated, tenant-isolated
install. Roughly an hour, most of it waiting for images to build.

_Last walked through end to end on 2026-11-07, against an empty PostgreSQL at
backend migration `0059`: migrations, login roles, API start, first account,
workspace, first admin and the fail-closed check below._

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
| Object storage | S3-compatible (AWS, MinIO, …) or Azure Blob |
| PostgreSQL 16 | the `database` module, or a managed server with `vector`, `pg_trgm` and `btree_gin` available |
| An LLM / embeddings key | only for the optional agent, labelling and embedding modules |

The two DNS names may be anything — `api.example.com` and `app.example.com`, or
`acme-api.example.com` and `acme-app.example.com`. They are independent
hostnames, not a shared base domain.

**All configuration is environment.** Every module reads a `.env` next to its
compose file (copy the shipped `.env.example`); no hostname, credential or
bucket name is baked into an image or a template. Changing a domain means
editing `.env` and recreating the container, not rebuilding it.

---

## Order of operations

Six of these steps depend on an earlier one in a way that is not obvious until
it fails:

```
 1. DNS ─────────────────► must resolve before certbot will issue
 2. certificates ────────► must exist before nginx will start
 3. wholegrain-network ──► nginx, database and frontend join it as external
 4. nginx ───────────────► creates wholegrain-edge, which the frontend joins
 5. database ────────────► must be healthy before migrations
 6. migrations to 0002 ──► create the schemas the login roles are granted on
 7. RLS login roles ─────► must exist BEFORE the remaining migrations run
 8. API, then frontend   (the API runs the remaining migrations, and on every start)
 9. first account ───────► register over HTTP; nothing special about it
10. first admin ─────────► SQL, once: role assignment is itself admin-only
11. connector secrets ───► optional: credentials for mail, SharePoint, Microsoft 365
12. document ingestion ──► optional: your own service, or the document-service modules
```

Step 10 is the only one that touches the database by hand after setup, and it
is a genuine chicken-and-egg rather than an omission.

### 1. Clone the modules

```sh
git clone https://github.com/wholegrainfinance/nginx.git
git clone https://github.com/wholegrainfinance/database.git      # optional: or a managed server
git clone https://github.com/wholegrainfinance/backend.git
git clone https://github.com/wholegrainfinance/frontend.git
git clone https://github.com/wholegrainfinance/cli.git           # optional
```

Some of these repositories may still be private; ask for access if a clone is
refused.

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

### 4. The shared network

```sh
docker network create wholegrain-network
```

The backend's compose file creates `wholegrain-network` with a pinned name; the
nginx, database and frontend compose files join it as `external: true`. Creating
it once by hand removes the ordering problem: without it, `docker compose up`
for nginx or the database refuses to start with "network declared as external,
but could not be found". If it already exists, the command fails harmlessly.

### 5. nginx

```sh
cd nginx && cp .env.example .env
#   API_DOMAIN, APP_DOMAIN, CERT_DIR, CSP_API_ORIGINS, BLOB_CSP_ORIGIN, VHOSTS
docker compose up -d
```

Before the frontend, because it creates the `wholegrain-edge` network. That
network is `internal: true` — no route off the host — and the static frontends
live on it alone with nginx. A container that only ever serves files needs no
outbound access, and denying it turns a foothold into a dead end.

The vhosts are templates. At start the stock nginx entrypoint runs `envsubst`
over them with the values from `.env`, writes the result to a tmpfs, and loads
it; the image itself stays read-only and identical for every install.
Upstreams (`BACKEND_UPSTREAM`, `FRONTEND_UPSTREAM`, …) are resolved at request
time, so a container that is not running yet costs a 502 on its own route and
nothing else. Certificates, by contrast, are read at load time — a vhost whose
certificate is missing stops nginx from starting at all.

**⚠︎ Set `VHOSTS`.** The template set covers more public names than a small
install has (alternate API and app names, an add-in, redirect-only domains).
`VHOSTS` keeps only the ones you list, e.g.
`VHOSTS="00-shared 01-default-deny 05-http-redirect 10-api 20-app"`. Left empty,
every vhost is rendered, each needs a certificate for its name, and nginx
refuses to start on the first one that is missing.

**⚠︎ An unset variable silently keeps the compose default.** Every variable the
templates use has a default in `docker-compose.yml`, because `envsubst` leaves an
undefined `${VAR}` as literal text that nginx then rejects. The price is that a
forgotten line in `.env` does not fail — it quietly renders someone else's
default. Set every domain you mean, then check the rendered config:
`docker exec wholegrain-nginx nginx -T | grep server_name`.

**⚠︎ `BLOB_CSP_ORIGIN`** is the scheme-and-host of your object storage, no bucket
and no path. Wrong or missing, document previews fail in the browser with a CSP
violation while working perfectly in `curl` — the two disagree because only the
browser enforces CSP.

### 6. Database

**Bundled** — the `database` module: PostgreSQL 16 with pgvector, `pg_trgm` and
`btree_gin` created on first start, memory settings taken from env, and backup /
restore scripts.

```sh
cd database && cp .env.example .env    # POSTGRES_PASSWORD, PG_* tuning, PG_MEM_LIMIT, PG_CPUS
docker compose up -d
```

**Managed** — any PostgreSQL 16 where `vector`, `pg_trgm` and `btree_gin` can be
created. Nothing else changes: the API and the pipeline take a connection URL,
and a managed server is just a different host in it. Keep it in the same region
as the host and reachable only from it (see the [README](README.md#hardening)).

### 7. The first two migrations

```sh
cd backend && cp .env.example .env
#   DATABASE_URL           -> the OWNER role for now (app_api does not exist yet)
#   MIGRATION_DATABASE_URL -> the OWNER role
docker compose run --rm wholegrain-backend alembic upgrade 0002_create_app_trusted_role
```

**⚠︎ Only up to `0002`, not `heads`.** `0001` creates the schemas and `0002` the
`app_trusted` group; the login roles of the next step are granted on both. Every
migration after that grants its new tables to `app_api`, `app_worker` and
`app_sql_ro`, and often narrows them again (append-only tables lose `DELETE`) —
but only *if the role already exists*, because a migration cannot invent a
password. Run all migrations first and every one of those statements is
skipped without an error: the document service cannot read its connectors,
`app_api` may delete rows that are meant to be permanent, and ad-hoc SQL fails
with `permission denied to set role "app_sql_ro"`.

Migrations run as the database **owner**. The application will not. The
migration connection is resolved in this order: `MIGRATION_DATABASE_URL`, then
`DB_MIGRATION_USER` / `DB_MIGRATION_PASSWORD` on the API's host and database,
then `DATABASE_URL` as a last resort — which is only correct in development,
where the API user *is* the owner.

`heads`, plural: the migration graph has branch points and a downstream install
may carry its own chain, so there can legitimately be more than one head. With
`head` the command refuses as soon as that happens.

The remaining migrations run in step 9, and from then on you never run them by
hand: **the backend container runs `alembic upgrade heads` on every start**,
before it starts serving. Upgrading is pulling the new code and recreating the
container. That also means a migration that fails stops the API from starting —
read the container log, not the browser, when an upgrade "does nothing".

### 8. Create the RLS login roles

```sh
psql "$OWNER_URL" -v ON_ERROR_STOP=1 \
     -v api_password="'<api-secret>'" -v worker_password="'<worker-secret>'" \
     -f backend/db/sql/rls_roles_setup.sql
```

It creates `app_api` (the API, subject to RLS) and `app_worker` (the document
service, bypasses RLS through `app_trusted`), and grants both what they need in
the `public` and `app` schemas. Everything later — `app_sql_ro` for `app_api`
(the read-only role behind `POST /query/sql`, `wholegrain sql` and the MCP
`run_sql` tool), the connector tables for `app_worker`, the narrowed grants on
append-only tables — the remaining migrations add in step 9, because the roles
now exist.

**⚠︎ Run it once, and never again on a database in use** — not even to rotate a
password. Its blanket grants re-open every table a later migration narrowed.
Rotate with `ALTER ROLE app_api WITH PASSWORD '...'` (and the same for
`app_worker`).

Then point the API at the scoped role, keeping the owner for migrations:

```ini
DATABASE_URL=postgresql://app_api:<api-secret>@wholegrain-db:5432/wholegrain
MIGRATION_DATABASE_URL=postgresql://<owner>:<pw>@wholegrain-db:5432/wholegrain
```

**⚠︎ This step is the one that matters most, and the one nothing forces you to
take.** See [Tenant isolation](#tenant-isolation-read-this-one).

### 9. API, then frontend

```sh
cd backend  && docker compose up -d --build      # runs migrations 0003 … head first
cd frontend && cp .env.example .env && docker compose up -d --build
```

The rest of the backend's `.env` is grouped by concern in `.env.example`:
`JWT_SECRET`; the public URLs (`PUBLIC_BASE_URL` is this API's external origin
and the OAuth issuer, `APP_BASE_URL` the dashboard, `CORS_ORIGINS` the browser
origins allowed); the MCP / OAuth signing keypair; transactional mail
(`EMAIL_PROVIDER=console` until you configure SMTP or Graph); storage (below);
the secret store (step 13); and the optional embeddings provider.

**⚠︎ `OAUTH_RS256_PRIVATE_KEY` empty** makes the API generate a throwaway key
per process — fine in development, and in production it silently invalidates
every MCP connection on each restart. Generate a keypair once (the command is
in `.env.example`).

The frontend takes `API_URL`, `APP_URL`, `BRAND` and `SUPPORT_EMAIL` (plus a
few optional branding values) from the container environment at *start* time,
not build time — its entrypoint writes them into `config.js`, so one built image
serves any install. Changing a domain means recreating the container, not
rebuilding it.

### 10. Create the first account

Self-registration is open, and the first account is not special:

```sh
curl -X POST https://$API_DOMAIN/auth/register-user \
     -H 'Content-Type: application/json' \
     -d '{"email":"you@example.com","password":"..."}'
```

Every registered account receives the `default` role, which carries the member
column of the permission matrix: it can create a workspace and work in the
workspaces it belongs to.

**⚠︎ Registration is ungated.** Anyone who can reach the API can create an
account. They land in no workspace and therefore see nothing, but if that is not
acceptable, put the API behind an allowlist before it is public.

Then create a workspace (`POST /workspace/`, or the dashboard), and you have a
working install. The creator becomes its `owner`.

### 11. Bootstrap the first admin — the one step that needs SQL

Role assignment requires the `admin` role, so the first admin cannot be made
through the API: nobody holds it yet. The role rows themselves — `admin`,
`member`, `default`, `agent`, `external` — and their permission matrix are seeded
by migration `0011_role_permissions`; what is missing is one membership. Break
the cycle once, directly in the database, and everything after this is
API-only.

Run as the database **owner**, substituting the email you registered above:

```sql
INSERT INTO user_roles (user_id, role_id)
SELECT u.id, r.id
FROM users u, roles r
WHERE u.email = 'you@example.com'          -- <-- your registered address
  AND r.name  = 'admin'
ON CONFLICT DO NOTHING;
```

Idempotent, so re-running it is safe. Roles travel in the access token, so log
in again before checking:

```sh
curl -s https://$API_DOMAIN/roles/roles/me -H "Authorization: Bearer $TOKEN"
# {"roles":["default","admin"], ...}
```

From here every further role is an API call:

```sh
curl -X POST https://$API_DOMAIN/roles/roles/assign \
     -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -d '{"user_email":"colleague@example.com","roles":["admin"]}'
```

Note the doubled segment: `/roles/roles/assign`, not `/roles/assign`. The
permission matrix itself is managed the same way — `GET /roles/roles/matrix`,
`PUT /roles/roles/matrix/{role}` — and only the `admin` role may change it.

### 12. Adding people to a workspace

An API call, by a workspace **owner**:

```sh
curl -X POST https://$API_DOMAIN/settings/add-user/$WORKSPACE_ID \
     -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
     -d '{"email":"colleague@example.com","role":"member"}'
```

`role` is a property of the membership, not of the person — one account can own
one workspace and only read another:

| membership role | can |
|---|---|
| `owner` | everything a member can, plus manage members and workspace settings |
| `member` | read and write the workspace's data |
| `read_only` | read only — enforced by the database's write policies, not by the routes |

An email with no account yet gets a passwordless one: the person signs in with
an emailed one-time code, so mail delivery must be configured for invitations to
work. Members are listed, re-roled and removed under `/settings/users/{workspace_id}`;
the last owner can be neither demoted nor removed.

Membership is precisely what RLS resolves — `app.tenant_visible()` looks the user
up in `user_workspaces` — so the new member can read the workspace immediately,
with no restart and no cache to clear.

One rule the database enforces for you: a user holding the `external` role
cannot be made a workspace member. A trigger rejects the insert. External users
are dataroom guests, granted folders; they are not tenants.

### 13. Connector secrets and Microsoft 365 (optional)

A **connector** is one source or target per workspace — a mailbox, a SharePoint
site, a cloud folder — kept in the `connectors` table and managed under
*Settings → Connectors* or `POST /connectors/{workspace_id}`. Its credentials
never go into that table, into `.env` or into git: they live in the **secret
store**, chosen by `SECRET_STORE_BACKEND`:

| backend | where the values are | needs |
|---|---|---|
| `db` (default) | the `connector_secrets` table, envelope-encrypted | `SECRET_KEK` — base64 of 32 random bytes |
| `azure_vault` | Azure Key Vault, through the host's managed identity | `AZURE_VAULT_URL`, no secret in the env |

**⚠︎ `SECRET_KEK` is the only key to every stored credential.** Generate it once
(the command is in `.env.example`) and back it up outside the host. A lost KEK
cannot be recovered: every connector credential has to be provisioned again. A
document-service container that reads credentials needs the same KEK.

Values are written with the secrets tool, inside the backend container. It asks
for the value twice without echoing it, stores only that, and writes an audit
row; it has no delete and never prints a value back:

```sh
docker exec -it wholegrain-backend python -m tools.secrets list
docker exec -it wholegrain-backend python -m tools.secrets set <namespace> <workspace_id> --by you@example.com
docker exec -it wholegrain-backend python -m tools.secrets check <namespace> <workspace_id>
```

**Microsoft 365 through your own app.** Instead of an app registration in every
client's tenant, an install can own one multi-tenant Microsoft app; a client
then connects an account under *Settings → Connectors → Apps*, and SharePoint
connectors read as that account (read-only). The refresh token stays in the
secret store; the browser never sees a token. To set it up, in the Entra tenant
that should own the app:

1. *App registrations → New registration*: accounts in **any organizational
   directory (multitenant)**; redirect URI of type **Web**:
   `<PUBLIC_BASE_URL>/apps/microsoft/callback`.
2. *API permissions → Microsoft Graph → Delegated*: `openid`, `profile`, `email`,
   `offline_access`, `User.Read`, `Mail.Read.Shared`, `Sites.Read.All`. All of
   them read-only; no admin consent needed in your own tenant.
3. *Certificates & secrets → New client secret*, and store it — not in `.env`:
   `docker exec -it wholegrain-backend python -m tools.secrets set-app microsoft --by you@example.com`
4. `MICROSOFT_APP_CLIENT_ID=<application (client) id>` in the backend `.env`,
   then recreate the container. `GET /apps` now reports `"configured": true`.

Delegated access sees exactly what the signed-in account sees. Have each client
sign in with a service account that can open only the shared mailbox and the
sites you should read, not with a person's own account.

### 14. Document ingestion (optional)

The core install has documents as soon as people upload them through the
dashboard. Bulk ingestion — mirroring a document tree, extracting text, building
search chunks — is a separate service, outside the backend:

- *sync* mirrors a directory tree into `folders` / `files` and object storage.
  The first directory level is the workspace. Identity is the SHA-256 of the
  bytes, so moves and renames are detected and nothing is re-extracted needlessly.
- *ingesters* pull mail (Microsoft Graph, mail exports on disk), WhatsApp
  messages and cloud folders, driven by the `connectors` table.
- *extract* turns files marked extractable into `file_extractions` (PyMuPDF,
  Docling, OCR).
- *chunk/embed* writes `file_chunks` with embeddings for vector search
  (optional; without it search is lexical-only).

The reference implementation is the `sync`, `ingester` and `extraction`
repositories (ask for access). They connect as **`app_worker`** and must use the
same storage backend and bucket mode as the API. Extraction carries Docling,
LibreOffice and OCR models and does not fit a small host; run it elsewhere
against the same database and storage.

**⚠︎ A missing file is an archived file.** Sync soft-archives every file it can no
longer find on disk, and archived rows are deleted after a grace period. A
directory that is silently empty — an unmounted volume after a reboot, a
synchroniser that stopped, the wrong mount path — produces a run that looks
completely ordinary and archives the workspace. Mount the tree read-only, check
it is populated before each run, and watch the archive count. Rows created
through the API (`origin = 'api'`, inherited by subfolders) are never archived
by sync, whatever is on disk.

**Your own document service.** Anything that writes the same tables — `folders`,
`files`, `file_extractions`, and `tasks` for ingested emails or messages — and
puts the bytes into the same object-storage layout (below) is a complete
substitute. Wholegrain only reads what it writes. Give it its own database role
modelled on `app_worker`, and keep it on the same migration head as the API: the
schema is the contract.

### 15. Copying a workspace between servers (optional)

The `mirror` module pulls one workspace from a source server into a workspace on
a target server and keeps it current: folders, file metadata and documents,
extracted text, the ontology with its file links, tasks and comments. Search
chunks are rebuilt on the target. It is one-way — the source is only read —
incremental, and deletes on the target only rows that came from that source.

Both sides enable the module on the workspace, each issues a token bound to that
one workspace and one direction (`export` on the source, `import` on the
target), and the `wholegrain` CLI runs `wholegrain mirror pull <name>`, usually
from cron on the target. Both servers must be on the same migration head.
Setup step by step: `docs/MIRROR.md` in the backend repository.

---

## Tenant isolation (read this one)

Isolation rests on row-level-security policies on every workspace-scoped table.
They are enforced by the database, but **only for a role that is subject to
them**, and it is entirely possible to deploy a system where they are switched
off without anything appearing wrong.

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

Four roles, by trust tier:

| Role | Used by | RLS |
|---|---|---|
| owner | migrations only | bypass (owner) |
| `app_api` | the API | **scoped** — per-request `app.current_user_id` (or a single pinned workspace) |
| `app_sql_ro` | `POST /query/sql`, switched into for one statement | **scoped**, and SELECT on workspace tables only |
| `app_worker` | pipeline / document service | bypass, via `app_trusted` membership |

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

`STORAGE_BACKEND` is `s3` (AWS, MinIO or any S3-compatible provider) or `azure`.
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

The API, the pipeline and any external document service must all use the same
backend and mode.

**⚠︎ The mode is fixed at install.** Object keys are persisted in
`files.original_blob_name` and `files.pdf_blob_name`. Switching modes on a
database that already holds files orphans every one of them — the rows keep the
old keys and the code looks for the new ones. Migrating means rewriting both the
objects and those columns.

**⚠︎ The S3 region must match the endpoint**, not the marketing name of the
region. SigV4 signs the region string, so a provider whose endpoint is
`region-1.example.com` generally wants `region-1`, and a plausible-looking `eu-central`
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

**`RBAC_ENFORCED=false` refuses to start on its own.** The permission matrix can
be switched to observe-only for a rollout, but only together with a second,
deliberately long acknowledgement variable. A single mistyped line cannot open
every route to every role.

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
- Document extraction (Docling, LibreOffice, OCR models) will not fit. Run it
  elsewhere — on a bigger box against the same database and storage, or as a
  managed job.

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

# Ad-hoc SQL works (app_sql_ro granted) and is read-only
wholegrain sql "select count(*) from entities"

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
keep the frontends on `wholegrain-edge`, and set a real OAuth signing key.

See [SECURITY.md](SECURITY.md) for what the shipped configuration does and does
not defend against.
