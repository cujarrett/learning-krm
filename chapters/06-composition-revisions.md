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

Each XR controls how it handles new revisions via `spec.compositionUpdatePolicy`:

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
  compositionUpdatePolicy: Manual          # Pin to current revision
  compositionRevisionRef:
    name: webservice-composition-abc12     # The specific revision name
  image: nginx:alpine
  replicas: 2
```

Setting `Manual` without specifying `compositionRevisionRef` means the XR pins to whatever revision it reconciled against most recently.

---

## Selecting a Composition by Labels

Instead of naming a Composition directly with `compositionRef`, XRs can select one dynamically using labels:

```yaml
spec:
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

This pattern lets you run `channel: stable` and `channel: canary` Compositions side by side and migrate XRs by changing the label selector rather than the `compositionRef` name.

---

## Hands-On: Create and Inspect Revisions

```bash
mkdir -p practice/ch06
```

### Step 1: Apply the WebService XRD and a Starting Composition

If Chapter 03 and 04 resources are gone:

```bash
kubectl apply -f practice/ch03/webservice-xrd.yaml
kubectl apply -f practice/ch04/function-pat.yaml
kubectl get xrds --watch
# Wait for ESTABLISHED=True, then Ctrl+C
kubectl get functions.pkg.crossplane.io --watch
# Wait for HEALTHY=True, then Ctrl+C
```

Apply the Chapter 04 composition (this becomes Revision 1):

```bash
kubectl apply -f practice/ch04/webservice-composition.yaml
```

### Step 2: Inspect the First Revision

```bash
kubectl get compositionrevisions
```

Expected output:

```
NAME                                REVISION   XR-KIND      READY   AGE
webservice-composition-<hash>       1          WebService   True    10s
```

Get the full details:

```bash
kubectl describe compositionrevision -l crossplane.io/composition-name=webservice-composition
```

Look for:
- `Revision: 1`
- `Current: true` — this is the revision currently in use

### Step 3: Create a WebService with Manual Update Policy

Create `practice/ch06/pinned-webservice.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: pinned-svc
  namespace: default
spec:
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
kubectl get webservice pinned-svc -n default -o jsonpath='{.spec.compositionRevisionRef.name}'
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

Make a small change to `practice/ch04/webservice-composition.yaml`. Find the ConfigMap resource base section and add a label:

```yaml
      # ─── ConfigMap ─────────────────────────────────────────────────────────
      - name: config
        base:
          apiVersion: v1
          kind: ConfigMap
          metadata:
            labels:
              revision: "2"           # ← add this line
          data: {}
```

Apply the updated Composition:

```bash
kubectl apply -f practice/ch04/webservice-composition.yaml
```

### Step 6: Inspect the Two Revisions

```bash
kubectl get compositionrevisions --sort-by=.spec.revision
```

Expected:

```
NAME                                REVISION   XR-KIND      READY   AGE
webservice-composition-<hash1>      1          WebService   True    5m
webservice-composition-<hash2>      2          WebService   True    10s
```

Revision 2 is now `Current: true`.

### Step 7: Verify the Manual XR Did NOT Update

```bash
kubectl get webservice pinned-svc -n default -o jsonpath='{.spec.compositionRevisionRef.name}'
# Should still point to revision 1 hash
```

Check the ConfigMap resource created from `pinned-svc` — it should NOT have the `revision: "2"` label:

```bash
kubectl get configmap -n default -l app=pinned-svc -o yaml | grep -A 5 "labels:"
```

### Step 8: Verify the Automatic XR DID Update

```bash
kubectl get webservice auto-svc -n default -o jsonpath='{.spec.compositionRevisionRef.name}'
# Should point to revision 2 hash
```

Check its ConfigMap for the new label:

```bash
kubectl get configmap -n default -l app=auto-svc -o yaml | grep -A 5 "labels:"
# Should show "revision: 2"
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
  -p "{\"spec\":{\"compositionRevisionRef\":{\"name\":\"$REV2\"}}}"
```

Watch it reconcile:

```bash
kubectl get events -n default --sort-by=.metadata.creationTimestamp | tail -10
```

Verify the ConfigMap now has the revision 2 label:

```bash
kubectl get configmap -n default -l app=pinned-svc -o yaml | grep "revision:"
# Should now show revision: "2"
```

### Step 10: Clean Up

```bash
kubectl delete -f practice/ch06/pinned-webservice.yaml
kubectl delete -f practice/ch06/auto-webservice.yaml
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
