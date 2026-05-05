# eToro App

A public, read-only portfolio analytics surface for eToro users (v1), built around the **eToro public API**. An existing MCP server (a wrapper over the same public API) sits alongside as an **optional sibling surface** — its concrete use case is TBD and a future ADR will document it when it crystallizes. Spec-driven via [GitHub Spec-Kit](https://github.com/github/spec-kit).

> **Status:** Foundation phase. Constitution and first spec are written. Stack is locked. Repo is not yet scaffolded.

---

## Foundation Documents

Read these in order before writing any code:

1. [`.specify/memory/constitution.md`](./.specify/memory/constitution.md) — non-negotiable principles.
2. [`specs/001-portfolio-view/spec.md`](./specs/001-portfolio-view/spec.md) — first thin spec (the surface the stack is hardened against).
3. [`docs/adr/0001-tech-stack.md`](./docs/adr/0001-tech-stack.md) — locked stack decisions and trade-offs.
4. [`docs/architecture.md`](./docs/architecture.md) — system topology, request flows, deployment.

---

## Stack at a glance

| Tier         | Choice                                                      |
|--------------|-------------------------------------------------------------|
| Frontend     | Next.js 15, React 19, TypeScript strict, Tailwind, shadcn/ui |
| Auth         | Better Auth (in `apps/web`), JWT validated by `.NET API`     |
| Backend API  | .NET 9 Minimal API, EF Core 9, Polly, Quartz.NET, Serilog    |
| MCP server   | Existing service, optional sibling surface (use case TBD)    |
| Database     | PostgreSQL 16 (Railway, vanilla)                             |
| Cache        | Redis 7 (Railway, vanilla)                                   |
| Hosting      | **Railway** (web, api, mcp, Postgres, Redis) + **Cloudflare** in front |
| CI/CD        | GitHub Actions; container images on GHCR; Railway pulls from GHCR |
| Observability| Grafana Cloud (LGTM + Faro) via OpenTelemetry + Better Stack synthetics |
| Portability  | Constitution Principle XIII: Docker-only, vanilla data tier, OTLP-only observability — Railway is a one-day swap to Render |

See ADR-0001 for full detail and alternatives considered.

---

## Repo Layout (target)

```
etoro-app/
├── apps/
│   ├── web/                  # Next.js 15 + React 19
│   ├── api/                  # .NET 9 minimal API
│   └── mcp/                  # existing MCP server (optional sibling, use case TBD)
├── packages/
│   ├── ui/                   # shared shadcn primitives
│   └── tsconfig/             # shared TS configs
├── infra/
│   ├── docker/               # Dockerfiles for api & mcp
│   ├── fly/                  # fly.toml per service
│   └── github/               # composite actions
├── specs/                    # Spec-Kit specs
├── docs/
│   ├── adr/                  # decision records
│   └── architecture.md
├── .specify/                 # Spec-Kit memory & templates
└── .github/workflows/        # CI/CD
```

---

## Prerequisites

- Node.js 22 LTS + pnpm 9
- .NET 9 SDK
- Docker Desktop (required — Constitution Principle XIII mandates `docker compose up` parity locally)
- `uv` (for the `specify` CLI): `pipx install uv` or `winget install astral-sh.uv`
- Railway CLI (`npm i -g @railway/cli`) for deploys and one-off env work
- (Optional) Cloudflare account for the production CDN/edge tier
- A GitHub account with a repo created for this project

---

## Running Locally

Per **Constitution Principle XIII**, every developer can bring up the full stack on their machine with Docker Compose. There are three modes, all driven by a single `compose.yml` at the repo root (no separate compose files).

| Mode | Command | What runs |
|------|---------|-----------|
| **Canonical (default)** | `docker compose up` | web, api, postgres, redis, otel-lgtm |
| **MCP-included** | `docker compose --profile mcp up` | Canonical + apps/mcp |
| **Infra-only** | `docker compose up postgres redis otel-lgtm` | Just the stateful + observability services — for use when running web/api on the host with `pnpm dev` and `dotnet watch` |

Once the canonical mode is up:

- Web app: <http://localhost:3001>
- API: <http://localhost:5001>
- Grafana (local LGTM observability): <http://localhost:3000>
- Postgres: `localhost:5432` (user/password documented in `.env.example`)
- Redis: `localhost:6379`

Hot reload is on by default — Next.js HMR for the web tier, `dotnet watch` for the API. Source is bind-mounted into the containers.

**Local observability is fully self-contained.** A `grafana/otel-lgtm` container hosts Loki, Tempo, Mimir, and Grafana — no traffic leaves your machine. Production sends OTLP to Grafana Cloud; local sends OTLP to `otel-lgtm:4318`. Only the `OTLP_ENDPOINT` env var changes.

**Local hits the real eToro public API.** There is no mock by default. Heavy local loops share the upstream rate-limit budget with production — tread accordingly. A future ADR may add a record/replay or fixture-mode toggle.

**Limitations vs. production:** no Cloudflare, no Railway, no synthetic monitors, no multi-region. ISR/cache behavior in `next dev` differs from `next start` — for end-to-end cache testing, build the production image locally and run it.

Full topology, env-var contract, and edge cases live in [`docs/architecture.md`](./docs/architecture.md) §11 "Local Development Topology."

---

## Bootstrap (one-time)

The Constitution and first spec are already in place. The next steps are:

### 1. Initialize the Spec-Kit toolchain

The `specify` CLI installs Spec-Kit slash commands tailored for Claude Code (or your AI agent of choice) into this repo.

```powershell
# From the repo root:
uvx --from git+https://github.com/github/spec-kit.git specify init --here --ai claude --force
```

This creates / updates `.specify/` with the templates that power the `/constitution`, `/specify`, `/clarify`, `/plan`, `/tasks`, `/analyze`, `/implement` slash commands. Our existing `.specify/memory/constitution.md` is preserved.

> If you prefer a different agent, replace `--ai claude` with one of `copilot`, `cursor`, `gemini`, `windsurf`, etc. See the Spec-Kit docs.

### 2. Run the spec-kit phases against spec 001

In Claude Code (or your chosen agent), with this repo open:

```text
/clarify  specs/001-portfolio-view/spec.md
/plan     specs/001-portfolio-view/spec.md
/tasks    specs/001-portfolio-view/spec.md
/analyze
```

`/plan` will reference ADR-0001 and `docs/architecture.md` as the source of truth for the stack.

### 3. Scaffold the apps

After `/plan` produces a concrete plan and `/tasks` produces task breakdowns, the implementation tasks will scaffold:

- `apps/web` via `pnpm create next-app@latest apps/web --ts --tailwind --app --no-src-dir --eslint --use-pnpm`
- `apps/api` via `dotnet new web -n EtoroApp.Api -o apps/api` and `dotnet new sln && dotnet sln add apps/api`
- Better Auth, Drizzle, EF Core, OpenTelemetry SDKs, Grafana Faro Web SDK, Playwright, Vitest, xUnit

Do not scaffold ahead of `/plan`. The plan is what tells us *which* options to enable.

### 4. Wire infra

- Create the GitHub repo and push.
- Create a Railway project. Inside it, provision: `apps/web` service, `apps/api` service, `apps/mcp` service, Postgres database, Redis instance.
  - For each service, configure Railway to **build from `Dockerfile`** (not Nixpacks) and pull the runtime image from GHCR after CI publishes it.
  - Enable Railway's PR preview environments for `apps/web` and `apps/api`.
- Add a Cloudflare zone for the production domain and point it at the Railway public hostname for `apps/web`. Configure Page Rules / Cache Rules per ADR-0008 (future).
- Create a Grafana Cloud stack (free tier) — note the OTLP endpoint, username, and API token; Faro app token will be created from the same stack.
- Create a Better Stack project for synthetic monitors against `/health` on `apps/api` and `apps/mcp`, and a public status page hosted off-platform.
- Add secrets to GitHub Actions and Railway. The same env-var keys (`DATABASE_URL`, `REDIS_URL`, `OTLP_ENDPOINT`, `JWKS_URL`, etc.) appear locally in `.env`, in CI, and in Railway — no platform-specific names.

Future ADRs will document: ADR-0002 (auth topology), ADR-0008 (Cloudflare config in front of Railway), ADR-0009 (Railway-exit playbook).

---

## Spec-Kit Workflow Reminder

```
Constitution  →  Specify  →  Clarify  →  Plan  →  Tasks  →  Analyze  →  Implement
   (done)        (done¹)
```
¹ For spec 001 only. Future features start from `/specify`.

Skipping phases violates Constitution Principle I.

---

## Contributing

- Conventional Commits.
- Branch naming: `feat/{spec-id}-{slug}`, `fix/{slug}`, `chore/{slug}`.
- Every PR links the spec it implements (or the ADR it amends).
- CI must be green before merge.

---

## License

TBD — to be decided before public release.
