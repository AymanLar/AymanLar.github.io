---
title: "Phase 3 — Local Kubernetes Deployment (k3d)"
description: "Running the SecureCloud full stack on a single-node k3d cluster with non-root workloads, secrets boundary, ingress, and verifiable end-to-end checks."
date: 2026-10-10
tags:
  - kubernetes
  - k3d
  - non-root
  - postgres
  - seccomp
  - networkpolicy
  - ingress
  - phase3
  - securecloud
---

# Phase 3 — Local Kubernetes Deployment (k3d)

_Phase 1 & 2 set the app, the tests and a hard "green PR only" pipeline. Phase 3 asks: can it run on real Kubernetes, on a dev laptop, with no paid services, and still satisfy concrete verifiable checks?_

## Objective

- Full local deployment of frontend, backend and PostgreSQL under Kubernetes.
- Use **k3d** — creates k3s in Docker with Traefik and `local-path` storage preinstalled. Less plumbing than kind for a "just works" dev loop.
- Keep manifests portable, production-shaped and least-privilege.
- Prove the deployment works: six end-to-end tests (`T1–T6`) must all pass.

No Kyverno/Falco/Cosign/Prometheus in this phase — purely the runnable workload + smoke checks.

## Architecture (local)

```text
              Windows + Docker Desktop + Git Bash
┌──────────────────────────────────────────────────────────────────┐
│                     k3d cluster "securecloud"                    │
│                                                                      │
│  http://localhost:8080 → k3d loadbalancer → Traefik (ingress)       │
│                                                 ├── / → frontend    │
│                                                 └── /api → backend  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ Next.js 3000 │─▶│ FastAPI 8000  │─▶│ PostgreSQL 5432 (PGDATA   │  │
│  │ non-root 1001│  │ non-root 999  │  │ /var/lib/postgresql/pgdata)│  │
│  │ RO tmp only   │  │ RO fs + tmp   │  │ StatefulSet + local-path   │  │
│  └──────────────┘  └──────────────┘  └───────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Why k3d, not kind

- **One command cluster** — no separate ingress install and no clunky extra manifests for port exposure.
- **Traefik out of the box** — works transparently with hostless Ingress.
- **local-path storage** — gives `RWX`/`RWO` semantics that behave reasonably on a single node without extra volume provisioners.
- **Portable objects** — everything is plain `apps/v1`, `core/v1`, `networking.k8s.io/v1`, `rbac.authorization.k8s.io/v1`. Swap the CNI to Calico/Cilium on a real cluster and it still makes sense.

## Manifests at a glance

- `kubernetes/base/` — namespace, configmap (non-secret), backend/frontend Deployments+Services, Postgres StatefulSet+headless Service.
- `kubernetes/rbac/` — per-workload ServiceAccounts (`securecloud-backend`, `securecloud-frontend`, `securecloud-postgres`) bound to a **deliberately empty** Role → cannot call the Kubernetes API.
- `kubernetes/network-policies/` — default-deny + allow-frontend→backend + allow-backend→postgres. **Declared but not enforced** by flannel on k3s — still valid and portable.
- `kubernetes/ingress/` — Traefik Ingress; routes `/api` to backend and `/` to frontend, hostless (`*`).
- `kubernetes/overlays/local/` — kustomize overlay: namespace, references, and `secretGenerator` that turns `secret.env` (gitignored) into the Opaque Secret consumed by Deployments/StatefulSet.

## Least privilege & security posture

- `runAsNonRoot: true` on all pods; fixed UIDs (`backend/postgres: 999`, `frontend: 1001`).
- `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`.
- `readOnlyRootFilesystem: true` for backend with an `emptyDir /tmp`; frontend has an `emptyDir /tmp` for caching.
- Per-workload ServiceAccount + empty Role (no API privileges).
- Secrets: only `DATABASE_URL` and `POSTGRES_PASSWORD` generated at deploy-time; never stored in the repo (`kubernetes/overlays/local/secret.env` is gitignored).
- Image tags pinned (`securecloud-backend:3.0.0`, `securecloud-frontend:3.0.0`, `postgres:16.15-alpine`). No `latest`.
- Resource requests/limits and real probes: backend readiness checks `/api/health/db` so traffic doesn't land before the DB is usable.

## The hard part: running PostgreSQL non-root (and not breaking initdb)

The official `postgres:16.15-alpine` has an opinionated `VOLUME /var/lib/postgresql/data` (owned by uid 70). With `runAsUser: 999`, two problems collide:

1. Mounting the PVC directly at `/var/lib/postgresql/data` — the local-path PV root is owned by `root` (fsGroup only changes group), so `initdb` can't `chmod` the dir it doesn't own.
2. Mounting at the parent `/var/lib/postgresql` but keeping default `PGDATA` — it hits the image's built-in `data` directory, which lives on a throwaway runtime volume and is owned by uid 70, not the pod user.

The fix: mount the PVC at `/var/lib/postgresql`, set `PGDATA=/var/lib/postgresql/pgdata`, and add an `emptyDir` for `/var/run/postgresql`. The `postgres` user creates `pgdata` under the PVC and owns/chmods it — so `initdb` succeeds without any initContainer running as root. This was confirmed empirically by inspecting the image entrypoint and via a debug pod.

## Scripts & Makefile

All logic lives in `scripts/` and `make` wraps them (works on macOS/Linux; on Windows run the `.sh` scripts in Git Bash):

| Command | Purpose |
| --- | --- |
| `scripts/cluster-up.sh` / `make cluster-up` | Create k3d cluster `securecloud` (ports 8080:80, 8443:443), wait for nodes |
| `scripts/deploy.sh` / `make deploy` | Generate `secret.env` once (reuse from `.env` or `openssl rand -hex 24`), build images with `NEXT_PUBLIC_API_URL=http://localhost:8080`, import into cluster, `kubectl apply -k overlays/local`, wait for rollouts |
| `scripts/status.sh` / `make status` | Nodes/workloads/Services/Ingress/Config/Secrets/SAs/NPs/PVCs + image versions |
| `scripts/logs.sh` / `make logs SERVICE=<backend\|frontend\|postgres>` | Tail logs by selector |
| `scripts/verify.sh` / `make verify` | Run T1–T6 |
| `scripts/cluster-down.sh` / `make cluster-down` | Delete cluster and all data |

