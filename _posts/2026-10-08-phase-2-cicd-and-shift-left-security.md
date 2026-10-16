---
title: "Phase 2 — A CI/CD pipeline that actually shifts security left"
date: 2026-10-08
categories: [projects]
tags: [portfolio, github-actions, semgrep, gitleaks, trivy, shift-left]
---

# Phase 2 — A CI/CD pipeline that actually shifts security left

This is the second build log for SecureCloud. Phase 1 shipped a working
FastAPI + Next.js + PostgreSQL stack in Docker. Phase 2 is where the project
becomes a real exercise in shift-left security: every commit goes through a
pipeline that lints, tests, static-analyzes, scans for secrets, and scans
dependencies and container images — and any finding fails the build.

The goal was never "look at how many tools I glued together." It was: *a
pipeline with zero fake gates, where each scanner is something I can run on
my laptop with one command, and where every failure has a fix.*

## First, a housekeeping restructure

Before adding any CI, the repo needed a sane layout. The project had grown as
a flat tree, and the pragmatic structure for a two-front-two-back repo is
monorepo-ish: application code under `app/`, glue and docs at the root.

```
SecureCloud/
├── app/
│   ├── backend/          # FastAPI + tests
│   └── frontend/         # Next.js
├── .github/workflows/    # CI
├── docs/                 # security pipeline docs
├── docker-compose.yml
├── requirements-tools.txt
└── README.md
```

The compose file lives at the root and now points its build contexts at
`./app/backend` and `./app/frontend`. Everything else (Dockerfiles, the
`.env.example`, `.gitignore`) moved into or beside their services. This is
the layout GitHub Actions sees, so the pipeline and the repo structure had to
agree before I could even write the first job.

## Pin everything, so the scanner can read the truth

Trivy (the dependency and container scanner, later) can only tell you about
*vulnerabilities in a specific version* if it can resolve an exact version.
Sloppy ranges like `fastapi>=0.100` aren't parseable — and worse, they mean
the lockfile isn't the source of truth, so CI and your laptop can drift.

So every manifest is pinned with `==`:

```text
# app/backend/requirements.txt
fastapi==0.143.0
uvicorn==0.54.0
psycopg==3.3.6
SQLAlchemy==2.1.4
```

Two decisions here are worth explaining:

1. **Dev vs runtime deps are split.** `requirements-dev.txt` holds test-only
   packages (`pytest`, `httpx`) on top of the runtime closure via
   `-r requirements.txt`. The production image installs only the runtime file.
2. **Tooling is pinned separately** in a root `requirements-tools.txt`
   (`ruff` and `semgrep`). The tools are pinned once, in one file, with a
   comment telling you to keep it in sync with the workflow's `env` block.

There was also a subtle bug I caught while pinning: the original `uvicorn[standard]`
extras pulled in `uvloop`, which is Unix-only and can't be pinned from a Windows
machine — its wheel doesn't exist there. Pinning `uvicorn==0.54.0` plain (no
extras) removed the unpinnable dependency and produced a closure that resolves
identically everywhere. That's exactly the class of problem that makes
"works on my machine" pipelines fail in CI.

## Backend lint: ruff

`ruff check` and `ruff format` with a `pyproject.toml` config. One rule
deserves a mention because it's a FastAPI footgun: `B008` (do not use mutable
default arguments) fires on the *standard* FastAPI pattern

```python
def list_users(session: Session = Depends(get_session), limit: int = Query(50)):
```

The B-feature set's `extend-immutable-calls` exists for exactly this — it lets
you declare FastAPI's DI callables as expected immutable defaults so the rule
stays on but stays quiet for idiomatic FastAPI:

```toml
[tool.ruff.lint.flake8-bugbear.extend-immutable-calls]
"fastapi.Depends" = true
"fastapi.Query" = true
```

The project now has zero `# noqa` exceptions on the table, which keeps the
lint gate honest.

## Frontend lint: the ESLint 9 flat-config saga

Next.js 16 removed `next lint`, so linting is the plain `eslint .` CLI with a
flat config. My first attempt used the *legacy* compat recipe: `FlatCompat`
with `eslint-config-next`'s old-style exports. It blew up with a wonderfully
opaque failure:

```text
TypeError: Converting circular structure to JSON
    --> starting at object with constructor 'Object'
    |     property 'configs' -> object with constructor 'Object'
    ...
```

The root cause: `eslint-config-next` v16 is a *flat* config now, and
`FlatCompat` tries to validate it through the legacy pipeline, which chokes on
the plugin's internal circular references.

