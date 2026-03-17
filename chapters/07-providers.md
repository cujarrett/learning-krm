# Chapter 07: Providers & Managed Resources

> **You will build:** A `BugReport` XRD backed by `provider-github` that files GitHub Issues from a `kubectl apply`.

Up to now, every resource Crossplane has created has been a Kubernetes resource — Deployments, Services, ConfigMaps. This chapter introduces **Providers**, which extend Crossplane to manage **external** systems: GitHub, AWS, GCP, Helm releases, or anything with an API.

The GitHub provider is the perfect first provider: no cloud credentials, no cost, and you can watch Crossplane create real GitHub issues and repositories from Kubernetes YAML.

---

## What is a Provider?

A Provider is a Crossplane package that ships two things:

1. **CRDs** — new `kind:` types for external resources (e.g., `Issue`, `Repository`)
2. **A controller** — a reconciler that watches those CRDs and calls the external API to make reality match spec

```
                     Provider package installed
                             │
              ┌──────────────┴──────────────┐
              │                             │
         CRDs installed              Controller running
    (Issue, Repository,           (watches for Issue CRs,
      IssueLabel, ...)             calls GitHub REST API)
              │                             │
              ▼                             ▼
   You create:                    Controller creates/updates/deletes
   kind: Issue                    the issue on github.com
   spec.forProvider:
     repository: my-repo
     title: "bug: ..."
```

The pattern is always the same regardless of the provider:

```
Provider install  →  ProviderConfig (auth)  →  Managed Resource (the actual thing)
```

---

## Managed Resources vs Compositions

| | Managed Resource | Composition |
|--|-----------------|-------------|
| **What it is** | A CRD for one external resource | A platform API that may render many resources |
| **Created by** | Provider package | You (Chapter 04) |
| **Renders** | One real external thing | Kubernetes resources OR Managed Resources |
| **Example** | `kind: Issue` → one GitHub issue | `kind: BugReport` → one `Issue` MR via template |

A Composition's Go template can render Managed Resource YAML just as easily as it renders Deployment YAML — the template produces the spec, the provider controller creates the real thing.

---

## Upbound Marketplace

Upbound hosts provider packages at `marketplace.upbound.io`. Browse to see what's available:

- `provider-github` — Repositories, Issues, Labels, Teams
- `provider-kubernetes` — create K8s resources in other clusters
- `provider-helm` — manage Helm releases as CRDs
- `provider-aws`, `provider-gcp`, `provider-azure` — cloud infrastructure

Every provider page shows the package reference you paste into the `Provider` manifest.

---

## Hands-On: GitHub Issues From Kubernetes YAML

You will install `provider-github`, authenticate with a Personal Access Token, create a GitHub Issue directly as a Managed Resource, and then wrap it in a `BugReport` XRD so any developer can file a standardised issue with a single YAML apply.

```bash
mkdir -p practice/ch07
```

### Step 1: Create a GitHub Personal Access Token

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**
2. Click **Generate new token (classic)**
3. Give it a descriptive name: `crossplane-learning`
4. Set expiry (7 days is fine for learning)
5. Select only the scopes you need:
   - `public_repo` — create/edit issues on public repositories
   - `repo` — if you plan to use private repositories
6. Click **Generate token** and copy the value — you will not see it again

### Step 2: Store the Token in a Kubernetes Secret

The Upbound GitHub provider expects the token in a JSON object:

```bash
kubectl create secret generic github-credentials \
  --namespace crossplane-system \
  --from-literal=credentials='{"token":"ghp_YOUR_TOKEN_HERE"}'
```

Verify it was created (do NOT print the value):

```bash
kubectl get secret github-credentials -n crossplane-system
```

### Step 3: Install the GitHub Provider

Check the current version at `marketplace.upbound.io/providers/upbound/provider-github`, then create `practice/ch07/provider-github.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-github
spec:
  package: xpkg.upbound.io/upbound/provider-github:v0.3.0
```

> **Version note:** Replace `v0.3.0` with the latest version shown on the Upbound Marketplace. The installation process won't change.

Apply it and wait for healthy:

```bash
kubectl apply -f practice/ch07/provider-github.yaml
kubectl get providers.pkg.crossplane.io --watch
# Wait for HEALTHY=True, INSTALLED=True, then Ctrl+C
```

While it installs, check what CRDs it added:

```bash
kubectl get crds | grep github.upbound.io
```

You should see entries for `issues.github.upbound.io`, `repositories.github.upbound.io`, `issuelabels.github.upbound.io`, and others.

### Step 4: Create the ProviderConfig

A `ProviderConfig` tells the provider controller which credentials to use. Create `practice/ch07/provider-config.yaml`:

```yaml
apiVersion: github.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: github-credentials
      key: credentials
```

The name `default` is special — any Managed Resource without an explicit `providerConfigRef` uses the config named `default`.

```bash
kubectl apply -f practice/ch07/provider-config.yaml
```

### Step 5: Create a GitHub Issue Directly as a Managed Resource

This is a raw Managed Resource — no XRD, no Composition. You are talking directly to the provider.

Create `practice/ch07/test-issue.yaml`. Replace `YOUR_GITHUB_USERNAME` and `YOUR_REPO_NAME` with a real repo you own (or your `learning-crossplane` repo if it is on GitHub):

```yaml
apiVersion: github.upbound.io/v1alpha1
kind: Issue
metadata:
  name: crossplane-test-issue
spec:
  forProvider:
    repository: YOUR_REPO_NAME  # just the repo name, not the full URL
    title: "Test: Issue created by Crossplane"
    body: |
      This issue was created automatically by Crossplane running on minikube.

      **How:**
      - Kubernetes resource: `kind: Issue` from `provider-github`
      - Applied with: `kubectl apply -f test-issue.yaml`
      - Reconciler: `provider-github` controller called the GitHub REST API

      You can close this issue — it was created for learning purposes.
  providerConfigRef:
    name: default
```

Apply it:

```bash
kubectl apply -f practice/ch07/test-issue.yaml
```

Watch the Managed Resource reconcile:

```bash
kubectl get issues.github.upbound.io crossplane-test-issue -w
# Watch READY and SYNCED columns. Ctrl+C when both are True.
```

Go to GitHub — the issue should be open on your repository.

```bash
# See the full status, including the GitHub issue number written back
kubectl describe issue.github.upbound.io crossplane-test-issue
# Look at Status.atProvider: issueNumber, url
```

The provider wrote the GitHub issue number and URL back into the Managed Resource's status — the same observed → desired pattern as everything else in Crossplane.

### Step 6: Update the Issue — Edit the Title

```bash
kubectl patch issue.github.upbound.io crossplane-test-issue \
  --type merge \
  -p '{"spec":{"forProvider":{"title":"Test: Issue created by Crossplane (updated)"}}}'
```

Refresh the GitHub issue page — the title should update within ~30 seconds.

### Step 7: Delete the Issue

```bash
kubectl delete issue.github.upbound.io crossplane-test-issue
```

The controller will close (or delete) the issue on GitHub. Check your repository.

> By default, the GitHub provider closes issues on delete rather than hard-deleting them since GitHub does not support issue deletion via API.

---

## Wrapping a Managed Resource in a Composition

Direct Managed Resources are useful, but the real power is giving developers a clean XRD that hides the provider details. A `BugReport` XRD could create a GitHub issue with a standardised template so all bugs are filed consistently.

### Step 8: Define a `BugReport` XRD

Create `practice/ch07/bugreport-xrd.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: bugreports.platform.example.io
spec:
  group: platform.example.io
  names:
    kind: BugReport
    plural: bugreports
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [repository, title, description]
            properties:
              repository:
                type: string
                description: GitHub repository name (not the full URL)
              title:
                type: string
                description: Issue title — one line summary
              description:
                type: string
                description: Full description of the bug
              severity:
                type: string
                enum: [low, medium, high, critical]
                default: medium
              component:
                type: string
                description: Optional — which service or component is affected
          status:
            type: object
            properties:
              issueNumber:
                type: integer
              issueUrl:
                type: string
```

Apply it:

```bash
kubectl apply -f practice/ch07/bugreport-xrd.yaml
kubectl get xrds --watch
# Wait for ESTABLISHED=True, then Ctrl+C
```

### Step 9: Write the Composition

The Composition renders a `github.upbound.io/v1alpha1 Issue` using Go templates. The body is built from the XR's structured fields.

