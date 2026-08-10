# Strategy Optimizer Agent — Living Architecture

> **Status:** brainstorm / design  
> **Last updated:** 2026-08-10  
> Keep this document updated as the design evolves. It is the source of truth for high-level structure, important classes/functions, and trade-offs.

---

## 1. Product idea (one sentence)

A always-on local agent that, for each goal, repeatedly **strategizes** (propose processes) and **optimizes** (score, refine, prune) using a local LLM plus gathered external knowledge, until it converges on a strong process—and keeps improving as new information arrives.

---

## 2. Core loop (the product)

```
                 ┌─────────────────────────────────────────┐
                 │              Goal Store                 │
                 │  (many goals, states, priorities)       │
                 └─────────────────┬───────────────────────┘
                                   │ pick / schedule
                                   ▼
┌──────────────┐   context    ┌──────────────┐   candidates   ┌──────────────┐
│  Knowledge   │─────────────▶│  Strategist  │───────────────▶│   Process    │
│  Gatherer    │              │  (LLM)       │                │   Store      │
└──────▲───────┘              └──────────────┘                └──────┬───────┘
       │                             ▲                               │
       │ new queries                 │ critique / revise             │ score
       │                             │                               ▼
┌──────┴───────┐              ┌──────┴───────┐                ┌──────────────┐
│  Sources     │◀─────────────│  Optimizer   │◀───────────────│  Evaluator   │
│  (web/files/ │   gaps       │  (LLM +      │   metrics      │  (rubric /   │
│   APIs/RAG)  │              │   search)    │                │   sim / LLM) │
└──────────────┘              └──────────────┘                └──────────────┘
```

For every active goal, the daemon cycles:

1. **Understand** — restates the goal, constraints, success criteria.
2. **Gather** — pulls external facts relevant to gaps in the current strategy.
3. **Strategize** — LLM proposes or revises processes (step graphs, not vague advice).
4. **Evaluate** — scores processes on feasibility, cost, risk, evidence strength, goal fit.
5. **Optimize** — mutate / merge / prune strategies; request more info where weak.
6. **Persist & sleep** — save artifacts, schedule next cycle (or next goal).

This is continuous: strategies are living objects that improve over time, not one-shot plans.

---

## 3. Recommended stack (small local server)

| Layer | Choice | Why |
|-------|--------|-----|
| Runtime | Python 3.11+ | Fastest path for LLM tooling, scrapers, async loops |
| Local LLM | Ollama (or llama.cpp server) | Simple local HTTP API; swap models without code changes |
| API / UI | FastAPI + minimal local dashboard | Lightweight control plane for goals & strategy inspection |
| Persistence | SQLite (+ optional Chroma/LanceDB) | Zero ops on a small box; good enough for many goals |
| Scheduler | asyncio loop + priority queue | Always-on; fair multi-goal scheduling |
| Packaging | Docker Compose optional | Ollama + app as two services |

**Trade-off:** Python + SQLite is not “enterprise scale,” but matches “small local server, always running.” We can later swap SQLite → Postgres and the gatherer → richer connectors without changing the strategist/optimizer contracts.

---

## 4. Knowledge feeding (how information gets in)

You were unsure whether it should be “web or something else.” Use a **pluggable Knowledge Gatherer** with multiple adapters. Start narrow; expand later.

### 4.1 Sources (priority order for v1)

1. **Local corpus / notes** (user-provided files, markdown, PDFs) — highest signal, private.
2. **Web search + fetch** (e.g. DuckDuckGo / SearXNG + readability extract) — broad external knowledge.
3. **Structured APIs** (optional per goal: prices, docs, GitHub, calendars) — precise facts.
4. **LLM internal knowledge** — always used as prior; never the only source for factual claims that must be current.

### 4.2 How gather works each cycle

