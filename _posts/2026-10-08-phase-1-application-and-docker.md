---
title: "Phase 1 — Building SecureCloud: a runnable, boring-by-design cloud platform"
date: 2026-10-08
tags: [portfolio, python, fastapi, nextjs, docker, postgresql]
---

# Phase 1 — Building SecureCloud: a runnable, boring-by-design cloud platform

This is a build log for the first phase of SecureCloud, a small cloud deployment
management platform I built as a portfolio project. I'm writing it partly for
myself (so I can point back at *why* things are the way they are) and partly in
the spirit of a DevSecOps engineer: everything in this project is documented,
runnable, and verifiable — nothing here exists on trust.

## Why another "cloud platform" demo?

I'm aiming for a junior DevSecOps / Cloud / DevOps role. Portfolio projects in
that space usually fall into two traps:

1. **The fake cloud project** — screenshots of dashboards that never existed,
   "architecture diagrams" for systems that don't run.
2. **The tutorial-shaped project** — a copy of a quickstart with no reasoning
   behind it, that breaks the moment you change one detail.

I wanted neither. So I set hard constraints up front and designed a project
that would be genuinely useful to *reason about* in an interview, not just
pretty.

## The constraints that shaped everything

- **Runnable locally on one machine.** Anything I show someone must start with
  one command.
- **$0.** No AWS, no Azure, no GCP, no paid SaaS. If it isn't open source and
  free, it isn't in the project.
- **No fake security.** "Secure" is a claim that has to be backed by something
  you can verify. Where the project makes a security claim, there's a check
  that proves it.
- **Production-style, not overcomplicated.** A real code layout, real config,
  real health checks — but no invented complexity for its own sake.
- **Everything documented must actually work.** This one turned out to be the
  most important, and it caught me once (more on that below).

## Choosing the stack

For a small, single-owner platform the choices were honestly easy:

| Layer     | Choice            | Why                                                        |
|-----------|-------------------|------------------------------------------------------------|
| Backend   | Python + FastAPI  | Fast, typed via Pydantic, first-class OpenAPI              |
| ORM       | SQLAlchemy 2      | The sane default; declarative models map cleanly to DDL    |
| Database  | PostgreSQL 16     | The industry default; worth matching from day one          |
| Frontend  | Next.js (App Router) + TypeScript | Realistic for cloud UIs; strict typing edges out bugs |
| Runtime   | Docker Compose    | One command brings up the whole system locally             |

The domain — *users create projects and trigger deployments for them* — is
small but honest: it's a believable slice of what a cloud platform does, and it
exercises real patterns (relationships, validation, distinct service roles)
without pretending to be a multi-service monster.

## The backend

I used an **application factory** pattern: `create_app(settings)` builds the
FastAPI app, and a module-level `app = create_app()` keeps `uvicorn
app.main:app` happy while tests create their own app from their own settings.

```python
# app/main.py (simplified)
app = create_app()  # uvicorn entry point

def create_app(settings: Settings) -> FastAPI:
    app = FastAPI(title=settings.app_name, lifespan=lifespan)
    ...
```

Three details were deliberate:

1. **Configuration is injected, not imported.** `config.py` loads
   `Settings` from environment variables (`DATABASE_URL`, `CORS_ORIGINS`, …).
   Nothing is hardcoded, which makes the same app safe to run in tests, dev,
   and CI with different settings.
2. **The database is created with a retry loop at startup.** The backend
   container can start before PostgreSQL is ready; instead of racing, the app
   retries the schema creation a handful of times and only then serves
   requests. This is the kind of small thing that keeps `docker compose up`
   reliable on a cold start.
3. **A typed router split.** `health`, `users`, `projects`, `deployments`
   each get their own router, Pydantic schemas define request/response shapes,
   and the DB session is a FastAPI dependency.

### The tests

