---
title: Cluster Placement
---

The `placement` sub-package provides a type-safe Go API for expressing label-based conditions, logical combinators (`All`, `Any`, `Not`), and evaluation logic. Constraints are evaluated at `vela def apply-module` time against cluster labels — definitions that don't match are skipped and never applied.

## Why Cluster Placement?

In multi-cluster environments, not all definitions make sense on every cluster. For example:

- An **AWS Load Balancer** component should only deploy to AWS EKS clusters, not GCP or Azure
- A **GPU workload** trait should only apply on clusters with GPU nodes
- A **persistent storage** component should not run on ephemeral virtual clusters
- A **production ingress** should only deploy to production clusters, not development environments

Placement constraints let you encode these rules directly in your definition. When you run `vela def apply-module`, the CLI evaluates each definition's constraints against the target cluster's labels and skips definitions that don't match — preventing misconfiguration before it happens.

## Label Conditions

`placement.Label(key)` creates a builder that supports equality, set membership, and existence checks against the target cluster's labels.

Import: `"github.com/oam-dev/kubevela/pkg/definition/defkit/placement"`

Applies to: **All Definition Types**

### Label & LabelConditionBuilder

| Function / Method | Description |
|---|---|
| `placement.Label(key string)` | Starts a condition based on a cluster label key. Chain with comparison methods below. |
| `.Eq(value)` | Label must equal value. |
| `.Ne(value)` | Label must not equal value. |
| `.In(values...)` | Label must be one of the listed values. |
| `.NotIn(values...)` | Label must not be any of the listed values. |
| `.Exists()` | Label key must be present (any value). |
| `.NotExists()` | Label key must not be present. |

```go title="Go — defkit"
import "github.com/oam-dev/kubevela/pkg/definition/defkit/placement"

placement.Label("provider").Eq("aws")
placement.Label("provider").Ne("aws")
placement.Label("cluster-type").In("eks", "self-managed")
placement.Label("env").NotIn("development", "staging")
placement.Label("gpu").Exists()
placement.Label("deprecated").NotExists()
```

```cue title="Evaluated at apply time"
// Placement constraints are evaluated by the CLI
// before applying definitions to the cluster.
// They do NOT appear in generated CUE or YAML output.
```

## Logical Combinators

Combine conditions with logical operators. `All` requires every condition to match. `Any` requires at least one. `Not` negates a condition. Combinators can be nested to express complex placement rules.

Applies to: **All Definition Types**

### Combinator Functions

| Function | Description |
|---|---|
| `placement.All(conditions...)` | All conditions must match (AND). |
| `placement.Any(conditions...)` | At least one must match (OR). |
| `placement.Not(condition)` | Invert a condition. |

```go title="Go — defkit"
// All conditions must match
placement.All(
    placement.Label("provider").Eq("aws"),
    placement.Label("env").Eq("production"),
)

// At least one must match
placement.Any(
    placement.Label("provider").Eq("aws"),
    placement.Label("provider").Eq("gcp"),
)

// Negate
placement.Not(placement.Label("cluster-type").Eq("vcluster"))

// Nested
placement.All(
    placement.Any(
        placement.Label("provider").In("aws", "gcp"),
    ),
    placement.Label("env").Eq("production"),
)
```

```cue title="Evaluation logic"
// Evaluation rules (applied by CLI):
// 1. No constraints → eligible everywhere
// 2. All RunOn conditions must match
// 3. No NotRunOn conditions may match
// 4. Eligible = (RunOn matches || empty) &&
//              NOT (NotRunOn matches)
```

## `.RunOn()` / `.NotRunOn()`

Wire placement constraints into any definition type. `.RunOn()` specifies conditions that must ALL match for the definition to be applied. `.NotRunOn()` specifies conditions that must NOT match. Multiple calls accumulate (AND semantics).

Applies to: **All Definition Types**

```go title="Go — defkit"
// AWS-only component
comp := defkit.NewComponent("aws-load-balancer").
    Description("AWS Application Load Balancer").
    RunOn(
        placement.Label("provider").Eq("aws"),
        placement.Label("cluster-type").In("eks", "self-managed"),
    ).
    NotRunOn(
        placement.Label("cluster-type").Eq("vcluster"),
    ).
    Workload("apps/v1", "Deployment").
    Params(defkit.String("image"))

// Exclude virtual clusters only
comp2 := defkit.NewComponent("persistent-storage").
    NotRunOn(placement.Label("cluster-type").Eq("vcluster"))

// GPU trait
trait := defkit.NewTrait("gpu-resource").
    RunOn(placement.Label("gpu").Exists()).
    AppliesTo("deployments.apps")
```

