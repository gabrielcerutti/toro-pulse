# Architecture Overview

**Status:** Draft v1
**Date:** 2026-04-30

This document is the single source of truth for how the system is composed at a high level. It is the companion to ADR-0001 (Tech Stack) and is referenced by Spec-Kit `/plan` outputs.

---

## 1. System Topology

The application is built around the **eToro public API**. Both the .NET API and the MCP server are independent clients of that public API. All application services and stateful infrastructure run on **Railway**, fronted by **Cloudflare**.

```
                          ┌────────────────────────┐
                          │        Browser         │
                          └──────────┬─────────────┘
                                     │ HTTPS
                                     ▼
                          ┌────────────────────────┐
                          │       Cloudflare       │
                          │  - CDN / edge cache    │
                          │  - Image optimization  │
                          │  - DDoS / WAF          │
                          └──────────┬─────────────┘
                                     │ HTTPS
  ╔══════════════════════════════════▼══════════════════════════════════╗
  ║                              Railway                                ║
  ║                                                                     ║
  ║   ┌────────────────────────┐                                        ║
  ║   │  apps/web (Next.js 15) │                                        ║
  ║   │  - UI / RSC            │                                        ║
  ║   │  - Better Auth         │                                        ║
  ║   │  - JWKS endpoint       │                                        ║
  ║   │  - thin BFF route      │                                        ║
  ║   │  - Faro Web SDK        │                                        ║
  ║   └──────────┬─────────────┘                                        ║
  ║              │ private net (JWT)                                    ║
  ║              ▼                                                      ║
  ║   ┌────────────────────────┐                                        ║
  ║   │  apps/api (.NET 9)     │                                        ║
  ║   │  - Domain logic        │─── HTTPS ──┐                           ║
  ║   │  - eToro public API    │            │                           ║
  ║   │    client              │            ▼                           ║
  ║   │  - EF Core             │   ┌──────────────────┐                 ║
  ║   │  - Quartz jobs         │   │  eToro PUBLIC API│ ◀────────────┐  ║
  ║   └────┬──────────┬────────┘   └──────────────────┘              │  ║
  ║        │          │                                              │  ║
  ║   ┌────▼─────┐ ┌──▼───────────┐                                  │  ║
  ║   │ Railway  │ │ Railway      │                                  │  ║
  ║   │ Postgres │ │ Redis        │                                  │  ║
  ║   │ auth/app │ │ cache + RL   │                                  │  ║
  ║   └──────────┘ └──────────────┘                                  │  ║
  ║                                                                  │  ║
  ║   ┌────────────────────────┐                                     │  ║
  ║   │  apps/mcp (sibling)    │ ─── HTTPS ──────────────────────────┘  ║
  ║   │  optional, no v1 use   │                                        ║
  ║   │  case                  │                                        ║
  ║   └────────┬───────────────┘                                        ║
  ╚════════════│════════════════════════════════════════════════════════╝
               │ MCP (HTTPS / stdio)
               ▼
  ┌────────────────────────┐
  │  AI Agent / LLM client │
  └────────────────────────┘
```

Web does **not** call the MCP server. The MCP server is a parallel, optional surface that AI agents may use directly; it has **no committed use case in v1** (see ADR-0001 §"MCP Server"). Both `apps/api` and `apps/mcp` reach the eToro public API independently. Drift between the two eToro clients is acceptable for v1.

**Cloud-agnostic posture (Constitution Principle XIII):** every service runs from a `Dockerfile`. Postgres and Redis are vanilla — no Railway-specific extensions. All inter-service communication uses standard env vars (`DATABASE_URL`, `REDIS_URL`, etc.). Migration to Render is a one-day cleanup; Fly.io / Coolify+VPS / Kubernetes are one-week migrations. The Cloudflare ↔ Railway boundary is the only platform-specific seam, and it is documented for replacement in the Railway-exit playbook (ADR-0009, future).

## 2. Request Flow — Public Portfolio View (`/u/{username}`)

