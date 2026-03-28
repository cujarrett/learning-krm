# Learning Crossplane

![logo](./logo.png)

A hands-on learning path for building Crossplane custom APIs from scratch — using **minikube on macOS**, **no cloud provider required**, and **Go templating** to generate Kubernetes resources.

All work is local: write YAML and Go templates, apply them to minikube, test, iterate.

No CI/CD, no cloud. Crossplane runs as pods inside minikube and creates standard Kubernetes resources (Deployments, Services, ConfigMaps). By Chapter 09 you will write your own Composition Function in Go.

---

## Chapters

| Chapter | Key Concepts | Est. Time |
|---------|--------------|-----------|
| [01 - Setup & The Local Workflow](chapters/01-setup-and-big-picture.md) | minikube, Helm, Crossplane install, starter project tour | ~45 min |
| [02 - Kubernetes Resources Refresher](chapters/02-kubernetes-refresher.md) | GVK, CRDs, Deployments, Services, labels | ~30 min |
| [03 - XRDs — Composite Resource Definitions](chapters/03-xrds.md) | XRD schema, versions, spec vs status | ~45 min |
| [04 - Compositions & Go Templating](chapters/04-compositions.md) | Pipeline mode, how Functions work (gRPC protocol), P&T as a read-once reference, first Go template hands-on | ~45 min |
| [05 - Go Templating Deep Dive](chapters/05-go-templating.md) | Sprig helpers, nil-safe `default dict`, status writeback, `define`/`include` blocks, conditional HPA | ~60 min |
| [06 - Composition Revisions](chapters/06-composition-revisions.md) | CompositionRevision objects, Automatic vs Manual update policy | ~30 min |
| [07 - Providers & Managed Resources](chapters/07-providers.md) | Upbound provider model, `provider-github`, ProviderConfig, direct MRs (`Branch` + `RepositoryFile`), `FeatureBranch` XRD pattern | ~45 min |
| [08 - Namespace Isolation & RBAC](chapters/08-claims-and-rbac.md) | Namespaced XRs, Roles, RoleBindings, `kubectl auth can-i` | ~30 min |
| [09 - Advanced Go Templating](chapters/09-advanced-go-templating.md) | HPA conditionals, nil-safe patterns, loops, MicroService XRD | ~60 min |
| [10 - Write a Composition Function in Go](chapters/10-write-function-in-go.md) | Custom Go function, RunFunction handler, local image load | ~90 min |
| [11 - Functions with HTTP](chapters/11-functions-with-http.md) | Outbound HTTP calls from RunFunction, graceful degradation, httptest unit tests | ~45 min |

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

Create your practice files under `practice/` as you go:

```
practice/
  ch01/    ← files you apply in Chapter 01
  ch02/    ← files you apply in Chapter 02
  ch03/
  ...
```

Work through chapters in order — each one builds on the previous.

## The Local Development Loop

Every chapter follows the same rhythm:

```
  Write (XRD, Composition, or Go template)
          │
          ▼
  kubectl apply -f <file.yaml>
          │
          ▼
  Crossplane reconciles
          │
          ▼
  kubectl get / describe / logs
          │
          └─ tweak and repeat
```

---

Start with [Chapter 01 →](chapters/01-setup-and-big-picture.md)
