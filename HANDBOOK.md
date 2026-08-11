# THE AURA HANDBOOK
### Learn to build an enterprise-grade agentic AI framework — from the absolute basics to cutting-edge patterns

This handbook is your study companion for the AURA codebase. It assumes you are a
beginner who learns fast. Read it top to bottom once, then keep it open beside the
code. Every section maps to real files you can run and modify.

A note on honesty up front: AURA as shipped is a **working foundation**, not a
finished commercial product. The point is that you *understand and own every line*,
so you can grow it. This handbook teaches the understanding; §12 is the roadmap for
turning it into something production-hard.

---

## Table of contents
0. How to use this handbook
1. Mental models: what an "agent" actually is
2. Configuration & BYOK (why no key is ever hardcoded)
3. Observability: tracing, spans, and the dashboard
4. The LLM boundary: caching and token efficiency
5. Memory: short-term, long-term, and the knowledge graph
6. Knowledge: ontology + the "knowledge-fluid" idea
7. Context engineering & dynamic compression
8. Agents, tools, and discovery
9. Orchestration: hub-and-spoke, reflection, debate
10. Evaluation: metrics and LLM-as-judge
11. Token efficiency: the full checklist
12. Roadmap: from foundation to enterprise
13. Deep-dive learning resources (papers, repos, videos, blogs)
14. Your daily learning loop (with the Tutor Skill)

---

## 0. How to use this handbook
Three passes:
1. **Read** a section.
2. **Open** the file it references and read the code with the section beside it.
3. **Change** one thing and re-run `example.py` + the dashboard to see the effect.

Understanding sticks when you break things on purpose. Lower a budget, raise a
threshold, delete a span — watch what the dashboard shows.

---

## 1. Mental models: what an "agent" actually is
Strip away the hype. An **LLM** is a function: text in, text out. It has no memory,
no tools, no goals. An **agent** is everything you wrap around that function to make
it *act*:

- **A role** (system prompt): who it is and how it behaves.
- **Context**: what information you feed it on each call.
- **Tools**: functions it can call to affect the world (search, create a ticket).
- **Memory**: what it remembers between calls.
- **A loop**: think → act → observe → repeat.

A **multi-agent system** is several such agents coordinated by an **orchestrator**.
That's the whole field. Everything else — RAG, reflection, debate, graphs — is a
technique for improving one of those five ingredients.

Why multi-agent at all? Because one prompt doing everything is hard to steer, hard
to debug, and burns tokens. Splitting work into specialists with a coordinator gives
you separation of concerns — the same reason we split code into functions.

> File to read next: none yet — just hold this model in your head.

---

## 2. Configuration & BYOK — `aura/config.py`
**BYOK = Bring Your Own Key.** The framework never stores an API key; it reads
`ANTHROPIC_API_KEY` from the environment at call time (`settings.require_key()`).

Why this matters for *you specifically*: you mentioned using a company key. Hardcoding
a key, or baking it into a shared service, is how keys leak and how usage gets billed
to the wrong account. BYOK means: your learning uses *your* key; if you ever let other
people use a deployment, *they* supply *their* key. Get written permission before any
company key powers anything personal or revenue-generating.

Two model tiers live here too:
- `model` — your smart model (Sonnet) for real reasoning.
- `utility_model` — a cheap, fast model (Haiku) for routing, summarising, judging.

**Token-efficiency lesson #1:** never use your most expensive model for cheap jobs.
Routing a task to a worker is a one-word answer — Haiku does it for a fraction of the
cost. AURA wires this in by default.

*Experiment:* change `AURA_UTILITY_MODEL` and watch the routing span in the dashboard.

---

## 3. Observability — `aura/tracing/`
This is the heart of the project and the part most people skip. You cannot improve
what you cannot see. **Observability** means: for any run, you can answer *what
happened, in what order, how long it took, how many tokens it cost, and where it broke.*

Three concepts:
- **Trace** (`schema.py`): one top-level request (e.g. "answer a support ticket").
- **Span**: one unit of work inside it (an agent step, an LLM call, a tool call).
  Spans nest — a supervisor span contains an agent span contains a tool span.
- **Store** (`store.py`): SQLite persistence so traces survive restarts.

The clever bit is in `tracer.py`: spans nest **automatically** using a Python
`contextvars.ContextVar`. When you open a span inside another, AURA reads the current
span id and records it as the parent. You never pass parent ids by hand. That parent
→ child structure is exactly what the dashboard turns into the **execution graph**.

