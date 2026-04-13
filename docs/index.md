# AI Engineering Portfolio

> Staff SWE learning AI engineering in public. Every project ships here.

I'm a Staff Software Engineer building a portfolio of production-grade AI systems.
Not tutorials. Not demos. Real systems with real tradeoffs, documented honestly.

## What I'm Building

6 phases. 15+ projects. Each one builds on the last.

| Phase | Focus | Projects | Status | Learnings |
|---|---|---|---|---|
| **1 — Foundation & Tooling** | Claude API deep dive, portfolio site | API lab, portfolio | 🔄 In progress | — |
| **2 — Prompt Engineering** | Prompts as code, security, injection defense | Prompt registry, Injection lab | ⏳ Planned | — |
| **3 — RAG & Data Pipelines** | Hybrid retrieval, embeddings, freshness | RAG system, Embedding pipeline | ⏳ Planned | — |
| **4 — Evals** | Measurement before features. Data, not vibes. | Eval harness, Eval-driven iteration | ⏳ Planned | — |
| **5 — Agents, Harness & Orchestration** | 12-component agent harness built part by part | Agent skeleton, Saga/tools, Cache/scheduler, Multi-agent CRDTs, Exec assistant + MCP | 🔄 In progress | [Part 1 →](learnings/agents/part1-agent-skeleton.md) |
| **6 — Production Systems** | Observability, reliability, cost governance, capstone | Observability stack, Reliability patterns, Full-stack capstone | ⏳ Planned | — |

### Phase 5 — Agent Systems Detail

Phase 5 is structured as a 7-part build series, each part adding one distributed systems concept to the previous:

| Part | What it builds | Distributed systems concept | Learnings |
|---|---|---|---|
| Part 0 | Architecture blueprint, 7 decisions | Harness design before code | — |
| **Part 1** | Async TAO loop, circuit breaker, model fallback router, structured observability | **Fault tolerance (#5), Observability (#6)** | [**Read →**](learnings/agents/part1-agent-skeleton.md) |
| Part 2 | Idempotent tool calls, saga orchestrator, checkpoint/resume | Transactions & sagas (#2) | — |
| Part 3 | Semantic cache, token-budget scheduler, priority queue | Caching (#4), Scheduling (#3), Rate limiting (#7) | — |
| Part 4 | Multi-agent CRDTs, Planner → Workers → Critic | Consistency models (#1), Consensus (#9) | — |
| Part 5 | Prompt injection defense, privilege separation | Security & trust boundaries (#10) | — |
| Part 6 | Hierarchical span tracing, LLM-as-judge eval harness, regression gate | Observability depth (#6) | — |
| Part 7 | FastAPI service, ADR, full eval run | Integration capstone | — |

## Principles I'm Following

- **No vibe coding** — I understand every line before it ships
- **Tests first** — Red/green TDD with Claude Code
- **Compound loop** — after every agent session, `CLAUDE.md` gets updated
- **Build in public** — every failure documented alongside every win
- **Distributed systems lens** — every agent design decision maps to a classical concept that breaks for agents, and is documented as such

## Start Here

- [All projects →](projects/index.md)
- [Concepts explained →](concepts/index.md)
- [Learnings log →](learnings/index.md)
