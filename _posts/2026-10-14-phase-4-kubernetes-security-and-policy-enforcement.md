---
title: "Phase 4 — Kubernetes Security & Policy Enforcement (Kyverno)"
description: "Turning cluster security guidelines into enforced admission policy: eight Kyverno policies tested with bad/good manifests, an audit-to-enforce workflow, and a least-privilege RBAC review."
date: 2026-10-14
tags:
  - kubernetes
  - kyverno
  - admission-control
  - policy-as-code
  - rbac
  - least-privilege
  - security
  - phase4
  - securecloud
---

# Phase 4 — Kubernetes Security & Policy Enforcement (Kyverno)

_Phase 3 got the stack running on Kubernetes and proved it with T1–T6. But "it runs" is not "it's safe." Phase 4 turns a list of cluster hardening wishes into code the API server actually enforces — and proves every rule with a deliberately bad manifest and a compliant one._

## Objective

- Enforce a secure-by-default baseline on every workload admitted to the cluster.
- Express it as **policy as code**: versioned YAML in Git, reviewed like any other change.
- Ship, for each rule, a **violating manifest** and a **compliant manifest**, and test both.
- Use an **audit → enforce** workflow: measure impact first, block second.
- Review **RBAC** for genuine least privilege.

No Falco (runtime detection) and no Cosign (supply-chain signing) — those complement admission policy and are a later phase.

## Why a policy engine at all

Kubernetes ships admission *hooks* but no opinions. A cluster is a concentration of authority, so the interesting question is how a single compromised workload turns into a bigger problem:

| Surface | What the attacker gains |
| --- | --- |
| Root containers | Kernel/runtime escape becomes host escalation |
| `allowPrivilegeEscalation` | setuid + file capabilities turn a process into root |
| Retained capabilities (`NET_RAW`, `SYS_ADMIN`) | Packet spoofing, container escape |
| No resource limits | Starve the node — DoS / noisy neighbour |
| `:latest` tags | Non-reproducible deploys; an unrelated push changes what runs |
| Any registry | Typosquatting / dependency confusion runs attacker code |
| `hostNetwork` | Reach the kubelet and node services; bypass NetworkPolicy |
| `hostPath` | Mount the runtime socket — the classic escape |
| Over-privileged RBAC | A leaked token reads secrets, creates pods, escalates |

The goal: **the safe configuration is the one you get by default**, and the dangerous one requires an explicit, reviewable exception.

## Why Kyverno

- **Kubernetes-native** — policies are Kubernetes resources, evaluated by admission webhooks. No new DSL to learn or run outside the cluster.
- **Policy as code** — YAML in Git, diffable, testable, pipeline-friendly.
- **Autogen** — a Pod policy is automatically projected to Deployment/StatefulSet/DaemonSet/Job/Rollout rules, so you write the rule once.
- **Two modes with one switch** — `Audit` records violations in `PolicyReport` objects; `Enforce` blocks them. Perfect for a staged rollout.

## The eight policies

All cluster-scoped, all excluding the system namespaces that legitimately need host access (`kube-system`, `kube-public`, `kube-node-lease`, `kyverno`). Base is **Audit**; the `enforce/` kustomize overlay flips all eight to **Enforce**.

| # | Policy | Blocks |
| --- | --- | --- |
| P1 | `securecloud-disallow-root` | `runAsNonRoot: false`, `runAsUser: 0` |
| P2 | `securecloud-disallow-privilege-escalation` | `allowPrivilegeEscalation: true` |
| P3 | `securecloud-require-drop-all-capabilities` | anything other than `capabilities.drop: [ALL]` |
| P4 | `securecloud-require-resource-limits` | containers without CPU/memory limits |
| P5 | `securecloud-disallow-latest-tag` | `:latest` and missing tags |
| P6 | `securecloud-restrict-image-registries` | images outside `securecloud-*`, `postgres:*`, `docker.io/library/*`, `ghcr.io/*` |
| P7 | `securecloud-disallow-host-network` | `hostNetwork: true` |
| P8 | `securecloud-disallow-host-path` | any `hostPath` volume |

Two patterns were worth learning the hard way:

- **Single-element array patterns match *all* containers.** Writing the container securityContext check once covers every container and init container.
- **`deny` rules with JMESPath error on missing keys.** A pod with no volumes made ``length(volumes)`` blow up and *deny the compliant pod*. The fix is a null-safe default: `length(volumes || \`[]\`)` for lists, `|| false` for scalars. The hostPath and hostNetwork policies use this and have autogen disabled in favour of explicit Pod/controller rules.

The five "shape" policies (P1–P5) use declarative `pattern` rules and rely on autogen; the two deny-based policies (P7, P8) spell out controller kinds explicitly.

## Installation without Helm

