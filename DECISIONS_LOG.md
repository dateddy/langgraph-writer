# Decisions Log

> **Append-only.** Never delete an entry. Never edit an entry after it's been written, except to add a "superseded by" link at the end. The value of this file is its historical integrity — a new Claude session needs to trust that what's here was actually decided, not retconned.
>
> **Read this when**: someone asks "why did we choose X" or "didn't we decide Y already". This file is the answer.
>
> **Format**: Newest decisions at the top. Each entry: title, date, status, context, decision, alternatives, rationale, consequences.

---

## Entry Template (copy this when adding a new entry)

```markdown
## [YYYY-MM-DD] D-NNN · short title

**Status**: Active | Superseded by D-XXX | Reversed

**Context**: What triggered this decision. What problem are we solving?

**Decision**: What we chose, stated as a clear directive.

**Alternatives considered**:
- Option A: brief description + why rejected
- Option B: brief description + why rejected

**Rationale**: Why this choice is correct given the context. What we'd lose by choosing otherwise.

**Consequences**:
- What this affects (files, skills, scope, timeline)
- What follow-up work is needed
- What we'll need to revisit if circumstances change
```

---

## Decisions

---

## [2026-06-14] D-016 - Anthropic for dev smoke from W2 onward

**Status**: Active

**Context**: Supersedes D-015. Ollama was selected as a free local development provider, but the owner's hardware made smoke validation too slow for useful W2 iteration. The attempted Gemini path was abandoned because Google migrated the owner's account to the new `AQ.` key format, which the current `langchain-google-genai` path does not support for this project setup. The owner topped up Anthropic credits with the minimum paid amount to unblock validation directly against the production-fidelity provider.

**Decision**: Use Anthropic for development smoke from W2 onward. Keep `LLM_PROVIDER=anthropic` for real research-pipeline validation, with Sonnet as the default model and Haiku as the cheap model where configured. The free-tier dev-provider strategy is abandoned for now.

**Alternatives considered**:
- Continue with Ollama: avoids API spend but runs too slowly on the owner's hardware to support practical smoke validation
- Continue with Gemini: blocked by the owner's Google `AQ.` key migration and current LangChain Google provider compatibility
- Skip live smoke: violates the W2 acceptance gate and would leave S02-S04 unvalidated against real external services
- Use OpenAI for dev smoke: introduces another paid hosted provider without matching the production-fidelity target

**Rationale**: The project already selected Anthropic as the primary production provider. A small Anthropic top-up validates the exact provider path needed for launch, removes local hardware uncertainty, and avoids losing more W2 time to provider abstraction churn. This is a conscious cost trade-off inside the project's `<$50/month` constraint.

**Consequences**:
- D-015's Ollama default-dev-provider choice is superseded
- Ollama provider code remains as an alternative path, but it is not the active dev smoke provider
- Gemini integration phases are skipped; no Gemini client/config changes are required
- Anthropic billing becomes part of the development workflow from W2 onward
- Full-scope smoke should run with `LLM_PROVIDER=anthropic`, `LLM_MODEL=claude-sonnet-4-5`, and `LLM_MODEL_CHEAP=claude-haiku-4-5-20251001`

---

## [2026-06-13] D-015 - Ollama as dev-time LLM provider; Anthropic for production smoke

**Status**: Superseded by D-016

**Context**: Anthropic credits are required for live API calls, but the project is in a cost-conscious development phase under the `<$50/month` constraint. The owner explicitly chose Ollama with `qwen2.5:7b` for local development to unblock W2 smoke validation without additional API spend.

**Decision**: Support `LLM_PROVIDER=ollama` in `client.py` for development. Development can run both default and cheap model settings as `qwen2.5:7b`. Production validation will switch back to Anthropic before launch.

**Alternatives considered**:
- Pay for Anthropic credits immediately: preserves production fidelity but spends money before W2 dev validation is complete
- Switch to Gemini free tier: adds another hosted provider dependency and key-management path
- Defer smoke testing: keeps code unchanged but violates the W2 acceptance gate