```cue title="CUE output (unchanged)"
"aws-load-balancer": {
    type: "component"
    description: "AWS Application Load Balancer"
    attributes: workload: definition: {
        apiVersion: "apps/v1"
        kind:       "Deployment"
    }
}
// Placement constraints are NOT
// visible in the generated CUE/YAML
```

## `placement.Evaluate()` / `placement.ValidatePlacement()`

`placement.Evaluate(spec, labels)` programmatically checks whether a cluster (given its label map) satisfies a `PlacementSpec`. Returns a result with `.Eligible` bool and `.Reason` string. `placement.ValidatePlacement(spec)` detects logically conflicting constraints (e.g., same label in RunOn and NotRunOn) at definition-build time.

Applies to: **All Definition Types**

| Function | Description |
|---|---|
| `placement.Evaluate(spec, labels) PlacementResult` | Programmatically checks if a cluster satisfies a `PlacementSpec`. Returns `.Eligible` bool and `.Reason` string. |
| `placement.ValidatePlacement(spec) error` | Detects logically conflicting constraints at definition-build time. |

```go title="Go — defkit"
spec := placement.PlacementSpec{
    RunOn: []placement.Condition{
        placement.Label("provider").Eq("aws"),
    },
    NotRunOn: []placement.Condition{
        placement.Label("cluster-type").Eq("vcluster"),
    },
}

labels := map[string]string{
    "provider":     "aws",
    "cluster-type": "eks",
}

result := placement.Evaluate(spec, labels)
// result.Eligible == true
// result.Reason describes why

// Validate for logical conflicts
err := placement.ValidatePlacement(spec)
```

```go title="Go — unit test"
It("should be eligible on AWS EKS clusters", func() {
    comp := defkit.NewComponent("aws-only").
        RunOn(placement.Label("provider").Eq("aws"))

    spec := comp.GetPlacement()
    result := placement.Evaluate(spec, map[string]string{
        "provider": "aws",
    })
    Expect(result.Eligible).To(BeTrue())
})

It("should be ineligible on GCP clusters", func() {
    comp := defkit.NewComponent("aws-only").
        RunOn(placement.Label("provider").Eq("aws"))

    spec := comp.GetPlacement()
    result := placement.Evaluate(spec, map[string]string{
        "provider": "gcp",
    })
    Expect(result.Eligible).To(BeFalse())
})
```

:::info
Placement constraints are evaluated by `vela def apply-module` by reading cluster labels from the `vela-cluster-identity` ConfigMap in the `vela-system` namespace. Definitions that do not match are **skipped** silently — no error is reported, and the definition is not applied.
:::

## Real-World Examples

### AWS-Only Component

```go title="Go — defkit"
comp := defkit.NewComponent("aws-load-balancer").
    RunOn(placement.Label("provider").Eq("aws"))
```

```cue title="Evaluated at apply time"
// Placement constraints do not appear in the generated CUE.
// The CLI skips this definition on clusters where
// the "provider" label is not "aws".
```

### Exclude Virtual Clusters

```go title="Go — defkit"
comp := defkit.NewComponent("persistent-storage").
    NotRunOn(placement.Label("cluster-type").Eq("vcluster"))
```

```cue title="Evaluated at apply time"
// Placement constraints do not appear in the generated CUE.
// The CLI skips this definition on clusters where
// the "cluster-type" label equals "vcluster".
```

### Production Multi-Cloud

Combine `placement.All()` and `placement.Any()` to express complex constraints: run only on production clusters that are hosted on a major cloud provider.

```go title="Go — defkit"
comp := defkit.NewComponent("enterprise-ingress").
    Description("Enterprise ingress controller").
    RunOn(
        placement.All(
            placement.Any(
                placement.Label("provider").In("aws", "gcp", "azure"),
            ),
            placement.Label("env").Eq("production"),
        ),
    ).
    Workload("apps/v1", "Deployment").
    Params(defkit.String("image"))
```

## Module-Wide Placement (`module.yaml`)

The Go API above sets placement on a single definition. For constraints that should apply to **every** definition in a module, declare them in `module.yaml` under `spec.placement`. They are evaluated by `vela def apply-module` at module-load time, before any per-definition rules.

Where this is useful:

- A module of AWS-only definitions that should never apply to GCP/Azure clusters — declare it once at the module level instead of repeating on each definition.
- A module that must never run on virtual clusters (`vcluster`) — one `notRunOn` rule blocks the whole module.

### YAML schema

