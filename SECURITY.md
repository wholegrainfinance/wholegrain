# Security

Wholegrain holds a firm's entire document archive and its ledgers. This describes
what the shipped configuration does about that, and — as importantly — what it
does not, so you can judge the gap yourself rather than assume it is covered.

Much of what follows was learned from a compromise of a sibling deployment: a
container was used to fetch and run a cryptominer. Each control below is stated
with the step it would have broken, because a control nobody can explain is a
control nobody maintains.

---

## Containers

**Non-root.** The API runs as **uid 10001**, not root. Root is what turns code
execution into persistence: as root a foothold can write `/bin`, install a cron
entry and survive a restart. As an unprivileged user the image's own filesystem
is unwritable and none of that works.

**Read-only root filesystems.** The API, nginx and the frontend mount their root
read-only, with the minimum tmpfs each genuinely needs — `/tmp` for the API;
`/var/cache/nginx`, `/run` and the rendered-config directory for the nginx-based
ones. The compromise wrote a binary into `/bin` and a miner into `/tmp`. Neither
lands on a read-only root, and anything that does reach tmpfs is gone at the next
restart. The database container cannot be read-only (it writes its data
directory); it runs with `no-new-privileges` instead.

**Two networks, one of them sealed.**

```
internet ──443──► nginx ──┬── wholegrain-edge   internal: true — NO egress
                          │      └─ browser frontend (static)
                          │
                          └── wholegrain-network  routable
                                 └─ API (Postgres, object storage, LLM)
```

A static frontend needs no outbound access at all: the browser calls the API
directly, the container only ever serves files. On an `internal: true` network the
payload fetch fails outright — no DNS to the internet, no route to a mining pool.
nginx is the only member of both, and it creates the edge network the frontends
then join.

`internal: true` still permits Docker's embedded DNS and traffic between members,
so nginx reaches the frontends normally. It does **not** permit published host
ports — Docker configures no gateway on an internal network, so a `ports:`
mapping on a member silently does nothing. That is the intended shape: those
containers are reachable only through nginx.

**Resource limits.** Every service has `mem_limit` and `cpus`. A miner that cannot
take the whole box is both less damaging and more visible — in the incident it was
the box falling over, not the mining, that eventually raised the alarm.

**Nothing heavy runs inside a request.** Document conversion and extraction are
background work, not part of upload. LibreOffice, Docling and their models live
in the separate pipeline image (or in an external document service), not in the
API image: the smaller the image that faces the internet, the less there is to
exploit.

---

## Database

**Tenant isolation is enforced by Postgres, not by application code.** Every
workspace-scoped table carries row-level security; the policies call
`app.tenant_visible(...)` for reads and `app.tenant_writable(...)` for writes,
both `SECURITY DEFINER`. A missing `WHERE workspace_id = …` in a query is
therefore not a data breach — the database refuses regardless of what the
caller asked for. The API connects as `app_api`, a role subject to those
policies; with no principal set it sees nothing (fail-closed).

**Read-only memberships are enforced there too.** A workspace membership can be
`owner`, `member` or `read_only`. A `read_only` member is refused every write by
the write policies, not by the routes — so the guarantee covers every endpoint,
the MCP server and any module added later, and a new endpoint cannot default to
writable.

