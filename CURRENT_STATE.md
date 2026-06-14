# Current State Dashboard

> **Read this when**: you need to know what's happening RIGHT NOW. What's the active task? What's blocked? What files are mid-edit?
>
> **Mutability**: Updated frequently. Always update the `Last updated` timestamp.

---

## Last Updated

**Date**: 2026-06-14T15:38:02.7718248+07:00  
**Session**: Anthropic smoke retry after D-016  
**Updated by**: Codex

---

## Active Focus

**Current Skill**: S02 Query Generator + S03 Web Researcher + S04 Source Summarizer  
**Current Phase**: Week 2 research validation  
**Current File(s)**: `backend/app/graph/nodes/research.py`, `backend/app/llm/client.py`, `backend/app/config.py`, `scripts/smoke_research.py`  
**Status**: In Progress - full-scope Anthropic smoke blocked by invalid API key

---

## Now Doing

Gemini integration was abandoned by owner decision. D-016 now supersedes D-015 and makes Anthropic the dev smoke provider from W2 onward.

- D-012 through D-015 were appended to `DECISIONS_LOG.md`.
- D-016 was appended to `DECISIONS_LOG.md`; D-015 status was changed to `Superseded by D-016`.
- `langchain-anthropic` is pinned to `==1.4.5`.
- `backend/app/llm/client.py` now supports `LLM_PROVIDER=ollama`.
- `backend/app/config.py` now includes `ollama_base_url` and `ollama_model`.
- `.env.example` now includes Ollama settings.
- `backend/app/graph/nodes/research.py` now has env-driven research scope knobs and flushed per-phase diagnostic prints.
- `scripts/smoke_research.py` now uses `graph.astream()` and flushed node-level progress output; Windows stdout/stderr are reconfigured to avoid UnicodeEncodeError.
- `uv run ruff check app/` passes.
- `uv run ruff format --check app/` passes: 23 files already formatted.
- `uv run mypy app/ --ignore-missing-imports` passes: no issues in 23 source files.
- `uv run pytest tests/unit/graph/nodes/test_research.py -v` passes: 6/6.
- S04 map summarization now uses `settings.llm_model_cheap`; reduce stays on the default model.
- `.env` is back to `LLM_PROVIDER=anthropic`, `LLM_MODEL=claude-sonnet-4-5`, and `LLM_MODEL_CHEAP=claude-haiku-4-5-20251001`.
- `RESEARCH_MAX_SOURCES_TOTAL` and `RESEARCH_MAX_RESULTS_PER_QUERY` were commented out so smoke runs full scope.
- Full-scope Anthropic smoke reached the planner call and failed with `anthropic.AuthenticationError` HTTP 401: invalid x-api-key.
- Sanitized key check: `ANTHROPIC_API_KEY` starts with `sk-ant-` but length is only 10, consistent with a placeholder rather than a real key.

**Next concrete action**: replace `ANTHROPIC_API_KEY` in `.env` with the real topped-up key, then rerun `uv run python scripts/smoke_research.py` from the repo root. Do not promote S02-S04 until smoke passes or the owner explicitly waives the gate.

---

## Blockers

### B-001: Anthropic smoke blocked by invalid API key

- **What**: Full-scope Anthropic smoke fails during the planner call before research runs.
- **Why blocked**: Anthropic returns HTTP 401 `authentication_error` with message `invalid x-api-key`.
- **Attempted workarounds**: D-016 logged; D-015 superseded; `.env` provider reverted to Anthropic; research scope overrides removed; smoke script Unicode output fixed; static validation rerun successfully; smoke executed full scope.
- **Owner action needed**: Replace `ANTHROPIC_API_KEY` in `.env` with the real topped-up key. The current value starts with `sk-ant-` but has length 10, which is too short for a real key.
- **Since**: 2026-06-14

### B-002: Anthropic billing/credit state not yet validated by smoke

- **What**: Owner reports Anthropic credits are topped up, but the smoke cannot reach billing validation because authentication fails first.
- **Why blocked**: Current API key is invalid/placeholder, so the request fails before any credit-balance check.
- **Owner action needed**: After replacing the key, rerun the full-scope smoke and confirm the billing blocker is gone.
- **Since**: 2026-06-13

---

## Files Mid-Edit

No files are mid-edit. Latest cleanup touched:

| File | Latest change | Resumption hint |
|---|---|---|
| `backend/app/llm/client.py` | Added `model_override`, current Anthropic constructor args, `SecretStr`, and Ollama provider branch | Recheck if dependency version changes |
| `backend/app/config.py` | Added `llm_model_cheap`, `ollama_base_url`, and `ollama_model` | Keep `.env.example` aligned |
| `backend/app/graph/nodes/research.py` | Added structured-output casts, cheap-model map step, env-driven diagnostic scope knobs, and flushed per-phase prints | Remove/keep diagnostic prints by owner decision after diagnosis |
| `backend/tests/unit/graph/nodes/test_research.py` | Added assertion that map step uses cheap model override | Unit test passes |
| `.env.example` | Added `LLM_MODEL_CHEAP`, Ollama settings, and commented research scope overrides | Do not edit real `.env` here |
| `scripts/smoke_research.py` | Rewritten as diagnostic `graph.astream()` smoke with flushed output; added Windows UTF-8/backslashreplace output guard | Rerun after real Anthropic key is set |
| `DECISIONS_LOG.md` | Added D-012 through D-016; D-015 superseded | Append-only |
| `pyproject.toml`, `uv.lock` | Added Ollama dependency and pinned `langchain-anthropic==1.4.5` | Keep dependency changes through `uv add` |

---

## Environment State

| Component | State |
|---|---|
| Local repo | Present, many files untracked |
| `.env` | Provider/model set to Anthropic; research scope overrides unset; `ANTHROPIC_API_KEY` appears placeholder-length |
| Dependencies | Required research dependencies present; `langchain-ollama` available |
| LangChain Anthropic | Installed `langchain-anthropic==1.4.5`; constructor uses `model_name` and `max_tokens_to_sample` |
| Ollama | Superseded by D-016 as dev smoke provider; branch remains available |
| LangSmith | Trace URL not captured |
| Tavily | Key previously present; not reached because planner auth failed first |
| Validation | ruff check pass; ruff format check pass; mypy pass; research pytest pass |
| Live smoke | FAIL: Anthropic HTTP 401 invalid x-api-key during planner |

---

## Open Questions

| ID | Question | Target resolution week |
|---|---|---|
| Q1 | Should S02-S04 be promoted after a successful dev smoke, or wait for the W2 eval set too? | W2 |
| Q2 | Should tracing be disabled locally until LangSmith credentials are confirmed valid? | W2 |
| Q3 | Should diagnostic print instrumentation remain after Anthropic smoke passes, or be removed/converted to logger output? | W2 |
| Q4 | Free tier limit: 3 generations/month vs 3/week? | W6 |
| Q5 | Cache research results across users for same topic? Privacy vs cost. | W5 |
| Q6 | Pricing: $9/mo starter or $19/mo? Survey 5 potential users. | W6 |
| Q7 | Should fact-check threshold be user-configurable in paid tier? | W5 |
| Q8 | Domain name + brand identity | W6 |
| Q9 | Vietnamese-specific embedding model for source dedup? | W3 |

---

## Milestones Tracker

- [ ] **W1** - Project scaffold + Planner node (S01) working end-to-end with LangSmith trace
- [ ] **W2** - Research pipeline (S02 + S03 + S04) producing summarized sources
- [ ] **W3** - Outline (S05) + Writer (S06) streaming Vietnamese article end-to-end
- [ ] **W4** - Fact-check (S07) + rewrite loop + RAGAS eval harness functional
- [ ] **W5** - FastAPI complete + Supabase auth + per-user persistence
- [ ] **W6** - Next.js UI + Stripe sandbox + deploy live (Vercel + Railway)
- [ ] **W7** - Polish + portfolio writeup + soft launch to 5+ external users

**Current week**: W2 static validation complete; live smoke blocked by invalid Anthropic API key

---

## Active Risks

| Risk | Likelihood | Impact | Current mitigation status |
|---|---|---|---|
| Anthropic key/config blocks production-fidelity validation | High | High | Owner must replace placeholder-length key with real topped-up key |
| Anthropic billing may still block after key replacement | Medium | High | Owner reports credits topped up; verify after auth succeeds |
| LangSmith trace access may still be invalid | Medium | Medium | Recheck after smoke can complete |
| LLM costs spiral | Medium | High | S04 map now uses cheap model; source count capped |
| Vietnamese fact-check is hard | High | Medium | Deferred to W4 |
| Eval harness eats W4 time | High | Medium | W2 static debt is now clean |

---

## Quick Recovery Notes

1. Read `HANDOFF.md` first for the previous integration summary.
2. Current cleanup leaves static validation clean.
3. D-016 supersedes D-015: Anthropic is the dev smoke provider from W2 onward.
4. Full-scope smoke currently fails at planner with Anthropic HTTP 401 invalid x-api-key.
5. Replace `ANTHROPIC_API_KEY` with the real topped-up key, then rerun `uv run python scripts/smoke_research.py` from the repo root.
6. Then rerun `uv run ruff check app/`, `uv run ruff format --check app/`, `uv run mypy app/ --ignore-missing-imports`, and `uv run pytest tests/unit/graph/nodes/test_research.py -v` from `backend/` if code changes are made.
7. Do not mark S02-S04 implemented until the live smoke passes or the owner explicitly waives that gate.
