# Chapter 10: Write a Composition Function in Go

> **You will build:** A custom gRPC Composition Function in Go — built, loaded into minikube, and tested locally.

You have used `function-go-templating` and `function-patch-and-transform` — pre-built functions created by the Crossplane community. This chapter shows you how to build your own.

A Composition Function is a **Go program that implements a single gRPC handler**: `RunFunction`. Crossplane calls it during each reconcile with the observed state of the XR and composed resources. Your function returns the desired state — the resources that should exist after this reconcile.

This is where your Go knowledge from `~/Developer/learning-go/101` becomes directly applicable.

---

## How a Function Fits In

```
Crossplane Controller
        │
        │  gRPC call: RunFunctionRequest
        │  {
        │    observed.composite.resource  ← the XR (spec, metadata, status)
        │    observed.composed.resources  ← existing child resources
        │    input                        ← the YAML under your Composition step's input:
        │  }
        ▼
  Your Go Function Pod
  func (f *Function) RunFunction(ctx, req) (*RunFunctionResponse, error) {
        │
        │  1. Read the XR: req.Observed.Composite.Resource
        │  2. Build the resources you want to exist
        │  3. Return them in the response
        │
  }
        │
        │  gRPC response: RunFunctionResponse
        │  {
        │    desired.composed.resources  ← resources to create/update/keep
        │    desired.composite.resource  ← status fields to write back to XR
        │  }
        ▼
Crossplane diffs against cluster state and applies changes
```

---

## The Function SDK

The Crossplane team publishes a Go SDK that handles all the gRPC plumbing:

```
github.com/crossplane/function-sdk-go
```

The SDK provides:
- `request.GetObservedCompositeResource(req)` — parse the XR from the request into a typed struct
- `response.SetDesiredComposedResources(rsp, resources)` — set the resources to create
- `response.Fatal(rsp, err)` — mark the reconcile as failed with an error
- `fnv1.RunFunctionRequest` / `fnv1.RunFunctionResponse` — the protobuf types

You write the `RunFunction` handler. The SDK handles the gRPC server startup, health checks, and TLS.

---

## The Function Lifecycle

```
1. Write the Go code  (fn.go)
2. crossplane xpkg build  →  OCI image containing your compiled binary
3. minikube image load    →  image available inside minikube (no registry needed)
4. kubectl apply function.yaml  →  tells Crossplane about the function
5. Crossplane runs your image as a pod inside crossplane-system
6. kubectl apply composition.yaml  →  Composition references your function by name
7. kubectl apply xr.yaml  →  XR triggers the pipeline → RunFunction is called
```

---

## Prerequisites

Install the Crossplane CLI (needed for `crossplane xpkg build`):

```bash
brew install crossplane/tap/crossplane
crossplane version
```

Verify Go 1.21+:

```bash
go version
```

---

## Hands-On: Build a Function That Creates a ConfigMap

You will write a function that reads three fields from an XR — `spec.appName`, `spec.environment`, `spec.owner` — and creates a ConfigMap containing those values plus a computed field. The goal is to learn the function structure, not complex business logic.

```bash
mkdir -p practice/ch09
```

### Step 1: Scaffold the Function

```bash
cd practice/ch09
crossplane beta xpkg init runtime function-template-go \
  --directory function-app-config
cd function-app-config
```

The scaffold creates:

```
function-app-config/
  fn.go           ← your RunFunction handler — edit this
  fn_test.go      ← unit tests using the SDK test helpers
  main.go         ← starts the gRPC server (rarely need to touch)
  go.mod / go.sum
  package/
    crossplane.yaml
  Dockerfile
```

### Step 2: Define the XRD

From the `practice/ch09/` folder (not inside function-app-config), create `practice/ch09/appconfig-xrd.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: appconfigs.platform.example.io
spec:
  scope: Namespaced
  group: platform.example.io
  names:
    kind: AppConfig
    plural: appconfigs
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
            properties:
              appName:
                type: string
              environment:
                type: string
                default: production
                enum: [development, staging, production]
              owner:
                type: string
            required:
            - appName
            - owner
          status:
            type: object
            properties:
              configMapName:
                type: string
```

### Step 3: Write the Function Handler

Replace the contents of `practice/ch09/function-app-config/fn.go` with:

