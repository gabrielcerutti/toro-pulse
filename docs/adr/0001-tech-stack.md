# ADR 0001 — Tech Stack

**Status:** Accepted
**Date:** 2026-04-30
**Decider:** Gabriel Cerutti
**Supersedes:** —

## Context

We are starting a new application built around the **eToro public API**. The owner also maintains an existing MCP server that wraps the same public API; this MCP is treated as an **optional sibling surface** in v1 with no committed use case (a future ADR will document it when one lands). The owner has a Microsoft-heavy background (.NET, Azure, Docker, Kubernetes) and React/TypeScript depth, and is open to adopting current ecosystem leaders. We chose to apply Spec-Kit's spec-driven workflow, which means this ADR locks the stack decisions referenced in the `/plan` phase.

Goals shaping the decision:
- Single-developer velocity in v1 with a credible path to scale.
- Production-grade type safety end-to-end.
- A presentation tier that is delightful to build and fast for users.
- A backend tier that gives us strong tooling for HTTP clients, OpenTelemetry, EF migrations, and background work.
- Observability and CI/CD that are first-class on day one.
- Reversible choices wherever possible.

## Decision

### Frontend (`apps/web`)
- **Framework:** Next.js 15 (App Router) on React 19.
- **Language:** TypeScript with `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` enabled.
- **Styling:** TailwindCSS 4 + shadcn/ui.
- **Client data:** TanStack Query for client-side fetching; Server Components for data-heavy initial render.
- **Charts:** Recharts (pragmatic), with the option to swap in Tremor for dashboard-style composites later.
- **Forms / validation:** React Hook Form + Zod.
- **Auth (client surface):** Better Auth (Next.js plugin), email/password + Google + GitHub providers, session cookie + signed JWT.
- **Tooling:** Biome (lint + format), Vitest (unit), Playwright (E2E), Storybook (deferred until component count justifies it).

### Backend API (`apps/api`)
- **Runtime:** .NET 9 Minimal API (ASP.NET Core).
- **Language:** C# 13, nullable reference types enabled, warnings-as-errors in CI.
- **HTTP clients:** `IHttpClientFactory` + Polly (retry, circuit breaker, bulkhead) for the eToro client.
- **ORM / data:** Entity Framework Core 9 + Npgsql provider.
- **Validation:** FluentValidation.
- **Mapping:** Riok.Mapperly (source-generated, no runtime reflection).
- **Auth (server):** Microsoft.AspNetCore.Authentication.JwtBearer validating tokens minted by Better Auth via a JWKS endpoint exposed by `apps/web`.
- **Background work:** Quartz.NET hosted in the same process for v1; peel out to a worker service when justified.
- **Logging:** Serilog → OpenTelemetry exporters (OTLP).
- **Tests:** xUnit + Testcontainers for Postgres-backed integration tests.

### MCP Server (`apps/mcp`)
- **Status:** Existing implementation, kept as-is. **Optional sibling surface.**
- **Role:** A parallel wrapper over the same eToro public API that `apps/api` consumes. Web app does **not** call it. Deployed independently. Shares no runtime code with `apps/api`.
- **v1 use case:** Not committed. The MCP is included in the repo and architecture diagrams as a placeholder so it stays visible, but no v1 feature depends on it. A future ADR (working title: "MCP role and use case") will define a concrete role — internal ops/dev tool, end-user agent surface, or in-app AI feature — when warranted.
- **Future:** If/when MCP endpoints diverge meaningfully from the .NET API, that's expected and acceptable. Both clients of the eToro public API will drift; we accept this cost for v1.

### Persistence
- **Primary database:** PostgreSQL 16 on **Railway Postgres** (managed).
  - Vanilla Postgres only — no platform-specific extensions baked into schema.
  - DB-per-PR branching is acknowledged as a nice-to-have, not a v1 requirement (see Constitution Principle XIII consequences). Local development uses `docker compose up` with a Postgres container.
  - Migration path to Neon, Supabase, Fly Postgres, or self-hosted Postgres is preserved by the vanilla-Postgres rule.
- **Ownership rules:**
  - Better Auth owns the `auth` schema (its own tables).
  - .NET API owns the `app` schema (business tables) via EF Core migrations.
  - The two services do not read each other's tables. The .NET API receives a verified user ID via JWT and treats it as an opaque key.
- **Cache:** **Railway Redis** (managed). StackExchange.Redis from .NET, `ioredis` from Next.js (used sparingly — most caching happens server-side in .NET). Vanilla Redis only — no Railway-specific features.

