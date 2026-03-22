# Chapter 06: Composition Revisions

> **You will build:** Pin an XR to a specific Composition version and test a safe rollout.

Every time you update a Composition, Crossplane creates a **CompositionRevision** — an immutable snapshot of that Composition at a point in time. This gives you a safe rollout mechanism: new XR instances pick up the latest revision automatically, but existing XRs can be pinned to an older revision until you decide to migrate them.

---

## Why Revisions Exist

Imagine you have 50 teams each with a `WebService` XR. You update the Composition to change a default. Without revisions, all 50 XRs would re-reconcile immediately with the new Composition — potentially changing running workloads without team consent. With revisions you can:

- Publish a new Composition version
- Let new deployments automatically use it
- Let existing deployments keep running the old version
- Migrate teams one at a time by updating their `compositionRevisionRef`

---

## The Revision Model

```
  Composition "webservice-composition"
  (mutable — you edit this)
          │
          │ Crossplane creates an immutable snapshot on each change
          ▼
  CompositionRevision "webservice-composition-abc12"   ← revision 1 (oldest)
  CompositionRevision "webservice-composition-def34"   ← revision 2
  CompositionRevision "webservice-composition-ghi56"   ← revision 3 (latest)

  XR "my-api" ──── compositionRevisionRef: ghi56  (automatic: always latest)
  XR "legacy" ──── compositionRevisionRef: abc12  (manual: pinned to revision 1)
```

CompositionRevisions are **read-only**. You never edit them. You edit the Composition and Crossplane creates a new revision.

---

## XR Composition Update Policy

Each XR controls how it handles new revisions via `spec.crossplane.compositionUpdatePolicy`:

| Value | Behavior |
|-------|---------|
| `Automatic` | (default) XR always tracks the latest CompositionRevision. On every reconcile it uses the most recent revision. |
| `Manual` | XR stays on its current `compositionRevisionRef` until you explicitly update that field. |

Setting it on an individual XR:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: my-stable-api
  namespace: default
spec:
  crossplane:
    compositionUpdatePolicy: Manual          # Pin to current revision
    compositionRevisionRef:
      name: webservice-composition-abc12     # The specific revision name
  image: nginx:alpine
  replicas: 2
```

Setting `Manual` without specifying `compositionRevisionRef` means the XR pins to whatever revision it reconciled against most recently.

---

## Selecting a Composition by Labels

Instead of naming a Composition directly with `spec.crossplane.compositionRef`, XRs can select one dynamically using labels:

```yaml
spec:
  crossplane:
    compositionSelector:
      matchLabels:
        channel: stable        # Only use Compositions labelled channel=stable
```

You then label your Compositions:

```yaml
metadata:
  name: webservice-composition-v2
  labels:
    channel: stable
```

This pattern lets you run `channel: stable` and `channel: canary` Compositions side by side and migrate XRs by changing the label selector rather than the `spec.crossplane.compositionRef` name.

---

## Hands-On: Create and Inspect Revisions

```bash
mkdir -p practice/ch06
```

### Step 1: Apply the WebService XRD and a Starting Composition

If Chapter 03 and 04 resources are gone:

```bash
kubectl apply -f practice/ch03/webservice-xrd.yaml
kubectl apply -f practice/ch04/function-go-templating.yaml
kubectl get xrds --watch
# Wait for ESTABLISHED=True, then Ctrl+C
kubectl get functions.pkg.crossplane.io --watch
# Wait for HEALTHY=True, then Ctrl+C
```

This chapter relies on `webservice-composition` starting at **Revision 1**. If you've worked through earlier chapters, previous applies will have already incremented the revision counter. Delete the Composition (and its revisions) first so you get a clean slate:

```bash
kubectl delete composition webservice-composition --ignore-not-found
kubectl delete composition webservice-advanced-composition --ignore-not-found
```

This chapter edits the Composition to create a second revision. Copy it into `practice/ch06/` first so the ch04 file stays untouched:

```bash
cp practice/ch04/webservice-composition.yaml practice/ch06/webservice-composition.yaml
kubectl apply -f practice/ch06/webservice-composition.yaml
```

### Step 2: Inspect the First Revision

```bash
kubectl get compositionrevisions
```

Expected output:

```
NAME                                REVISION   XR-KIND      XR-APIVERSION                    AGE
webservice-composition-<hash>       1          WebService   platform.example.io/v1alpha1     10s
```

Get the full details:

```bash
kubectl describe compositionrevision -l crossplane.io/composition-name=webservice-composition
```

Look for:
- `Revision: 1` in the Spec section

### Step 3: Create a WebService with Manual Update Policy

Create `practice/ch06/pinned-webservice.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: pinned-svc
  namespace: default
