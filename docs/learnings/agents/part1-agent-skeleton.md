---
title: "Building a Production-Grade Agent Skeleton in Python"
date: 2025-04-10
description: "Four components every agent needs before you add any features: an async TAO loop, structured observability, a circuit breaker, and a model fallback router — and exactly how each one maps to the distributed systems concepts that break for agents."
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

Production agents are fundamentally different from classical distributed systems. Classical systems assume short-lived, stateless, bounded requests. Agents are long-lived, stateful, unbounded, and non-deterministic. Every design decision in this skeleton is a consequence of that gap — and every component maps to a specific distributed systems concept that breaks when you naively apply it to agents.

This is Part 1 of my Distributed Agent Systems project series. It builds the skeleton every subsequent part layers onto, and deliberately addresses two of the ten classical concepts that must change for agent-driven development — while scaffolding three more for later parts.

---

## What We're Building

Four components, four files:

```
agent-skeleton/
├── agent.py           # Async TAO loop — the heartbeat
├── tracer.py          # Structured JSON observability  (Concept #6)
├── circuit_breaker.py # Three-state failure protection (Concept #5)
└── router.py          # Model fallback chain            (Concept #5)
```

**Concepts fully implemented here:** #5 Fault Tolerance, #6 Observability

**Concepts scaffolded here, deepened in later parts:** #1 Consistency (Part 4), #2 Sagas (Part 2), #3 Scheduling (Part 3)

---

## Component 1: The Async TAO Loop

### The distributed systems context

Classical distributed systems assume request-response: one bounded operation, one result. Agents break this assumption. An agent task might run for 45 minutes, call 20 tools, and accumulate intermediate state across every turn. You can't model that as a request.

The TAO loop — **Thought → Action → Observation** — is the correct primitive. It's a resumable unit of execution, not a request. Each turn appends to `messages[]`, which is the agent's working memory (the in-context tier of a four-tier memory hierarchy that Parts 2–8 complete).

The design principle that matters: **the loop is deliberately dumb.** No planning logic, no routing logic in the harness. All intelligence lives in the model. This is also why this skeleton scaffolds Concept #1 (Consistency): the strongly-consistent session boundary is `messages[]` — everything inside one agent run is strongly consistent. Multi-agent consistency with CRDTs comes in Part 4.

```python
@dataclass
class AgentResult:
    agent_id:   str
    task_id:    str
    content:    str
    tokens_in:  int
    tokens_out: int
    latency_ms: float
    cost_usd:   float       # scaffolds Concept #3: token-budget scheduling
    status:     str         # success | error | timeout

class BaseAgent:
    MODEL       = "claude-sonnet-4-20250514"
    INPUT_COST  = 3.00  / 1_000_000
    OUTPUT_COST = 15.00 / 1_000_000

    async def run(self, task: str) -> AgentResult:
        messages = [{"role": "user", "content": task}]

        while True:                             # resumable — not a request
            response = await self.client.messages.create(
                model=self.MODEL, max_tokens=1024, messages=messages
            )
            if response.stop_reason != "tool_use":
                break                           # final answer — exit
            # tool_use: execute tools and loop (Part 2 adds full tool system)
            observations = await self._execute_tools(response.content)
            messages += [
                {"role": "assistant", "content": response.content},
                {"role": "user",      "content": observations},
            ]
        # capture tokens, cost, latency → return AgentResult
```

`AsyncAnthropic` is non-negotiable here. The synchronous client blocks Python's event loop — one API call at a time. The async client lets the loop switch to other coroutines while waiting for the API response, which is what makes `asyncio.gather()` actually concurrent:

```python
agents  = [BaseAgent(f"agent-{i}") for i in range(3)]
tasks   = ["Explain CAP theorem", "What is idempotency?", "Define a saga pattern"]
results = await asyncio.gather(*[a.run(t) for a, t in zip(agents, tasks)])
# total wall time ≈ slowest single call — not 3×
```

The `AgentResult` struct is also doing important scaffolding work. Its `cost_usd` and `tokens_out` fields are the telemetry a token-budget-aware scheduler runs on — you can't implement Concept #3 (preemptible, token-budget scheduling) without per-call cost data, so this is where that data collection starts.

---

## Component 2: Structured Observability

### The distributed systems context — Concept #6

Classical observability: trace the request. That's insufficient for agents.

The document is specific about what changes: agents need **agent step tracing + cost attribution + quality scores** — not just request tracing. The reason is that agent failures are often not failures in the traditional sense. The API returned 200. The circuit breaker didn't trip. But the agent ran 12 turns instead of 3, spent $0.40 on a task that should cost $0.003, and produced a hallucinated answer. None of that shows up in a request trace.

