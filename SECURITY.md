# Security

Wholegrain holds a firm's entire document archive and its ledgers. This describes
what the shipped configuration does about that, and — as importantly — what it
does not, so you can judge the gap yourself rather than assume it is covered.

Most of what follows was learned from a compromise of a sibling deployment: a
container was used to fetch and run a cryptominer. Each control below is stated
with the step it would have broken, because a control nobody can explain is a
control nobody maintains.

---

## Containers

**Non-root.** The API runs as **uid 10001**, not root. Root is what turns code
execution into persistence: as root a foothold can write `/bin`, install a cron
entry and survive a restart. As an unprivileged user the image's own filesystem
is unwritable and none of that works.

**Read-only root filesystems.** Every container mounts its root read-only, with
the minimum tmpfs each genuinely needs — `/tmp` for the API, `/var/cache/nginx`
and `/run` for the nginx-based ones. The compromise wrote a binary into `/bin`
and a miner into `/tmp`. Neither lands on a read-only root, and anything that
does reach tmpfs is gone at the next restart.

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
nginx is the only member of both, so it must start first; it creates the edge
network the frontends then join.

`internal: true` still permits Docker's embedded DNS, traffic between members and
published host ports, so nothing legitimate breaks.

**Resource limits.** Every service has `mem_limit` and `cpus`. A miner that cannot
take the whole box is both less damaging and more visible — in the incident it was
the box falling over, not the mining, that eventually raised the alarm.

**Nothing expensive runs inside a request.** Document conversion and extraction
are background work, not part of upload. That keeps LibreOffice and OCR stacks out
of the API image entirely: the smaller the image, the less there is to exploit.

---

## Database

**Tenant isolation is enforced by Postgres, not by application code.** Fifteen
tables carry row-level security; policies call `app.tenant_visible(...)`, which is
`SECURITY DEFINER`. A missing `WHERE workspace_id = …` in a query is therefore not
a data breach — the database refuses regardless of what the caller asked for.

This is the single most important property in the system, and the reason
[DATABASE.md](DATABASE.md) warns that a schema rebuilt from ORM models alone is
dangerous: models describe tables, not policies, and a database with the right
tables and no policies looks correct while letting every tenant read every other.

**Reach the database privately.** No public firewall rule, and in particular not
"allow all cloud services" — on most providers that admits *every* customer of
that cloud, not just you. Prefer a private endpoint.

**Keep compute and database in the same region.** Not only a security control, but
it removes an internet path that would otherwise carry every query.

---

## Application

**Authentication is the API's own**, not a federated identity provider — local
users, hashed credentials, its own tokens.

**The frontend is static.** React built to files, served by nginx. There is no
Node runtime in the shipped image: no `node`, no `npm`, nothing to execute. This
is the difference that mattered most in the incident — the compromised container
ran a live JavaScript server, so it had an interpreter, a package manager and
outbound access. A static container has none of the three.

**Secrets live in the environment**, never in the image or the repository.

---

## What is deliberately not covered

Stated plainly, because a security document that lists only strengths is worse
than none.

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
- **Backups are the operator's job.** Nothing here backs up your database.

---

## Reporting a vulnerability

Open a private security advisory on the repository rather than a public issue.