### Hosting and Deploy
- **Platform:** **Railway** for all application services and stateful infrastructure.
- **`apps/web`:** Railway service (Docker), Production + Preview environment per PR.
- **`apps/api`:** Railway service (Docker), Production + Preview environment per PR.
- **`apps/mcp`:** Railway service (Docker), Production environment only in v1 (sibling surface, no committed use case).
- **Database:** Railway Postgres (managed).
- **Cache:** Railway Redis (managed).
- **Container registry:** GitHub Container Registry (`ghcr.io`). Railway pulls from `ghcr.io` rather than relying on Railway's Nixpacks builder, per Constitution Principle XIII.
- **Build:** every service has a `Dockerfile` at the repo root of its app folder. Railway is configured to build from the Dockerfile, not via Nixpacks.
- **Front-of-app CDN:** **Cloudflare** in front of `apps/web` for edge caching, image optimization (or Next.js's built-in Sharp pipeline), and DDoS protection. Closes the perceptible gap vs Vercel without coupling to a Vercel-specific runtime.
- **Migration posture (Principle XIII):** Render is the closest functional swap (a few days of work). Fly.io is a one-week swap (more "infrastructure" feel, multi-region trivial). Coolify on Hetzner is a budget swap. Kubernetes anywhere is the scale-out swap.

### CI/CD and Quality
- **CI:** GitHub Actions.
- **Pipelines:** lint → type check → unit tests → build → integration tests (Testcontainers) → Playwright E2E (against preview deploy) → Lighthouse (against preview).
- **Deploy:** GitHub Actions builds and pushes container images to GHCR on merge to `main`. A subsequent workflow step calls `railway up` (or Railway's image-redeploy webhook) to roll services. Preview environments deploy automatically on PR open via Railway's GitHub integration, also pulling images from GHCR.
- **Pre-commit:** lefthook runs Biome and `dotnet format` on staged files.

### Observability
- **Stack:** **Grafana Cloud (free tier)** as the v1 destination — managed LGTM + Faro: **Loki** (logs), **Grafana** (visualization), **Tempo** (traces), **Mimir** (metrics), **Faro** (frontend RUM + browser error tracking).
- **Wire protocol:** **OpenTelemetry / OTLP** for everything. The Grafana Cloud OTLP endpoint is the only collector we configure in v1. If we ever outgrow the free tier or the managed offering, swapping to self-hosted LGTM, Honeycomb, Datadog, or Axiom is a config change rather than code work.
- **Frontend RUM + errors:** Grafana Faro Web SDK in `apps/web`. Captures Web Vitals, console errors, unhandled exceptions, and traces. **Sentry is not used** in v1 — Faro covers the same ground and consolidates observability under one vendor. If Faro's error-grouping or release-tracking UX disappoints in practice, ADR-0006 will revisit.
- **Backend traces / metrics / logs:** OpenTelemetry SDKs in Next.js (Node) and .NET → OTLP → Grafana Cloud. Serilog in .NET is wired to the OpenTelemetry log signal exporter.
- **Uptime:** Better Stack synthetic monitors on `/health` for `apps/api` and `apps/mcp` (Grafana Synthetic Monitoring is a viable later swap).
- **Migration posture:** OpenTelemetry is a hard requirement; Grafana Cloud is the current destination. The first is non-negotiable, the second is reversible.

### Repo
- **Layout:** Mixed-language monorepo, pnpm workspaces for the JS/TS apps, .NET solution for the C# app, shared at the file-system level.
  ```
  etoro-app/
  ├── apps/
  │   ├── web/                  # Next.js 15 + React 19 + TS
  │   ├── api/                  # .NET 9 minimal API
  │   └── mcp/                  # existing MCP server
  ├── packages/
  │   ├── ui/                   # shared shadcn primitives (web only)
  │   └── tsconfig/             # shared TS configs
  ├── infra/
  │   ├── docker/               # Dockerfiles for api & mcp
  │   ├── fly/                  # fly.toml per service
  │   └── github/               # composite actions
  ├── specs/                    # Spec-Kit specs (one folder per spec)
  ├── docs/
  │   ├── adr/                  # Architecture Decision Records
  │   └── architecture.md
  ├── .specify/                 # Spec-Kit templates and memory
  ├── .github/workflows/        # CI/CD pipelines
  └── README.md
  ```
- **Versioning:** Conventional Commits, Changesets for `apps/web` if/when we publish UI packages.

## Alternatives Considered

| Decision Area    | Option                                | Why not (for v1)                                                                  |
|------------------|---------------------------------------|------------------------------------------------------------------------------------|
| Backend runtime  | Node + Fastify/Hono                   | Loses .NET leverage; would re-implement eToro client in TS without strong gain.    |
| Backend runtime  | Go                                    | Smaller ecosystem fit for the owner; learning curve cost outweighs perf gain here. |
| Frontend framework | Remix                               | Excellent, but Next.js has the broader ecosystem and stronger Server Components story for our data-heavy pages. |
| ORM (.NET)       | Dapper                                | Faster, but loses migrations and change tracking. EF Core wins for v1 velocity.    |
| Auth             | Auth0 / Clerk                         | Faster to ship but introduces vendor lock-in for a sensitive surface. Better Auth keeps full control with comparable DX. |
| Auth             | ASP.NET Core Identity end-to-end      | Heavier on the web side; loses the modern Next.js auth ergonomics.                 |
| Hosting          | Vercel + Fly.io + Neon + Upstash      | Best-in-class per tier but four vendors, four bills, four dashboards. Original v1 plan; rejected on consolidation grounds. |
| Hosting          | Render                                 | Functionally equivalent to Railway, slightly more mature ops, fewer surprise pricing changes. Tied for best fit; Railway chosen by owner preference. Discipline keeps Render a one-day swap. |
| Hosting          | Fly.io for everything                  | Pure Docker, multi-region trivial, easy migration. More "infrastructure" feel than Railway. Strong fallback if Railway disappoints. |
| Hosting          | Coolify on a Hetzner/DO VPS           | Most truly cloud-agnostic, cheapest. Real ops cost (backups, monitoring, OS updates). Reserved as the "cost-down" exit path. |
| Hosting          | Azure Container Apps + Azure DB for PostgreSQL + Azure Cache for Redis | Strong MS alignment, enterprise compliance posture. Locks to Azure; rejected on cloud-agnostic grounds. |
| Hosting          | Self-hosted Kubernetes (DOKS/AKS/EKS) | Most agnostic + most scalable. Premature ops cost for v1 single-developer MVP. Reserved as scale-out exit path. |
| DB host          | Neon                                   | Excellent DB-per-PR branching, but separate vendor — conflicts with consolidation. Preserved as easy migration target via vanilla-Postgres rule. |
| DB host          | Supabase                              | Strong, but Supabase Auth would conflict with Better Auth.                       |
| Monorepo tool    | Nx                                    | Powerful but heavy for this size; pnpm workspaces + Turborepo (later) is enough.   |
| Observability    | Sentry + Axiom                         | Mature error UX (Sentry) and fast trace ingest (Axiom), but two vendors and weaker portability story than the Grafana stack. |
| Observability    | Datadog                                | Best-in-class but expensive even at small scale; no free tier sufficient for v1.   |
| Observability    | Honeycomb                              | Excellent for traces, weaker on logs/metrics breadth; revisit if trace volume grows. |
| Observability    | Self-hosted LGTM in containers        | Right answer at scale or for compliance; premature for v1 single-developer ops budget. |
| Frontend RUM     | Sentry Browser SDK                     | Mature; not chosen because Faro covers RUM + errors under the same vendor as backend traces. |

## Consequences

**Positive**
- Operational simplicity: one platform (Railway) for application + data + cache, one bill, one dashboard.
- Velocity: Railway preview environments per PR cover the full stack including DB and Redis.
- Type safety end-to-end with OpenAPI-generated clients.
- Two-language stack matches owner's strengths (TS frontend, C# backend) without forcing context switches mid-feature.
- Reversible by Constitution Principle XIII: Docker-only, vanilla Postgres, vanilla Redis, OTLP-only observability. Render is a days-long swap; Fly.io / Coolify / Kubernetes are week-long swaps.

**Negative**
- **Single-platform blast radius.** Railway down means web, api, mcp, Postgres, and Redis are all down. Mitigation: Better Stack synthetic monitors, status page on a separate provider, documented runbook for emergency Render/Fly migration.
- **No native DB-per-PR branching.** Railway Postgres lacks Neon-style copy-on-write branching. Acceptable for single-developer v1; revisit when team grows.
- **Vercel-class Next.js edge optimization is forfeit.** Cloudflare-in-front mitigates most of the practical impact (caching, image optimization, DDoS), but ISR-on-the-edge and Vercel Edge Middleware are not on the table. We will need to be deliberate about cache headers and ISR revalidation strategy.
- **Constitution Principle XIII has a real cost.** Refusing Railway-specific buildpacks, Postgres extensions, and managed-feature shortcuts means writing a few more lines of Dockerfile and avoiding some Railway sugar. We pay this on every service.
- Polyglot ops: two language toolchains, two test frameworks, two CI lanes.
- No shared code between web and API beyond OpenAPI-derived types. Domain logic must not creep into the web tier.
- Better Auth + JWT validation in .NET requires a JWKS endpoint and rotation discipline.
- MCP server is a second client of the eToro public API; both will drift over time. We accept this and will revisit if drift becomes painful or if the MCP gains a committed v1 use case.
- Grafana Cloud free tier has caps (10k metrics series, 50GB logs, 50GB traces, 50k Faro sessions/month). Hitting them forces either an upgrade or a self-host migration sooner than ideal — but OTel keeps that migration cheap.
- Faro is younger than Sentry; if its error-grouping or source-map workflow disappoints, we'll feel it in incident triage before we feel it elsewhere.

## Follow-ups

- ADR-0002: Authentication topology (Better Auth → JWT → .NET JWKS validation flow).
- ADR-0003: eToro public API client design and rate-limit strategy.
- ADR-0004: Caching strategy (TTLs, invalidation, stale-while-revalidate).
- ADR-0005: OpenAPI and typed-client generation pipeline.
- ADR-0006: MCP role and use case (when one is committed).
- ADR-0007: Observability — Faro evaluation outcome and Grafana Cloud → self-hosted LGTM migration trigger.
- ADR-0008: Cloudflare configuration in front of Railway (cache headers, ISR strategy, image pipeline).
- ADR-0009: Railway-exit playbook — formal runbook for migrating to Render or Fly.io if Railway proves the wrong choice.