1. Browser requests `https://app.example.com/u/gabipins`.
2. Cloudflare resolves and either serves a cached HTML response (if the route allows it) or forwards to Railway.
3. Railway routes to the `apps/web` Next.js container; a Server Component for that route runs.
4. Server Component invokes a typed RPC to `apps/api` via Railway's private network: `GET /v1/portfolios/{username}` with `Authorization: Bearer <service-jwt>` for trace correlation (no user identity required for public endpoint).
5. `.NET API` checks Redis for `portfolio:{username}`. On hit, deserializes and returns. On miss:
   - Calls the eToro public API through the typed client (Polly: 3 retries with exponential backoff; circuit breaker after 5 consecutive failures).
   - Validates the response into the domain model.
   - Writes to Redis (TTL: 15min portfolio, 5min prices, 1h profile).
   - Persists a snapshot row in `app.portfolio_snapshots` for analytics (best-effort, async-fire).
6. Server Component renders the page; non-critical panels stream in via Suspense boundaries.
7. Browser hydrates; subsequent in-page interactions (sort, range switch) hit a Next.js Route Handler that proxies to `.NET API` with TanStack Query handling client cache.

**P95 budget end-to-end:** 2.5s cold, 800ms warm.

## 3. Request Flow — AI Agent via MCP (optional, no v1 use case)

This flow is documented for completeness. In v1, no shipped feature triggers it.

1. Agent (e.g., Claude Desktop) connects to `apps/mcp`.
2. Agent invokes a tool, e.g., `etoro_get_user_portfolio({ username: "gabipins" })`.
3. MCP server reaches the eToro public API directly. No call to `apps/api`. No call to Postgres or Redis.
4. MCP server returns a structured tool result.

The MCP server is stateless in v1. If a future use case calls for shared caching with `apps/api`, Redis is the seam — but introducing that coupling is an ADR-0006 decision, not an architecture default.

## 4. Authentication Topology

```
  ┌────────────┐    1. login          ┌──────────────────┐
  │  Browser   │ ───────────────────▶ │  Better Auth     │
  │            │ ◀─────────────────── │  (apps/web)      │
  │            │    2. session cookie └──────────────────┘
  │            │                              │
  │            │                              │  3. issues JWT (RS256)
  │            │                              │     signed with rotating key
  │            │                              ▼
  │            │                      ┌──────────────────┐
  │            │     4. browser holds │  JWKS endpoint   │
  │            │        cookie only;  │  (apps/web)      │
  │            │        no JWT in JS  └──────────────────┘
  │            │
  │            │    5. fetch via Next.js BFF route
  │            │ ──────────────────────────────────────▶ apps/web BFF
  │            │                                            │
  │            │                                            │  6. attach JWT from
  │            │                                            │     server-side session
  │            │                                            ▼
  │            │                                   ┌──────────────────┐
  │            │                                   │  .NET API        │
  │            │                                   │  - validates JWT │
  │            │                                   │    via JWKS      │
  │            │                                   └──────────────────┘
```

Key properties:
- The browser never holds a JWT. Only an HttpOnly, Secure, SameSite=Lax session cookie issued by Better Auth.
- The .NET API never reads from auth tables. It receives `sub` (user ID) and standard claims via signed JWT.
- JWKS rotation is automatic; the .NET API caches keys with a short TTL.
- Public endpoints (e.g., portfolio view) accept service JWTs for tracing but do not require a user.

## 5. Data Ownership

| Schema  | Owner            | Migrations    | Read by                    | Write by                |
|---------|------------------|---------------|----------------------------|-------------------------|
| `auth`  | Better Auth      | Better Auth CLI | apps/web only            | apps/web (Better Auth)  |
| `app`   | .NET API         | EF Core        | .NET API                   | .NET API                |

Cross-schema joins are forbidden. The user-ID claim from JWT is the only bridge.

## 6. Caching Strategy

| Resource                      | Layer        | TTL     | Invalidation                                |
|-------------------------------|--------------|---------|---------------------------------------------|
| Portfolio composition         | Redis        | 15 min  | TTL only in v1                              |
| Instrument prices             | Redis        | 5 min   | TTL only                                    |
| Profile metadata              | Redis        | 60 min  | TTL only                                    |
| Search results (autocomplete) | Redis        | 5 min   | TTL only                                    |
| Page output (Next.js RSC)     | Next.js Data Cache + Cloudflare edge | 60 sec  | `revalidateTag('portfolio:{username}')` on background refresh; Cloudflare Cache Rules align TTLs |

Stale-while-revalidate is enabled at the Next.js layer for `/u/{username}`.

## 7. Background Work

Hosted in `apps/api` via Quartz.NET in v1:
- Pre-warm cache for the top 100 most-viewed usernames every 10 minutes.
- Persist daily portfolio snapshots for trending users.
- Reconcile cache hit metrics → OpenTelemetry.