I wrote 19 tests using `httpx`/`TestClient` against an **in-memory SQLite**
database. SQLite here is a deliberate trick: the app is written against
SQLAlchemy and Postgres-specific features aren't used in these endpoints, so
the whole suite runs in under a second with zero infrastructure. It covers
every endpoint including the error paths — `404` for unknown resources,
`422` for validation failures, `409` for duplicate emails, `503` when the
database is unreachable.

The error-path tests are part of the "no fake claims" rule: the README lists
every error code the API can return, and each one has a test.

## The frontend

Next.js 16 (App Router) with TypeScript. The whole UI is one dashboard page in
`lib/api.ts` + `app/page.tsx`:

- a **typed API client** with a `fetch` layer that turns non-2xx responses
  into a `ApiError` carrying the server's `detail`;
- a form-driven UI for users → projects → deployments, mirroring the domain;
- an "API online" badge wired to the health endpoint, so connectivity problems
  are visible instead of mysterious.

The page is a client component that loads data on mount and refreshes after
each mutation. Nothing clever, but it exercises the API, the error paths, and
the CORS configuration — the three things that go wrong first when wiring a
frontend to a backend.

## Containerizing

Both Dockerfiles follow the same recipe:

- **multi-stage builds** — build tools stay in the builder stage;
- **non-root users** in the runtime image (`uvicorn`/`next` as an unprivileged user);
- **a real `HEALTHCHECK`** — the backend probes `/api/health`, the frontend
  probes its own port with `wget`. `docker compose ps` reports `(healthy)`
  or `(unhealthy)`, and the frontend/backend images are honest about their own liveness.

The dashboard's kicker is the compose health gating:

- `db` has `pg_isready` as its healthcheck;
- `backend` starts only after `db` is healthy (`condition: service_healthy`);
- `frontend` starts only after `backend` is healthy.

One command, and the whole stack comes up in the right order every time.

## Things that went wrong (the honest part)

This is the section I actually want to remember, because it's where the
"documented means run" rule earned its keep.

**1. Docker's CLI wasn't on PATH.** `docker` was nowhere to be found — Docker
Desktop's real binary lives under its install directory. A quick `where docker`
after checking the Desktop install location fixed it. Fine. Lesson: environment
discovery before assuming tooling is broken.

**2. I killed Docker Desktop by mistake.** A leftover `next dev` server was
holding port 3000, so I ran a PowerShell `Stop-Process` on "the process owning
the port". It turned out that process was Docker Desktop's backend, not the
dev server. Docker daemon went down; I restarted Docker Desktop and the
`restart: unless-stopped` containers came back. Lesson: verify the process
identity before killing something by port, especially when the port is
published by Docker itself.

**3. The README was wrong, and verification caught it.** My first draft of the
README said the backend "fall[s] back to SQLite automatically" when no
`DATABASE_URL` is set. That was a nice story — and it was a lie. `DATABASE_URL`
is *required*. The verification pass that I'd committed to caught the
inaccuracy, and I fixed the README to describe what the code actually does
(dev instructions now point `DATABASE_URL` at a local SQLite file explicitly,
which is the real, working workflow). Lesson: write docs from what the code
does, and re-run what you document.

## Verification discipline

Phase 1 shipped with a verification table in the README, and every row was
actually run:

- `docker compose up --build` → all three containers `healthy`
- every API endpoint + error code + CORS behavior smoke-tested with `curl`
- schema auto-created in PostgreSQL, rows inserted via the API, verified
  directly with `psql`, and confirmed to survive a container restart
- 19/19 tests passing
- clean production `next build`
- documented local dev flow re-verified end-to-end

That table is the backbone of the whole project. It's what turns "I built a
cloud platform" into "here is a system whose every documented claim is a
resolved checkmark."

## Where this leaves the project

Phase 1 produced a working, containerized, tested three-service platform that
runs anywhere Docker does. The structure was ready for the hard part: Phase 2,
where I stopped manually checking "does this have obvious problems" and
started building a pipeline that does it automatically.

[Continue to Phase 2 — A CI/CD pipeline that actually shifts security left →](./2026-10-08-phase-2-cicd-and-shift-left-security.md)