This component implements the cost attribution layer. Every invocation emits a structured JSON log line:

```json
{
  "agent_id": "agent-0",
  "task_id":  "a3f9c2",
  "model":    "claude-sonnet-4-20250514",
  "tokens_in":  42,
  "tokens_out": 187,
  "latency_ms": 1823.4,
  "cost_usd":   0.003171,
  "status":     "success"
}
```

The implementation is an async context manager that wraps every `run()` call:

```python
class AgentTracer:
    async def __aenter__(self):
        self._t0 = time.monotonic()
        return self.trace        # caller fills token/cost fields mid-run

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.trace.latency_ms = round((time.monotonic() - self._t0) * 1000, 2)
        self.trace.status     = "error" if exc_type else "success"
        print(json.dumps(asdict(self.trace)))  # → Langfuse/OTel in Part 6
        return False
```

Two design decisions embedded here that matter for later parts:

**The `print()` is intentional.** It's a deliberate placeholder. The structure of `AgentTrace` is stable — `agent_id`, `tokens_in`, `tokens_out`, `latency_ms`, `cost_usd`, `status`. Part 6 swaps the `print()` for a Langfuse span or OpenTelemetry exporter without changing the data shape. Designing the schema before the sink means you're not retrofitting observability into a system that wasn't built for it.

**`cost_usd` is the scaffolding for Concept #3.** Once this data is flowing, you have the input signal for a token-budget-aware priority queue scheduler. The scheduler in Part 3 reads per-call cost from these traces to make preemption decisions. The document is explicit: a scheduler that doesn't understand token budgets produces silent degradation — agents hit rate limits and degrade silently rather than triggering an alert.

---

## Component 3: The Circuit Breaker

### The distributed systems context — Concept #5

This is the concept where the classical pattern directly applies — and where agents make it harder.

The classic circuit breaker (Hystrix, Resilience4j) is binary: CLOSED (normal) or OPEN (failing). That's fine when downstream services are homogeneous. It breaks for agents because LLM APIs aren't homogeneous — there are multiple model tiers with different latency and cost profiles. A single open/closed state can't express "Sonnet is down but Haiku is available."

The circuit breaker here stays classical — three states, failure threshold, reset timeout — because it wraps individual model calls. The model-tier awareness lives in the router (Component 4), which sits in front of the breaker.

```
CLOSED ──(3 failures)──► OPEN ──(30s elapsed)──► HALF_OPEN
  ▲                                                    │
  └──────────────(1 success)──────────────────────────┘
```

- **CLOSED**: normal operation, calls pass through
- **OPEN**: failing — reject immediately in microseconds, no timeout wait
- **HALF_OPEN**: one test call to check if the service has recovered

```python
async def call(self, fn, *args, **kwargs):
    if self.state == CBState.OPEN:
        if self._should_attempt_reset():
            self.state = CBState.HALF_OPEN
        else:
            raise CircuitOpenError("fast-fail")   # μs, not seconds

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

The critical behavior is what happens in OPEN state. Without the breaker, every call to a failing LLM API blocks for its full timeout (5–30 seconds). Under any meaningful load, these timeouts stack. The circuit breaker converts that from a latency cascade into an immediate rejection — the caller gets a `CircuitOpenError` in microseconds and can react (queue the task, return a stub, retry later) rather than stalling.

This also matters for Concept #2 (Sagas). The document says agent tool calls are often non-compensable — you can't un-send an email. The circuit breaker prevents a corrupted intermediate state from propagating: if the LLM is returning garbage before the circuit trips, the breaker stops the saga from proceeding to a non-compensable step on bad data. Part 2 makes this explicit with the saga's human approval gate for non-compensable steps.

---

## Component 4: The Model Fallback Router

### The distributed systems context — Concept #5 continued

The document names this exactly: "Model fallback routing: if Claude Sonnet is unavailable, fall back to Claude Haiku, then to a cached response, then graceful degradation. Your circuit breaker needs to understand model tiers, not just service health."

The router implements that three-tier fallback chain, with per-tier timeouts that reflect each model's latency profile:

```python
FALLBACK_CHAIN = [
    ("claude-sonnet-4-20250514",  5.0),    # Tier 1: best quality, strict SLA
    ("claude-haiku-4-5-20251001", 10.0),   # Tier 2: fast + cheap, looser SLA
]

