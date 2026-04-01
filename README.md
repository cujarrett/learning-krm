# Learning KRM

A hands-on learning path for building declarative Kubernetes APIs — covering the foundational Kubernetes Resource Model (KRM), then two tools that build on it: **Crossplane** and **kro (Kube Resource Orchestrator)**.

All work is local: write YAML, apply to minikube, test, iterate. No CI/CD, no cloud account required.

---

## How to Use This Guide

Start with **Foundations** (one chapter, ~45 min) to understand the CRD → CR → Controller model that every tool in this repo builds on. Then pick a track — or do both.

```
Foundations
    └── 01: CRDs, CRs & Controllers
            │
            ├──── Crossplane Track (ch01–11)
            │         XRD → Composition → Functions → Providers
            │
            └──── kro Track (ch01–…)
                      ResourceGraphDefinition → CEL expressions
```

---

## Foundations

Tool-agnostic. Read this before any track.

| Chapter | Key Concepts | Est. Time |
|---------|--------------|-----------|
| [00 - YAML Primer](chapters/foundations/00-yaml-basics.md) | Objects, arrays, multi-line strings, reading compositions | ~20 min |
| [01 - CRDs, CRs & Controllers](chapters/foundations/01-crds-crs-controllers.md) | KRM three-layer model, reconciliation loop, spec/status contract, when to use each tool | ~45 min |

---

## Crossplane

[Crossplane](https://crossplane.io) lets you define custom platform APIs (XRDs) and wire them to real resources through Compositions and Functions — without writing a Go controller for the common case.

| Chapter | Key Concepts | Est. Time |
|---------|--------------|-----------|
| [01 - Setup & The Local Workflow](chapters/crossplane/01-setup-and-big-picture.md) | minikube, Helm, Crossplane install, starter project tour | ~45 min |
| [02 - Kubernetes Resources Refresher](chapters/crossplane/02-kubernetes-refresher.md) | GVK, CRDs, Deployments, Services, labels | ~30 min |
| [03 - XRDs — Composite Resource Definitions](chapters/crossplane/03-xrds.md) | XRD schema, versions, spec vs status | ~45 min |
| [04 - Compositions & Go Templating](chapters/crossplane/04-compositions.md) | Pipeline mode, how Functions work (gRPC protocol), P&T as a read-once reference, first Go template hands-on | ~45 min |
| [05 - Go Templating Deep Dive](chapters/crossplane/05-go-templating.md) | Sprig helpers, nil-safe `default dict`, status writeback, `define`/`include` blocks, conditional HPA | ~60 min |
| [06 - Composition Revisions](chapters/crossplane/06-composition-revisions.md) | CompositionRevision objects, Automatic vs Manual update policy | ~30 min |
| [07 - Providers & Managed Resources](chapters/crossplane/07-providers.md) | Upbound provider model, `provider-github`, ProviderConfig, direct MRs (`Branch` + `RepositoryFile`), `FeatureBranch` XRD pattern | ~45 min |
| [08 - Namespace Isolation & RBAC](chapters/crossplane/08-claims-and-rbac.md) | Namespaced XRs, Roles, RoleBindings, `kubectl auth can-i` | ~30 min |
| [09 - Advanced Go Templating](chapters/crossplane/09-advanced-go-templating.md) | HPA conditionals, nil-safe patterns, loops, MicroService XRD | ~60 min |
| [10 - Write a Composition Function in Go](chapters/crossplane/10-write-function-in-go.md) | Custom Go function, RunFunction handler, local image load | ~90 min |
| [11 - Functions with HTTP](chapters/crossplane/11-functions-with-http.md) | Outbound HTTP calls from RunFunction, graceful degradation, httptest unit tests | ~45 min |

---

## kro

[kro](https://kro.run) (Kube Resource Orchestrator) is an official Kubernetes SIG project. You write a `ResourceGraphDefinition` in YAML with CEL expressions — kro infers the dependency graph, generates a CRD, and runs a controller. No Go required.

| Chapter | Key Concepts | Est. Time |
|---------|--------------|-----------|
| [01 - Intro, Setup & Your First RGD](chapters/kro/01-intro-and-setup.md) | ResourceGraphDefinition, SimpleSchema, CEL expressions, dependency ordering, kro vs Crossplane | ~45 min |

---

## When to Use Each

| Situation | Reach for |
|-----------|-----------|
| Logic fits YAML/CEL, no Go needed, want simplest authoring | **kro** |
| Need to provision cloud resources (AWS, GCP, Azure, GitHub…) | **Crossplane** + provider package |
| Need complex logic YAML can't express | **Crossplane** + Go Function |
| Want automatic dependency ordering from expressions alone | **kro** |
| Need composition revision tracking and rollback | **Crossplane** |
| Prefer OpenAPI v3 schema precision | **Crossplane** |
| Prefer terse inline schema (`string \| default=nginx`) | **kro** |
| Resource lifecycle doesn't map to create/update/delete | Write your own controller |
| Need admission webhooks, real-time event handling, or complex state machines | Write your own controller |

kro and Crossplane are not mutually exclusive — you can run both in the same cluster and use kro for simple internal APIs while Crossplane manages cloud provider resources.

---

## Prerequisites

- **macOS** with [Homebrew](https://brew.sh) installed
- **Docker Desktop** running (minikube uses it as the driver)
- **Go familiarity**
- Basic `kubectl` knowledge (`get`, `apply`, `describe`, `logs`)

---

## How To Use This Guide

Each chapter follows a consistent pattern:

1. **Read** — work through the theory top to bottom (concepts, diagrams, examples)
2. **Hands-On** — follow the numbered steps, copy-paste the YAML, run the `kubectl` commands
3. **Test** — every hands-on ends with verification commands so you can see it working

Create your practice files under `practice/` as you go. Each track has its own subfolder:

```
practice/
  crossplane/
    ch01/    ← files you apply in Crossplane Chapter 01
    ch02/
    ...
  kro/
    ch01/
    ...
```

## The Local Development Loop

Every hands-on section follows the same rhythm:

```
  Write (YAML — XRD, Composition, RGD, etc.)
          │
          ▼
  kubectl apply -f <file.yaml>
          │
          ▼
  Controller reconciles
          │
          ▼
  kubectl get / describe / logs
          │
          └─ tweak and repeat
```

---

Start with [Foundations 00 →](chapters/foundations/00-yaml-basics.md)
