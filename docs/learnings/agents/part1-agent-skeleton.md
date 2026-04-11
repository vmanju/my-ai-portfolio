---
title: "Building a Production-Grade Agent Skeleton in Python"
date: 2025-04-10
description: "Four components every agent needs before you add any features: an async TAO loop, structured observability, a circuit breaker, and a model fallback router."
tags:
  - agents
  - python
  - asyncio
  - distributed-systems
  - ai-engineering
authors:
  - vmanju
---

# Building a Production-Grade Agent Skeleton in Python

Most agent tutorials end at "get a response from the API."

Production agents are different. Before you add tools, memory, or any feature — there are four things every agent needs from day one. Skip them and you'll be retrofitting into working code later.

This is Part 1 of my Distributed Agent Systems project series: the skeleton every subsequent part builds on.

---

## What We're Building

Four components, four files:

```
agent-skeleton/
├── agent.py           # Async TAO loop — the heartbeat
├── tracer.py          # Structured JSON observability
├── circuit_breaker.py # Three-state failure protection
└── router.py          # Model fallback chain
```

All runnable with just an Anthropic API key.

---

## Component 1: The Async TAO Loop

The TAO loop — **Thought → Action → Observation** — is the heartbeat of every agent. The design principle that matters most: **the loop is deliberately dumb**. No planning logic, no routing logic inside the harness. All intelligence lives in the model.

```python
@dataclass
class AgentResult:
    agent_id:   str
    task_id:    str
    content:    str
    tokens_in:  int
    tokens_out: int
    latency_ms: float
    cost_usd:   float
    status:     str   # success | error | timeout

class BaseAgent:
    MODEL       = "claude-sonnet-4-20250514"
    INPUT_COST  = 3.00  / 1_000_000
    OUTPUT_COST = 15.00 / 1_000_000

    def __init__(self, agent_id: str = None):
        self.agent_id = agent_id or str(uuid.uuid4())[:8]
        self.client   = AsyncAnthropic()  # async — never the sync client

    async def run(self, task: str) -> AgentResult:
        messages = [{"role": "user", "content": task}]

        while True:  # TAO loop — dumb by design
            response = await self.client.messages.create(
                model=self.MODEL, max_tokens=1024, messages=messages
            )
            if response.stop_reason != "tool_use":
                break  # final answer — exit
            # tool_use branch: execute and loop (Part 2 adds this)
            observations = await self._execute_tools(response.content)
            messages += [
                {"role": "assistant", "content": response.content},
                {"role": "user",      "content": observations},
            ]
        # ... capture tokens, cost, latency and return AgentResult
```

`AsyncAnthropic` is critical. The synchronous client blocks Python's event loop — only one API call at a time. The async client lets the loop switch to other coroutines while waiting on the API.

**Running 3 agents concurrently with `asyncio.gather()`:**

```python
agents  = [BaseAgent(f"agent-{i}") for i in range(3)]
tasks   = ["Explain CAP theorem", "What is idempotency?", "Define a saga pattern"]
results = await asyncio.gather(*[a.run(t) for a, t in zip(agents, tasks)])
```

Total wall time ≈ the slowest single call — not 3× the average.

---

## Component 2: Structured Observability

Cost surprises are one of the most common production AI issues. The fix is to instrument every invocation from day one — not after your first unexpected bill.

Every agent run emits a JSON log line:

```json
{
  "agent_id": "agent-0",
  "task_id": "a3f9c2",
  "model": "claude-sonnet-4-20250514",
  "tokens_in": 42,
  "tokens_out": 187,
  "latency_ms": 1823.4,
  "cost_usd": 0.003171,
  "status": "success"
}
```

The implementation is an async context manager:

```python
class AgentTracer:
    async def __aenter__(self):
        self._t0 = time.monotonic()
        return self.trace        # caller fills in token/cost fields

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.trace.latency_ms = round((time.monotonic() - self._t0) * 1000, 2)
        self.trace.status     = "error" if exc_type else "success"
        print(json.dumps(asdict(self.trace)))  # swap for Langfuse in Part 6
        return False
```

