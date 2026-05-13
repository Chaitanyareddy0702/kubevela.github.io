---
title: Health & Status DSL
---

Builder DSL for defining health checks and custom status messages. Generates the `status.healthPolicy` and `status.customStatus` CUE blocks consumed by the KubeVela controller.

**Health policy** — a boolean expression evaluated by the controller to determine if a workload is healthy. Drives the `DispatchHealthy` → `Healthy` component status transition.

**Custom status** — a message string extracted from the workload and surfaced in `Application.status.components[*].message`. Lets users see meaningful status without reading raw Kubernetes resources.

Both are optional. If omitted, the controller uses built-in defaults based on the workload type.

## Preset Builders

The fastest way to add health and status to common workload types.

| Preset | Type | Description |
|---|---|---|
| `DeploymentHealth()` | Health | Checks replicas == readyReplicas == updatedReplicas |
| `DeploymentStatus()` | Status | Shows "Ready: X/Y" (readyReplicas vs spec.replicas) |
| `StatefulSetHealth()` | Health | Checks readyReplicas == updatedReplicas == spec.replicas, plus observedGeneration |
| `StatefulSetStatus()` | Status | Shows "Ready: X/Y" for StatefulSets |
| `DaemonSetHealth()` | Health | Checks numberReady == desiredNumberScheduled == currentNumberScheduled == updatedNumberScheduled, plus observedGeneration |
| `DaemonSetStatus()` | Status | Shows "Ready: X/Y" (numberReady vs desiredNumberScheduled) |
| `JobHealth()` | Health | Checks status.succeeded == spec.parallelism |
| `CronJobHealth()` | Health | Always healthy if the CronJob resource exists |

String helpers: `StatusEq(left, right)`, `StatusGte(left, right)`, `StatusOr(conditions...)`, `StatusAnd(conditions...)`.

### `defkit.DeploymentHealth()` / `defkit.DaemonSetHealth()` / `defkit.JobHealth()`

Pre-built health policy builders for the three common workload types.

```go title="Go — defkit"
return defkit.NewComponent("task").
    Workload("batch/v1", "Job").
    HealthPolicy(defkit.JobHealth().Build())

return defkit.NewComponent("webservice").
    Workload("apps/v1", "Deployment").
    HealthPolicy(defkit.DeploymentHealth().Build())

return defkit.NewComponent("daemon").
    Workload("apps/v1", "DaemonSet").
    HealthPolicy(defkit.DaemonSetHealth().Build())
```

```cue title="CUE — JobHealth().Build()"
succeeded: *0 | int
if context.output.status.succeeded != _|_ {
    succeeded: context.output.status.succeeded
}
isHealth: succeeded == context.output.spec.parallelism
```

```cue title="CUE — CronJobHealth().Build()"
isHealth: true
```

```cue title="CUE — DeploymentHealth().Build()"
ready: {
    updatedReplicas:    *0 | int
    readyReplicas:      *0 | int
    replicas:           *0 | int
    observedGeneration: *0 | int
} & {
    if context.output.status.updatedReplicas != _|_ {
        updatedReplicas: context.output.status.updatedReplicas
    }
    if context.output.status.readyReplicas != _|_ {
        readyReplicas: context.output.status.readyReplicas
    }
    if context.output.status.replicas != _|_ {
        replicas: context.output.status.replicas
    }
    if context.output.status.observedGeneration != _|_ {
        observedGeneration: context.output.status.observedGeneration
    }
}
_isHealth: (context.output.spec.replicas == ready.readyReplicas) && (context.output.spec.replicas == ready.updatedReplicas) && (context.output.spec.replicas == ready.replicas) && (ready.observedGeneration == context.output.metadata.generation || ready.observedGeneration > context.output.metadata.generation)
isHealth: *_isHealth | bool
if context.output.metadata.annotations != _|_ {
    if context.output.metadata.annotations["app.oam.dev/disable-health-check"] != _|_ {
        isHealth: true
    }
}
```

