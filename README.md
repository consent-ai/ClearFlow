# ClearFlow

[![codecov](https://codecov.io/gh/consent-ai/ClearFlow/graph/badge.svg?token=29YHLHUXN3)](https://codecov.io/gh/consent-ai/ClearFlow)
[![PyPI](https://badge.fury.io/py/clearflow.svg)](https://pypi.org/project/clearflow/)
![Python](https://img.shields.io/badge/Python-3.13%2B-blue)
![License](https://img.shields.io/badge/License-MIT-yellow)

Type-safe async workflow orchestration for LLM agents. **Explicit routing, immutable state, zero dependencies.** Read the entire source in 5 minutes.

---

## Why ClearFlow?

- **Predictable control flow** – explicit routes, no hidden magic  
- **Immutable, typed state** – frozen state passed via `NodeResult`  
- **One exit rule** – exactly one termination route enforced  
- **Tiny surface area** – one file, three concepts: `Node`, `NodeResult`, `Flow`  
- **100% test coverage** – every line tested  
- **Zero runtime deps** – bring your own clients (OpenAI, Anthropic, etc.)  

---

## Installation

```bash
pip install clearflow
```

---

## 60-second Quickstart

```python
from typing import TypedDict
from clearflow import Flow, Node, NodeResult

# 1) Define typed state
class ChatState(TypedDict):
    messages: list[dict[str, str]]

# 2) Define a node
class ChatNode(Node[ChatState]):
    async def exec(self, state: ChatState) -> NodeResult[ChatState]:
        # Call your LLM here
        # reply = await llm.chat(state["messages"])
        reply = {"role": "assistant", "content": "Hello!"}
        new_state: ChatState = {"messages": [*state["messages"], reply]}
        return NodeResult(new_state, outcome="success")

# 3) Build flow with explicit routing
chat = ChatNode()
flow = (
    Flow[ChatState]("ChatBot")
    .start_with(chat)
    .route(chat, "success", None)  # terminate on success
    .build()
)

# 4) Run it
async def main() -> None:
    result = await flow({"messages": [{"role": "user", "content": "Hi"}]})
    print(result.state["messages"][-1]["content"])  # "Hello!"

import asyncio
asyncio.run(main())
```

---

## Core Concepts

### `Node[T]`

A unit that transforms state of type `T`.

- `prep(state: T) -> T` – optional pre-work/validation  
- `exec(state: T) -> NodeResult[T]` – **required**; return new state + outcome  
- `post(result: NodeResult[T]) -> NodeResult[T]` – optional cleanup/logging  

Nodes are `async` and **pure** (no shared mutable state).

### `NodeResult[T]`

Holds the **new state** and an **outcome** string used for routing.

### `Flow[T]`

A fluent builder that wires nodes together with **explicit routing**:

```python
Flow[T]("Name")
  .start_with(a)
  .route(a, "ok", b)
  .route(b, "done", None)  # exactly one termination
  .build()                 # -> returns a Node[T] you can await
```

**Routing**: next node is `(from_node.name, outcome)`. If no name set, uses class name.  
**Nested flows**: a built flow is itself a `Node[T]` – compose flows within flows.

---

## Example: Multi-step Pipeline

```python
from typing import TypedDict
from clearflow import Flow, Node, NodeResult

class State(TypedDict):
    value: int

class Validate(Node[State]):
    async def exec(self, s: State) -> NodeResult[State]:
        return NodeResult(s, "valid" if s["value"] >= 0 else "invalid")

class Process(Node[State]):
    async def exec(self, s: State) -> NodeResult[State]:
        return NodeResult({"value": s["value"] * 2}, "success")

class Output(Node[State]):
    async def exec(self, s: State) -> NodeResult[State]:
        print("Final:", s["value"])
        return NodeResult(s, "done")

flow = (
    Flow[State]("Pipeline")
    .start_with(Validate())
    .route(Validate(), "valid", Process())
    .route(Validate(), "invalid", Output())  # route invalid to output
    .route(Process(), "success", Output())
    .route(Output(), "done", None)  # single termination point
    .build()
)

await flow({"value": 21})  # Final: 42
```

See more: [Chat example](examples/chat/) | [Structured output](examples/structured_output/)

---

## Testing Example

Nodes are easy to test in isolation because they are pure functions over typed state:

```python
import pytest
from clearflow import Node, NodeResult

class N(Node[int]):
    async def exec(self, x: int) -> NodeResult[int]:
        return NodeResult(x + 1, "ok")

@pytest.mark.asyncio
async def test_n() -> None:
    res = await N()(0)
    assert res.state == 1 and res.outcome == "ok"
```

---

## When to Use ClearFlow

- LLM workflows where you need explicit control  
- Systems requiring clear error handling paths  
- Projects with strict dependency requirements  
- Applications where debugging matters  

---

## ClearFlow vs PocketFlow

| Aspect | ClearFlow | PocketFlow |
|--------|-----------|------------|
| **State** | Immutable, passed via `NodeResult` | Shared store (mutable dict) |
| **Routing** | Explicit `(node, outcome)` routes | Graph with labeled edges |
| **Termination** | Exactly one `None` route enforced | Multiple exit patterns |
| **Type safety** | Full Python 3.13+ generics | Dynamic |
| **Lines** | 166 | 100 |

Both are minimalist. ClearFlow emphasizes **type safety and explicit control**. PocketFlow emphasizes **brevity and shared state**.

---

## Recipes

- **Guardrails**: validate node routes `"invalid"` → termination  
- **Retries**: node returns `"retry"` outcome → routes back to itself  
- **Sub-flows**: build child flow, use as node in parent  
- **Parallel**: multiple validate nodes → single process node  

---

## Development

```bash
# Install uv (if not already installed)
pip install --user uv   # or: pipx install uv

# Clone and set up development environment
git clone https://github.com/consent-ai/ClearFlow.git
cd ClearFlow
uv sync --group dev      # Creates venv and installs deps automatically
./quality-check.sh       # Run all checks
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)

---

## License

[MIT](LICENSE)

---

## Acknowledgments

Inspired by [PocketFlow](https://github.com/The-Pocket/PocketFlow)'s Node-Flow-State pattern.