```go
package main

import (
	"context"
	"fmt"

	"github.com/crossplane/function-sdk-go/errors"
	"github.com/crossplane/function-sdk-go/logging"
	fnv1 "github.com/crossplane/function-sdk-go/proto/v1"
	"github.com/crossplane/function-sdk-go/request"
	"github.com/crossplane/function-sdk-go/resource"
	"github.com/crossplane/function-sdk-go/response"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime"
)

// Function implements the Crossplane function gRPC server.
type Function struct {
	fnv1.UnimplementedFunctionRunnerServiceServer
	log logging.Logger
}

// RunFunction is called by Crossplane on every reconcile of an AppConfig XR.
func (f *Function) RunFunction(_ context.Context, req *fnv1.RunFunctionRequest) (*fnv1.RunFunctionResponse, error) {
	f.log.Info("Running function", "tag", req.GetMeta().GetTag())

	// Build the base response object.
	rsp := response.To(req, response.DefaultTTL)

	// ── 1. Read the observed XR ────────────────────────────────────────────
	oxr, err := request.GetObservedCompositeResource(req)
	if err != nil {
		return nil, errors.Wrap(err, "cannot get observed composite resource")
	}

	// ── 2. Extract fields from spec ────────────────────────────────────────
	appName, err := oxr.Resource.GetString("spec.appName")
	if err != nil {
		response.Fatal(rsp, errors.Wrap(err, "cannot get spec.appName"))
		return rsp, nil
	}

	environment, err := oxr.Resource.GetString("spec.environment")
	if err != nil {
		environment = "production"
	}

	owner, err := oxr.Resource.GetString("spec.owner")
	if err != nil {
		response.Fatal(rsp, errors.Wrap(err, "cannot get spec.owner"))
		return rsp, nil
	}

	namespace := oxr.Resource.GetNamespace()
	xrName := oxr.Resource.GetName()
	configMapName := fmt.Sprintf("%s-config", xrName)

	// ── 3. Build the ConfigMap ─────────────────────────────────────────────
	cm := &corev1.ConfigMap{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "v1",
			Kind:       "ConfigMap",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      configMapName,
			Namespace: namespace,
			Labels: map[string]string{
				"app":        appName,
				"env":        environment,
				"owner":      owner,
				"managed-by": "crossplane",
			},
		},
		Data: map[string]string{
			"APP_NAME":      appName,
			"ENVIRONMENT":   environment,
			"OWNER":         owner,
			"APP_FULL_NAME": fmt.Sprintf("%s-%s", appName, environment), // computed
		},
	}

	// ── 4. Convert to unstructured and register as a desired resource ──────
	cmObj, err := runtime.DefaultUnstructuredConverter.ToUnstructured(cm)
	if err != nil {
		return nil, errors.Wrap(err, "cannot convert ConfigMap to unstructured")
	}

	desired := map[resource.Name]*resource.DesiredComposed{
		"app-configmap": {
			Resource: &resource.Composed{
				Unstructured: unstructured.Unstructured{Object: cmObj},
			},
		},
	}

	if err := response.SetDesiredComposedResources(rsp, desired); err != nil {
		return nil, errors.Wrap(err, "cannot set desired resources")
	}

	f.log.Info("Desired ConfigMap set", "name", configMapName)
	return rsp, nil
}
```

Pull dependencies:

```bash
cd practice/ch09/function-app-config
go mod tidy
go build ./...
```

Fix any compile errors before continuing. Common issue: missing `k8s.io/api` or `k8s.io/apimachinery` — `go mod tidy` resolves these automatically.

### Step 4: Run the Unit Tests

The scaffold's `fn_test.go` already has test structure. Replace its test content with:

```go
func TestRunFunction(t *testing.T) {
	f := &Function{log: logging.NewNopLogger()}

	req := &fnv1.RunFunctionRequest{
		Meta: &fnv1.RequestMeta{Tag: "test"},
		Observed: &fnv1.State{
			Composite: &fnv1.Resource{
				Resource: resource.MustStructJSON(`{
					"apiVersion": "platform.example.io/v1alpha1",
					"kind": "AppConfig",
					"metadata": {"name": "payments", "namespace": "team-alpha"},
					"spec": {
						"appName": "payments-service",
						"environment": "production",
						"owner": "team-alpha"
					}
				}`),
			},
		},
	}

	rsp, err := f.RunFunction(context.Background(), req)
	if err != nil {
		t.Fatalf("RunFunction error: %v", err)
	}
	if len(rsp.GetDesired().GetResources()) != 1 {
		t.Errorf("expected 1 desired resource, got %d", len(rsp.GetDesired().GetResources()))
	}
}
```

Run them:

```bash
go test ./... -v
```

The test runs entirely in-process — no cluster, no Docker. This is one of the biggest advantages of the Go function model: fast, reliable unit tests.

### Step 5: Build the OCI Image

```bash
# From inside practice/ch09/function-app-config/
crossplane xpkg build \
  --package-root=package \
  --embed-runtime-image=runtime \
  --output=../function-app-config.xpkg
```

