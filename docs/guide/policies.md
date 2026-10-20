# Policy Engine

Provena's policy engine lets you define governance rules that are evaluated
automatically on every logged record. Policies give you fine-grained control
over which context is accepted, flagged, or rejected — without changing your
application logic.

## How Policies Work

When `trail.log()` or a `@trail.track()` decorated function produces a record,
the policy engine evaluates all registered policies against it. The decision
follows a **forbid-overrides-permit** model: if any BLOCK-level policy fails,
the overall decision is `DENY`, regardless of how many other policies pass.

The evaluation happens **after** the record is written to storage. A blocked
record is still persisted to the audit trail — the `PolicyViolation` exception
is raised as a signal, but the evidence of the rejected input remains in your
audit log.

## Enforcement Levels

Every policy has one of three enforcement levels:

| Level | Constant | Behavior on failure |
|---|---|---|
| `log` | `EnforcementLevel.LOG` | Record is accepted. Nothing is logged — the result is silently included in the `PolicyEvaluation`. |
| `warn` | `EnforcementLevel.WARN` | Record is accepted. A `WARNING` message is emitted to the `provena.policy` logger. |
| `block` | `EnforcementLevel.BLOCK` | Record is accepted **and persisted**, then `PolicyViolation` is raised. |

!!! important "Blocked records are still persisted"
    A `BLOCK`-level violation does **not** prevent the record from being
    written to storage. The record is saved, then the exception is raised.
    This is intentional: the audit trail must contain evidence of rejected
    inputs for governance and compliance purposes.

    ```python
    from provena import ContextTrail
    from provena.policy import provenance_check, EnforcementLevel, PolicyViolation

    trail = ContextTrail(
        backend="memory",
        policies=[provenance_check(status="MISSING", enforcement=EnforcementLevel.BLOCK)],
    )
    try:
        trail.log("no provenance", source="retriever")
    except PolicyViolation:
        pass

    print(trail.summary()["total"])  # 1 — the record was persisted despite the block
    trail.close()
    ```

## Built-in Policies

Provena ships four ready-to-use policy factory functions. All accept a
`enforcement` keyword argument.

### `freshness_check(status, enforcement)`

Fails when a record's freshness status matches the given string.

```python
from provena.policy import freshness_check, EnforcementLevel

# Warn when context is stale (older than max_age_days)
policy = freshness_check(status="STALE", enforcement=EnforcementLevel.WARN)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `status` | `str` | `"STALE"` | Freshness status that causes a failure |
| `enforcement` | `EnforcementLevel` | `EnforcementLevel.LOG` | What happens on failure |

**Policy name:** `freshness:{status}` (e.g. `freshness:STALE`)

The check passes (`True`) when:
- The record has no freshness result (`freshness_result is None`), or
- The record's freshness status is **not** equal to `status`.

!!! tip
    Freshness is only evaluated when the trail has a `max_age_days` configured
    and the record includes provenance with a `created_at` field, or when
    `temporal_detection=True` detects a date in the content. See the
    [Freshness Checking guide](freshness.md) for details.

---

### `provenance_check(status, enforcement)`

Fails when a record's provenance status matches the given string.

```python
from provena.policy import provenance_check, EnforcementLevel

# Block records with missing provenance
policy = provenance_check(status="MISSING", enforcement=EnforcementLevel.BLOCK)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `status` | `str` | `"MISSING"` | Provenance status that causes a failure |
| `enforcement` | `EnforcementLevel` | `EnforcementLevel.LOG` | What happens on failure |

**Policy name:** `provenance:{status}` (e.g. `provenance:MISSING`)

The check passes (`True`) when:
- The record has no provenance result (`provenance_result is None`), or
- The record's provenance status is **not** equal to `status`.

---

### `require_signing(enforcement)`

Fails when the trail does not have HMAC signing enabled.

```python
from provena.policy import require_signing, EnforcementLevel

# Block any log call on an unsigned trail
policy = require_signing(enforcement=EnforcementLevel.BLOCK)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enforcement` | `EnforcementLevel` | `EnforcementLevel.BLOCK` | What happens on failure |

**Policy name:** `require_signing`

The check is wired to the trail's actual signing state via a mutable reference
(`_signed_ref`) that the trail sets at construction time. This means the check
correctly reflects whether `signing_key` was provided:

```python
from provena import ContextTrail
from provena.policy import require_signing, EnforcementLevel

# Signed trail — policy passes
trail = ContextTrail(
    backend="memory",
    signing_key="my-secret-key",
    policies=[require_signing(enforcement=EnforcementLevel.BLOCK)],
)
record = trail.log("data", source="retriever")
print(record is not None)  # True
trail.close()
```

!!! warning "Standalone usage without a ContextTrail"
    When you build a `PolicyEngine` via `PolicyEngine.from_config(...)` without
    passing a `_signed_ref`, the `require_signing` check always returns `False`
    (unsigned). To wire it correctly, provide the trail's signing reference:

    ```python
    engine = PolicyEngine.from_config(config, _signed_ref=[trail._hasher.is_signed])
    ```

    In normal usage through `ContextTrail`, this wiring happens automatically.

---

### `source_allowlist(allowed, enforcement)`

Fails when a record's source type is not in the allowed list.

```python
from provena.policy import source_allowlist, EnforcementLevel

# Only permit retriever and tool sources; block everything else
policy = source_allowlist(
    allowed=["retriever", "tool"],
    enforcement=EnforcementLevel.BLOCK,
)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `allowed` | `list[str]` | (required) | Source type strings that are permitted |
| `enforcement` | `EnforcementLevel` | `EnforcementLevel.BLOCK` | What happens on failure |

**Policy name:** `source_allowlist`

The check compares `record.entry.source.value` against the frozen set of
allowed strings. Source values correspond to `ContextSource` enum members:
`"retriever"`, `"tool"`, `"agent"`, `"human"`, `"synthetic"`.

```python
trail = ContextTrail(
    backend="memory",
    policies=[
        source_allowlist(allowed=["retriever", "tool"], enforcement=EnforcementLevel.BLOCK)
    ],
)

trail.log("ok", source="retriever")   # passes
trail.log("ok", source="tool")        # passes

from provena.policy import PolicyViolation
try:
    trail.log("blocked", source="agent")   # fails — "agent" not in allowlist
except PolicyViolation as exc:
    print(exc.policy_name)   # "source_allowlist"
trail.close()
```

---

## Configuring Policies on a Trail

### Passing Policy objects directly

```python
from datetime import datetime, timezone
from provena import ContextTrail, ProvenanceMetadata
from provena.policy import (
    freshness_check,
    provenance_check,
    source_allowlist,
    EnforcementLevel,
)

trail = ContextTrail(
    backend="memory",
    max_age_days=90,
    policies=[
        freshness_check(status="STALE", enforcement=EnforcementLevel.WARN),
        provenance_check(status="MISSING", enforcement=EnforcementLevel.BLOCK),
        source_allowlist(
            allowed=["retriever", "tool"],
            enforcement=EnforcementLevel.BLOCK,
        ),
    ],
)

prov = ProvenanceMetadata(
    source_url="https://docs.example.com",
    created_at=datetime.now(timezone.utc),
)
record = trail.log("fresh, sourced context", source="retriever", provenance=prov)
print(record is not None)  # True
trail.close()
```

### Using a config dict

The same setup works via the `config=` dict parameter, mirroring what the TOML
parser produces:

```python
trail = ContextTrail(
    config={
        "storage": {"backend": "memory"},
        "policies": [
            {"check": "provenance", "status": "MISSING", "enforcement": "block"},
        ],
    }
)
```

### Using a TOML file

Save a `provena.toml` file (see `provena.example.toml` for the full template):

```toml
[storage]
backend = "sqlite"
path = "provena.db"

[freshness]
max_age_days = 90

[[policies]]
check = "freshness"
status = "STALE"
enforcement = "warn"

[[policies]]
check = "provenance"
status = "MISSING"
enforcement = "block"

[[policies]]
check = "source_allowlist"
sources = ["retriever", "tool"]
enforcement = "block"