`DeploymentHealth()` uses `.WithDefault()` (the `_isHealth` / `isHealth: *_isHealth | bool` pattern) and `.WithDisableAnnotation("app.oam.dev/disable-health-check")` so applications can opt out via metadata.

```cue title="CUE — DaemonSetHealth().Build()"
ready: {
    replicas: *0 | int
} & {
    if context.output.status.numberReady != _|_ {
        replicas: context.output.status.numberReady
    }
}
desired: {
    replicas: *0 | int
} & {
    if context.output.status.desiredNumberScheduled != _|_ {
        replicas: context.output.status.desiredNumberScheduled
    }
}
current: {
    replicas: *0 | int
} & {
    if context.output.status.currentNumberScheduled != _|_ {
        replicas: context.output.status.currentNumberScheduled
    }
}
updated: {
    replicas: *0 | int
} & {
    if context.output.status.updatedNumberScheduled != _|_ {
        replicas: context.output.status.updatedNumberScheduled
    }
}
generation: {
    metadata: context.output.metadata.generation
    observed: *0 | int
} & {
    if context.output.status.observedGeneration != _|_ {
        observed: context.output.status.observedGeneration
    }
}
isHealth: (desired.replicas == ready.replicas) && (desired.replicas == updated.replicas) && (desired.replicas == current.replicas) && (generation.observed == generation.metadata || generation.observed > generation.metadata)
```

### `defkit.DeploymentStatus()` / `defkit.DaemonSetStatus()`

Pre-built custom status builders for Deployment and DaemonSet workloads.

```go title="Go — defkit"
return defkit.NewComponent("worker").
    Workload("apps/v1", "Deployment").
    CustomStatus(defkit.DeploymentStatus().Build()).
    HealthPolicy(defkit.DeploymentHealth().Build())

return defkit.NewComponent("daemon").
    Workload("apps/v1", "DaemonSet").
    CustomStatus(defkit.DaemonSetStatus().Build()).
    HealthPolicy(defkit.DaemonSetHealth().Build())
```

```cue title="CUE — DeploymentStatus().Build()"
ready: {
    readyReplicas: *0 | int
} & {
    if context.output.status.readyReplicas != _|_ {
        readyReplicas: context.output.status.readyReplicas
    }
}
message: "Ready:\(ready.readyReplicas)/\(context.output.spec.replicas)"
```

```cue title="CUE — DaemonSetStatus().Build()"
ready: {
    replicas: *0 | int
} & {
    if context.output.status.numberReady != _|_ {
        replicas: context.output.status.numberReady
    }
}
desired: {
    replicas: *0 | int
} & {
    if context.output.status.desiredNumberScheduled != _|_ {
        replicas: context.output.status.desiredNumberScheduled
    }
}
message: "Ready:\(ready.replicas)/\(desired.replicas)"
```