The dashboard (`dashboard/`) gives you two signature views:
- **Waterfall timeline**: each span is a bar positioned by start time and sized by
  duration — like a browser's network tab. Instantly shows what ran when, what was
  slow, and (hatched bars) what was served from cache.
- **Execution graph**: the parent/child tree as a left-to-right node graph — who
  called whom.

> Debugging a multi-agent system *is* reading these two views. Run `example.py`, open
> the dashboard, click a bar, read its input/output.

*Why SQLite, not Postgres?* Zero setup, one file, perfect for local/intranet. The
store is the only place that knows it's SQLite — §12 explains the swap to Postgres.

---

## 4. The LLM boundary — `aura/llm/`
All Claude calls go through one wrapper (`client.py`). Centralising this is an
architecture decision with big payoffs: every call is automatically traced, every
call can be cached, and cost control lives in one auditable place.

**Semantic cache** (`cache.py`): before paying for a call, AURA checks whether a
*semantically similar* prompt was already answered. Exact matches are instant; near
matches use embeddings + cosine similarity above a threshold (`AURA_CACHE_SIM`, default
0.92). A hit returns the stored answer for **zero tokens** and is flagged `cached=True`
in its span (the hatched bar in the timeline).

Caution: set the threshold *high*. A cache that returns "close enough" answers to
*different* questions is a correctness bug. When in doubt, prefer a miss.

**Anthropic prompt caching:** when you pass a `system` prompt, AURA marks it with
`cache_control: ephemeral`. Long, stable system prompts then cost far less on repeated
calls. This is different from the semantic cache — it's Anthropic caching the *prefix*
of your prompt server-side.

---

## 5. Memory — `aura/memory/`
Three kinds, because human memory has kinds too:

- **Short-term** (`short_term.py`): the rolling conversation buffer for one session.
  Small on purpose; a `deque` with a max length.
- **Long-term** (`long_term.py`): facts/experiences stored with an embedding and
  retrieved by semantic similarity (`recall`). This is the engine of RAG.
- **Knowledge / memory graph** (`graph.py`): entities as nodes, typed relations as
  edges, using NetworkX. Lets agents do *multi-hop* reasoning — "alice raised
  ticket-42 which is about billing" — that flat text can't.

**Embeddings** (`embeddings.py`) underpin both long-term memory and the cache. The
default `lexical` backend is a pure-python bag-of-words hash so the framework runs
with no installs; switch to `fastembed` for real neural embeddings on CPU (no GPU, no
torch). The abstraction means calling code never changes when you upgrade quality.

*Why a graph view of memory?* Because relationships are knowledge. "Who else raised
tickets about billing?" is one hop in a graph and a painful scan in flat text.

---

## 6. Knowledge — `aura/knowledge/`  — the "knowledge-fluid" core
This is the idea you cared most about: one framework, many domains, just by swapping
knowledge.

- **Ontology** (`ontology.py`): the *schema* of knowledge — which entity types exist
  (Customer, Ticket, Product) and which relations are legal between them (a Customer
  `raised` a Ticket). It validates facts and gives the LLM a description of the domain
  to reason with. The export is JSON-LD-flavoured for interoperability.
- **KnowledgeBase** (`knowledge_base.py`): bundles ontology + graph + long-term memory
  into one swappable object. `assert_fact` enforces the ontology; `context_for`
  assembles grounded context for an agent.

**Knowledge fluid in practice:** to retarget AURA from "support" to, say, "legal
research", you write a new ontology (Case, Statute, Judge, `cites`, `overrules`) and
load that domain's facts. *No agent, orchestration, or observability code changes.*
That separation is the whole architectural point.

> Accuracy note on "OKF (Open Knowledge Format) from Google Cloud": I could not verify
> that as an established public standard. AURA's ontology layer is deliberately
> format-agnostic and RDF/JSON-LD-aligned. If OKF turns out to be real and you must
> conform to it, confirm its schema (use your web-search account + the Tutor Skill) and
> map it onto `Ontology`/`to_jsonld`. Don't trust a spec you haven't verified.

---

## 7. Context engineering & compression — `aura/context/`
Most "the model is dumb" problems are really "the context was bad" problems. **Context
engineering** is deciding *what* goes into each prompt and *in what order*, under a
token budget.

- **`engineer.py`** (`ContextEngineer`): you `add()` labelled blocks with a priority;
  `build()` packs them into the budget, dropping the lowest-priority blocks first, then
  renders them in logical order. Role and task are priority 1; retrieved knowledge is
  priority 2; history is lower.