[[policies]]
check = "require_signing"
enforcement = "block"
```

Load it with:

```python
trail = ContextTrail(config="provena.toml")
```

#### TOML policy fields reference

| Field | Type | Required | Description |
|---|---|---|---|
| `check` | string | yes | One of: `freshness`, `provenance`, `require_signing`, `source_allowlist` |
| `enforcement` | string | no | One of: `log`, `warn`, `block` (default: `log`) |
| `status` | string | for `freshness`/`provenance` | Status value to match: `STALE`, `MISSING`, etc. |
| `sources` | list of strings | for `source_allowlist` | Allowed source type strings |

!!! warning "Config overrides constructor parameters"
    When `config=` is provided, it takes precedence over all other constructor
    arguments. Passing both `config=` and `policies=` emits a warning and the
    `policies=` list is ignored.

    ```python
    # The policies= argument is silently ignored here
    trail = ContextTrail(
        config="provena.toml",
        policies=[...],    # WARNING: this is overridden by config=
    )
    ```

---

## Per-Decorator Policy Overrides

`@trail.track()` accepts a `policies=` keyword argument that **replaces** the
trail-level policy engine for calls through that specific decorator. Direct
`trail.log()` calls are not affected.

```python
from provena import ContextTrail
from provena.policy import provenance_check, EnforcementLevel, PolicyViolation

trail = ContextTrail(backend="memory")  # no trail-level policies

strict_policies = [
    provenance_check(status="MISSING", enforcement=EnforcementLevel.BLOCK)
]

@trail.track(source="retriever", policies=strict_policies)
def fetch_docs(query: str) -> str:
    return f"Docs for: {query}"

# This call is subject to the strict_policies override
try:
    fetch_docs("k8s deployment")   # raises PolicyViolation (no provenance)
except PolicyViolation as exc:
    print(exc.policy_name)  # "provenance:MISSING"

# Direct log calls are NOT affected by the decorator override
record = trail.log("direct data", source="retriever")
print(record is not None)  # True

trail.close()
```

!!! tip
    Use per-decorator overrides when only certain data sources in your pipeline
    require stricter governance than the trail default. This avoids having to
    split context across multiple trails.

---

## Writing Custom Policies

A `Policy` is a frozen dataclass with a `check` callable:

```python
from provena.policy import Policy, EnforcementLevel

# Custom policy: reject context shorter than 10 characters
def _min_length_check(record) -> bool:
    content = record.entry.content
    if isinstance(content, bytes):
        return len(content) >= 10
    return len(str(content)) >= 10

short_content_policy = Policy(
    name="min_content_length",
    check=_min_length_check,
    enforcement=EnforcementLevel.WARN,
    description="Content must be at least 10 characters",
)

trail = ContextTrail(backend="memory", policies=[short_content_policy])
```

`Policy` fields:

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | (required) | Unique identifier shown in violations and logs |
| `check` | `Callable[[TrailRecord], bool]` | (required) | Returns `True` if the record passes |
| `enforcement` | `EnforcementLevel` | `EnforcementLevel.LOG` | What happens on failure |
| `description` | `str` | `""` | Human-readable explanation, included in violation details |

!!! warning "Exceptions in check functions are treated as failures"
    If your `check` callable raises an exception, the policy engine catches it,
    logs a `WARNING` to `provena.policy`, and treats the check as failed. If
    the policy is `BLOCK`-level, a `PolicyViolation` is raised.

---

## PolicyViolation

When a `BLOCK`-level policy fails, `PolicyViolation` is raised:

```python
from provena.policy import PolicyViolation

try:
    trail.log("missing provenance", source="retriever")
except PolicyViolation as exc:
    print(exc.policy_name)   # "provenance:MISSING"
    print(exc.details)       # "Provenance must not be MISSING"
    print(exc.record.id)     # Persisted record ID (>= 1)
    print(str(exc))          # "Policy 'provenance:MISSING' blocked context: Provenance must not be MISSING"
```

`PolicyViolation` attributes:

| Attribute | Type | Description |
|---|---|---|
| `policy_name` | `str` | Name of the policy that triggered the block |
| `record` | `TrailRecord` | The trail record that was blocked (already persisted) |
| `details` | `str` | Human-readable explanation of the violation |

When multiple `BLOCK`-level policies fail on the same record, only the first
failing `BLOCK` policy is reported in the exception. All violations are
available in the `PolicyEvaluation.violations` tuple if you need them.

---

## Chain Integrity After a Block

Blocked records are written to the hash chain before the exception is raised.
This means the chain remains intact and subsequent records chain off the
blocked record correctly:

```python
trail = ContextTrail(
    backend="memory",
    policies=[
        source_allowlist(allowed=["retriever"], enforcement=EnforcementLevel.BLOCK)
    ],
)

