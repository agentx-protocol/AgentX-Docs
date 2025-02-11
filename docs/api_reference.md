# AgentX-Core API Reference

**Version:** 0.2.1
**Python:** 3.10+

---

## Table of Contents

- [Agent](#agent)
- [AgentResult](#agentresult)
- [Tool](#tool)
- [ShortTermMemory](#shorttermmemoryx)
- [Utilities](#utilities)
- [Exceptions](#exceptions)

---

## Agent

```python
class Agent(
    name: str = "agentx-agent",
    model: str = "gpt-4o",
    tools: list[Tool] | None = None,
    system_prompt: str = "",
    memory: ShortTermMemory | None = None,
    max_iterations: int = 15,
    runtime: SolanaRuntime | None = None,
    verbose: bool = False,
)
```

The core autonomous agent class. Implements a ReAct (Reason → Act → Observe) loop powered by an LLM backend.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | `"agentx-agent"` | Human-readable agent identifier |
| `model` | `str` | `"gpt-4o"` | LLM model ID. Supports OpenAI (`gpt-*`, `o1-*`) and Anthropic (`claude-*`) models |
| `tools` | `list[Tool]` | `[]` | List of tools the agent can use |
| `system_prompt` | `str` | built-in | System instructions that define agent persona and behavior |
| `memory` | `ShortTermMemory` | auto | Conversation buffer. Pass a shared instance for multi-turn conversations |
| `max_iterations` | `int` | `15` | Hard cap on ReAct loop iterations |
| `runtime` | `SolanaRuntime` | `None` | Solana runtime for on-chain operations |
| `verbose` | `bool` | `False` | Print each iteration step to stdout |

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `agent_id` | `str` | 8-char UUID fragment (unique per instance) |
| `on_chain_address` | `str \| None` | Solana PDA address after `deploy()` |

### Methods

#### `run(task: str) -> AgentResult`

Run the agent on a task string. The agent loops using ReAct until it produces a `FINAL ANSWER:` or hits `max_iterations`.

```python
result = agent.run("What is the current price of SOL?")
print(result.output)   # "The current price of SOL is $145.20"
```

**Parameters:**
- `task` — natural language task description

**Returns:** [`AgentResult`](#agentresult)

---

#### `run_until_done(task: str, timeout_seconds: float = 120.0) -> AgentResult`

Like `run()`, but retries automatically on rate-limit errors with exponential back-off until `timeout_seconds` is reached.

```python
result = agent.run_until_done("Draft a 500-word report on SOL.", timeout_seconds=60.0)
```

---

#### `deploy() -> str`

Register this agent on the AgentX Solana program. Requires `runtime` to be set.

```python
from agentx.runtime import SolanaRuntime

agent = Agent(
    name="my-agent",
    runtime=SolanaRuntime(rpc_url="https://api.devnet.solana.com"),
)
address = agent.deploy()
print(address)   # "AxPDA..."
```

**Returns:** `str` — the agent's on-chain PDA address

**Raises:** `RuntimeError` if `runtime` is not configured

---

## AgentResult

```python
@dataclass
class AgentResult:
    output: str
    iterations: int
    tool_calls: list[dict]
    success: bool
    error: str | None
    metadata: dict
```

Structured output returned by `Agent.run()`.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `output` | `str` | Final answer text from the agent |
| `iterations` | `int` | Number of ReAct loop iterations used |
| `tool_calls` | `list[dict]` | List of `{"tool": name, "args": {...}, "result": ...}` |
| `success` | `bool` | `True` if agent produced a `FINAL ANSWER:` |
| `error` | `str \| None` | Error message if `success=False` |
| `metadata` | `dict` | Extra data (e.g. `{"attempts": 2}` from `run_until_done`) |

### Example

```python
result = agent.run("Get the SOL price and suggest a trade.")

if result.success:
    print(result.output)
    print(f"Used {result.iterations} iterations and {len(result.tool_calls)} tool calls")
else:
    print(f"Failed: {result.error}")
```

---

## Tool

```python
class Tool(func: Callable, name: str = "", description: str = "")
```

Wraps any Python callable as an AgentX tool that the agent can invoke.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `func` | `Callable` | Python function to wrap |
| `name` | `str` | Tool name (defaults to `func.__name__`) |
| `description` | `str` | Tool description for the LLM (defaults to docstring) |

### Methods

#### `__call__(**kwargs) -> Any`

Invoke the tool. Errors are caught and returned as `"ERROR: ..."` strings.

#### `schema() -> dict`

Returns OpenAI-compatible function schema for tool use.

### `@tool` decorator

```python
from agentx.core import tool

@tool(name="get_price", description="Fetch token price in USD")
def get_token_price(symbol: str) -> str:
    """Fetch price for a given token symbol."""
    return f"{symbol}: $100.00"  # replace with real API
```

### Example

```python
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression."""
    try:
        return str(eval(expression, {"__builtins__": {}}, {}))
    except Exception as e:
        return f"Error: {e}"

calc_tool = Tool(calculator, name="calculator", description="Evaluate math expressions")

agent = Agent(name="math-agent", tools=[calc_tool])
result = agent.run("What is 137 * 842 + 17?")
```

---

## ShortTermMemory

```python
class ShortTermMemory(max_tokens: int = 8192)
```

In-process conversation buffer. Pass a single instance to an agent for multi-turn interactions.

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_tokens` | `int` | `8192` | Approximate token budget. Older messages are dropped when exceeded |

### Methods

| Method | Description |
|--------|-------------|
| `add(message: Message)` | Append a message to the buffer |
| `get_history() -> list[Message]` | Return all messages |
| `clear()` | Reset the buffer |

### Example: Multi-turn conversation

```python
from agentx.core import Agent, ShortTermMemory

memory = ShortTermMemory(max_tokens=16384)
agent = Agent(name="assistant", memory=memory)

r1 = agent.run("My name is Alice.")
r2 = agent.run("What is my name?")   # Agent remembers "Alice"
print(r2.output)  # "Your name is Alice."
```

---

## Utilities

### `agentx.utils`

#### `get_logger(name: str) -> structlog.BoundLogger`

Return a namespaced structlog logger.

```python
from agentx.utils import get_logger
logger = get_logger(__name__)
logger.info("event", key="value")
```

#### `call_llm_api(model, messages, tools, temperature, max_tokens, api_key) -> dict`

Low-level LLM call. Normalised response:
```python
{
    "content": "...",        # text response
    "tool_calls": [...],     # list of tool call objects
    "error": None,           # error string or None
}
```

#### `http_get(url, params, timeout) -> dict`

Safe GET with error handling. Returns `{"data": ..., "error": ...}`.

#### `http_post(url, payload, headers, timeout) -> dict`

Safe POST with error handling.

#### `@timed`

Decorator that logs execution time:
```python
from agentx.utils import timed

@timed
def my_slow_function():
    time.sleep(1)
```

---

## Exceptions

| Exception | Module | When raised |
|-----------|--------|-------------|
| `RuntimeError` | built-in | `Agent.deploy()` called without runtime |
| `requests.exceptions.HTTPError` | requests | LLM API returns 4xx/5xx (after retries) |

---

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENAI_API_KEY` | For OpenAI models | — | OpenAI API key |
| `ANTHROPIC_API_KEY` | For Claude models | — | Anthropic API key |
| `AGENTX_MODEL` | No | `gpt-4o` | Default model for examples |
| `AGENTX_ENV` | No | `development` | Set to `production` for JSON logging |
| `AGENTX_LOG_LEVEL` | No | `20` (INFO) | Python logging level integer |
