# AutoGen

Provena integrates with AutoGen through a message-processing hook that logs
inter-agent messages to your audit trail while passing them through unchanged.

## Installation

```bash
pip install provena[autogen]
```

This installs `autogen-agentchat>=0.4`. Note that `ProvenaAutoGenHook` interacts with AutoGen's standard hook registration API and can be used with any agent that supports hook registration.

## Quick start

```python
from provena import ContextTrail
from provena.integrations.autogen import ProvenaAutoGenHook

# Initialize the trail and hook
trail = ContextTrail(storage_path="autogen_audit.db")
hook = ProvenaAutoGenHook(trail=trail)

# Register the hook with an AutoGen agent
agent.register_hook(
    "process_message_before_send",
    hook.process_message,
)
```

## How it works

`ProvenaAutoGenHook` implements a callback compatible with AutoGen's `process_message_before_send` hook specification:

```python
def process_message(
    self, sender: Any, message: Any, recipient: Any, silent: bool
) -> Any:
    if isinstance(message, dict):
        content = message.get("content", str(message))  # (1)!
    else:
        content = str(message)  # (2)!

    sender_name = getattr(sender, "name", "unknown")  # (3)!
    recipient_name = getattr(recipient, "name", "unknown")  # (4)!

    self._trail.log(
        content=content,
        source=ContextSource.AGENT,
        source_name=f"autogen:{sender_name}",
        metadata={"recipient": recipient_name},
    )
    return message  # (5)!
```

1. If the message is a dictionary (such as `{"content": "...", "role": "assistant"}`), the hook extracts the `content` field. If `content` is missing, it falls back to stringifying the dictionary.
2. If the message is a string or other type, it is converted to a string.
3. The sender name is read from `sender.name` (defaults to `"unknown"` if absent).
4. The recipient name is read from `recipient.name` (defaults to `"unknown"` if absent).
5. **Pass-through behavior**: The hook returns `message` completely unchanged, ensuring the conversation flow and message structure are not altered.

### Logged attributes

| Attribute | Logged Value | Description |
|---|---|---|
| `content` | Extracted message string | Raw conversation text or payload |
| `source` | `ContextSource.AGENT` (`"agent"`) | Identifies the context input as agent communication |
| `source_name` | `autogen:<sender_name>` | Prefix plus sender's agent name |
| `metadata` | `{"recipient": "<recipient_name>"}` | Preserves communication topology |

!!! info "Multi-agent conversation tracking"
    Registering the hook on multiple agents in a group chat allows Provena to record every exchange between agents, creating a cryptographic audit trail of who said what to whom.

## Provenance tracking & limitations

AutoGen messages represent generated agent communications rather than ingested source documents with external origins.

- **No provenance auto-extraction**: `ProvenaAutoGenHook` does not set `ProvenanceMetadata` fields (`source_url`, `author`, `created_at`).
- **Default status**: All recorded agent messages receive a provenance status of `MISSING`.
- **Audit trail vs. data lineage**: The hook provides tamper-evident logging of agent dialog and decision-making chains, but does not trace source citations back to external documents.

!!! tip "Tracking external documents in AutoGen"
    If an AutoGen agent retrieves files or web content using tools, log the retrieved content with `trail.log(content=..., source=ContextSource.RETRIEVER, provenance=...)` inside your tool definition to establish data lineage for external materials.

## Full working example

```python
from autogen import AssistantAgent, UserProxyAgent
from provena import ContextTrail
from provena.integrations.autogen import ProvenaAutoGenHook

# Initialize trail and hook
trail = ContextTrail(storage_path="autogen_audit.db")
hook = ProvenaAutoGenHook(trail=trail)

# Configure AutoGen agents
config_list = [{"model": "gpt-4o", "api_key": "YOUR_API_KEY"}]

assistant = AssistantAgent(
    name="planner",
    llm_config={"config_list": config_list},
)

user_proxy = UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=1,
)

# Register the hook on agents before starting communication
assistant.register_hook("process_message_before_send", hook.process_message)
user_proxy.register_hook("process_message_before_send", hook.process_message)

# Initiate conversation
user_proxy.initiate_chat(
    assistant,
    message="Draft a plan for our data retention compliance audit.",
)
```

## Verifying the trail

Inspect and verify the cryptographic chain of agent messages:

```python
# Summary of audit trail
summary = trail.summary()
print(f"Total records: {summary['total']}")
print(f"Sources: {summary['sources']}")
print(f"Provenance: {summary['provenance']}")

# Verify hash chain integrity
verdict = trail.verify_chain()
assert verdict.intact, f"Chain broken: {verdict.details}"
print(f"Chain intact: {verdict.total_records} records verified")

# Inspect recorded dialog
records = trail.query(source="agent", limit=10)
for r in records:
    print(f"  [{r['source_name']}] -> recipient: {r.get('metadata', {}).get('recipient')}")
    print(f"    Hash: {r['content_hash'][:12]}")

trail.close()
```

Expected output:

```text
Total records: 2
Sources: {'agent': 2}
Provenance: {'MISSING': 2}
Chain intact: 2 records verified
  [autogen:user_proxy] -> recipient: planner
    Hash: a1b2c3d4e5f6
  [autogen:planner] -> recipient: user_proxy
    Hash: 7b8c9d0e1f2a
```

## Context manager pattern

```python
with ContextTrail(storage_path="autogen_audit.db") as trail:
    hook = ProvenaAutoGenHook(trail=trail)
    assistant.register_hook("process_message_before_send", hook.process_message)
    user_proxy.register_hook("process_message_before_send", hook.process_message)

    user_proxy.initiate_chat(assistant, message="Hello!")

    verdict = trail.verify_chain()
    print(f"Chain intact: {verdict.intact}")
# trail.close() is called automatically
```