In Part 6 this `print()` becomes a Langfuse or OpenTelemetry span. The structure stays identical — only the sink changes.

---

## Component 3: The Circuit Breaker

LLM APIs go down. Rate limits get hit. Without a circuit breaker, every call to a failing service blocks for the full timeout — and timeouts stack, cascading into a full agent stall.

Three states:

```
CLOSED ──(3 failures)──► OPEN ──(30s elapsed)──► HALF_OPEN
  ▲                                                    │
  └──────────────(1 success)──────────────────────────┘
```

- **CLOSED**: normal — calls pass through
- **OPEN**: failing — reject calls immediately, no timeout wait
- **HALF_OPEN**: let one test call through to check recovery

```python
async def call(self, fn, *args, **kwargs):
    if self.state == CBState.OPEN:
        if self._should_attempt_reset():
            self.state = CBState.HALF_OPEN
        else:
            raise CircuitOpenError("Circuit OPEN — fast-fail")

    try:
        result = await fn(*args, **kwargs)
        self.state         = CBState.CLOSED
        self.failure_count = 0
        return result
    except Exception:
        self.failure_count += 1
        if self.failure_count >= self.failure_threshold:
            self.state      = CBState.OPEN
            self._opened_at = time.monotonic()
        raise
```

Once OPEN, calls are rejected in microseconds instead of waiting through a multi-second timeout. Under load, this is the difference between graceful degradation and cascading failure.

---

## Component 4: The Model Fallback Router

The circuit breaker handles failure. The fallback router handles graceful degradation — trying cheaper or faster alternatives before giving up entirely.

```python
FALLBACK_CHAIN = [
    ("claude-sonnet-4-20250514",  5.0),   # Tier 1: best quality
    ("claude-haiku-4-5-20251001", 10.0),  # Tier 2: fast + cheap
]

async def run(self, task: str) -> tuple[str, str]:
    for model, timeout in FALLBACK_CHAIN:
        try:
            async with asyncio.timeout(timeout):
                msg = await self.client.messages.create(
                    model=model, max_tokens=512,
                    messages=[{"role": "user", "content": task}]
                )
            return msg.content[0].text, model
        except asyncio.TimeoutError:
            print(f"[Router] {model} timed out — trying next tier")
        except Exception as e:
            print(f"[Router] {model} failed — trying next tier")

    return "⚠️ Stub: all models unavailable.", "stub"
```

The `model_used` return value is logged in the trace. If Haiku starts getting called frequently, that's an early signal that Sonnet is degraded — visible in logs before it becomes a formal incident.

---

## Wiring It Together

```python
class InstrumentedAgent(BaseAgent):
    def __init__(self, agent_id=None):
        super().__init__(agent_id)
        self.cb     = CircuitBreaker(failure_threshold=3, reset_timeout=30)
        self.router = ModelRouter()

    async def run(self, task: str) -> AgentResult:
        task_id = str(uuid.uuid4())[:8]
        async with AgentTracer(self.agent_id, task_id, self.MODEL) as trace:
            content, model_used = await self.cb.call(self.router.run, task)
            trace.model  = model_used
            trace.status = "success"
        return AgentResult(...)
```

Every request flows: **AgentTracer → CircuitBreaker → ModelRouter → TAO loop**. Failures are fast-failed by the breaker, degraded by the router, and every outcome is logged as structured JSON.

---

## The Interview Anchor

When asked *"How would you handle LLM API outages in production?"*:

> "My agent wraps every API call in a three-state circuit breaker. After 3 consecutive failures it trips OPEN and fast-fails all subsequent calls — no timeout stacking. Behind that is a model fallback router: Sonnet with a 5-second timeout, Haiku as Tier 2 at 10 seconds, then a cached stub. The router logs which tier was hit on every call, so I can see degradation in traces before it becomes an incident."

That answer comes from building it — not reading about it.

---

## What's Next

Part 2 adds tools and the saga pattern: idempotent tool calls and checkpoint-based workflow recovery that survives arbitrary mid-run crashes.

Full code: [github.com/vmanju/my-ai-portfolio](https://github.com/vmanju/my-ai-portfolio)
