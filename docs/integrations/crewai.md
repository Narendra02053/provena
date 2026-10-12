# CrewAI

Provena integrates with CrewAI through an event listener that automatically
captures tool executions and agent task completions to your tamper-evident audit trail.

## Installation

```bash
pip install provena[crewai]
```

This installs `crewai>=1.0.0`. If you already have CrewAI installed, the extra ensures version compatibility with CrewAI's class-based event system.

## Quick start

```python
from crewai import Agent, Crew, Task
from provena import ContextTrail
from provena.integrations.crewai import ProvenaCrewListener

# Initialize the trail and listener
trail = ContextTrail(storage_path="crew_audit.db")
listener = ProvenaCrewListener(trail=trail)

# Run your Crew as usual — events are automatically captured
crew = Crew(
    agents=[...],
    tasks=[...],
)
result = crew.kickoff()
```

## How it works

`ProvenaCrewListener` subclasses CrewAI's `BaseEventListener` (from `crewai.events`). When initialized, it implements `setup_listeners(crewai_event_bus)` to subscribe to two core lifecycle events on CrewAI's event bus:

### `ToolUsageFinishedEvent` -- tool outputs

When a tool finishes running, the listener extracts the output and records a trail entry with source type `TOOL`:

```python
def _log_tool_usage_finished(self, event: ToolUsageFinishedEvent) -> None:
    output = getattr(event, "output", None)
    if output is None:
        return
    tool_name = getattr(event, "tool_name", "unknown")
    self._trail.log(
        content=str(output),  # (1)!
        source=ContextSource.TOOL,  # (2)!
        source_name=f"crewai:{tool_name}",  # (3)!
    )
```

1. Non-string outputs (e.g., dictionaries or result objects) are safely coerced to strings.
2. Logged with `ContextSource.TOOL` (`"tool"`).
3. The source name format is `crewai:<tool_name>`, or `crewai:unknown` if the tool name attribute is absent.

If `event.output` is `None`, the event is skipped and no record is created.

### `AgentExecutionCompletedEvent` -- agent outputs

When an agent completes a task, the listener captures the result and records a trail entry with source type `AGENT`:

```python
def _log_agent_execution_completed(
    self, event: AgentExecutionCompletedEvent
) -> None:
    output = getattr(event, "output", None)
    if output is None:
        return
    agent = getattr(event, "agent", None)
    agent_role = getattr(agent, "role", None) or "unknown"
    self._trail.log(
        content=str(output),  # (1)!
        source=ContextSource.AGENT,  # (2)!
        source_name=f"crewai:{agent_role}",  # (3)!
    )
```

1. The output content is coerced to a string.
2. Logged with `ContextSource.AGENT` (`"agent"`).
3. The source name format is `crewai:<role>`, taking the agent's role (e.g., `crewai:Researcher`), or `crewai:unknown` if unspecified.

If `event.output` is `None`, no record is logged.

!!! info "Automatic event bus registration"
    CrewAI's `BaseEventListener` registers listeners directly with the global `crewai_event_bus` upon instantiation. You do not need to manually attach the listener to the `Crew` or `Agent` objects—instantiating `ProvenaCrewListener(trail=trail)` before calling `crew.kickoff()` is sufficient.

## Provenance tracking & limitations

Unlike document retrievers (such as LangChain or LlamaIndex) where documents carry metadata keys like `source` or `author`, CrewAI's event objects do not contain structured provenance metadata.

- **No automatic provenance extraction**: Records created by `ProvenaCrewListener` do not populate `ProvenanceMetadata` fields (`source_url`, `author`, `created_at`).
- **Default compliance status**: Because provenance fields are omitted, all records logged via `ProvenaCrewListener` are marked with provenance status `MISSING`.
- **Audit-trail logging vs. provenance**: `ProvenaCrewListener` provides tamper-evident audit logging (SHA-256 content hashing, HMAC signing, hash chaining, and timestamping) of what tools and agents produced, but does not track the original external source of documents unless manually logged.

!!! tip "Handling provenance requirements"
    If your compliance policy requires `VALID` provenance status (e.g., EU AI Act Article 10 data governance), you can manually log external data inputs with `trail.log(content=..., source="...", provenance=ProvenanceMetadata(...))` before passing them to agents, or annotate recorded entries using `trail.annotate()`.

## Full working example

```python
from crewai import Agent, Crew, Process, Task
from crewai.tools import tool

from provena import ContextTrail
from provena.integrations.crewai import ProvenaCrewListener

# Define a tool
@tool("search_market_data")
def search_market_data(query: str) -> str:
    """Look up market statistics for a query."""
    return f"Market data for '{query}': Revenue grew by 18% in Q3."

# Set up the Provena audit trail and listener
trail = ContextTrail(storage_path="crew_audit.db")
listener = ProvenaCrewListener(trail=trail)

# Define agents
analyst = Agent(
    role="Financial Analyst",
    goal="Analyze market conditions",
    backstory="Senior analyst evaluating quarterly performance.",
    tools=[search_market_data],
    verbose=False,
)

# Define tasks
analysis_task = Task(
    description="Research market conditions for enterprise cloud software.",
    expected_output="A 2-sentence summary of market performance.",
    agent=analyst,
)

# Create crew and execute
crew = Crew(
    agents=[analyst],
    tasks=[analysis_task],
    process=Process.sequential,
)

output = crew.kickoff()
print(f"Crew output: {output}")
```

## Verifying the trail

After execution, verify the cryptographic integrity of the actions taken by your crew:

```python
# Check recorded summary
summary = trail.summary()
print(f"Total records: {summary['total']}")
print(f"Sources: {summary['sources']}")
print(f"Provenance: {summary['provenance']}")

# Verify SHA-256 hash chain
verdict = trail.verify_chain()
assert verdict.intact, f"Chain broken: {verdict.details}"
print(f"Chain intact: {verdict.total_records} records verified")

# Query recorded entries
records = trail.query(limit=10)
for r in records:
    print(f"  [{r['source']}] {r['source_name']} - {r['content_hash'][:12]}")

trail.close()
```

Expected output:

```text
Total records: 2
Sources: {'tool': 1, 'agent': 1}
Provenance: {'MISSING': 2}
Chain intact: 2 records verified
  [tool] crewai:search_market_data - e3b0c44298fc
  [agent] crewai:Financial Analyst - 4f53cda18c2b
```

## Context manager pattern

Use `ContextTrail` as a context manager to ensure database connections and buffers are automatically flushed and closed:

```python
with ContextTrail(storage_path="crew_audit.db") as trail:
    listener = ProvenaCrewListener(trail=trail)
    crew = Crew(agents=[analyst], tasks=[analysis_task])
    crew.kickoff()

    verdict = trail.verify_chain()
    print(f"Chain intact: {verdict.intact}")
# trail.close() is called automatically
```