**Rationale**: Local Ollama gives zero-cost smoke testing for graph wiring, tool calls, and structured-output paths. It is not a replacement for final Claude quality validation, but it is sufficient to unblock development feedback loops.

**Consequences**:
- Adds/keeps `langchain-ollama` as a runtime dependency
- Adds an Ollama branch in `app/llm/client.py`
- Adds `ollama_base_url` and `ollama_model` settings
- Smoke output quality and structured-output reliability may be lower than Claude; production fidelity check remains pending

---

## [2026-06-13] D-014 - `langchain-anthropic` 1.4.5 API drift requires param renames

**Status**: Active

**Context**: The reference implementation used `ChatAnthropic(model=..., max_tokens=...)` with `api_key: str`. Investigation during the Surgical Audit showed `langchain-anthropic==1.4.5` expects `model_name`, `max_tokens_to_sample`, and `api_key: SecretStr`.

**Decision**: Pin `langchain-anthropic` to `==1.4.5` and use the current constructor arguments in `client.py`. Wrap provider API keys with `SecretStr`.

**Alternatives considered**:
- Pin to an older library version: may restore old parameter names but risks other compatibility issues
- Keep old parameter names: breaks the current runtime package
- Leave a broad version range: invites surprise drift on the next dependency resolution

**Rationale**: The code should track the installed runtime API, and a tight pin prevents a known drift point from recurring during the sprint.

**Consequences**:
- Affects `pyproject.toml`, `uv.lock`, and `app/llm/client.py`
- Future Anthropic upgrades require rechecking constructor names and mypy behavior

---

## [2026-06-13] D-013 - `cast()` pattern for LangChain structured-output typing

**Status**: Active

**Context**: LangChain's `with_structured_output(Schema)` typing can surface `dict | BaseModel` even when runtime returns the passed Pydantic schema instance. `mypy` reported false-positive structured-output errors in `research.py`.

**Decision**: Use local `cast(Schema, await llm.ainvoke(messages))` calls for LangChain structured outputs. Do not use `# type: ignore` for these lines.

**Alternatives considered**:
- `# type: ignore`: hides more than the intended structured-output typing gap
- Accept mypy failures: violates the project validation rules
- Wait for upstream type improvements: leaves the repo blocked

**Rationale**: `cast()` documents the runtime contract at the exact call site and keeps other type errors visible.

**Consequences**:
- Three scoped casts remain in `backend/app/graph/nodes/research.py`
- The `_raw_facts` temporary field is represented by a private `_SourceDocWithRawFacts` TypedDict instead of a broad cast

---

## [2026-06-13] D-012 - Haiku for S04 map step, Sonnet for reduce step

**Status**: Active

**Context**: S04 map summarization runs once per source and can dominate cost. The reduce/relevance pass requires stronger judgment and controls what downstream nodes consume.

**Decision**: Use `settings.llm_model_cheap` for `_map_summarize`. Keep the default model for `_reduce_summarize`.

**Alternatives considered**:
- All Sonnet: higher cost for modest map-step quality gain
- All Haiku: cheaper but weaker for relevance judgment
- Single hardcoded model: violates the no-hardcoded-model rule

**Rationale**: Map summarization is high-volume and lower judgment; reduce filtering is lower-volume and higher judgment. Matching model class to task protects cost without throwing away quality where it matters.

**Consequences**:
- Adds `llm_model_cheap` in settings
- Adds `model_override` support in `get_llm()`
- Keeps S04 behavior configurable per environment

---

## [2026-06-13] D-011 · Relevance threshold 0.3 for source filtering

**Status**: Active

**Context**: S04 needs to discard source summaries that are only weakly related to the requested topic before downstream outline/writer nodes consume them. The reference implementation uses a per-source `RelevanceJudgment` and filters after the reduce pass.

**Decision**: Keep a minimum `relevance_score` threshold of `0.3` in `research_node` for enriched sources.

**Alternatives considered**:
- No threshold: preserves recall but allows off-topic facts to leak into the writer
- Higher threshold such as `0.7`: aligns with the S03 target relevance metric but risks dropping useful early-MVP sources before eval calibration exists

