# OpenAI Agents SDK

Provena integrates with the OpenAI Agents SDK through run hooks that record
tool outputs and agent handoff events to a tamper-evident audit trail asynchronously.

## Installation

```bash
pip install provena[openai-agents]
```

This installs `openai-agents>=0.1`.

## Quick start

```python
from agents import Agent, Runner
from provena import ContextTrail
from provena.integrations.openai_agents import ProvenaRunHooks

# Initialize the trail and hooks
trail = ContextTrail(storage_path="agents_audit.db")
hooks = ProvenaRunHooks(trail=trail)

agent = Agent(name="researcher", instructions="Conduct research and use tools.")

# Pass the hooks instance to Runner.run
result = await Runner.run(agent, input="Find latest report", hooks=hooks)
```

## How it works

`ProvenaRunHooks` extends `agents.RunHooks[Any]`. It hooks into two core execution lifecycle events:

### `on_tool_end` -- tool completion

When a tool finishes execution, `on_tool_end` logs the tool result:

```python
async def on_tool_end(
    self,
    context: RunHooksContext[Any],
    agent: Any,
    tool: Any,
    result: str,
) -> None:
    tool_name = getattr(tool, "name", "unknown")  # (1)!
    agent_name = getattr(agent, "name", "unknown")  # (2)!
    await asyncio.to_thread(  # (3)!
        self._trail.log,
        content=result,
        source=ContextSource.TOOL,
        source_name=f"openai:{tool_name}",
        metadata={"agent": agent_name},
    )
```

1. Tool name is resolved from `tool.name` (fallback `"unknown"`).
2. Agent name is resolved from `agent.name` (fallback `"unknown"`).
3. **Non-blocking asynchronous execution**: Logging is executed on a worker thread via `asyncio.to_thread()`, ensuring database writes do not block the active asyncio event loop.
4. Recorded with source type `TOOL`, source name `openai:<tool_name>`, and metadata `{"agent": agent_name}`.

### `on_handoff` -- agent delegation

When execution is handed off from one agent to another, `on_handoff` captures the transition:

```python
async def on_handoff(
    self,
    context: RunHooksContext[Any],
    from_agent: Any,
    to_agent: Any,
) -> None:
    from_name = getattr(from_agent, "name", "unknown")  # (1)!
    to_name = getattr(to_agent, "name", "unknown")  # (2)!
    await asyncio.to_thread(  # (3)!
        self._trail.log,
        content=f"Handoff from {from_name} to {to_name}",
        source=ContextSource.AGENT,
        source_name=f"openai:{from_name}",
        metadata={"to_agent": to_name},
    )
```

1. Source agent name from `from_agent.name` (fallback `"unknown"`).
2. Target agent name from `to_agent.name` (fallback `"unknown"`).
3. Offloaded to a worker thread via `asyncio.to_thread()`.
4. Recorded with source type `AGENT`, source name `openai:<from_name>`, content `"Handoff from <from_name> to <to_name>"`, and metadata `{"to_agent": to_name}`.

### Logged attributes summary

| Event | Source Type | Source Name | Logged Content | Metadata |
|---|---|---|---|---|
| Tool execution | `ContextSource.TOOL` (`"tool"`) | `openai:<tool_name>` | String tool result | `{"agent": "<agent_name>"}` |
| Agent handoff | `ContextSource.AGENT` (`"agent"`) | `openai:<from_name>` | `"Handoff from X to Y"` | `{"to_agent": "<to_agent>"}` |

!!! info "Thread safety and async design"
    Because Provena's backend storage and hash calculation perform synchronous disk I/O, `ProvenaRunHooks` uses `asyncio.to_thread` for all trail logging operations. This guarantees high-throughput asynchronous execution without event loop latency spikes.

## Provenance tracking & limitations

- **No provenance auto-extraction**: Neither `on_tool_end` nor `on_handoff` extracts `ProvenanceMetadata` fields (`source_url`, `author`, `created_at`).
- **Default status**: Records logged through `ProvenaRunHooks` have provenance status `MISSING`.
- **Scope**: The hooks record tool outputs and agent handoffs. Model inputs and internal chain-of-thought tokens are not captured unless passed through tool responses.

!!! tip "Adding provenance to tool results"
    If your tools query external APIs or databases, you can log the raw external response with full `ProvenanceMetadata` before returning the formatted string to the agent.

## Full working example

```python
import asyncio
from agents import Agent, Runner, function_tool
from provena import ContextTrail
from provena.integrations.openai_agents import ProvenaRunHooks

# Define a tool
@function_tool
def fetch_system_status() -> str:
    """Check the health status of system services."""
    return "All systems operational. Latency 14ms."

# Define specialist agents
triage_agent = Agent(
    name="triage_agent",
    instructions="Review requests and hand off to specialist.",
)

status_agent = Agent(
    name="status_agent",
    instructions="Report system status using available tools.",
    tools=[fetch_system_status],
)

# Allow triage agent to hand off to status agent
triage_agent.handoffs = [status_agent]

async def main():
    trail = ContextTrail(storage_path="agents_audit.db")
    hooks = ProvenaRunHooks(trail=trail)

    # Execute workflow with hooks
    result = await Runner.run(
        triage_agent,
        input="Check the system status please.",
        hooks=hooks,
    )
    print(f"Final output: {result.final_output}")

    # Verify audit trail
    verdict = trail.verify_chain()
    print(f"Chain intact: {verdict.intact} ({verdict.total_records} records verified)")
    trail.close()

if __name__ == "__main__":
    asyncio.run(main())
```

## Verifying the trail

After execution, verify the cryptographic trail of tools and handoffs:

```python
summary = trail.summary()
print(f"Total records: {summary['total']}")
print(f"Sources: {summary['sources']}")

verdict = trail.verify_chain()
assert verdict.intact, f"Chain broken: {verdict.details}"
print(f"Chain intact: {verdict.total_records} records verified")

records = trail.query(limit=10)
for r in records:
    print(f"  [{r['source']}] {r['source_name']}: {r['content'][:40]}...")

trail.close()
```

Expected output:

```text
Total records: 2
Sources: {'agent': 1, 'tool': 1}
Chain intact: 2 records verified
  [agent] openai:triage_agent: Handoff from triage_agent to status_agent...
  [tool] openai:fetch_system_status: All systems operational. Latency 14ms...
```

## Context manager pattern

```python
with ContextTrail(storage_path="agents_audit.db") as trail:
    hooks = ProvenaRunHooks(trail=trail)
    result = await Runner.run(agent, input="Task description", hooks=hooks)
    verdict = trail.verify_chain()
    print(f"Chain intact: {verdict.intact}")
# trail.close() is called automatically
```
