# Chapter 05: Go Templating Deep Dive

> **You will build:** That same Composition rewritten using Go templates (`function-go-templating`).

You wrote your first Go template Composition in Chapter 04. This chapter covers the patterns you will reach for as templates grow more complex: Sprig helpers, nil-safe map access, writing status back to the XR, and reusable template blocks.

---

## Sprig: The Helper Library Built Into `function-go-templating`

[Sprig](http://masterminds.github.io/sprig/) provides over 100 helper functions available in every `function-go-templating` template. Here are the ones you will use most:

### `default` — Safe Fallback Values

```
{{ $spec.replicas | default 1 }}
{{ $spec.environment | default "production" }}
{{ $spec.image | default "nginx:alpine" }}
```

Without `default`, accessing a field that was not provided in the XR spec will render as `<no value>` in the YAML, which usually breaks the resource.

### `quote` — Wrap a Value in Quotes

```
{{ $val | quote }}
```

Use this in ConfigMap `data:` and anywhere the value must be a YAML string, not an unquoted scalar.

### `upper` / `lower` / `title` — Case Transforms

```
{{ $spec.environment | upper }}        # development → DEVELOPMENT
{{ $spec.name | lower }}
{{ $spec.tier | title }}               # web → Web
```

### `trunc` — Truncate to the Kubernetes Label Limit

Kubernetes label values must be 63 characters or fewer:

```
{{ $name | trunc 63 | trimSuffix "-" }}
```

The `trimSuffix` removes a trailing dash that `trunc` may leave if the name ends mid-word.

### `printf` — Format Strings

```
{{ printf "%s-%s" $ns $name }}
{{ printf "app/%s:v%s" $spec.image $spec.version }}
```

Equivalent to `fmt.Sprintf` in Go.

### `int` — Convert to Integer for Arithmetic

```
{{ $spec.replicas | int | mul 2 }}     # double the replica count
{{ add 1 ($spec.port | int) }}         # port + 1
```

### `has` — Check if a Value is in a List

```
{{- if has "production" (list "production" "staging") }}
  # only render in production or staging
{{- end }}
```

### `typeOf` — Inspect the Type of a Value

Useful when debugging templates that behave unexpectedly:

```
{{ typeOf $spec.replicas }}   # outputs "float64" (JSON numbers decode as float64)
```

---

## Nil-Safe Map Access With `default dict`

When an XR spec contains a nested object that the user might not provide, accessing its fields directly will panic the template engine:

```yaml
# XRD schema defines:
spec:
  autoscaling:
    enabled: bool
    minReplicas: int
    maxReplicas: int
```

If a user creates an XR without `spec.autoscaling`, then `$spec.autoscaling.enabled` panics with a nil pointer error.

**The fix: `default dict`**

```
{{- $autoscaling := $spec.autoscaling | default dict }}
```

`dict` produces an empty map `{}`. `default dict` returns that empty map if `$spec.autoscaling` is nil or missing. After this line, `$autoscaling.enabled` safely returns `<no value>` / falsy instead of panicking.

Full pattern:

```
{{- $autoscaling := $spec.autoscaling | default dict }}
{{- if $autoscaling.enabled }}
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ $name }}
  namespace: {{ $ns }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ $name }}
  minReplicas: {{ $autoscaling.minReplicas | default 2 }}
  maxReplicas: {{ $autoscaling.maxReplicas | default 10 }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
{{- end }}
```

Apply `default dict` to any optional nested object in `$spec` before accessing its sub-fields.

---

## Writing Status Back to the XR

Your Composition can write computed values back to the XR's `status` fields. This surfaces runtime data (ClusterIP, actual replica count, readiness) to anything watching the XR.

The special resource name `composite` in the desired composed resources targets the XR itself:

```yaml
template: |
  {{- $name := .oxr.resource.metadata.name }}
  {{- $ns   := .oxr.resource.metadata.namespace }}
  {{- $spec := .oxr.resource.spec }}
  ---
  apiVersion: apps/v1
  kind: Deployment
  ...
  ---
  # Write status back to the XR. The name "composite" is special.
  apiVersion: {{ .oxr.resource.apiVersion }}
  kind: {{ .oxr.resource.kind }}
  metadata:
    name: {{ $name }}
    namespace: {{ $ns }}
    annotations:
      gotemplating.fn.crossplane.io/composition-resource-name: composite
  status:
    endpoint: "http://{{ $name }}.{{ $ns }}.svc.cluster.local:8080"
    environment: {{ $spec.environment | default "production" }}
```

After applying the XR and waiting for reconcile:

```bash
kubectl get webservice my-webservice -n default -o jsonpath='{.status.endpoint}'
# http://my-webservice.default.svc.cluster.local:8080
```

You can define whatever status fields you want as long as they are declared in the XRD's `spec.versions[].schema.openAPIV3Schema.properties.status.properties` block.

---

## Reusable Template Blocks With `define` and `include`

When the same YAML fragment appears in multiple resources (standard labels, owner references, resource limits), you can define it once and include it:

```yaml
template: |
  {{- $name := .oxr.resource.metadata.name }}
  {{- $ns   := .oxr.resource.metadata.namespace }}
  {{- $spec := .oxr.resource.spec }}

  {{/* Define a reusable labels block. 'dot' is passed as context. */}}
  {{- define "commonLabels" }}
  labels:
    app: {{ .name }}
    environment: {{ .env | default "production" }}
    managed-by: crossplane
  {{- end }}

  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: {{ $name }}
    namespace: {{ $ns }}
    {{- include "commonLabels" (dict "name" $name "env" $spec.environment) | indent 4 }}
  spec:
    replicas: {{ $spec.replicas | default 1 }}
    selector:
      matchLabels:
        app: {{ $name }}
    template:
      metadata:
        {{- include "commonLabels" (dict "name" $name "env" $spec.environment) | indent 8 }}
      ...
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: {{ $name }}
    namespace: {{ $ns }}
    {{- include "commonLabels" (dict "name" $name "env" $spec.environment) | indent 4 }}
  ...
```

`include` returns the rendered string which you can pipe through `indent N` to get the right YAML indentation. `dict` constructs an ad-hoc map to pass as the template context.

This pattern is identical to Helm's `_helpers.tpl` approach — if you have used Helm before, it is the same idea.

---

## Environment-Driven Replica Defaults

A common pattern: different replica defaults per environment, overridable by the user:

```
{{- $env      := $spec.environment | default "production" }}
{{- $replicas := $spec.replicas | default 0 | int }}

{{/* Set environment-appropriate defaults if the user did not specify */}}
{{- if eq $replicas 0 }}
  {{- if eq $env "production" }}
    {{- $replicas = 3 }}
  {{- else if eq $env "staging" }}
    {{- $replicas = 2 }}
  {{- else }}
    {{- $replicas = 1 }}
  {{- end }}
{{- end }}
```

Then use `$replicas` in the Deployment spec:

```
spec:
  replicas: {{ $replicas }}
```

Note that `$replicas = 3` (without `:=`) is a reassignment of an existing variable. This only works in Go templates if the variable was declared with `:=` in the same or outer scope.

---

## Hands-On: A Multi-Environment WebService With Status Writeback

You will extend the WebService Composition from Chapter 04 with environment-driven replica defaults, nil-safe HPA, and status writeback. This uses the same XRD from Chapter 03 — no schema changes needed.

```bash
mkdir -p practice/ch05
```

### Step 1: Write the Enhanced Composition

Create `practice/ch05/webservice-advanced-composition.yaml`:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: webservice-advanced-composition
  labels:
    channel: advanced
spec:
  compositeTypeRef:
    apiVersion: platform.example.io/v1alpha1
    kind: WebService
  mode: Pipeline
  pipeline:
  - step: render-webservice
    functionRef:
      name: function-go-templating
    input:
      apiVersion: gotemplating.fn.crossplane.io/v1beta1
      kind: GoTemplate
      source: Inline
      inline:
        template: |
          {{- $name := .oxr.resource.metadata.name }}
          {{- $ns   := .oxr.resource.metadata.namespace }}
          {{- $spec := .oxr.resource.spec }}
          {{- $env  := $spec.environment | default "production" }}

          {{/* Environment-driven replica default */}}
          {{- $replicas := $spec.replicas | default 0 | int }}
          {{- if eq $replicas 0 }}
            {{- if eq $env "production" }}
              {{- $replicas = 3 }}
            {{- else if eq $env "staging" }}
              {{- $replicas = 2 }}
            {{- else }}
              {{- $replicas = 1 }}
            {{- end }}
          {{- end }}

          {{/* Nil-safe autoscaling object */}}
          {{- $autoscaling := $spec.autoscaling | default dict }}

          ---
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            labels:
              app: {{ $name }}
              environment: {{ $env }}
              managed-by: crossplane
          spec:
            {{- if not $autoscaling.enabled }}
            replicas: {{ $replicas }}
            {{- end }}
            selector:
              matchLabels:
                app: {{ $name }}
            template:
              metadata:
                labels:
                  app: {{ $name }}
                  environment: {{ $env }}
              spec:
                containers:
                - name: app
                  image: {{ $spec.image }}
                  ports:
                  - containerPort: {{ $spec.port | default 80 }}
          ---
          apiVersion: v1
          kind: Service
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            labels:
              app: {{ $name }}
          spec:
            selector:
              app: {{ $name }}
            ports:
            - port: 8080
              targetPort: {{ $spec.port | default 80 }}
              protocol: TCP
          {{- if $autoscaling.enabled }}
          ---
          apiVersion: autoscaling/v2
          kind: HorizontalPodAutoscaler
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            labels:
              app: {{ $name }}
          spec:
            scaleTargetRef:
              apiVersion: apps/v1
              kind: Deployment
              name: {{ $name }}
            minReplicas: {{ $autoscaling.minReplicas | default 2 }}
            maxReplicas: {{ $autoscaling.maxReplicas | default 10 }}
            metrics:
            - type: Resource
              resource:
                name: cpu
                target:
                  type: Utilization
                  averageUtilization: 70
          {{- end }}
          {{- if $spec.config }}
          ---
          apiVersion: v1
          kind: ConfigMap
          metadata:
            name: {{ $name }}-config
            namespace: {{ $ns }}
            labels:
              app: {{ $name }}
          data:
          {{- range $key, $val := $spec.config }}
            {{ $key }}: {{ $val | quote }}
          {{- end }}
          {{- end }}
          ---
          # Write computed values back to XR status
          apiVersion: {{ .oxr.resource.apiVersion }}
          kind: {{ .oxr.resource.kind }}
          metadata:
            name: {{ $name }}
            namespace: {{ $ns }}
            annotations:
              gotemplating.fn.crossplane.io/composition-resource-name: composite
          status:
            endpoint: "http://{{ $name }}.{{ $ns }}.svc.cluster.local:8080"
            environment: {{ $env }}
            scalingMode: {{ if $autoscaling.enabled }}hpa{{ else }}fixed{{ end }}
```

Apply it:

```bash
kubectl apply -f practice/ch05/webservice-advanced-composition.yaml
```

### Step 2: Create a Production WebService (No Autoscaling)

Create `practice/ch05/svc-production.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: svc-production
  namespace: default
spec:
  compositionRef:
    name: webservice-advanced-composition
  image: nginx:alpine
  port: 80
  environment: production
  config:
    LOG_LEVEL: warn
    APP_NAME: svc-production
```

```bash
kubectl apply -f practice/ch05/svc-production.yaml
kubectl get deployments,services --watch
# Ctrl+C when svc-production Deployment shows 3/3 READY
```

No `spec.replicas` was set — the environment-driven default of 3 should apply:

```bash
kubectl get deployment svc-production -o jsonpath='{.spec.replicas}'
# 3
```

### Step 3: Check the Status Writeback

```bash
kubectl get webservice svc-production -n default -o yaml | grep -A 5 "status:"
```

Expected:

```yaml
status:
  endpoint: http://svc-production.default.svc.cluster.local:8080
  environment: production
  scalingMode: fixed
```

### Step 4: Create a Staging WebService With Autoscaling

Create `practice/ch05/svc-staging.yaml`:

```yaml
apiVersion: platform.example.io/v1alpha1
kind: WebService
metadata:
  name: svc-staging
  namespace: default
spec:
  compositionRef:
    name: webservice-advanced-composition
  image: nginx:alpine
  port: 80
  environment: staging
  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 5
```

```bash
kubectl apply -f practice/ch05/svc-staging.yaml
kubectl get deployments,services,hpa --watch
# Ctrl+C when HPA appears
```

Verify the Deployment has **no static replica count** (HPA manages it):

```bash
kubectl get deployment svc-staging -o jsonpath='{.spec.replicas}'
# <no value> — the template omits replicas: when autoscaling is enabled
```

Check the HPA:

```bash
kubectl get hpa svc-staging -n default
```

Check status writeback:

```bash
kubectl get webservice svc-staging -n default -o jsonpath='{.status.scalingMode}'
# hpa
```

### Step 5: Update Autoscaling Config In-Place

```bash
kubectl patch webservice svc-staging -n default \
  --type merge \
  -p '{"spec":{"autoscaling":{"minReplicas":2,"maxReplicas":8}}}'
```

```bash
kubectl get hpa svc-staging -n default
# MINPODS should now show 2, MAXPODS 8
```

### Step 6: Clean Up

```bash
kubectl delete webservice svc-production svc-staging -n default
```

---

## What You Built

- Environment-driven replica defaults using template variable reassignment
- Nil-safe `default dict` pattern for optional nested spec objects
- Conditional HPA: created when `spec.autoscaling.enabled` is true, absent otherwise
- Deployment `replicas:` field omitted when HPA is active (so HPA can own it)
- Status writeback: computed values (`endpoint`, `scalingMode`) surfaced on the XR
- The `range` loop and conditional `ConfigMap` pattern from Chapter 04, now in a larger template

Chapter 06 covers Composition Revisions — how to roll out Composition changes safely and pin specific XR instances to a specific version.

---

➡️ [Chapter 06: Composition Revisions](06-composition-revisions.md)
