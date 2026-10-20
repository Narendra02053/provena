# Google ADK

Provena integrates with the Google Agent Development Kit (ADK) through a tool
callback that records tool invocations and responses to your audit trail.

## Installation

```bash
pip install provena[google-adk]
```

This installs `google-adk>=1.0`. `ProvenaADKCallback` is designed to be lightweight with no heavy internal dependencies, adhering directly to the Google ADK callback interface.

## Quick start

```python
from google.adk.agents import Agent
from provena import ContextTrail
from provena.integrations.google_adk import ProvenaADKCallback

# Initialize the trail and callback
trail = ContextTrail(storage_path="adk_audit.db")
callback = ProvenaADKCallback(trail=trail)

# Attach callback to the agent
agent = Agent(
    name="research-agent",
    after_tool_callback=callback.after_tool_call,
)
```

## How it works

`ProvenaADKCallback` implements the `after_tool_call` method, designed to be passed to an ADK Agent's `after_tool_callback` parameter:

```python
def after_tool_call(
    self,
    tool: Any,
    args: dict[str, Any],
    tool_context: Any,
    tool_response: Any,
) -> Any | None:
    tool_name = (
        getattr(tool, "name", None) or getattr(tool, "__name__", "unknown")  # (1)!
    )
    content = str(tool_response) if tool_response is not None else ""  # (2)!
    if content:  # (3)!
        self._trail.log(
            content=content,
            source=ContextSource.TOOL,
            source_name=f"adk:{tool_name}",  # (4)!
            metadata={"args": args} if args else {},  # (5)!
        )
    return None  # (6)!
```

1. **Tool name resolution**: Extracts `tool.name` if present (for class/object-based tools), falls back to `tool.__name__` (for function-based tools), or `"unknown"`.
2. **Response serialization**: Converts `tool_response` to string.
3. **Empty response handling**: If `tool_response` is `None` or yields an empty string, nothing is logged to the trail.
4. **Source naming**: Logged with source `ContextSource.TOOL` (`"tool"`) and source name `adk:<tool_name>`.
5. **Argument metadata**: If tool invocation arguments `args` are present, they are stored in the record's metadata dictionary as `{"args": args}`.
6. **Pass-through return behavior**: Returns `None`. Under Google ADK's callback protocol, returning `None` preserves the original `tool_response` and passes it through to the agent without modification.

### Logged attributes

| Attribute | Logged Value | Description |
|---|---|---|
| `content` | `str(tool_response)` | Stringified tool response |
| `source` | `ContextSource.TOOL` (`"tool"`) | Identifies input as tool execution |
| `source_name` | `adk:<tool_name>` | Identifies the tool invoked |
| `metadata` | `{"args": args}` | Call arguments supplied to the tool (or `{}` if empty) |

## Provenance tracking & limitations

- **No provenance auto-extraction**: `ProvenaADKCallback` does not automatically populate `ProvenanceMetadata` fields (`source_url`, `author`, `created_at`).
- **Default status**: Tool output records receive provenance status `MISSING`.
- **Pass-through behavior**: The callback only observes responses; it does not filter or sanitize tool output.

!!! tip "Auditing tool inputs and outputs"
    Because `args` is saved in the record metadata, Provena allows you to audit both what parameters were passed to the tool and the resulting output hash.

## Full working example

```python
from google.adk.agents import Agent
from provena import ContextTrail
from provena.integrations.google_adk import ProvenaADKCallback

# Define a tool function
def weather_lookup(location: str) -> str:
    """Fetch current weather for a given city."""
    return f"Weather in {location}: 22°C, Sunny."

# Set up the Provena audit trail and callback
trail = ContextTrail(storage_path="adk_audit.db")
callback = ProvenaADKCallback(trail=trail)

# Create an agent with the callback attached
agent = Agent(
    name="weather-assistant",
    tools=[weather_lookup],
    after_tool_callback=callback.after_tool_call,
)

# Agent execution (tool calls trigger after_tool_call automatically)
# response = agent.run("What's the weather in Seattle?")
```

## Verifying the trail

After execution, verify the cryptographic integrity of recorded tool responses:

```python
# Summary of audit trail
summary = trail.summary()
print(f"Total records: {summary['total']}")
print(f"Sources: {summary['sources']}")
print(f"Provenance: {summary['provenance']}")

# Verify hash chain
verdict = trail.verify_chain()
assert verdict.intact, f"Chain broken: {verdict.details}"
print(f"Chain intact: {verdict.total_records} records verified")

# Inspect records with argument metadata
records = trail.query(source="tool", limit=10)
for r in records:
    print(f"  [{r['source_name']}] args={r.get('metadata', {}).get('args')}")
    print(f"    Hash: {r['content_hash'][:12]}")

trail.close()
```

Expected output:

```text
Total records: 1
Sources: {'tool': 1}
Provenance: {'MISSING': 1}
Chain intact: 1 records verified
  [adk:weather_lookup] args={'location': 'Seattle'}
    Hash: 3c8e4a1b0f5d
```

## Context manager pattern

```python
with ContextTrail(storage_path="adk_audit.db") as trail:
    callback = ProvenaADKCallback(trail=trail)
    agent = Agent(
        name="assistant",
        tools=[weather_lookup],
        after_tool_callback=callback.after_tool_call,
    )
    # Run agent...
    verdict = trail.verify_chain()
    print(f"Chain intact: {verdict.intact}")
# trail.close() is called automatically
```
