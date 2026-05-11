# toro-pulse — Agent Instructions

Public, read-only portfolio analytics for eToro users (v1). Spec-driven via [GitHub Spec-Kit](https://github.com/github/spec-kit). Stack is **locked** in [docs/adr/0001-tech-stack.md](docs/adr/0001-tech-stack.md); the constitution is **non-negotiable**.

## Read first (in order)

1. [.specify/memory/constitution.md](.specify/memory/constitution.md) — 13 non-negotiable principles.
2. [docs/adr/0001-tech-stack.md](docs/adr/0001-tech-stack.md) — locked stack.
3. [docs/architecture.md](docs/architecture.md) — system topology and local-dev rules (§11).
4. [specs/](specs/) — feature specs. Always work against a specific spec; never freelance.

## Workflow gate (Constitution Principle I)

> *No implementation work begins without an approved Specify document and a Plan derived from it. Skipping phases is forbidden.*

Order: **Constitution → Specify → Clarify → Plan → Tasks → Analyze → Implement.** "Tiny" features still need a thin spec — a paragraph counts.

Use Spec-Kit slash commands (installed in `.claude/skills/`):

- `/speckit-specify` — create a baseline spec
- `/speckit-clarify` — resolve `[NEEDS CLARIFICATION]` markers (use *before* `/speckit-plan`)
- `/speckit-plan` — produce an implementation plan grounded in ADR-0001 + architecture.md
- `/speckit-tasks` — break the plan into actionable tasks
- `/speckit-analyze` — cross-artifact consistency check (after `/speckit-tasks`, before `/speckit-implement`)
- `/speckit-checklist` — generate quality checklists
- `/speckit-implement` — execute the plan

If a user request would require touching code without a spec, push back and offer to start with `/speckit-specify`.

## Stack guardrails (one-liners)

- **TypeScript:** `strict: true`, `noUncheckedIndexedAccess: true`, `exactOptionalPropertyTypes: true`. No `any` without a comment justifying it. (Constitution II)
- **C#:** `<Nullable>enable</Nullable>`; warnings-as-errors in CI. (Constitution II)
- **API contracts:** .NET API publishes OpenAPI; web generates a typed client from it. Never hand-write API types. (Constitution II)
- **API-first:** all business logic and eToro integration live in `apps/api`. `apps/web` is a thin presentation + BFF layer (session, auth, request shaping only). MCP is a parallel surface that shares **no runtime code** with web/api. (Constitution III)
- **Two-schema DB:** `auth` schema is owned by Better Auth (writes only from `apps/web`). `app` schema is owned by EF Core in `apps/api`. **No cross-schema joins.** Bridge user identity via JWT `sub` claim only.
- **Vanilla data tier:** Postgres 16 + Redis 7. No platform-specific extensions, no managed-feature couplings. (Constitution XIII)
- **Docker-only deploys:** every service ships from a `Dockerfile` in the repo. No Nixpacks, Heroku detection, or Vercel build steps. (Constitution XIII)
- **OTLP-only observability:** no platform-native log/metric APIs. (Constitution XIII)
- **API versioning:** all endpoints under `/v1/` from day one. Breaking changes get `/v2/`. (Constitution X)
- **Idempotency:** non-idempotent write endpoints require `Idempotency-Key` header. External calls tolerate retries. (Constitution XI)

## Test rules (Constitution IV)

Every feature ships with **all three**:

1. **Unit tests** — non-trivial functions and domain logic.
2. **Integration tests** — every API endpoint, against a real Postgres via **Testcontainers**. Never a mocked database.
3. **Playwright E2E** — at least one test exercising the user-visible flow.

CI is the gate. **A red CI is a stop-the-line event.** Don't ship around it; fix it.

Performance budgets (Constitution VI) and accessibility (V, WCAG 2.1 AA, axe-core passing) regressions also block merge.

## Local development

Per Constitution XIII, `docker compose up` from the repo root brings up the canonical stack. Three modes (see [docs/architecture.md §11](docs/architecture.md)):

| Mode         | Command                                       | Services                                              |
| ------------ | --------------------------------------------- | ----------------------------------------------------- |
| Canonical    | `docker compose up`                           | web, api, postgres, redis, otel-lgtm                  |
| MCP-included | `docker compose --profile mcp up`             | + apps/mcp                                            |
| Infra-only   | `docker compose up postgres redis otel-lgtm`  | host runs web/api with `pnpm dev` / `dotnet watch`    |

- Local observability is fully self-contained (`grafana/otel-lgtm` container). Only `OTLP_ENDPOINT` differs between local and prod.
- Local hits the **real eToro public API** — no mock by default. Heavy loops share the upstream rate-limit budget; tread accordingly.

## Git / PR conventions

- **Conventional Commits.** No exceptions.
- **Branches:** `feat/{spec-id}-{slug}`, `fix/{slug}`, `chore/{slug}` (e.g., `feat/001-portfolio-view-search-input`).
- **PRs link a spec or amended ADR.** No untethered changes.
- **Constitution edits** require a version bump and an amendment-log row.

## Don't

- ❌ `git commit --no-verify` — pre-commit hooks exist for a reason.
- ❌ `git push --force` to `main`.
- ❌ Store auth tokens client-side. Browser gets a session cookie only. (Constitution VII)
- ❌ `any`-cast eToro responses into the domain. They are untrusted input — validate, narrow types. (Constitution VII)
- ❌ Introduce platform-specific bindings (Railway templates, Postgres extensions, Vercel build steps) without an explicit ADR. Constitution XIII treats this as amendment-level. (XIII)
- ❌ Capture PII in analytics without an explicit consent flow. (Constitution VIII)
- ❌ Ship an API endpoint without OpenTelemetry traces, structured logs, and metrics. *If it isn't traced, it isn't shipped.* (Constitution IX)

## Useful MCP servers (already attached, user-scope)

- **`etoro-api-docs`** — authoritative source for every eToro API question (auth, rate limits, payload shapes). Reach for this during `/speckit-clarify` against any spec touching eToro.
- **`context7`** — current docs for Next.js 15, React 19, EF Core 9, Polly, Better Auth, etc.
- **`Microsoft Learn`** — .NET 9, EF Core, OpenTelemetry .NET SDK.
- **`Playwright`** — browser automation for E2E tests against a running app.

## Repo layout (target — most still TBD)

```text
apps/web        # Next.js 15 + React 19 (presentation + BFF)
apps/api        # .NET 9 Minimal API (business logic + eToro client)
apps/mcp        # MCP server (sibling, no shared code)
packages/ui     # shadcn primitives shared by web
packages/tsconfig
infra/docker    # Dockerfiles
.specify/       # Spec-Kit memory & templates
specs/          # one folder per feature: NNN-slug/
docs/           # ADRs, architecture
```

Currently scaffolded: docs, specs, .specify. Apps/packages/infra arrive after `/speckit-plan` for spec 001.

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->