This is the single most important property in the system, and the reason
[DATABASE.md](DATABASE.md) warns that a schema rebuilt from ORM models alone is
dangerous: models describe tables, not policies, and a database with the right
tables and no policies looks correct while letting every tenant read every other.
It is equally defeated by running the API as the table owner, which bypasses RLS
without any visible symptom — see
[DEPLOYMENT.md](DEPLOYMENT.md#tenant-isolation-read-this-one).

**Ad-hoc SQL is three walls, not one.** `POST /query/sql` (and the CLI and MCP
tools built on it) runs the caller's statement in a read-only transaction, as the
`app_sql_ro` role that holds SELECT on the workspace-scoped tables only, after an
`EXPLAIN`-based check that refuses any plan touching the credential, membership-
graph or mirror-token tables. Read-only is enforced by the executor, not by
parsing; the credential tables are unreachable by absence of grant; and the plan
check is an independent second reason to refuse.

**Changes are attributed.** On entities, events, transactions, measurements,
tasks, groups, datarooms and files, an audit trigger records the old and new row,
the acting user, the channel (`api`, `mcp`, `sql`) and — for every write by an
agent or the CLI — a required reason. The old row is kept, so a change is
reversible from the audit trail; the MCP tools additionally cannot delete at all.

**Reach the database privately.** No public firewall rule, and in particular not
"allow all cloud services" — on most providers that admits *every* customer of
that cloud, not just you. Prefer a private endpoint.

**Keep compute and database in the same region.** Not only a security control, but
it removes an internet path that would otherwise carry every query.

---

## Application

**Three gates per request.** Authentication; then an **RBAC matrix**
(`role_permissions`: create/read/update/delete per role and resource) decides
*what* a role may do, default-deny, with any authenticated route that is not
mapped to a resource refused; then RLS decides *which rows*. Owner-only actions
(membership, workspace settings) add an ownership check. Humans and agents are
the same kind of principal: an agent is a user with a role, and restriction is a
role you assign, not an identity the code special-cases. Only the `admin` role can
change roles or the matrix. Switching enforcement off requires a second, explicit
acknowledgement variable, or the API refuses to start.

**Authentication is the API's own**, not a federated identity provider — local
users with hashed credentials, short-lived HS256 access tokens carrying only role
*names* (so a permission change applies on the next request), and rotating,
revocable refresh tokens. Emailed one-time codes and a device-code flow cover
passwordless and CLI sign-in.

**The MCP server is a separate OAuth 2.1 resource**, one per workspace
(`/mcp/<workspace-id>`), with RS256 tokens bound to that workspace. Every
connection is pinned to its one workspace at the database level, even for a user
who belongs to several. Its write surface is narrow: typed tools for tasks and
processes, each with a required reason and the same RBAC check the REST API
applies; and a generic `call_api` that goes through the API as the user — so its
ceiling is that user's own permissions — and refuses `DELETE` and the
`/external`, `/auth`, `/oauth`, `/workspace` and `/settings` surfaces outright.

**Bank lines are admin-write.** Statement lines live in their own resource,
`cashflows`, which only an admin may write. An agent carries a member's rights and
can therefore read them, but cannot invent one.

**Entity contacts cannot hold secrets.** The `contacts` object refuses unknown
keys at every level; a portal entry has a `credential_ref` (where a secret is
kept), never a password field.

**The frontend is static.** React built to files, served by nginx. There is no
Node runtime in the shipped image: no `node`, no `npm`, nothing to execute. This
is the difference that mattered most in the incident — the compromised container
ran a live JavaScript server, so it had an interpreter, a package manager and
outbound access. A static container has none of the three.

**Secrets live in the environment**, never in the image or the repository. Each
module's `.dockerignore` excludes `.env`.

---

## What is deliberately not covered

Stated plainly, because a security document that lists only strengths is worse
than none.

- **Registration is open.** Anyone who can reach the API can create an account.
  It lands in no workspace and sees nothing, but if that is not acceptable, put
  the API behind an allowlist.
- **Not every table is under RLS.** Identity and credential tables, the role
  tables, operational logs and some feature-module tables carry no policy and
  rely on the application's checks alone. The ad-hoc SQL walls above exist
  because of this; a table that should be tenant-scoped belongs under a policy.
- **The services that need egress have no allowlist.** The API can reach any host
  on the internet. Restricting it properly needs a forward proxy or cloud service
  tags — DNS names cannot be matched by firewall rules, and managed-service IPs
  rotate. This is the most valuable remaining control.
- **No image scanning, no rebuild cadence.** Nothing here runs Trivy or Grype and
  nothing rebuilds on a schedule. The image in the incident was four months old.
- **Build-time supply chain.** Runtime hardening does not help if a dependency is
  malicious at build time — by the time the container serves those bytes they are
  already yours. Commit lockfiles, install from them (`npm ci`, pinned Python
  requirements), and build on a known runner.
- **No intrusion detection.** Nothing watches for a process that should not exist.
- **Backups are the operator's job.** The `database` module ships backup and
  restore scripts; nothing schedules them, and a managed server's point-in-time
  restore is only as good as its retention window.

---

## Reporting a vulnerability

Open a private security advisory on the repository rather than a public issue.