Note: on some Windows setups `k3d` lives outside PATH — scripts assume `k3d` is available; in practice it's easy to `export PATH="/tmp/k3d-tool:$PATH"` (or just put it on PATH).

## End-to-end verification

`scripts/verify.sh` is the gate:

- **T1** — all pods labeled `app.kubernetes.io/part-of=securecloud` are Ready (180s timeout).
- **T2** — frontend reachable via ingress: `curl -sS -o /dev/null -w '%{http_code}' http://localhost:8080/` → `200`.
- **T3** — backend health via ingress: `GET /api/health` contains `"ok"`.
- **T4** — backend → PostgreSQL via app: `GET /api/health/db` contains `"ok"` (proves the app opened a DB connection using the Secret DSN).
- **T5** — in-cluster: frontend pod `wget -qO- http://securecloud-backend:8000/api/health` returns `"ok"`.
- **T6** — in-cluster: backend pod connects to PostgreSQL with `psycopg` using `DATABASE_URL` from the Secret (scheme normalized from SQLAlchemy `postgresql+psycopg://` to plain `postgresql://` for the raw psycopg test — the app itself uses SQLAlchemy, which accepts the + driver form).

All six pass.

## Bringing it up (from scratch)

```bash
# after cloning
make cluster-up
make deploy
make verify    # should print ALL CHECKS PASSED
make status
curl http://localhost:8080/        # frontend
curl http://localhost:8080/api/health/db  # backend->DB
make cluster-down  # when done
```

## Takeaways

- **Portable, runnable hardening** — fsGroup alone isn't enough for non-root `initdb`; the correct PGDATA placement under the PVC is the reliable fix for the official image.
- **Deterministic smoke tests** — T1–T6 turn "it works" from an assertion into a script you can run before/after any change to manifests or scripts.
- **Small operational surface** — one overlay, one secret file, thin scripts. Easy to reason about and easy to tear down.
- **Docs in lockstep** — `docs/PHASE-3-KUBERNETES.md` captures the why, not just the what, so future "Kubernetes" phases don't rediscover these pitfalls.

Phase 3 closes the loop: the same app that passed a strict CI pipeline now runs as a real, constrained Kubernetes workload on a laptop and proves it with concrete checks.