**Rationale**: A low threshold is a conservative MVP filter: it removes clearly irrelevant sources while keeping enough material for S05/S06 to work. It should be revisited once source relevance evals exist.

**Consequences**:
- Affects `backend/app/graph/nodes/research.py`
- Downstream nodes receive fewer but more relevant sources
- Needs calibration against the W2 eval set before marking S04 implemented

---

## [2026-06-13] D-010 · Fail-soft scrape with Tavily snippet fallback

**Status**: Active

**Context**: Web scraping Vietnamese news and institutional pages is unreliable: pages may block requests, timeout, or produce too little readable text. A hard failure on any single source would make the research phase brittle.

**Decision**: `fetch_clean_text()` returns `None` on fetch/parse failure, and the research node falls back to Tavily snippet content when the snippet is at least 200 characters.

**Alternatives considered**:
- Fail the whole research node on scrape errors: stricter but too fragile for an external-web pipeline
- Keep failed sources with empty content: simpler but wastes summarization calls and creates downstream noise

**Rationale**: The fallback preserves useful search snippets while still skipping sources with insufficient content. This matches the project's cost and reliability constraints for a solo MVP.

**Consequences**:
- Affects `backend/app/tools/scraper.py` and `backend/app/graph/nodes/research.py`
- Some summaries may be based on snippets rather than full article text
- Future audit should expose whether a source came from full scrape or fallback

---

## [2026-06-13] D-009 · Single research node for S02-S04

**Status**: Active

**Context**: Week 2 implements query generation, web research, and source summarization. These can be modeled as three graph nodes or as one graph node with internal async helpers.

**Decision**: Use one LangGraph node, `research_node`, to orchestrate S02, S03, and S04 internally.

**Alternatives considered**:
- Three separate graph nodes: more observable at graph level but adds state transitions and edge complexity before the rest of the pipeline exists
- A non-graph script: simpler locally but violates the graph-first project invariant

**Rationale**: A single node keeps the public graph topology simple for W2 while preserving testable internal helper functions. The design can be split later if tracing or retry policies need per-skill graph boundaries.

**Consequences**:
- Affects `backend/app/graph/nodes/research.py` and `backend/app/graph/builder.py`
- S02 and S04 are sub-steps inside `research_node`, not standalone graph nodes
- Skill status in `SKILLS.md` must be tracked at sub-step granularity

---

## [2025-W0] D-008 · English-primary for project meta-documentation

**Status**: Active

**Context**: Need to decide language for `RULES.md`, `SKILLS.md`, `CONTEXT.md`, and other project meta-files. Project owner is Vietnamese, but Codex GPT is the primary consumer of these files, and the GitHub repo will be public.

**Decision**: All meta-documentation files written in English. Vietnamese preserved only where semantically required: prompt content for Vietnamese-output agents, Vietnamese sample outputs, glossary entries naming Vietnamese-specific concepts.

**Alternatives considered**:
- Full Vietnamese: more natural for the owner, but Codex GPT follows English instructions more reliably and weakens portfolio impression with international recruiters
- Mixed (technical EN, prose VN): inconsistent reading experience, harder to maintain

**Rationale**: Codex/Claude were trained primarily on English-language documentation conventions. English instructions yield more reliable code generation. Portfolio value is also higher when international recruiters can read the docs directly.

**Consequences**:
- Owner reads English docs daily — minor friction acceptable
- Vietnamese prompts and sample outputs preserved verbatim in their files
- All future docs (HANDOFF, CURRENT_STATE, etc.) follow this convention

---

## [2025-W0] D-007 · Context management system with 5 files

**Status**: Active

**Context**: Need a system for Claude/Codex sessions to maintain continuity across the 7-week project. Single-file approaches mix mutable and immutable content; pure chat history is lost when conversations get compressed or new chats open.

**Decision**: Five-file system:
- `context.md` — immutable foundation (rarely changes)
- `CURRENT_STATE.md` — real-time dashboard (frequently updated)
- `DECISIONS_LOG.md` — append-only history (this file)
- `HANDOFF.md` — session-to-session bridge (overwritten each session)
- `compress.md` — conversation compression target (overwritten each compression)