This produces `practice/ch09/function-app-config.xpkg`.

### Step 6: Load Into Minikube

```bash
minikube image load practice/ch09/function-app-config.xpkg --profile crossplane

# Confirm the image is available inside minikube
minikube ssh --profile crossplane -- crictl images | grep function-app-config
```

### Step 7: Install the Function

Create `practice/ch09/function-install.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-app-config
spec:
  package: function-app-config:latest
  packagePullPolicy: IfNotPresent
```

Apply and wait:

```bash
kubectl apply -f practice/ch09/function-install.yaml
kubectl get functions.pkg.crossplane.io --watch
# Wait for function-app-config HEALTHY=True, then Ctrl+C
```

If the function stays unhealthy:

```bash
kubectl describe function function-app-config
kubectl get pods -n crossplane-system | grep app-config
kubectl logs <pod-name> -n crossplane-system
```

### Step 8: Write the Composition

Create `practice/ch09/appconfig-composition.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: appconfig-composition
spec:
  compositeTypeRef:
    apiVersion: platform.example.io/v1alpha1
    kind: AppConfig
  mode: Pipeline
  pipeline:
  - step: create-config
    functionRef:
      name: function-app-config
```

Apply both:

```bash
kubectl apply -f practice/ch09/appconfig-xrd.yaml
kubectl apply -f practice/ch09/appconfig-composition.yaml
kubectl get xrds --watch
# Wait for appconfigs.platform.example.io ESTABLISHED, then Ctrl+C
```

### Step 9: Test End to End

Create `practice/ch09/test-appconfig.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: AppConfig
metadata:
  name: payments
  namespace: default
spec:
  compositionRef:
    name: appconfig-composition
  appName: payments-service
  environment: production
  owner: team-alpha
```

Apply and watch:

```bash
kubectl apply -f practice/ch09/test-appconfig.yaml
kubectl get appconfigs,configmaps -n default --watch
# Ctrl+C when payments-config appears
```

Inspect what your Go function created:

```bash
kubectl get configmap payments-config -n default -o yaml
```

Expected `data`:

```yaml
data:
  APP_FULL_NAME: payments-service-production
  APP_NAME: payments-service
  ENVIRONMENT: production
  OWNER: team-alpha
```

`APP_FULL_NAME` was computed in your Go code — this is logic that no YAML patch syntax can express.

### Step 10: Iterate — Add a New Field Without Restarting the Cluster

Open `fn.go` and add one more key to the `Data` map:

```go
"REPLICA_HINT": fmt.Sprintf("%d", replicaHint(environment)),
```

Add the helper function at the bottom of `fn.go`:

```go
// replicaHint returns a sensible starting replica count for an environment.
func replicaHint(env string) int {
	switch env {
	case "production":
		return 3
	case "staging":
		return 2
	default:
		return 1
	}
}
```

Rebuild and reload:

```bash
go build ./...
crossplane xpkg build \
  --package-root=package \
  --embed-runtime-image=runtime \
  --output=../function-app-config.xpkg

minikube image load practice/ch09/function-app-config.xpkg --profile crossplane

# Restart the function pod to pick up the new image
kubectl rollout restart deployment \
  -n crossplane-system \
  -l pkg.crossplane.io/revision=function-app-config
```

Delete and re-create the XR to trigger a fresh reconcile:

```bash
kubectl delete -f practice/ch09/test-appconfig.yaml
kubectl apply  -f practice/ch09/test-appconfig.yaml
kubectl get configmap payments-config -n default -o yaml
# data should now include REPLICA_HINT: "3"
```

---

## What You Built

- A Go function implementing `RunFunction` with the Crossplane function SDK
- XR field access using `GetString` on the observed composite resource
- Kubernetes object construction in Go, converted to unstructured and added to the desired state
- Unit tests that run entirely without a cluster
- OCI image packaging with `crossplane xpkg build` and local loading with `minikube image load`
- End-to-end test from XR apply through Crossplane pipeline to created ConfigMap

---

## Where to Go Next

| Topic | What to Explore |
|-------|----------------|
| **Read observed composed resources** | Use `request.GetObservedComposedResources(req)` to make decisions based on current cluster state |
| **`crossplane beta render`** | Run your entire Composition pipeline locally without a cluster for rapid iteration |
| **Push to a real registry** | Replace `minikube image load` with `docker push`; update `spec.package` to the registry URL |
| **Multi-step pipelines** | Add `function-auto-ready` as a second step to automate readiness detection |
| **Kustomize** | Learn standalone; use it to manage your XR claim YAML across dev/staging/production |

---

⬅️ [Back to README](../README.md)