These builders emit field declarations and the `message:` line directly. The `customStatus: { ... }` wrapper around them is added by the definition renderer when these builders feed into `.CustomStatus()` (see the [Complete Example](#complete-example--database-component) at the bottom of this page).

## Health DSL — `defkit.Health()`

The `defkit.Health()` DSL provides two styles: the low-level `IntField/HealthyWhen` API and the higher-level composable expression API.

### Health Builder Methods

| Method | Description |
|---|---|
| `.IntField(name, sourcePath, defaultVal)` | Extracts an integer field from the output resource's status and stores it as a CUE variable for use in health conditions. The `sourcePath` is relative to `context.output`. |
| `.StringField(name, sourcePath, defaultVal)` | Same as `IntField` but for string values. |
| `.MetadataField(name, sourcePath)` | Extracts a field from the output resource's metadata rather than from status. |
| `.HealthyWhen(conditions ...string)` | Sets the `isHealth` expression from raw CUE condition strings. Each condition is ANDed together. Use string helpers like `StatusEq()`, `StatusGte()`. |
| `.HealthyWhenExpr(expr HealthExpression)` | Sets the `isHealth` expression from the type-safe HealthExpression DSL. |
| `.WithDefault()` | Enables the `_isHealth` intermediate pattern allowing the health result to be overridden via CUE unification. |
| `.WithDisableAnnotation(annotation)` | Allows the health check to be disabled via an annotation on the workload. |
| `.Build() string` | Compiles the health builder into a CUE string for the `status: healthPolicy:` block. |
| `.Policy(expr HealthExpression) string` | Generates a complete health policy CUE string directly from a HealthExpression, bypassing builder state. |
| `.RawCUE(cue)` | Escape hatch: replaces the entire health policy with raw CUE. |

### Low-level: `IntField` / `HealthyWhen`

```go title="Go — defkit"
return defkit.NewComponent("worker").
    HealthPolicy(defkit.Health().
        IntField("ready.updatedReplicas",   "status.updatedReplicas",   0).
        IntField("ready.readyReplicas",     "status.readyReplicas",     0).
        IntField("ready.replicas",          "status.replicas",          0).
        IntField("ready.observedGeneration","status.observedGeneration",0).
        HealthyWhen(
            defkit.StatusEq("context.output.spec.replicas", "ready.readyReplicas"),
            defkit.StatusEq("context.output.spec.replicas", "ready.updatedReplicas"),
            defkit.StatusEq("context.output.spec.replicas", "ready.replicas"),
            defkit.StatusOr(
                defkit.StatusEq("ready.observedGeneration", "context.output.metadata.generation"),
                "ready.observedGeneration > context.output.metadata.generation",
            ),
        ).Build())
```

```cue title="CUE — generated healthPolicy"
ready: {
    updatedReplicas:    *0 | int
    readyReplicas:      *0 | int
    replicas:           *0 | int
    observedGeneration: *0 | int
} & {
    if context.output.status.updatedReplicas != _|_ {
        updatedReplicas: context.output.status.updatedReplicas
    }
    if context.output.status.readyReplicas != _|_ {
        readyReplicas: context.output.status.readyReplicas
    }
    if context.output.status.replicas != _|_ {
        replicas: context.output.status.replicas
    }
    if context.output.status.observedGeneration != _|_ {
        observedGeneration: context.output.status.observedGeneration
    }
}
isHealth: (context.output.spec.replicas == ready.readyReplicas) && (context.output.spec.replicas == ready.updatedReplicas) && (context.output.spec.replicas == ready.replicas) && (ready.observedGeneration == context.output.metadata.generation || ready.observedGeneration > context.output.metadata.generation)
```

Each `IntField(name, sourcePath, default)` declaration emits a default in the first block and a guarded `if … != _|_ { … }` override in the sibling block — `name` is split on dots, so `ready.readyReplicas` lands under a single `ready: {…}` group. Each entry passed to `HealthyWhen()` is auto-parenthesized and joined with `&&`.

## Composable Health API

An alternative higher-level API using condition-based chaining. Wire into definitions with `.HealthPolicyExpr()`.

### HealthExpression DSL

| Method | Description |
|---|---|
| `.Condition(type string) *ConditionExpr` | Checks a Kubernetes status condition by type name (e.g. `"Ready"`). Returns a `*ConditionExpr` for chaining with `.IsTrue()`, `.IsFalse()`, etc. |
| `.Field(path) *HealthFieldExpr` | Checks a field value on the output resource. Path is relative to `context.output`. Returns a `*HealthFieldExpr` for comparison chains. |
| `.FieldRef(path) *HealthFieldRefExpr` | Creates a reference to a field for use in compound expressions (e.g. comparing two fields). |
| `.Phase(phases ...string)` | Checks if `context.output.status.phase` matches any of the listed phases. |
| `.PhaseField(path, phases...)` | Like `Phase()` but checks a custom field path instead of the default `status.phase`. |
| `.Exists(path)` / `.NotExists(path)` | Checks whether a path exists (or doesn't) on the output resource. Generates `path != _\|_`. |
| `.And(exprs...)` / `.Or(exprs...)` / `.Not(expr)` | Logical combinators for composing multiple HealthExpressions. |
| `.Always()` | Returns a HealthExpression that always evaluates to healthy. |
| `.AllTrue(condTypes ...string)` | Checks that all listed condition types have status `"True"`. The most common health pattern for CRDs. |
| `.AnyTrue(condTypes ...string)` | Checks that at least one of the listed condition types has status `"True"`. |

**ConditionExpr**: `.IsTrue()` — status is `"True"`. `.IsFalse()` — status is `"False"`. `.Is(status)` — exact match. `.Exists()` — condition type exists. `.ReasonIs(reason)` — reason field matches.

**HealthFieldExpr**: `.Eq(val)`, `.Ne(val)`, `.Gt(val)`, `.Gte(val)`, `.Lt(val)`, `.Lte(val)` — comparison operators. `.In(values...)` — field matches any listed value. `.Contains(substr)` — string field contains substring.

### Condition Checks

```go title="Go — defkit"
h := defkit.Health()

h.Condition("Ready").IsTrue()
h.Condition("Stalled").IsFalse()
h.Condition("Initialized").Exists()
h.Condition("Ready").ReasonIs("Available")

h.AllTrue("Ready", "Synced")
h.AnyTrue("Ready", "Available")

comp := defkit.NewComponent("operator").
    Workload("apps/v1", "Deployment").
    HealthPolicyExpr(h.AllTrue("Ready", "Synced"))
```

```cue title="CUE — generated"
_readyCond:  [ for c in context.output.status.conditions if c.type == "Ready"  { c } ]
_syncedCond: [ for c in context.output.status.conditions if c.type == "Synced" { c } ]
isHealth: (len(_readyCond) > 0 && _readyCond[0].status == "True") &&
          (len(_syncedCond) > 0 && _syncedCond[0].status == "True")
```

### Phase Checks

```go title="Go — defkit"
h := defkit.Health()

h.Phase("Running", "Succeeded")
h.PhaseField("status.currentPhase", "Active")
```

```cue title="CUE — generated"
isHealth: context.output.status.phase == "Running" ||
          context.output.status.phase == "Succeeded"
```

### Field Expressions

```go title="Go — defkit"
h := defkit.Health()

h.Field("status.state").Eq("active")
h.Field("status.replicas").Gt(0)
h.Field("status.availableReplicas").Gte(1)
h.Field("status.phase").In("Running", "Succeeded")
h.Field("status.message").Contains("ready")

h.Field("status.readyReplicas").Eq(h.FieldRef("spec.replicas"))

h.Exists("status.loadBalancer.ingress")
h.NotExists("status.error")
```

```cue title="CUE — generated"
isHealth: context.output.status.readyReplicas ==
          context.output.spec.replicas
```

### Composite Expressions: `h.And()` / `h.Or()` / `h.Not()`

```go title="Go — defkit"
h := defkit.Health()

comp := defkit.NewComponent("database").
    Workload("apps/v1", "StatefulSet").
    HealthPolicyExpr(h.And(
        h.Condition("Ready").IsTrue(),
        h.Not(h.Condition("Stalled").IsTrue()),
        h.Or(
            h.Field("status.replicas").Gte(1),
            h.Exists("status.endpoint"),
        ),
    ))
```

```cue title="CUE — generated"
_readyCond:   [ for c in context.output.status.conditions if c.type == "Ready"   { c } ]
_stalledCond: [ for c in context.output.status.conditions if c.type == "Stalled" { c } ]
isHealth: (len(_readyCond) > 0 && _readyCond[0].status == "True") &&
          (!(len(_stalledCond) > 0 && _stalledCond[0].status == "True")) &&
          ((context.output.status.replicas >= 1) || (context.output.status.endpoint != _|_))
```

## Status DSL — `defkit.Status()`

### Status Builder Methods

| Method | Description |
|---|---|
| `.IntField(name, sourcePath, defaultVal)` | Extracts an integer from the output resource and stores it as a CUE variable for the status message. |
| `.StringField(name, sourcePath, defaultVal)` | Extracts a string from the output resource for the status message. |
| `.Message(msg)` | Sets the status message template. Can reference variables using CUE interpolation. |
| `.Build() string` | Compiles into CUE for the `customStatus:` block. |
| `.RawCUE(cue)` | Escape hatch: replaces the status with raw CUE. |

### StatusExpression DSL

| Method | Description |
|---|---|
| `.Field(path) *StatusFieldExpr` | References a field relative to `context.output`. Returns a `*StatusFieldExpr` for chaining `.Default(val)` or comparison methods. |
| `.SpecField(path) *StatusFieldExpr` | Same as `Field()` — references a field relative to `context.output`. Functionally identical. |
| `.Condition(type) *StatusConditionAccessor` | Accesses a Kubernetes status condition by type name. Chain with `.StatusValue()`, `.Message()`, `.Reason()`, or `.Is(status)`. |
| `.Exists(path)` / `.NotExists(path)` | Checks whether a path exists (or doesn't) on the output resource. |
| `.Literal(value string)` | Creates a static string status expression. |
| `.Concat(parts ...any)` | Concatenates strings, field references, and other expressions into a single status message. |
| `.Format(template, args...)` | Creates a formatted status string using a template with positional arguments. |
| `.Case(condition, message)` | Creates a case for use in `Switch()`. |
| `.Default(message)` | Creates the default/fallback case for a `Switch()`. |
| `.Switch(casesAndDefault ...any)` | Evaluates cases in order. Generates CUE `if cond1 { msg1 } if cond2 { msg2 }`. |
| `.HealthAware(healthyMsg, unhealthyMsg)` | Creates a status message that varies based on `context.status.healthy`. |
| `.Detail(key, value)` | Creates a key-value detail entry for structured status reporting. |
| `.WithDetails(message, details...)` | Creates a status expression with both a message and structured key-value details. |

**StatusFieldExpr**: `.Default(val)` provides a fallback. `.Eq(val)`/`.Ne(val)`/`.Gt(val)`/`.Gte(val)`/`.Lt(val)`/`.Lte(val)` create StatusConditions for `Case()`/`Switch()`.

**StatusConditionAccessor**: `.StatusValue()` accesses `.status`. `.Message()` accesses `.message`. `.Reason()` accesses `.reason`. `.Is(status)` checks equality.

**Standalone**: `StatusPolicy(expr)` wraps a StatusExpression into the full `customStatus:` CUE block. `HealthPolicy(expr)` wraps a HealthExpression into the `healthPolicy:` block.

### Low-level: `IntField` / `Message`

```go title="Go — defkit"
return defkit.NewComponent("task").
    CustomStatus(defkit.Status().
        IntField("status.active",    "status.active",    0).
        IntField("status.failed",    "status.failed",    0).
        IntField("status.succeeded", "status.succeeded", 0).
        Message(`Active/Failed/Succeeded:\(status.active)/\(status.failed)/\(status.succeeded)`).
        Build())
```

```cue title="CUE — generated customStatus"
status: {
    active:    *0 | int
    failed:    *0 | int
    succeeded: *0 | int
} & {
    if context.output.status.active != _|_ {
        active: context.output.status.active
    }
    if context.output.status.failed != _|_ {
        failed: context.output.status.failed
    }
    if context.output.status.succeeded != _|_ {
        succeeded: context.output.status.succeeded
    }
}
message: "Active/Failed/Succeeded:\(status.active)/\(status.failed)/\(status.succeeded)"
```

`Status().IntField(name, sourcePath, default)` follows the same pattern as `Health().IntField` — name dots group fields under a single struct, defaults go in the first block, conditional overrides in the sibling block. The `customStatus: { … }` wrapper is added by the definition renderer; the builder itself emits the contents.

### `s.Format()` / `s.Switch()`

```go title="Go — defkit"
s := defkit.Status()

s.Format("Ready: %v/%v",
    s.Field("status.readyReplicas").Default(0),
    s.SpecField("spec.replicas"),
)

s.Switch(
    s.Case(s.Field("status.phase").Eq("Running"), "Service is running"),
    s.Case(s.Field("status.phase").Eq("Pending"), "Service is starting..."),
    s.Default("Unknown status"),
)
```

```cue title="CUE — generated (Format)"
_readyReplicas: *0 | int
if context.output.status.readyReplicas != _|_ {
    _readyReplicas: context.output.status.readyReplicas
}
message: "Ready: \(_readyReplicas)/\(context.output.spec.replicas)"
```

```cue title="CUE — generated (Switch)"
message: *"Unknown status" | string
if context.output.status.phase == "Running" {
    message: "Service is running"
}
if context.output.status.phase == "Pending" {
    message: "Service is starting..."
}
```

`Switch` lays the default down as the disjunction default (`*<default> | string`) and overrides it with one `if cond { message: ... }` block per case. The cases are evaluated in declaration order; in CUE the last unifying override wins, so put more specific cases later if they overlap.

`HealthAware(healthy, unhealthy)` follows the same shape with a single override:

```cue title="CUE — generated (HealthAware)"
message: *"Service degraded" | string
if context.status.healthy {
    message: "All systems operational"
}
```

### `.HealthPolicyExpr()` / `defkit.HealthPolicy(expr)` / `defkit.StatusPolicy(expr)`

- `.HealthPolicyExpr(expr)` — wires a composable health expression directly into the definition.
- `defkit.HealthPolicy(expr)` — converts a composed health expression to a raw CUE string.
- `defkit.StatusPolicy(expr)` — converts a composed status expression to a raw CUE string for `.CustomStatus()`.

```go title="Go — defkit"
h := defkit.Health()
s := defkit.Status()

comp := defkit.NewComponent("webservice").
    Workload("apps/v1", "Deployment").
    HealthPolicyExpr(h.Condition("Ready").IsTrue()).
    CustomStatus(defkit.StatusPolicy(
        s.Format("Ready: %v/%v",
            s.Field("status.readyReplicas").Default(0),
            s.SpecField("spec.replicas"),
        ),
    ))
```

```cue title="CUE — generated"
_readyCond: [ for c in context.output.status.conditions if c.type == "Ready" { c } ]
isHealth: len(_readyCond) > 0 && _readyCond[0].status == "True"

_readyReplicas: *0 | int
if context.output.status.readyReplicas != _|_ {
    _readyReplicas: context.output.status.readyReplicas
}
message: "Ready: \(_readyReplicas)/\(context.output.spec.replicas)"
```

## Complete Example — Database Component

```go title="Go — defkit"
h := defkit.Health()
s := defkit.Status()

comp := defkit.NewComponent("database").
    Workload("apps/v1", "StatefulSet").
    Params(
        defkit.String("image"),
        defkit.Int("replicas").Default(3),
    ).
    HealthPolicyExpr(h.And(
        h.Condition("Ready").IsTrue(),
        h.Field("status.readyReplicas").Eq(h.FieldRef("spec.replicas")),
    )).
    CustomStatus(defkit.StatusPolicy(
        s.Format("Ready: %v/%v",
            s.Field("status.readyReplicas").Default(0),
            s.SpecField("spec.replicas"),
        ),
    ))
```

```cue title="CUE — generated"
database: {
    type: "component"
    annotations: {}
    labels: {}
    description: ""
    attributes: {
        workload: {
            definition: {
                apiVersion: "apps/v1"
                kind:       "StatefulSet"
            }
            type: "statefulsets.apps"
        }
        status: {
            customStatus: #"""
                _readyReplicas: *0 | int
                if context.output.status.readyReplicas != _|_ {
                    _readyReplicas: context.output.status.readyReplicas
                }
                message: "Ready: \(_readyReplicas)/\(context.output.spec.replicas)"
                """#
            healthPolicy: #"""
                _readyCond: [ for c in context.output.status.conditions if c.type == "Ready" { c } ]
                isHealth: (len(_readyCond) > 0 && _readyCond[0].status == "True") && (context.output.status.readyReplicas == context.output.spec.replicas)
                """#
        }
    }
}
template: {
    parameter: {
        image: string
        replicas: *3 | int
    }
}
```