**Alternatives considered**:
- Single `PROJECT.md`: simple but mixes append-only history with mutable state — destroys clarity
- Git commit history as the record: not human-readable enough for an LLM to scan quickly
- External project management tool (Notion, Linear): adds friction, doesn't integrate with code repo

**Rationale**: Each file has a single mutability rule (immutable / mutable / append-only / overwrite). When Claude reads them, the structure tells it what to trust as current vs. historical. This separation is the whole point — it's worth the file count.

**Consequences**:
- Five files to maintain, but each is small and has a clear purpose
- Reading protocol documented in `context.md`
- Previous `CONTEXT.md` (capital) deprecated — content migrated to the new files

---

## [2025-W0] D-006 · Skills system as project's vocabulary

**Status**: Active

**Context**: Need a way to talk about agent capabilities consistently across commits, PRs, planning, and code. Without IDs, conversations drift into ambiguous references like "the research thing" or "that summarizer".

**Decision**: Define each agent capability as a numbered Skill (S01-S11+) in `SKILLS.md`. Use the ID in commit messages, branch names, PR titles, eval reports.

**Alternatives considered**:
- Just use node names: works internally but doesn't capture cross-cutting capabilities like S08 Style Editor that might span multiple nodes
- Use Linear/Jira issues: external dependency, doesn't live in the repo

**Rationale**: Skill IDs are stable references. When S03 is renamed from "Researcher" to "Web Researcher", all references still resolve. They also serve as a project taxonomy — `SKILLS.md` becomes a one-page mental model of the whole product.

**Consequences**:
- Every new capability gets an ID before code is written
- Skill IDs in branch names: `feat/s03-research-node`
- Status tracking lives in `SKILLS.md`, not in CURRENT_STATE.md (avoid duplication)

---

## [2025-W0] D-005 · SSE over WebSocket for streaming

**Status**: Active

**Context**: Need to stream graph output (each agent's progress + writer's tokens) from FastAPI backend to Next.js frontend in real time.

**Decision**: Use Server-Sent Events (sse-starlette) for streaming.

**Alternatives considered**:
- WebSocket: bidirectional, but adds complexity (heartbeats, reconnection, sticky sessions on Vercel)
- Polling: simple but inefficient and laggy
- Long-polling: outdated

**Rationale**: Generation is one-way (server → client). SSE works over standard HTTP, plays well with Vercel's edge network, no reconnection logic needed. WebSocket's bidirectional capability is unused — adding complexity for no gain.

**Consequences**:
- Cannot send mid-generation control signals from client (e.g., "stop"). Acceptable — abort is rare enough to handle via a separate `/cancel` endpoint if needed.
- Reverse proxy config (Railway, Vercel) must allow SSE pass-through. Verified compatible.

---

## [2025-W0] D-004 · Conditional rewrite loop with hard retry cap

**Status**: Active

**Context**: Fact-check node (S07) can detect issues in the draft. Need to decide whether to attempt fixes or just report.

**Decision**: If `score < 0.7`, loop back to Writer (S06) with the issues as context. Cap retries at `retry_count = 2`. After 2 retries, ship whatever we have with the score surfaced to the user.

**Alternatives considered**:
- Unbounded loop until score >= 0.7: classic LangGraph foot-gun, can burn unbounded tokens
- No loop, just report: cheaper but lower quality output
- Score-aware threshold (higher threshold = more retries): too complex for MVP

**Rationale**: Two rewrites give meaningful improvement; a third rarely does and burns tokens. Hard cap also protects against infinite-loop bugs from edge cases in the fact-check parser.

**Consequences**:
- Some low-quality outputs will ship. Acceptable for MVP — fact-check score is surfaced to the user so they can decide to manually edit or regenerate.
- `WriterState.retry_count` field added to track loop count.
- Edge in `builder.py` routes back to S06 when `should_rewrite(state) == "rewrite"`.

---

## [2025-W0] D-003 · Vietnamese-first, no parallel English pipeline

**Status**: Active