- **`compressor.py`**: when history exceeds the budget, keep the most recent turns
  verbatim and **summarise the older ones** with the cheap utility model. Lossy but
  smart — you keep decisions and open questions, drop the chatter.

**Token-efficiency lesson #2:** a bigger model rarely fixes a problem that better
context would. Engineer the context first.

*Note:* token counts here use a ~4-chars-per-token heuristic to avoid a dependency.
§12 covers switching to accurate counting.

---

## 8. Agents, tools, discovery — `aura/agents/`, `aura/tools/`
- **`Agent`** (`agents/base.py`): a name + role + system prompt + optional KB + tools.
  Its `run()` builds context via `ContextEngineer`, calls the LLM, and is wrapped in an
  `agent` span so it appears in the graph. Deliberately thin — coordination lives in the
  orchestrator, not inside agents.
- **Tool registry** (`tools/registry.py`): tools are functions with a name +
  description. `register` is a decorator; every tool call is traced. **Discovery**:
  `tools.discover("give money back")` ranks tools by semantic fit so an agent can find
  the right tool without hardcoding.
- **Agent discovery** (`agents/discovery.py`): same idea one level up — workers register
  their capabilities and the supervisor finds the right worker semantically.

"Agentic resource discovery" = an agent figuring out *which capabilities exist and which
fit the task*, rather than you wiring every path by hand. That's what these registries
provide.

---

## 9. Orchestration — `aura/agents/supervisor.py`, `aura/orchestration/`
**Hub-and-spoke (Supervisor)** — `supervisor.py`. The supervisor is the hub: it receives
the task, **routes** it to the best worker (LLM router + semantic fallback), delegates,
then **synthesises** the final answer. Why this pattern for a beginner: one clear
decision point, a trace that reads top-to-bottom, easy to debug. (Alternatives:
peer-to-peer, where agents message each other; and blackboard, where agents read/write
a shared state. Hub-and-spoke is the right first pattern.)

**Reflection** — `orchestration/reflection.py`. Answer → critic finds flaws → revise.
A cheap quality boost, capped at a few rounds so it can't burn unbounded tokens.

**Multi-agent debate** — `orchestration/debate.py`. Two agents argue independent
positions for a few rounds; a judge synthesises the most defensible answer. Reduces a
single model's blind spots on contested questions. Also token-bounded.

All three produce spans, so you can *see* the reflection rounds and debate turns in the
timeline — invaluable for understanding and for cost control.

---

## 10. Evaluation — `aura/evaluation/`
You can't improve quality you don't measure. Two complementary tools:

- **Deterministic metrics** (`metrics.py`): free, objective numbers computed from a
  trace — span count, llm calls, **cache hit rate**, tokens in/out/total, duration,
  errors. These drive the dashboard cards.
- **LLM-as-judge** (`evaluator.py`): scores an answer 1–5 against a fixed rubric
  (correctness, completeness, groundedness, conciseness). Use a *fixed* rubric so scores
  are comparable as you change prompts/models.

**The practice that matters:** build a small "golden set" of ~20 representative tasks
with known-good answers. Run them after every change. Track average judge scores and
token cost over time. This is how real teams keep agents from silently regressing.

---

## 11. Token efficiency — the full checklist
Everything AURA does to keep your bill (and latency) down, in one place:
1. **Two-tier models** — Haiku for routing/summarising/judging, Sonnet for reasoning.
2. **Semantic cache** — skip repeated/near-repeated calls entirely.
3. **Anthropic prompt caching** — cheap reuse of long stable system prompts.
4. **Context budget + priority packing** — never send more than needed.
5. **Dynamic compression** — summarise old history instead of resending it.
6. **Bounded loops** — reflection and debate have hard round caps.
7. **Grounding over guessing** — feed facts from the KB so the model doesn't waste
   tokens hedging or hallucinating.
8. **Measure it** — the dashboard's cache-hit-rate and token cards tell you if it's
   working. Optimise what you can see.

---

## 12. Roadmap — from foundation to enterprise
What to harden next, roughly in order. Each is a great learning project:
- **Accurate token counting** (swap the heuristic for real counting).
- **Retries & timeouts** around the LLM call (exponential backoff on 429/5xx).
- **Auth on the dashboard** (API key or SSO) before exposing it widely — even on an
  intranet, add at least a shared secret.