The dev host is Windows Git Bash with no `helm` (a Linux binary dropped in gives `Exec format error`), so Kyverno is installed from its pinned release manifest with a **server-side** apply. Client-side apply fails because the CRDs exceed the annotation size limit:

```
metadata.annotations: Too long: may not be more than 262144 bytes
```

```bash
curl -fsSL -o /tmp/kyverno-install.yaml \
  https://github.com/kyverno/kyverno/releases/download/v1.19.1/install.yaml
kubectl apply --server-side --force-conflicts -f /tmp/kyverno-install.yaml
kubectl -n kyverno wait --for=condition=Ready pod --all --timeout=300s
```

That's wrapped in `scripts/kyverno-install.sh` (or `make kyverno`) so it's reproducible. Kyverno's `clusterpolicies.kyverno.io/v1` API prints a deprecation notice pointing at the newer CEL-based `policies.kyverno.io` APIs; it works today and the manifests are shaped to migrate cleanly later.

## Audit → enforce in practice

Measure first. With policies in Audit mode, submit bad manifests and read the reports instead of fighting rejections:

```
$ kubectl -n securecloud get policyreport
Deployment/bad-run-as-root   PASS 14  FAIL 2
Pod/bad-latest-tag-...       PASS 15  FAIL 1
```

Then flip to Enforce and the same objects are refused at admission. The running SecureCloud workloads evaluate fully compliant (`PASS 16 / FAIL 0`) and never notice the switch.

## Proving the policies: bad in, good in

`scripts/policy-test.sh` applies the enforce overlay and submits every manifest with `--dry-run=server` — admission runs, nothing is persisted:

```
MANIFEST                          EXPECTED   GOT        RESULT
bad/01-run-as-root.yaml           rejected   rejected   PASS
bad/02-privilege-escalation.yaml  rejected   rejected   PASS
bad/03-capabilities.yaml          rejected   rejected   PASS
bad/04-no-resource-limits.yaml    rejected   rejected   PASS
bad/05-latest-tag.yaml            rejected   rejected   PASS
bad/06-unapproved-registry.yaml   rejected   rejected   PASS
bad/07-host-network.yaml          rejected   rejected   PASS
bad/08-host-path.yaml             rejected   rejected   PASS
good/compliant-deployment.yaml    accepted   accepted   PASS
=> ALL POLICY TESTS PASSED
```

Eight violations, one passing control. That control matters: it catches the classic failure mode where a policy is *too* strict and starts blocking legitimate workloads.

## RBAC: least privilege, verified

The app is a plain client/server stack and never calls the Kubernetes API, so its pods don't need a service-account token at all:

| Item | State |
| --- | --- |
| ServiceAccounts | one per workload, never `default` |
| ClusterRoleBindings | none |
| Namespaced Role | `securecloud-app`, **empty** |
| `automountServiceAccountToken` | **`false`** |
| `cluster-admin` | not used |

Setting the flag only affects *new* pods, so the workloads were rolled and the old pods retired — every running pod now has no `kube-api-access` volume. `scripts/rbac-review.sh` asserts all of it (`no ClusterRoleBinding`, `no cluster-admin`, `automount=false`, and `kubectl auth can-i` denying six sensitive verbs for each SA) and exits non-zero if any check fails — safe to use as a gate.

```
RBAC REVIEW PASSED (least privilege confirmed)
```

## Scripts & Makefile

| Command | Purpose |
| --- | --- |
| `scripts/kyverno-install.sh` / `make kyverno` | Install Kyverno (pinned, server-side apply) |
| `make policies-audit` | Apply policies in Audit mode; read `PolicyReport`s |
| `make policies-enforce` | Flip all policies to Enforce |
| `scripts/policy-test.sh` / `make policy-test` | Bad manifests rejected, good manifest accepted |
| `scripts/rbac-review.sh` / `make rbac-review` | Assert least-privilege RBAC |

Final state: `policy-test.sh`, `rbac-review.sh`, and the Phase 3 `verify.sh` (T1–T6) all pass together — the app is still fully functional under enforcement.

## Takeaways

- **A wiki rule is a wish; an admission policy is a gate.** Policy as code makes the baseline unskippable and reviewable.
- **Audit before enforce.** Reports let you find what you'd break before you break it.
- **Test both directions.** Every rule needs a manifest that must fail *and* one that must pass, or "too strict" policies slip through.
- **Null-safe by default.** Admission rules that error on missing fields deny innocent pods; guard every optional key.
- **RBAC is part of the perimeter.** No token beats a restricted token — and an empty Role is cheap insurance if one ever appears.
- **Exceptions should be objects, not edits.** The right answer to a real hostPath need is a narrow `PolicyException`, not weakening the rule for everyone.

Phase 4 makes the cluster enforce the properties Phase 3 merely configured — and proves it, violation by violation.