**Context**: Deciding whether to build the pipeline as language-agnostic from day 1 (with English as a parallel path) or to commit to Vietnamese-only for MVP.

**Decision**: Vietnamese only. No English pipeline. No language toggle.

**Alternatives considered**:
- Build bilingual from day 1: doubles prompt iteration cost, doubles eval cost, dilutes the Vietnamese quality story
- Build English-first because more training data is in English: undermines the entire differentiation strategy
- Build Vietnamese-only with hooks to add English later: even "hooks" add complexity; defer until validated demand

**Rationale**: Vietnamese is the moat. Most foreign tools (ChatGPT, Jasper, Copy.ai) handle Vietnamese poorly — translation-ese, wrong tone, missed cultural context. Owning Vietnamese quality is the entire pitch. Building English would dilute focus during the only 7-week sprint available.

**Consequences**:
- All prompts in `templates.py` written in Vietnamese (RULES.md §6 enforces no translation)
- All eval datasets in Vietnamese
- Cannot target international users in MVP. Acceptable — Vietnamese-speaking content market is large enough.
- If English demand emerges post-MVP, we revisit as a separate product, not a feature flag.

---

## [2025-W0] D-002 · All prompts centralized in `templates.py`

**Status**: Active

**Context**: Where do prompts live in the codebase? Inline with nodes? In a separate file per agent? In a single shared file?

**Decision**: All prompts in `app/prompts/templates.py` as module-level constants. No prompt text anywhere else in the codebase.

**Alternatives considered**:
- Inline with nodes: readable in context but scatters versioning, makes A/B testing harder, can't migrate to LangSmith Prompt Hub without refactor
- One file per agent: feels modular but adds 5+ tiny files, makes cross-prompt consistency (tone, format) harder to maintain
- External prompt management tool from day 1: premature, adds dependency, slows iteration

**Rationale**: Prompts iterate 10-50x more than code. Centralizing makes diffs scannable. Single file makes it trivial to enforce Vietnamese tone consistency across all agents. Migration path to LangSmith Prompt Hub is straightforward — change imports only.

**Consequences**:
- `templates.py` will grow large. Acceptable — better than 5 files with 50 lines each.
- Naming convention: `<AGENT>_SYSTEM`, e.g., `WRITER_SYSTEM`, `FACT_CHECK_SYSTEM`.
- Versioning: `WRITER_SYSTEM_V2` next to V1, never overwrite.

---

## [2025-W0] D-001 · Pure-function nodes, no class hierarchy

**Status**: Active

**Context**: LangGraph allows nodes as either plain functions or class methods. Need to pick a convention.

**Decision**: Nodes are plain `async def` functions. No `BaseAgent` class. No node-as-method.

**Alternatives considered**:
- BaseAgent class with `run()` method: feels OO-clean but adds indirection
- Dataclass + standalone function: hybrid that combines worst of both
- Per-node module with internal class: over-engineered for the scope

**Rationale**: LangGraph already provides composition via the graph. Adding an OO layer adds indirection without value. Functions are trivially testable (mock input → assert output), classes require mocking instance state. Functions also match the LangGraph community's idiomatic style — onboarding new devs is faster.

**Consequences**:
- Shared logic between nodes goes in `app/utils/` as plain helpers
- If shared logic across 3+ nodes becomes substantial, revisit
- One file per node in `nodes/`, exporting one function

---

## Decision Status Conventions

| Status | Meaning |
|---|---|
| **Active** | Currently in force. Code follows this decision. |
| **Superseded by D-XXX** | A later decision (D-XXX) reversed or replaced this one. Both entries remain. |
| **Reversed** | Decision was tried, didn't work, no specific successor — back to previous state. Explanation appended. |

---

## How to Supersede a Decision

When a new decision changes a previous one:

1. Add the new entry at the top, normally.
2. In the new entry's "Context", reference the old decision: `"Supersedes D-XXX"`.
3. In the OLD entry, edit ONLY the Status line to: `Superseded by D-NNN (link)`.
4. Do NOT edit the body of the old entry. Its historical value is preserved.

---

*Append entries above this line. Always at the top of the Decisions list, not the bottom.*