Create `practice/ch07/bugreport-composition.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: bugreport-composition
spec:
  compositeTypeRef:
    apiVersion: platform.example.io/v1alpha1
    kind: BugReport
  mode: Pipeline
  pipeline:
  - step: render-github-issue
    functionRef:
      name: function-go-templating
    input:
      apiVersion: gotemplating.fn.crossplane.io/v1beta1
      kind: GoTemplate
      source: Inline
      inline:
        template: |
          {{- $name     := .oxr.resource.metadata.name }}
          {{- $ns       := .oxr.resource.metadata.namespace }}
          {{- $spec     := .oxr.resource.spec }}
          {{- $severity := $spec.severity | default "medium" }}
          {{- $severityEmoji := "" }}
          {{- if eq $severity "critical" }}{{- $severityEmoji = "🔴" }}
          {{- else if eq $severity "high" }}{{- $severityEmoji = "🟠" }}
          {{- else if eq $severity "medium" }}{{- $severityEmoji = "🟡" }}
          {{- else }}{{- $severityEmoji = "🟢" }}
          {{- end }}
          ---
          apiVersion: github.upbound.io/v1alpha1
          kind: Issue
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            annotations:
              gotemplating.fn.crossplane.io/composition-resource-name: github-issue
          spec:
            forProvider:
              repository: {{ $spec.repository }}
              title: "{{ $severityEmoji }} [{{ $severity | upper }}] {{ $spec.title }}"
              body: |
                ## Bug Report

                **Severity:** {{ $severity | title }}
                {{- if $spec.component }}
                **Component:** {{ $spec.component }}
                {{- end }}
                **Reported by:** Crossplane / {{ $ns }}

                ---

                {{ $spec.description }}

                ---
                *This issue was filed automatically via the `BugReport` platform API.*
                *To update it, edit the `BugReport` resource in Kubernetes.*
            providerConfigRef:
              name: default
```

Apply it:

```bash
kubectl apply -f practice/ch07/bugreport-composition.yaml
```

### Step 10: File a Bug Report

Create `practice/ch07/my-bug-report.yaml`. Replace `YOUR_REPO_NAME`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: BugReport
metadata:
  name: login-timeout-bug
  namespace: default
spec:
  repository: YOUR_REPO_NAME
  title: "Login page times out after 30 seconds on mobile"
  severity: high
  component: auth-service
  description: |
    Users on mobile devices consistently see a timeout error when attempting
    to log in after the page has been open for ~30 seconds without interaction.

    Steps to reproduce:
    1. Open the login page on a mobile browser
    2. Wait 30 seconds without typing
    3. Enter credentials and submit

    Expected: login succeeds
    Actual: 504 Gateway Timeout
```

Apply it:

```bash
kubectl apply -f practice/ch07/my-bug-report.yaml
kubectl get bugreports -n default --watch
# Ctrl+C when READY=True
```

Go to GitHub — the issue should have appeared with the formatted body, severity label in the title, and component listed.

### Step 11: Escalate the Severity

```bash
kubectl patch bugreport login-timeout-bug -n default \
  --type merge \
  -p '{"spec":{"severity":"critical"}}'
```

The GitHub issue title should update to `🔴 [CRITICAL] Login page times out...` within ~30 seconds.

### Step 12: Clean Up

```bash
kubectl delete bugreport login-timeout-bug -n default
# GitHub issue will be closed

kubectl delete -f practice/ch07/bugreport-xrd.yaml
kubectl delete -f practice/ch07/provider-github.yaml
kubectl delete secret github-credentials -n crossplane-system
```

---

## What You Built

- Installed an Upbound Provider (`provider-github`) and understood what a Provider is
- Created a `ProviderConfig` to authenticate with the GitHub API
- Created a Managed Resource directly — a GitHub Issue — and watched Crossplane reconcile it
- Wrapped the Managed Resource in a `BugReport` XRD: developers file bugs with structured YAML, the Composition renders a formatted GitHub issue
- Observed that Managed Resources follow the same observed → desired model as everything else in Crossplane

**Extending this pattern:** The same approach works for any Upbound provider. Replace `provider-github` with `provider-helm` to manage Helm releases, `provider-kubernetes` to create resources in a remote cluster, or `provider-aws` when you are ready to provision real infrastructure. The Provider → ProviderConfig → Managed Resource pattern is always the same.

---

➡️ [Chapter 08: Namespace Isolation & RBAC](08-claims-and-rbac.md)