```yaml title="module.yaml — module-wide placement"
apiVersion: core.oam.dev/v1beta1
kind: DefinitionModule
metadata:
  name: my-platform
spec:
  placement:
    runOn:
      - key: provider
        operator: In
        values: ["aws", "gcp"]
    notRunOn:
      - key: cluster-type
        operator: Eq
        values: ["vcluster"]
```

| Field | Type | Description |
|---|---|---|
| `runOn` | `[]Condition` | All conditions must match for definitions in this module to be applied. Omit (or leave empty) to skip the gate. |
| `notRunOn` | `[]Condition` | If any condition matches, every definition in the module is skipped. |

Each `Condition` has the same shape as the Go-API `placement.Label(...)` chain, but expressed in YAML:

| Field | Type | Description |
|---|---|---|
| `key` | string | Cluster label key to match against. |
| `operator` | string | One of `Eq`, `Ne`, `In`, `NotIn`, `Exists`, `NotExists`. |
| `values` | `[]string` | Values to compare against. Required for `Eq`/`Ne`/`In`/`NotIn`. Ignored by `Exists`/`NotExists`. |

These operator strings come straight from `pkg/definition/defkit/placement/types.go:32-44` — they are case-sensitive and must match exactly. `Operator: equals` or `Operator: ==` will fail to parse.

### Module-wide vs per-definition — **override**, not merge

`module.yaml` placement and per-definition `.RunOn(...)` / `.NotRunOn(...)` do **not** combine. They override:

- If a definition has **any** per-definition placement, its module-wide rules are **ignored** completely (per `placement.GetEffectivePlacement` at `pkg/definition/defkit/placement/types.go:142-149`).
- If a definition has **no** per-definition placement, it inherits the module-wide rules.

So the module-wide block acts as a **default** for definitions that don't set their own constraints. To enforce a constraint on every definition in a module regardless of per-definition rules, you must include it in each definition's `.RunOn(...)` / `.NotRunOn(...)` — there is no global override.

Module hooks (`pre-apply` / `post-apply`) are **not** gated by placement; they always run regardless of whether any definitions were skipped.

The CLI logs each skip with the failing condition, e.g.:

```text
ComponentDefinition simple-deploy: skipped (runOn conditions not satisfied: provider = k3d)
```

### Where cluster labels come from

Placement does **not** read labels from Kubernetes Node objects. It reads from a **ConfigMap** named `vela-cluster-identity` in the `vela-system` namespace (see `pkg/definition/defkit/placement/cluster.go:30-65`):

```yaml title="vela-cluster-identity ConfigMap"
apiVersion: v1
kind: ConfigMap
metadata:
  name: vela-cluster-identity
  namespace: vela-system
data:
  provider: aws
  env: production
  cluster-type: eks
  gpu: "true"
```

Each key/value in `data:` is treated as one cluster label. If the ConfigMap is absent, the placement evaluator returns an empty label map (no error) — `runOn` rules will then fail (no labels to satisfy them) and `notRunOn` rules pass trivially.

`vela def apply-module` prints the labels at the start of every run so you can see exactly what placement is evaluating against:

```text
Checking placement constraints...
Cluster labels: provider=aws, env=production
```

A `Cluster labels: (no labels)` line means the ConfigMap is missing or empty.

### Live example — block virtual clusters from a whole module

```yaml title="module.yaml"
apiVersion: core.oam.dev/v1beta1
kind: DefinitionModule
metadata:
  name: defkit-demo
spec:
  description: Demo module — no vclusters allowed.

  placement:
    notRunOn:
      - key: cluster-type
        operator: Eq
        values: ["vcluster"]
```

On a cluster labeled `cluster-type=vcluster`, `vela def apply-module` reports the module is filtered out and applies nothing. On any other cluster, every definition in the module is evaluated against its own per-definition rules (if any) and applied normally.

:::tip Inspect cluster labels
`vela def apply-module` prints the labels it evaluates against at start of run:

```text
Checking placement constraints...
Cluster labels: provider=aws, env=production
```

If the line shows `(no labels)`, your cluster has none of the labels referenced by your placement constraints — `runOn` rules will fail unless they only use `NotExists`, and `notRunOn` rules will pass trivially.
:::

```cue title="CUE — generated"
"enterprise-ingress": {
    type: "component"
    annotations: {}
    labels: {}
    description: "Enterprise ingress controller"
    attributes: {
        workload: {
            definition: {
                apiVersion: "apps/v1"
                kind:       "Deployment"
            }
            type: "deployments.apps"
        }
    }
}
template: {
    parameter: {
        image: string
    }
}
```
