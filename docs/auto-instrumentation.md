# Auto-Instrumentation

Torrix v0.2.0 adds zero-config auto-instrumentation for the Python SDK. One `torrix.init()` call patches the OpenAI and Anthropic libraries at the module level so every LLM call made anywhere in your process is traced automatically — including calls made by agent frameworks that create their own clients internally.

## Why this matters

Frameworks like LangGraph, CrewAI, and AutoGen create OpenAI or Anthropic client instances internally. Calling `torrix.wrap(client)` on a client you don't control is impossible. Auto-instrumentation solves this by patching the underlying class method rather than individual instances.

## Usage

```python
import torrix

torrix.init(
    api_key="trxk_...",
    base_url="http://localhost:8088",
)

# From here on, every OpenAI and Anthropic call is traced automatically.
# No torrix.wrap() needed.

from openai import OpenAI
client = OpenAI(api_key="sk-...")
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello"}],
)
```

This works for:

- Clients you create directly
- Clients created by LangGraph nodes
- Clients created by CrewAI agents
- Clients created by AutoGen agents
- Any other code that imports and uses OpenAI or Anthropic after `torrix.init()` is called

## Explicit wrapping (still supported)

If `openai` or `anthropic` are not installed, `torrix.init()` completes without error and simply skips patching those libraries.

If you call `torrix.wrap(client)` after `torrix.init()`, it returns the original client unchanged to avoid double-tracing. Both patterns are safe to use together.

```python
# After torrix.init(), wrap() is a no-op — safe to call but not required
wrapped = torrix.wrap(client)  # returns client unchanged
```

## Example with a LangGraph-style pattern

```python
import torrix
from openai import OpenAI

torrix.init(api_key="trxk_...", base_url="http://localhost:8088")

def agent_node(state):
    # This client is created inside the node — auto-instrumentation captures it
    client = OpenAI(api_key="sk-...")
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=state["messages"],
    )
    return {"output": response.choices[0].message.content}
```

## Version history

- `0.2.0` — added `torrix.init()` auto-instrumentation via module-level monkey-patching
- `0.1.0` — explicit `torrix.wrap()` pattern for OpenAI and Anthropic clients
