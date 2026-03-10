# SupaHost Platform — single-slot deployments + shared tenant UI

This repository is the revised, runnable starter for a multi-tenant **Supabase hosting** platform with:

- **actual upstream Supabase self-hosting assets** vendored under `vendor/supabase-docker/`
- **Next.js + TypeScript + shadcn-style UI** for customer signup/login, shared tenant console, and admin console
- **Stripe checkout service** (optional)
- **Redis + NATS** control-plane state/events
- **Rust platform API** and **Rust provisioner**
- **Traefik** hostname routing for shared UI + tenant API gateways

## What changed in this bundle

This version focuses on the three operational gaps you called out:

- **Only one active local deployment per tenant** — every tenant now carries `deployment_mode=single` and an `active_deployment_count` constrained to `0/1`.
- **Pause / resume / delete in every lifecycle phase** — tenants and admins can queue lifecycle actions while a deployment is provisioning, paused, active, suspended, or reconciling.
- **One shared UI deployment for everyone (host port 3000 is reserved for the shared web app only)** — the dedicated per-tenant Supabase Studio container is removed in local mode. A single shared Next.js console handles all human-facing UI, while API prefixes on each tenant hostname still route to the tenant’s Kong gateway.

The default runnable path uses the **native Rust platform API service** instead of depending on `wash build` for the control plane. The wasmCloud component folder remains as a reference.

## Quickstart

### Prereqs

- Docker Desktop
- Rust toolchain (`cargo`)
- `jq` (optional)

### 1) Start local infrastructure

```bash
./scripts/dev-up.sh
```

That starts:

- NATS on `localhost:4222`
- Redis on `localhost:6379`
- Traefik on `localhost:80`
- platform Postgres on `localhost:5433`
- Next.js web UI on `http://localhost:3000`
- a one-shot `web-migrate` job that runs Prisma migrations + seeds the admin user

### 2) Start the Rust services

```bash
source .env
cargo run -p platform-api
```

In another terminal:

```bash
source .env
cargo run -p provisioner
```

Optional Stripe billing service:

```bash
source .env
cargo run -p billing-service
```

### 3) Open the shared UI

- customer UI: `http://localhost:3000`
- admin login is seeded from `.env`
  - default email: `admin@local`
  - default password: `adminadmin`

### 4) Create and manage a tenant

Use the UI or:

```bash
./scripts/example-create-tenant.sh dev@example.com starter
```

Once a tenant exists, you can:

- open the tenant host: `http://<tenant-id>.supabase.localhost`
- pause, resume, or delete the deployment from the shared UI
- use API paths like `http://<tenant-id>.supabase.localhost/rest/v1/`

## Shared UI routing model

Traefik now routes tenant subdomains in two layers:

- **root / non-API paths** → shared Next.js UI container
- **API prefixes** (`/auth`, `/rest`, `/graphql`, `/realtime`, `/storage`, `/functions`, `/pg`) → the tenant’s Kong gateway

That gives you one shared UI deployment while keeping each tenant’s actual Supabase data plane isolated.

## Notes

- The provisioner still spins up a **real per-tenant Supabase stack** from `vendor/supabase-docker/`.
- The local optimized tenant stack omits the dedicated Studio container because the shared UI replaces it.
- This is a strong starter, not a hardened production platform. You still need backups/PITR, quotas, network policy, secret backends, and stronger isolation for production.

## Troubleshooting

- If `./scripts/example-create-tenant.sh` says the Platform API is unreachable, start it with `source .env && cargo run -p platform-api`.
- If tenant root pages do not load, make sure `web` is running and Traefik is up; the shared UI serves tenant hostnames now.
- If tenant API routes do not work, confirm the provisioner started and that the tenant is `active` or `paused`/`resumed` as expected.

## Shared tenant UI routing

- Tenant subdomains no longer redirect to `localhost:3000`; they now open the tenant's Supabase Studio directly at `http://<tenant>.supabase.localhost/`.
- Host port `3000` remains exclusive to the shared web UI. No tenant Studio container is published on host port `3000` in local shared-UI mode.


## Important local networking note

The shared web UI runs in Docker and reaches the platform API through `host.docker.internal:8000`.
That means the host-run platform API must bind to `0.0.0.0:8000`, not `127.0.0.1:8000`.
The included `.env.example` already sets `PLATFORM_API_BIND=0.0.0.0:8000`.
If tenant creation fails from the UI, verify both of these: `curl http://localhost:8000/healthz` from the host and `docker exec $(docker ps -qf name=web) wget -qO- http://host.docker.internal:8000/healthz` from the container.


## Upgrade notes for shared-UI local mode
If you are upgrading from an earlier shared-UI bundle:

```bash
docker compose -f infra/local/platform/docker-compose.platform.yaml -f infra/local/web/docker-compose.web.yaml build --no-cache web-migrate web
```

Then recreate any tenant that was provisioned before this version so the corrected Traefik labels and Studio URL are applied.


## Repair an older tenant that still returns 404 on `*.supabase.localhost`

If a tenant was created from an older bundle, its generated `tenants/<id>/supabase/docker-compose.yml`
may still be missing the Studio route and Traefik labels. Reconcile it after upgrading:

```bash
source .env
./scripts/repair-tenant-studio.sh <tenant-id>
```

Then wait a few seconds and open:

```text
http://<tenant-id>.supabase.localhost/
```

If you still get a 404, check whether Traefik sees the tenant router on:

```text
http://localhost:8088/dashboard/#/http/routers
```
# supahost