The fix was to stop shimming and read the v16 docs: import the flat configs
directly and spread them.

```js
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";

const eslintConfig = defineConfig([
  ...nextVitals,
  ...nextTs,
  globalIgnores([".next/**", "out/**", "build/**", "next-env.d.ts"]),
]);

export default eslintConfig;
```

The config then immediately earned its keep by flagging a real React 19 issue:
`react-hooks/set-state-in-effect` fired because the initial data load called a
function that set state *synchronously within the effect body*. The fix was to
wait properly for the fetch inside the effect (`await refresh()`) so state
updates happen after the async boundary — a genuine improvement React's new
rule exists to enforce.

## The `braces` CVE: when "no fix exists" is the honest answer

npm `audit` flagged 5 HIGH vulnerabilities — all `braces`, a transitive
dependency of the *ESLint toolchain* (`micromatch` → `fast-glob` →
`@next/eslint-plugin-next`). I tried to fix it. The pinnacle is here:

- `braces@3.0.3` is already installed and is the newest release — and it is
  itself still flagged.
- `npm audit fix --force` "fixes" it by downgrading `eslint-config-next` to
  v14, a breaking downgrade that also doesn't make sense.

This is the moment where "I never ignore vulnerabilities" meets reality: the
vulnerability has **no fixed version** to upgrade to. The correct engineering
response is not to silently push a green build — it's to make the risk
*tracked and visible*:

- the packages are build-time dev tooling, never bundled into the shipped
  images, and unreachable from runtime code;
- Trivy's `--ignore-unfixed` reports the advisory but doesn't make the build
  fail on an unfixable advisory;
- the whole thing is written down in `docs/SECURITY-PIPELINE.md` under
  "Known limitations and tracked risk", with the exact reason and a note to
  revisit on upstream patch.

Setting that policy was probably the most important decision in Phase 2:
**scan thresholds are a policy, documented and defensible** — HIGH/CRITICAL
fails the build *only when a fix exists*; anything unfixable is reported,
reviewed, and logged. That is not "ignoring." Ignoring is what you do by
accident; this is what you do on purpose with a paper trail.

## SAST with Semgrep

Semgrep running the `p/ci` ruleset (the same ruleset its CI uses) over the
backend and frontend: Python injection, hardcoded credentials, SSRF, SQLi,
unsafe deserialization, and friends. The gate is `--error` — any finding fails
the job. On a first pass against this codebase: **0 findings in 41 files
across 50 rules**. Not because the code is special, but because it's small,
boring, and uses validated frameworks for everything dangerous (Pydantic for
input, SQLAlchemy for SQL). Shift-left scans being clean on a clean codebase is
the expected outcome, and that's a feature: the pipeline is now the thing that
*keeps* it clean instead of the thing that catches messes after they exist.

## Secret scanning with Gitleaks

Gitleaks scans both the working tree and git history for committed secrets —
cloud keys, tokens, private keys, and high-entropy strings. It runs redacted
(`--redact`) so a leak can't be echoed into the logs while it's being fixed.

The dependency between this job and the rest of the repo is the one you want
to think about: if someone commits a real key, *history rewriting is required*
because the key stays in `git log`. The fix playbook is in the docs: rotate it,
remove it from history, never just delete the file.

## Dependency scanning with Trivy: two quirks I hit and worked around

Trivy's filesystem scanner reads manifests and maps exact versions against
its vulnerability database. It found the Python and npm manifests clean —
with two sharp edges worth knowing:

**1. Trivy only auto-detects files named exactly `requirements.txt`.** The
`-r requirements.txt` reference in `requirements-dev.txt` makes that true of
the dev file too, but `requirements-tools.txt` was invisible to it. Quietly
not scanning a manifest is worse than failing loudly. The workflow stages the
extra manifests under detectable names for an explicit second scan:

```bash
mkdir -p deps-scan/extra deps-scan/tools
grep -v '^-r ' app/backend/requirements-dev.txt > deps-scan/extra/requirements.txt
cp requirements-tools.txt deps-scan/tools/requirements.txt
trivy fs ... deps-scan
```

**2. Trivy's dev-dependency heuristic.** A `requirements.txt` living under a
directory path that looks *dev-ish* gets silently treated as a dev manifest
and suppressed — even with `--include-dev-deps`. I staged the dev-manifest
contents under a neutral path (`extra/`) and confirmed they were scanned. This
exact behavior is documented in the pipeline doc so nobody "fixes" it later by
accident.