- Strategist / Optimizer emit **information needs** (`InfoNeed`: question, why it matters, preferred source type).
- Gatherer resolves needs → `Evidence` snippets with provenance (URL, file path, timestamp, confidence).
- Evidence is embedded into a small vector store **scoped per goal** so retrieval stays relevant.
- Next strategize/optimize call receives: goal + current best process + top-k evidence + critique notes.

### 4.3 Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| Web-only | Fresh, broad | Noisy, rate limits, hallucinations if scraped badly |
| Local files only | Private, high trust | Stale / incomplete for many goals |
| Hybrid (recommended) | Best of both; LLM fills gaps | More plumbing; need provenance & freshness rules |
| Human-in-the-loop inbox | Highest quality | Not fully autonomous |

**Decision for v1:** Hybrid — local corpus + web search, with every claim tagged `source=llm|web|file` so the optimizer can prefer evidenced steps.

---

## 5. Important domain objects

```text
Goal
  id, title, description, success_criteria[], constraints[],
  priority, status (active|paused|done|failed),
  created_at, updated_at

Process (a strategy / plan)
  id, goal_id, version, parent_process_id?,
  steps[] (ordered or DAG), assumptions[], risks[],
  score, score_breakdown{}, status (candidate|champion|retired)

Step
  id, title, action, inputs[], outputs[],
  dependencies[], estimated_effort?, evidence_ids[]

Evidence
  id, goal_id, info_need_id?, content, provenance,
  fetched_at, freshness_score, embedding?

InfoNeed
  id, goal_id, question, rationale, status (open|resolved|stale)

Critique
  process_id, weaknesses[], suggested_mutations[],
  missing_info[] → spawn InfoNeeds

CycleRun
  goal_id, phase, model, tokens, duration, artifacts
```

---

## 6. Important modules / classes / functions

Proposed package layout:

```text
agent/
  main.py                 # entry: start daemon + API
  config.py               # paths, model name, cycle intervals
  models/                 # pydantic / dataclass schemas (Goal, Process, …)
  store/
    db.py                 # SQLite schema + repositories
    vector.py             # per-goal embeddings
  llm/
    client.py             # Ollama chat / JSON mode wrapper
    prompts.py            # strategist, optimizer, evaluator prompts
  knowledge/
    gatherer.py           # orchestrates adapters
    adapters/
      local_files.py
      web_search.py
      web_fetch.py
    chunking.py
  agents/
    strategist.py         # propose / revise processes
    evaluator.py          # score processes
    optimizer.py          # mutate / select champion
    critic.py             # find gaps → InfoNeeds
  scheduler/
    loop.py               # always-on multi-goal scheduler
    priority.py           # which goal runs next
  api/
    routes.py             # CRUD goals, inspect strategies, pause/resume
  ui/                     # optional simple local dashboard
```

### Key functions (contracts)

| Function | Responsibility |
|----------|----------------|
| `Strategist.propose(goal, evidence, prior_processes) -> list[Process]` | Generate new process candidates (structured JSON). |
| `Strategist.revise(process, critique, evidence) -> Process` | Produce a new version from critique. |
| `Evaluator.score(process, goal, evidence) -> ScoreBreakdown` | Rubric: goal fit, feasibility, evidence coverage, cost/risk. |
| `Optimizer.select_champion(processes) -> Process` | Keep best; retire weak; archive lineage. |
| `Optimizer.mutate(process, critique) -> list[Process]` | Local search over plan space (merge steps, reorder, drop, specialize). |
| `Critic.find_gaps(process, evidence) -> list[InfoNeed]` | What must we learn next? |
| `Gatherer.fulfill(info_needs) -> list[Evidence]` | Pull external/local info. |
| `Scheduler.tick()` | Pick goal → run one cycle → persist → sleep. |
| `GoalStore.upsert / list_active` | Multi-goal persistence. |

---

## 7. Optimization model (how “optimize” is real, not just more chat)

Treat strategies as a **search problem** over process space:

- **Population:** keep top-N candidates per goal (e.g. 5) + one champion.
- **Operators:** revise step, split/merge steps, change order, add contingency, drop low-evidence steps.
- **Fitness:** weighted rubric; optionally LLM-as-judge + hard rules (must cite evidence for factual steps).
- **Exploration vs exploitation:** occasional wild new strategy vs refine champion.
- **Stopping / cooling:** lower cycle frequency when score plateaus; spike when new high-value evidence arrives.

This is closer to evolutionary / bandit search than a single chat thread.

---

## 8. Always-on multi-goal scheduling

- Priority queue: `priority * urgency * (1 - recent_progress)`.
- Fairness: don’t starve low-priority goals (round-robin budget of LLM tokens per hour).
- Phases can be short: one goal may only gather this tick; another may optimize.
- Crash-safe: each cycle writes `CycleRun` before/after; restart resumes from last incomplete cycle.

---

## 9. Local LLM usage pattern

- Prefer **structured JSON outputs** (schema-constrained) for Process / Critique / Score.
- Separate prompts per role (Strategist ≠ Optimizer ≠ Evaluator) to reduce confused outputs.
- Keep context small: goal summary + champion + top critiques + top-k evidence chunks.
- Model guidance: start with a solid instruct model via Ollama (e.g. Qwen2.5 / Llama3.x class); upgrade later if quality needs it.
- **Trade-off:** smaller local models need tighter schemas and more retrieval; they will not match frontier APIs on open-ended creativity—compensate with better evidence and clearer rubrics.

---

## 10. MVP build sequence

1. **Skeleton daemon** — FastAPI + SQLite + empty scheduler tick.
2. **Goal CRUD** — create/list/pause goals via API (and tiny UI or CLI).
3. **LLM client** — Ollama wrapper with JSON schema responses.
4. **Strategist v1** — one process proposal per goal from goal text + LLM knowledge only.
5. **Evaluator + Optimizer v1** — score + revise loop (no web yet).
6. **Local file gatherer** — drop docs into `./knowledge/<goal_id>/`.
7. **Web gatherer** — search + fetch for open InfoNeeds.
8. **Multi-goal scheduler** — priority + token budgets.
9. **Dashboard** — show champion process, lineage, evidence, cycle history.
10. **Hardening** — provenance, rate limits, plateau detection, backups.

---

## 11. Open decisions (to resolve with you)

1. **Domain of goals** — general life/work goals, research, coding, ops, business? Affects sources & success criteria.
2. **Autonomy level** — propose only vs also execute steps (execution is a much larger product).
3. **Human approval** — auto-promote champion vs require accept.
4. **Web privacy** — allow outbound search from the local server?
5. **UI** — CLI-first vs local web dashboard first.
6. **Success signal** — purely LLM rubric, or measurable outcomes you report back?

**Working assumption until you say otherwise:**  
propose/optimize **plans only** (no automatic real-world execution), hybrid knowledge, local web dashboard, human can mark goals done / inject feedback as new evidence.

---

## 12. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Local LLM drifts / vague plans | Strict JSON schemas; rubric scoring; require evidence links on factual steps |
| Web noise pollutes strategies | Provenance + freshness + critic rejects low-trust evidence |
| Infinite loop burning CPU/GPU | Plateau detection; per-goal cycle caps; sleep schedules |
| Many goals thrash context | Per-goal stores; token budgets; summarize old cycles |
| Hallucinated “optimization” | Optimizer must change structured process fields; evaluator checks delta |

---

## 13. Success criteria for the system itself

- Given a goal, after N cycles the champion process is more specific, better evidenced, and higher-scored than cycle 1.
- Multiple goals can run unattended for days without manual restart.
- Every step that depends on external fact has attached evidence or an open InfoNeed.
- You can inspect full lineage: why the champion beat alternatives.

---

## 14. Next concrete step

Confirm open decisions in §11 (especially domain + execution vs plan-only), then implement MVP steps 1–5 on this branch: daemon, Goal store, Ollama strategist/evaluator/optimizer loop with SQLite persistence.