When background load justifies it, peel out to `apps/worker` (separate Fly app, same .NET solution).

## 8. Observability Topology

All signals flow over **OpenTelemetry / OTLP** to **Grafana Cloud** (free tier, managed LGTM + Faro). OTel is the wire contract; Grafana Cloud is the destination — swappable with one config change.

```
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │ apps/web     │       │ apps/api     │       │ apps/mcp     │
  │ Faro Web SDK │       │ OTel .NET    │       │ OTel SDK     │
  │ + OTel Node  │       │ Serilog→OTel │       │              │
  └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
         │ OTLP                 │ OTLP                 │ OTLP
         │ (HTTP/protobuf)      │                      │
         ▼                      ▼                      ▼
              ┌────────────────────────────────────────┐
              │         Grafana Cloud (managed)        │
              │  ┌──────┐  ┌──────┐  ┌──────┐  ┌─────┐ │
              │  │ Loki │  │Tempo │  │Mimir │  │Faro │ │
              │  │(logs)│  │(trace│  │(metr)│  │(RUM)│ │
              │  └──────┘  └──────┘  └──────┘  └─────┘ │
              │                ↑                       │
              │             Grafana                    │
              │         (visualization)                │
              └────────────────────────────────────────┘
```

- **Traces:** OpenTelemetry SDK in Next.js (Node) + .NET → OTLP → **Tempo**.
- **Logs:** Pino (Next.js) + Serilog (.NET, via OpenTelemetry log signal exporter) → OTLP → **Loki**.
- **Metrics:** OpenTelemetry metrics SDK in both runtimes → OTLP → **Mimir**.
- **Frontend RUM + browser errors:** **Grafana Faro Web SDK** in `apps/web` (Web Vitals, console errors, unhandled exceptions, browser traces). **No Sentry** in v1.
- **Synthetic:** Better Stack against `/health` (apps/api) and `/health` (apps/mcp). Grafana Synthetic Monitoring is a viable later swap.
- **Dashboards & alerts:** Grafana managed dashboards; alerts route via Grafana OnCall (or Better Stack incidents) to email/Slack.

Correlation IDs:
- Inbound request gets a `x-request-id` (or one is generated).
- Header propagated to `apps/api`. Spans inherit the W3C trace context. Logs include `trace_id` and `span_id` so Loki ↔ Tempo deep-link works in Grafana.

Vendor-portability posture: every signal's wire format is OTLP. Migrating off Grafana Cloud (to self-hosted LGTM, Honeycomb, Datadog, or Axiom) is a change to the OTLP endpoint and credentials — no code changes.

## 9. Deployment Topology

| Service       | Platform       | Region (v1)        | Scaling                                       |
|---------------|----------------|--------------------|-----------------------------------------------|
| Edge / CDN    | Cloudflare     | Global             | Auto                                          |
| apps/web      | Railway        | us-east (default)  | Vertical, 1 replica v1; preview env per PR    |
| apps/api      | Railway        | us-east            | Vertical, 1 replica v1; preview env per PR    |
| apps/mcp      | Railway        | us-east            | Vertical, 1 replica; production only in v1    |
| Postgres      | Railway        | us-east            | Managed, vanilla Postgres 16                  |
| Redis         | Railway        | us-east            | Managed, vanilla Redis 7                      |
| Observability | Grafana Cloud  | (managed)          | Free tier; OTLP endpoint as the only ingress  |
| Synthetic     | Better Stack   | Global             | Per-monitor                                   |
| Container reg | GHCR           | Global             | Pulled by Railway; not bound to Railway       |

Multi-region is deferred. Cloudflare's global edge mitigates most latency for cached routes. When latency on dynamic routes outside us-east becomes a complaint, the migration trigger to Fly.io (which makes multi-region trivial) or a multi-region Railway setup will be evaluated in ADR-0009.

## 10. Failure Modes and Posture