async def run(self, task: str) -> tuple[str, str]:
    for model, timeout in FALLBACK_CHAIN:
        try:
            async with asyncio.timeout(timeout):
                msg = await self.client.messages.create(
                    model=model, max_tokens=512,
                    messages=[{"role": "user", "content": task}]
                )
            return msg.content[0].text, model    # return which tier was used
        except asyncio.TimeoutError:
            print(f"[Router] {model} timed out — next tier")
        except Exception:
            print(f"[Router] {model} failed — next tier")

    return "⚠️ Stub: all models unavailable.", "stub"
```

The `model_used` return value is the operational signal. It flows into `AgentTrace.model`, which means every log line records which tier served the request. When Haiku starts appearing in traces more frequently than baseline, that's Sonnet degrading — visible in your observability layer (Component 2) before it escalates to a formal incident. The document frames this as the goal: "your circuit breaker needs to understand model tiers, not just service health."

The Sonnet 5-second timeout is deliberate. Sonnet's p95 latency at normal load is well under 5 seconds. A 5-second timeout means you're not waiting out a genuine outage — you're catching tail latency that indicates service degradation and routing around it before the user notices.

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

Every request flows: **AgentTracer → CircuitBreaker → ModelRouter → TAO loop**. This ordering matters. The tracer wraps everything — including circuit open events and fallback tier switches — so every outcome is observable. The breaker gates the router call, so a tripped circuit never reaches the API. The router's tier selection is logged in the trace, so degradation patterns emerge from the data rather than from alerts.

---

## What This Scaffolds for Later Parts

Part 1 completes two of the ten classical concepts that must change for agents. Five more are scaffolded here — the data exists, the structure is right, but the full implementation waits until the concepts become necessary:

**Concept #1 — Consistency:** The strongly-consistent session boundary is `messages[]`. Part 4 adds CRDT-based shared scratchpad for multi-agent state, where merge safety becomes the hard problem.

**Concept #2 — Sagas:** `AgentResult` is the checkpoint shape. The TAO loop is the resumable unit. Part 2 adds JSON-backed saga state with idempotency keys on every tool call, and classifies tool calls as compensable vs non-compensable before execution.

**Concept #3 — Scheduling:** `cost_usd` per invocation is the input signal for a token-budget-aware scheduler. Part 3 builds the three-tier priority queue that uses this data to apply backpressure before hitting API rate limits.

**Concept #7 — Rate Limiting:** `tokens_in + tokens_out` per call is the meter. The router's per-tier timeouts are primitive rate-limit protection. Part 3 adds full two-sided rate limiting: inbound (protecting against user abuse) and outbound (protecting against upstream LLM provider limits).

**Concept #9 — Coordination:** `BaseAgent` is the simplest valid orchestrator — single authority, no peer coordination, no consensus problem. Part 4 introduces the Planner → Workers → Critic pattern where coordination overhead becomes real.

The three concepts not touched here — #4 (semantic caching), #8 (four-tier agent memory), and #10 (prompt injection defense) — are deferred because their value depends on having the cost data and tool system from Parts 1 and 2 first. Semantic caching ROI is a function of call volume and per-call cost. You can't make that argument without the `cost_usd` instrumentation from this part.

---

## The Interview Anchors

**On fault tolerance (Concept #5):** "I modeled the circuit breaker and the fallback router as separate layers — the breaker handles service health (binary: is this tier responding?), the router handles model-tier economics (Sonnet vs Haiku vs stub). Combining them into one component would mean the breaker has to understand pricing, which is the wrong coupling. The `model_used` field in my traces makes degradation visible before it becomes an incident."

**On observability (Concept #6):** "Classical request tracing doesn't catch the agent failure modes that matter — an agent that ran 12 turns instead of 3 and spent 100× the expected cost returns HTTP 200 with no visible error. My `AgentTracer` emits cost attribution per invocation from the start. The schema is stable; the sink (print → Langfuse → OpenTelemetry) upgrades in Part 6 without touching the data shape."

**On what's deferred and why:** "I didn't implement semantic caching in Part 1 because the investment needs to be justified by cost data. At $0.003 per call you need meaningful volume before caching ROI turns positive. Part 3 implements it once the tracer proves the economics at scale."

---

## What's Next

Part 2 turns the TAO loop into a saga-orchestrated workflow: idempotent tool calls with UUID deduplication, JSON checkpoint/resume that survives arbitrary mid-run crashes, non-compensable step classification, and a chaos test that verifies the system recovers from random failures.

Full code: [github.com/vmanju/my-ai-portfolio](https://github.com/vmanju/my-ai-portfolio)