spec:
  crossplane:
    compositionUpdatePolicy: Manual
  image: nginx:alpine
  replicas: 1
  port: 80
  environment: production
```

Apply it:

```bash
kubectl apply -f practice/ch06/pinned-webservice.yaml
```

Check what revision it was pinned to automatically:

```bash
kubectl get webservice pinned-svc -n default -o jsonpath='{.spec.crossplane.compositionRevisionRef.name}'
```

Crossplane fills in the `compositionRevisionRef` with the current latest revision at the time the XR was first reconciled. Because the policy is `Manual`, it will now stay on that revision even when you publish a new one.

### Step 4: Create a Second WebService with Automatic Policy (the Default)

Create `practice/ch06/auto-webservice.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: auto-svc
  namespace: default
spec:
  crossplane:
    compositionUpdatePolicy: Automatic
  image: nginx:alpine
  replicas: 1
  port: 80
  environment: staging
```

Apply it:

```bash
kubectl apply -f practice/ch06/auto-webservice.yaml
```

### Step 5: Update the Composition — Create Revision 2

Edit `practice/ch06/webservice-composition.yaml` (the copy you made in Step 1 — not the ch04 original). Find the Deployment's `metadata.labels` block and add a label:

```yaml
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            annotations:
              gotemplating.fn.crossplane.io/composition-resource-name: deployment
            labels:
              app: {{ $name }}
              environment: {{ $spec.environment | default "production" }}
              revision: "2"           # ← add this line
```

Apply the updated Composition:

```bash
kubectl apply -f practice/ch06/webservice-composition.yaml
```

### Step 6: Inspect the Two Revisions

```bash
kubectl get compositionrevisions --sort-by=.spec.revision
```

Expected:

```
NAME                                REVISION   XR-KIND      XR-APIVERSION                    AGE
webservice-composition-<hash1>      1          WebService   platform.example.io/v1alpha1     5m
webservice-composition-<hash2>      2          WebService   platform.example.io/v1alpha1     10s
```

Revision 2 is the latest — new XRs with `Automatic` policy will use it.

### Step 7: Verify the Manual XR Did NOT Update

```bash
kubectl get webservice pinned-svc -n default -o jsonpath='{.spec.crossplane.compositionRevisionRef.name}'
# Should still point to the older revision hash
```

Check the Deployment created from `pinned-svc` — it should NOT have the `revision: "2"` label:

```bash
kubectl get deployment pinned-svc -n default -o jsonpath='{.metadata.labels}'
# Should not contain revision:2
```

### Step 8: Verify the Automatic XR DID Update

```bash
kubectl get webservice auto-svc -n default -o jsonpath='{.spec.crossplane.compositionRevisionRef.name}'
# Should point to the latest revision hash
```

Check its Deployment for the new label:

```bash
kubectl get deployment auto-svc -n default -o jsonpath='{.metadata.labels}'
# Should contain revision:2
```

### Step 9: Manually Migrate the Pinned XR to Revision 2

Get the name of revision 2:

```bash
REV2=$(kubectl get compositionrevisions \
  --sort-by=.spec.revision \
  -o jsonpath='{.items[-1].metadata.name}')
echo $REV2
```

Patch the pinned XR to use revision 2:

```bash
kubectl patch webservice pinned-svc -n default \
  --type merge \
  -p "{\"spec\":{\"crossplane\":{\"compositionRevisionRef\":{\"name\":\"$REV2\"}}}}"
```

Watch it reconcile:

```bash
kubectl get events -n default --sort-by=.metadata.creationTimestamp | tail -10
```

Verify the Deployment now has the revision 2 label:

```bash
kubectl get deployment pinned-svc -n default -o jsonpath='{.metadata.labels}'
# Should now contain revision:2
```

### Step 10: Clean Up

```bash
# XRs (deletes all composed resources via cascade)
kubectl delete -f practice/ch06/pinned-webservice.yaml --ignore-not-found
kubectl delete -f practice/ch06/auto-webservice.yaml --ignore-not-found

# Composition applied in this chapter (also deletes its revisions)
kubectl delete composition webservice-composition --ignore-not-found
```

---

## What You Built

- Created two WebService XRs with different update policies: `Manual` and `Automatic`
- Updated the Composition and watched Crossplane auto-create Revision 2
- Verified the `Manual` XR stayed on Revision 1 while the `Automatic` XR upgraded immediately
- Manually migrated the pinned XR to Revision 2 by updating `compositionRevisionRef`

Composition Revisions are the safety net for platform changes at scale. The next chapter covers namespace isolation — restricting which teams can create XRs in which namespaces.

---

➡️ [Chapter 07: Providers & Managed Resources](07-providers.md)