- **Postgres store** (implement the same `TraceStore` interface against Postgres; the
  rest of the framework won't notice).
- **Streaming** responses for better UX on long generations.
- **Background workers / queue** so long runs don't block requests.
- **RBAC & multi-tenant** trace isolation if multiple teams share one instance.
- **Real vector index** (sqlite-vec / FAISS) when memory grows past a few thousand rows;
  the current cosine scan is linear.
- **Autonomous learning loop**: periodically mine high-scoring traces into long-term
  memory and refine system prompts from judge feedback. The pieces (judge, memory,
  traces) are already here — wiring them into a scheduled loop is your capstone.

Be honest in your own notes about which of these are done. "Enterprise-grade" is earned
one hardened edge at a time.

---

## 13. Deep-dive learning resources
These are real, well-known works. For exact links and anything newer than early 2026,
use your web-search Claude account + the Tutor Skill (it's built to fetch and verify).

**Foundational papers (search by title on arXiv):**
- *Chain-of-Thought Prompting Elicits Reasoning in LLMs* — Wei et al., 2022.
- *ReAct: Synergizing Reasoning and Acting in Language Models* — Yao et al., 2022.
- *Reflexion: Language Agents with Verbal Reinforcement Learning* — Shinn et al., 2023.
- *Self-Refine: Iterative Refinement with Self-Feedback* — Madaan et al., 2023.
- *Toolformer: Language Models Can Teach Themselves to Use Tools* — Schick et al., 2023.
- *Retrieval-Augmented Generation for Knowledge-Intensive NLP* — Lewis et al., 2020.
- *Generative Agents: Interactive Simulacra of Human Behavior* — Park et al., 2023.
- *MemGPT: Towards LLMs as Operating Systems* — Packer et al., 2023.
- *Constitutional AI: Harmlessness from AI Feedback* — Anthropic, 2022.
- *From Local to Global: A GraphRAG Approach* — Microsoft Research, 2024.
- *DSPy: Compiling Declarative LM Calls into Self-Improving Pipelines* — Khattab et al., 2023.

**GitHub repos to read (study the code, don't just install):**
- LangGraph (graph-based orchestration), AutoGen (Microsoft, multi-agent),
  CrewAI (role-based crews), DSPy (programmatic prompting), Letta/MemGPT (agent memory),
  Microsoft GraphRAG, Anthropic Cookbook (official Claude patterns).

**Docs that are worth a slow read:**
- Anthropic docs: tool use, prompt caching, the Cookbook (docs.claude.com / Anthropic
  GitHub). Verify current URLs via search.

**Patents:** I won't fabricate patent numbers. Instead, search Google Patents for terms
like "multi-agent orchestration language model", "retrieval augmented generation", and
"knowledge graph reasoning agent", then read the cited prior art. The Tutor Skill has a
workflow for turning a topic into a patent-search query.

**Channels/blogs (verify they're current):** Anthropic's engineering blog; talks from
the authors above; conference recordings (NeurIPS/ICLR) for the papers listed.

> Caution: link rot and new releases are constant. Treat the *titles and authors* above
> as the durable anchors; let the Tutor Skill fetch fresh links.

---

## 14. Your daily learning loop — with the Tutor Skill
You also have **`aura-tutor`**, a Claude Skill that turns this into a daily practice:
1. On your web-search-enabled account, load the skill and say what you want to learn.
2. It checks your progress log, picks the next topic, fetches *current* resources,
   teaches it against the AURA code, and gives you one hands-on exercise.
3. It appends what you covered to the progress log so tomorrow continues seamlessly.

Recommended 8-week path (the skill follows this by default):
- **Wk 1** Fundamentals: §1–2, run example, read every span in the dashboard.
- **Wk 2** Observability: §3 — add your own span kinds; break things and debug.
- **Wk 3** LLM & tokens: §4, §11 — measure cache hit rate; tune the threshold.
- **Wk 4** Memory & graph: §5 — load a real dataset into the knowledge graph.
- **Wk 5** Knowledge fluid: §6 — build a *new* ontology for a domain you care about.
- **Wk 6** Context: §7 — implement accurate token counting; tune the budget.
- **Wk 7** Orchestration: §8–9 — add a third worker; try a debate task.
- **Wk 8** Evaluation & capstone: §10, §12 — build a golden set; wire one roadmap item.

Learn one section, change one thing, measure the effect. That loop, repeated, is how you
go from beginner to building this for real.