trail.log("first",   source="retriever")   # passes, id=1
try:
    trail.log("blocked", source="agent")   # blocked, id=2 — still persisted
except PolicyViolation:
    pass
trail.log("third",   source="retriever")   # passes, id=3

verdict = trail.verify_chain()
print(verdict.intact)         # True
print(verdict.total_records)  # 3
trail.close()
```

---

## Inspecting Policy Results

The raw evaluation result is not exposed through `trail.log()`. To inspect
individual policy outcomes, use `PolicyEngine.evaluate()` directly:

```python
from provena import ContextTrail
from provena.policy import PolicyEngine, provenance_check, EnforcementLevel

trail = ContextTrail(backend="memory")
record = trail.log("test", source="retriever")

engine = PolicyEngine([
    provenance_check(status="MISSING", enforcement=EnforcementLevel.LOG),
])

evaluation = engine.evaluate(record)
print(evaluation.decision)           # "ALLOW" (LOG does not deny)
print(len(evaluation.violations))    # 1
print(evaluation.violations[0].policy_name)  # "provenance:MISSING"
trail.close()
```

`PolicyEvaluation` fields:

| Field | Type | Description |
|---|---|---|
| `decision` | `"ALLOW"` or `"DENY"` | `"DENY"` only when at least one `BLOCK` policy fails |
| `results` | `tuple[PolicyCheckResult, ...]` | One entry per evaluated policy |
| `violations` | (property) `tuple[PolicyCheckResult, ...]` | Only the failing entries |

---

## Common Mistakes and Troubleshooting

### Blocked records accumulating in the audit trail

**Symptom:** `trail.summary()["total"]` grows even though most log calls raise
`PolicyViolation`.

**Reason:** This is expected. Blocked records are persisted before the
exception is raised. Use `trail.query(provenance_status="MISSING")` or
`trail.query(source="agent")` to identify and investigate them.

### `require_signing` always fails

**Symptom:** A `require_signing(enforcement=EnforcementLevel.BLOCK)` policy
blocks all records even though you supplied a `signing_key`.

**Common cause:** The `signing_key` parameter was passed alongside `config=`,
which ignores it. Ensure the signing key is configured under `[hash_chain]` in
the TOML file:

```toml
[hash_chain]
signing_key_env = "PROVENA_SIGNING_KEY"
```

Or pass both through the constructor without `config=`:

```python
trail = ContextTrail(
    backend="memory",
    signing_key="my-key",
    policies=[require_signing(enforcement=EnforcementLevel.BLOCK)],
)
```

### Invalid enforcement level in TOML

**Symptom:** A policy in `provena.toml` is silently ignored.

**Reason:** An invalid `enforcement` value (e.g. `"blocked"` instead of
`"block"`) causes the entry to be skipped with a warning:

```
WARNING - Invalid enforcement level 'blocked' for policy check 'freshness' — skipping. Valid levels: log, warn, block
```

Check the `provena.policy` logger output and correct the spelling.

### Unknown check name in TOML

**Symptom:** A policy is silently ignored.

**Reason:** An unrecognised `check` value causes a warning and the entry is
skipped. Valid values are: `freshness`, `provenance`, `require_signing`,
`source_allowlist`.

```
WARNING - Unknown policy check 'freshness_check' — skipping. Valid checks: freshness, provenance, require_signing, source_allowlist
```

### Policies in `config=` dict are ignored

**Symptom:** Config dict policies are not enforced.

**Reason:** When using `config=`, the key must be `"policies"` (plural), and
the value must be a list of dicts — not `Policy` objects:

```python
# Correct
config = {
    "storage": {"backend": "memory"},
    "policies": [
        {"check": "provenance", "status": "MISSING", "enforcement": "block"}
    ],
}
# Incorrect — Policy objects cannot be used in the config dict path
config = {
    "policies": [provenance_check(status="MISSING", enforcement=EnforcementLevel.BLOCK)]
}
```

---

## See Also

- [Tracking Context](tracking.md) — how `trail.log()` and `@trail.track()` work
- [Provenance Validation](provenance.md) — provenance status values (`VALID`, `INCOMPLETE`, `MISSING`)
- [Freshness Checking](freshness.md) — freshness status values (`FRESH`, `STALE`, `UNKNOWN`)
- [Chain Verification](verification.md) — verifying hash-chain integrity after blocked records
- [Configuration](configuration.md) — full TOML configuration reference
