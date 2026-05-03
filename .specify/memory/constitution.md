# Project Constitution

**Version:** 1.1.0
**Ratified:** 2026-04-30
**Last amended:** 2026-04-30
**Owner:** Gabriel Cerutti

This document captures the non-negotiable principles for the eToro App project. Every spec, plan, ADR, and pull request is measured against it. Principles change only by explicit amendment with a recorded version bump.

---

## I. Spec-First Development

No implementation work begins without an approved Specify document and a Plan derived from it. We follow the GitHub Spec-Kit grammar: **Constitution → Specify → Clarify → Plan → Tasks → Analyze → Implement.** Skipping phases is forbidden. "Tiny" features still need a thin spec — a paragraph counts.

Rationale: this project will be built across multiple AI-assisted sessions. The spec is the durable contract; chat history is not.

## II. Type Safety End-to-End

- **TypeScript** is configured with `strict: true`, `noUncheckedIndexedAccess: true`, and `exactOptionalPropertyTypes: true`. No `any` outside well-justified, commented exceptions.
- **C#** projects enable nullable reference types (`<Nullable>enable</Nullable>`) and treat warnings as errors in CI.
- API contracts are typed at both ends. The .NET API publishes an OpenAPI document; the web app generates a typed client from it.

Rationale: bugs caught by the compiler are bugs that never reach a user.

## III. API-First, Web is Thin

The .NET API owns all business logic, all eToro integration, all persistence. The Next.js app is a presentation layer with a thin BFF for session, auth, and request shaping. If logic appears in two places, it lives in the API.

The MCP server is a parallel surface for AI agents. It does not share runtime code with the web stack and does not sit in the web's request path. Logic duplicated between MCP and API is acknowledged and minimized; both are clients of the same upstream eToro contract.

## IV. Test Discipline

Every feature ships with:
- Unit tests for non-trivial functions and domain logic.
- Integration tests for every API endpoint, executed against a real Postgres (Testcontainers) — never a mocked database.
- At least one Playwright end-to-end test exercising the user-visible flow.

CI is the gate. A red CI is a stop-the-line event.

Rationale: this stack spans two languages and three deploy targets. Manual testing does not scale.

## V. Accessibility is a Baseline, Not a Feature

All public UI meets WCAG 2.1 AA. Charts have keyboard navigation and screen-reader summaries. Color is never the sole carrier of meaning. New components ship with axe-core checks passing.

## VI. Performance Budgets

- Web: P95 page load under 2.5s on a simulated 4G connection. Lighthouse Performance score above 90 for every public route. Largest Contentful Paint under 2.0s.
- API: P95 latency under 300ms for cached eToro reads, under 1500ms for cold reads.
- Bundle: initial JS payload under 200KB gzipped per route.

Regressions block merge.

## VII. Security and Trust Boundaries

- eToro responses are untrusted input. Validate, narrow types, never `any`-cast into the domain.
- Auth tokens never leave the server. The browser receives only a session cookie issued by Better Auth.
- Secrets live in the deployment platform's secret store, never in repo or `.env` committed files.
- All write endpoints require auth and idempotency keys where the operation is non-idempotent.
- Rate limit every public endpoint. eToro upstream calls are token-bucketed with backoff.

## VIII. Privacy by Default

Phase 1 processes only public eToro data — usernames and the data those users have made public. PII boundaries:
- Auth tables (Better Auth) are the only place we store user PII.
- The .NET API never reads from auth tables; it receives a verified user ID via JWT and uses it as an opaque key.
- No analytics tooling captures PII without an explicit consent flow.

## IX. Observability is Mandatory

Every request through the .NET API and every MCP tool invocation emits:
- A structured log with correlation ID and user ID hash.
- A trace span (OpenTelemetry).
- A metric counter and latency histogram.

If it isn't traced, it isn't shipped.

## X. API Versioning from Day One

The .NET API is mounted under `/v1/` from the first endpoint. Breaking changes get `/v2/`. We never break consumers in place.

## XI. Idempotency and Safe Retries

External calls (eToro upstream, payment, email if added) tolerate retries. Internal write endpoints accept an `Idempotency-Key` header where the operation is not naturally idempotent.

## XII. Reversibility Beats Cleverness

Default to the boring choice. Prefer techniques and tools that we know how to undo. Avoid lock-in we cannot afford to pay for in migration cost.

## XIII. Platform-Agnostic Deployment Discipline

We host on a consolidated platform (Railway in v1) for operational simplicity, but we accept that benefit only if migrating off the platform stays a days-long exercise rather than a weeks-long one. The following are non-negotiable:

- **Docker is the only deployment unit.** Every service ships from a `Dockerfile` checked into the repo. No reliance on platform-specific buildpacks (Nixpacks, Heroku-style detection, Vercel build steps) for production builds.
- **Service discovery via standard env vars only.** `DATABASE_URL`, `REDIS_URL`, `OTLP_ENDPOINT`, etc. No platform-specific service references baked into application code.
- **Secrets via environment variables.** Platform-specific secret APIs are wrapped behind a thin abstraction or avoided.
- **Local parity.** `docker compose up` from the repo root must run the full stack (web, api, mcp, postgres, redis) using the same images that deploy to production.
- **Database is vanilla Postgres, cache is vanilla Redis.** No platform-specific extensions, no platform-specific managed features baked into schema or queries.
- **Observability is OTLP-only.** No platform-native log/metric APIs.

**Why:** consolidation buys us a single dashboard, single bill, and single mental model. It costs us blast-radius isolation and the ability to use platform-specific superpowers. We accept the trade only on the condition that the door stays open: any service should run on Render, Fly.io, Coolify+VPS, or Kubernetes after a single day of cleanup.

**How to apply:** every PR that introduces a platform-specific binding (a Railway templates feature, a managed-service-specific extension, a non-Docker build path) is a Constitution-amendment-level change and requires explicit recorded justification.

---

## Amendments

| Version | Date       | Change                                                            |
|---------|------------|-------------------------------------------------------------------|
| 1.0.0   | 2026-04-30 | Initial ratification.                                             |
| 1.1.0   | 2026-04-30 | Added Principle XIII — Platform-Agnostic Deployment Discipline.   |