## Container scanning: the npm lesson

`trivy image` on the built images found the backend clean but the frontend
image full of **10 fixable HIGH CVEs** — all in one place:

```text
usr/local/lib/node_modules/npm/node_modules/brace-expansion/...
usr/local/lib/node_modules/npm/node_modules/ip-address/...
usr/local/lib/node_modules/npm/node_modules/pacote/...
usr/local/lib/node_modules/npm/node_modules/picomatch/...
usr/local/lib/node_modules/npm/node_modules/sigstore/...
```

None of them are *our* code. The base `node:22-alpine` image bundles `npm`,
and npm vendors those packages. Our runtime never invokes npm — it runs
`next start` with plain `node`. So the standard fix is also the clean one:
**remove the base image's bundled npm from the runtime stage.**

```dockerfile
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npx && \
    rm -f /usr/local/bin/npm
```

Rebuild, rescan: green. This is shift-left applying to *base images*, which is
easy to forget when you're only scanning your own dependencies.

## The workflow itself

Everything above had to become a single, readable, fail-loud workflow. GitHub
Actions can't express `needs` across workflow files, so it's one file with
seven chained jobs:

```text
lint  ->  unit-tests  ->  sast  ->  secret-scan  ->  dependency-scan  ->  docker-build  ->  container-scan
```

Design decisions, each with a reason:

- **No `continue-on-error` anywhere.** If a scanner finds something, the
  pipeline is red. Green must mean green.
- **Least privilege.** Workflow-wide `permissions: contents: read`. No secrets
  are used; nothing needs write scope; there's nothing to leak.
- **Pinned actions by commit SHA.** Every `uses:` is a full 40-char SHA with a
  `# v4`-style tag comment, resolved via `git ls-remote` — so a compromised or
  changed tag can never silently replace what we run. (An `actions/` SHA
  changes on every tag; pinning the immutable SHA is the actually-safe form.)
- **Pinned tool versions** in one `env` block, matching the local manifests.
  Every binary download is verified with `sha256sum` before it runs.
- **The container scan gets pre-built images.** `docker-build` saves the two
  images to an artifact, `container-scan` downloads and `docker load`s them —
  so the scan job proves the *exact* artifact that passed the earlier jobs.
- **No hardcoded credentials.** The compose file interpolates `${POSTGRES_USER:?}`
  etc., and CI generates throwaway values with `openssl rand` at build time.
  There is nothing sensitive in the file to leak.

Validation of the workflow itself: `python -c "import yaml; yaml.safe_load(...)"`
for parsing, plus **actionlint** for semantic correctness, plus a tiny script
that asserts the job chain is exactly what the README claims. The workflow
lints and gates itself.

## The docs are part of the pipeline

`docs/SECURITY-PIPELINE.md` is the contract for the whole thing:

- what each scanner does and what it detects;
- **when** the pipeline fails, with the exact thresholds;
- **how to fix** a finding for each category;
- one-command local reproductions for every check, so a developer can iterate
  in seconds instead of round-tripping through CI.

The README got a Phase 2 section with the pipeline diagram, the thresholds
table, and a verification-status row for every check — all actually run.

## Results

Every check passed locally with the CI-identical command:

| Check            | Result                                              |
|------------------|-----------------------------------------------------|
| ruff + ESLint    | clean, zero disabled rules                          |
| pytest           | 19/19                                               |
| Semgrep `p/ci`   | 0 findings (50 rules, 41 files)                     |
| Gitleaks         | no leaks                                            |
| Trivy fs         | 0 fixable HIGH/CRITICAL (Python + npm manifests)    |
| Trivy image      | 0 fixable HIGH/CRITICAL on both images              |
| actionlint + YAML| clean                                               |

And the honest ledger of what's *not* green-washed: the `braces` advisory is
documented as tracked risk with no upstream fix and no runtime exposure, and
Trivy's manifest-detection quirks are documented so the pipeline's coverage is
verifiable, not assumed.

## What shift-left actually bought here

The most valuable thing Phase 2 delivered isn't the tools — it's the *loop*:
every change now reinflates to the same checks a human would run by hand,
and the checks are cheap enough and fast enough that they run on every single
pull request. A candidate-looking trap like a 10-CVE base image or a
circular-structure config error became the pipeline's job, caught during an
afternoon build instead of three weeks after a production deploy.

The natural next steps — authentication, IaC, Kubernetes — are deliberately
left out. When they come, they'll be added the same way: as checks that can be
run locally, gates that actually gate, and claims that are verified rather
than asserted.