- **eToro upstream down:** circuit breaker opens; serve from Redis if available; otherwise return a typed error and a "degraded mode" UI banner.
- **Redis down:** Bypass cache, log a warning, hit eToro directly. Web tier shows no error.
- **Postgres down:** Auth flows fail (apps/web). Public read paths still serve from Redis. Surface a clear status banner.
- **MCP server down:** No impact on web. Agents see standard MCP errors.
- **Cloudflare down:** Static cached responses still serve via Cloudflare's resilient cache layers; new dynamic requests fail. DNS fallback to direct Railway hostnames documented in ADR-0009.
- **Railway down (single-platform blast radius):** web, api, Postgres, and Redis all unavailable. This is the explicit cost of consolidation. Mitigation: Better Stack synthetic monitors + status page hosted off-platform; emergency Railway-exit playbook (ADR-0009) keeps a Render/Fly migration achievable in days, not weeks.
- **Grafana Cloud down:** No application impact; observability blind during outage. Logs/traces/metrics are buffered in OTel SDK queues and flushed on recovery.

## 11. Local Development Topology

Constitution Principle XIII requires a `docker compose up` path that mirrors production. The local topology supports three modes — canonical, MCP-included, and infra-only — chosen by Compose flags rather than separate orchestration tools.

### 11.1 Topology

```
  Developer browser  ──▶  http://localhost:3001  (apps/web container)
                          http://localhost:5001  (apps/api container, optional direct)
                          http://localhost:3000  (Grafana UI — local LGTM)

  ╔════════════════ Docker Compose (local) ═══════════════════════╗
  ║                                                               ║
  ║  ┌───────────────┐                                            ║
  ║  │  apps/web     │  Next.js (next dev) inside container       ║
  ║  │  :3001        │  - Better Auth                             ║
  ║  │  bind-mount   │  - JWKS at http://web:3001/api/auth/jwks   ║
  ║  │  on src/      │  - Faro disabled or pointed at local LGTM  ║
  ║  └──────┬────────┘                                            ║
  ║         │ http://api:5001                                     ║
  ║         ▼                                                     ║
  ║  ┌───────────────┐                                            ║
  ║  │  apps/api     │  .NET (dotnet watch) inside container      ║
  ║  │  :5001        │  - JwtBearer fetches JWKS from web:3001    ║
  ║  │  bind-mount   │  - eToro client points at PUBLIC eToro     ║
  ║  │  on src/      │    API (no mock by default)                ║
  ║  └───┬──────┬────┘                                            ║
  ║      │      │                                                 ║
  ║   ┌──▼─┐  ┌─▼────┐    ┌──────────────┐    ┌──────────────┐   ║
  ║   │ pg │  │redis │    │ otel-lgtm    │    │ apps/mcp     │   ║
  ║   │5432│  │6379  │    │ Grafana :3000│    │ (profile=mcp)│   ║
  ║   └────┘  └──────┘    │ OTLP :4317   │    │ :4000        │   ║
  ║                       └──────────────┘    └──────────────┘   ║
  ╚═══════════════════════════════════════════════════════════════╝
```

All inter-service URLs use Docker's internal DNS (`web:3001`, `api:5001`, `postgres:5432`, `redis:6379`, `otel-lgtm:4317`). Host-port mappings are only for the developer browser and external tools.

### 11.2 The three modes

| Mode               | Command                                              | What runs                                                         | When to use                                                                        |
|--------------------|------------------------------------------------------|-------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| **Canonical (default)** | `docker compose up`                              | web, api, postgres, redis, otel-lgtm                              | Day-to-day work. Maximum prod parity. Hot-reload via bind mounts.                  |
| **MCP-included**   | `docker compose --profile mcp up`                    | Canonical + apps/mcp                                              | Working on the MCP surface, or testing eToro client drift between api and mcp.     |
| **Infra-only**     | `docker compose up postgres redis otel-lgtm`         | Only stateful + observability services                            | Iterating quickly with `pnpm dev` (web) and `dotnet watch` (api) on the host.       |

The default profile is empty; `apps/mcp` carries `profiles: [mcp]` in `compose.yml` so it only starts when the profile is requested. There is no separate `compose.infra.yml` file — the infra-only mode is a service-list invocation against the same Compose file, which keeps the source of truth single.

### 11.3 Hot reload

Two bind-mount strategies, codified in `compose.yml`:

- **`apps/web`:** mount the repo into the container, run `pnpm dev` as the entrypoint. Next.js handles HMR. The container's `node_modules` are kept inside the container (named volume) so Linux-built modules don't conflict with the developer's host (typically Windows or macOS).
- **`apps/api`:** mount the repo into the container, run `dotnet watch run` as the entrypoint. .NET hot-reloads on source changes. NuGet caches are kept inside the container.

In infra-only mode, neither bind mount runs — the developer uses `pnpm dev` and `dotnet watch` natively on the host. The host-mode contract is: every service the developer is *not* running natively must be reachable on `localhost:<port>` with the same env-var keys as production.

### 11.4 Auth wiring

Better Auth in `apps/web` exposes a JWKS endpoint at `/api/auth/jwks`. In Compose mode this resolves as `http://web:3001/api/auth/jwks` from the api container. In infra-only mode, the api process running on the host hits `http://localhost:3001/api/auth/jwks`. The .NET API's `JwtBearerOptions.Authority` and `MetadataAddress` are configured exclusively from environment variables — `JWKS_URL` is the contract — so the same code runs in both modes without conditionals.

### 11.5 Local observability

The local OpenTelemetry destination is a **`grafana/otel-lgtm`** container (the official all-in-one distribution: Grafana, Loki, Tempo, Mimir). It exposes:
- Grafana UI at `http://localhost:3000` (default credentials documented in compose).
- OTLP/gRPC endpoint at `otel-lgtm:4317` (internal) / `localhost:4317` (host).
- OTLP/HTTP endpoint at `otel-lgtm:4318` / `localhost:4318`.

`OTLP_ENDPOINT` env var points at `http://otel-lgtm:4318` in compose mode and `http://localhost:4318` in infra-only mode. Production points at the Grafana Cloud OTLP endpoint. Same code, same SDK config, different value — exactly the OTel-as-portability-contract this architecture promises.

Faro Web SDK in `apps/web` uses a feature flag (`FARO_ENABLED`) that defaults to `false` locally to keep the dev console quiet.

### 11.6 Env-var contract

A single `.env.example` at the repo root enumerates every variable the stack needs. Developers copy it to `.env` and override per their machine. Keys are identical across local, CI, and Railway:

| Key                     | Local default                                  | Notes                                              |
|-------------------------|------------------------------------------------|----------------------------------------------------|
| `DATABASE_URL`          | `postgres://app:app@postgres:5432/app`         | Vanilla Postgres URL. Same shape in prod.          |
| `REDIS_URL`             | `redis://redis:6379`                           | Vanilla Redis URL. Same shape in prod.             |
| `JWKS_URL`              | `http://web:3001/api/auth/jwks`                | Better Auth issues; .NET API consumes.             |
| `OTLP_ENDPOINT`         | `http://otel-lgtm:4318`                        | Grafana Cloud endpoint in prod.                    |
| `ETORO_API_BASE`        | `https://www.etoro.com/api/...` (real)         | Local hits real eToro; rate-limit budget is shared with prod when running heavy local traffic. |
| `BETTER_AUTH_SECRET`    | dev-only random in `.env`                      | Per-developer; never checked in.                   |
| `FARO_ENABLED`          | `false`                                        | `true` in prod.                                    |

### 11.7 What's intentionally not in local

- **Cloudflare.** No local CDN. The browser hits `apps/web` directly on `localhost:3001`. Cache headers are exercised in prod and preview only.
- **Railway-specific anything.** No Railway CLI, no Railway env. The whole point.
- **Grafana Cloud.** Local LGTM only. Sending dev traces to the prod stack is forbidden by Principle XIII.
- **Synthetic monitoring.** Better Stack is prod-only.
- **Multi-region.** Not relevant locally.

### 11.8 Limitations vs prod

- ISR / Next.js Data Cache behavior in `next dev` differs from `next start`. For end-to-end cache testing, run `apps/web` with `pnpm build && pnpm start` inside the container — a `compose.prod-mode.yml` overlay (future) will make this a one-line switch.
- The eToro public API is shared between local and prod. Heavy local load can deplete the same upstream rate-limit budget the prod app uses. Rate-limit middleware in the .NET API includes a local-mode warning when a developer crosses a soft threshold.
- The local LGTM container does not persist data across `docker compose down -v`. This is intentional — we want clean dashboards each session.

## 12. Open Architectural Questions (for ADR-0002+)

1. JWT lifetime and refresh strategy.
2. eToro client retry budget tuning under load.
3. Whether to expose an internal admin API for cache busting.
4. Snapshotting cadence and retention for `app.portfolio_snapshots`.
5. Whether the MCP server should eventually call `apps/api` instead of eToro directly (would unify caching but reintroduces the coupling we explicitly avoided in v1).
6. Whether to add a `compose.prod-mode.yml` overlay that runs `next start` and a Release-build .NET binary for cache and perf testing, and how its output should diverge from canonical local mode.
