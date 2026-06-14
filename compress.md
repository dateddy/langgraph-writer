# Conversation Compression

> **Read this when**: a previous long conversation was compressed into a summary and you are continuing from that point.
>
> **Mutability**: Overwritten each compression. Archive previous to `compress/YYYY-MM-DD.md` if historically significant.

---

## Current Compression

**Compressed on**: 2026-06-14
**Conversation span**: ~32 turns — project bootstrap → W2 research integration → 3 provider attempts
**Compressed by**: Claude
**Status when compressed**: Paused before Anthropic top-up + .env revert + smoke retry

---

### What This Conversation Was About

Single-thread session that took the project from cold-start (project recommendations) through W0 (full context management system + RULES/SKILLS/scaffolding) to W2 (Research agent implementation + 3 Codex execution cycles). Most chat time was integration validation: Anthropic billing failed → Ollama too slow → Gemini blocked by Google's AQ. key migration. Session ends with pivot decision to top up Anthropic with $5 and run smoke against production stack instead of fighting free-tier providers.

---

### Decisions Made + Reasoning

All formally logged in `DECISIONS_LOG.md`. One-line summaries:

| ID | Decision | Reasoning |
|---|---|---|
| D-001 | Pure async functions for LangGraph nodes | Functions trivially testable; LangGraph provides composition |
| D-002 | All prompts in `app/prompts/templates.py` | Iterate 10-50x more than code; centralize for diffs |
| D-003 | Vietnamese-only output, NO English pipeline | Vietnamese is the moat; English dilutes focus |
| D-004 | Fact-check rewrite loop capped at `retry_count = 2` | Prevents unbounded token spend |
| D-005 | SSE (not WebSocket) for streaming | One-way push; simpler; Vercel-compatible |
| D-006 | Skills S01-S11 as project vocabulary | IDs survive renames; serve as taxonomy |
| D-007 | 5-file context management system | Each file has single mutability rule |
| D-008 | English-primary for meta-docs | Codex follows English better; portfolio international |
| D-009 | Single `research_node` for S02+S03+S04 | Simpler graph topology for W2 |
| D-010 | Fail-soft scrape with Tavily snippet fallback | One bad URL shouldn't kill batch |
| D-011 | Relevance threshold 0.3 for source filtering | Conservative MVP filter; recalibrate after evals |
| D-012 | Haiku for S04 map, Sonnet for reduce | Map = high-volume/low-judgment; cheap model fits |
| D-013 | `cast()` pattern for LangChain structured output | Library typing limit; better than `# type: ignore` |
| D-014 | `langchain-anthropic==1.4.5` pin with API drift fixes | Real API change: `model_name`, `max_tokens_to_sample`, `SecretStr` |
| D-015 | Ollama as dev LLM provider (free-tier) | $0 dev cost; aligned with budget memory |
| **D-016** *(pending log)* | **Anthropic for dev smoke from W2 onward, supersedes D-015** | **Ollama too slow on user hardware + Google AQ. migration blocks Gemini. $5 < 4hr debug time. Production fidelity now instead of W7.** |

D-016 marks D-015 status as `Superseded by D-016`. Free-tier dev strategy formally abandoned.

---

### Code/Outputs (Paths, Not Content)

**Backend code** (`backend/`):
- `app/graph/state.py` — `WriterState`, `SourceDoc`, `Fact`, `OutlineSection` TypedDicts
- `app/graph/nodes/research.py` — `research_node` + private async helpers (`_generate_queries`, `_execute_searches`, `_map_summarize`, `_reduce_summarize`) + env-driven scope knobs + diagnostic prints
- `app/graph/builder.py` — `planner → research → END` (W3 extends)
- `app/prompts/templates.py` — `PLANNER_SYSTEM`, `QUERY_GEN_*`, `SUMMARIZE_MAP_*`, `SUMMARIZE_REDUCE_*` (all Vietnamese)
- `app/models/schemas.py` — Pydantic v2: `QueryList`, `SourceSummary`, `RelevanceJudgment` + API req/resp
- `app/tools/tavily.py` — `AsyncTavilyClient` wrapper
- `app/tools/scraper.py` — httpx + BS4, UTF-8 force, VN noise selector removal
- `app/llm/client.py` — `get_llm()` factory, 3 provider branches (anthropic, openai, ollama) + `model_override` + `SecretStr`
- `app/config.py` — Settings with `llm_model_cheap`, `ollama_base_url`, `ollama_model`
- `app/utils/{logger,exceptions}.py` — cached logger factory + domain exceptions
- `tests/unit/graph/nodes/test_research.py` — 6 mocked tests, all pass
- `scripts/smoke_research.py` — `astream()` diagnostic version with per-node prints (Windows UTF-8 reconfigure pending)
- `pyproject.toml` — `langchain-anthropic==1.4.5` pinned, `langchain-ollama` added

**Project meta-documentation** (root):
- `context.md`, `RULES.md`, `SKILLS.md`, `SETUP_GUIDE.md`, `CURRENT_STATE.md`, `DECISIONS_LOG.md`, `HANDOFF.md`, `compress.md`

**Codex prompts** (used in order, all in outputs):
- `CODEX_AUDIT_PROMPT.md` → verdict NEEDS_REVIEW
- `RESEARCH_IMPL_BUNDLE.md` → single-file bundle of 10 .py files
- `CODEX_SURGICAL_PROMPT.md` → Path C cleanup
- `CODEX_OLLAMA_PROMPT.md` → Ollama integration
- `CODEX_DIAGNOSTIC_PROMPT.md` → instrumentation + reduced scope
- `CODEX_GEMINI_PROMPT.md` → abandoned at Phase 1

---

### Problems Encountered + Solutions

| # | Problem | Root cause | Solution |
|---|---|---|---|
| 1 | Audit prompt `[BLOCKED]`: workspace only has pasted-text.txt | Reference files not attached | Created `RESEARCH_IMPL_BUNDLE.md` with regex-extractable file markers |
| 2 | Anthropic smoke: `invalid x-api-key` | User pasted real API key in chat (security incident) | Key rotation; established never-paste-in-chat rule |
| 3 | mypy 10 errors on `ChatAnthropic(model=..., max_tokens=...)` | Real `langchain-anthropic==1.4.5` API drift, NOT typing noise | Codex inspected library; renamed params; `SecretStr` for api_key; version pin → D-014 |
| 4 | mypy 5 errors on `with_structured_output()` return type | LangChain typing limitation | `cast(Schema, await llm.ainvoke())` pattern → D-013 |
| 5 | Anthropic smoke: HTTP 400 credit balance too low | No credits in account | Pivot to free-tier providers (Ollama → Gemini) |
| 6 | Ollama smoke: exit 124 at 604.4s, no stdout | qwen2.5:7b on local hardware ~30s/call × 20+ sequential calls | Added diagnostic instrumentation (astream + per-phase prints); still too slow for user hardware |
| 7 | Codex `.env` shape check fails twice | First: `GEMINI_API_KEY` not `GOOGLE_API_KEY`; Second: key format `AQ.` not `AIza` | First: renamed env var to library convention. Second: **web-confirmed Google migrated user's account to new AQ. token format that no third-party tool supports yet** (rolled out past 2 weeks; June 19 2026 cutoff in 5 days). Abandoned Gemini. |
| 8 | Windows UnicodeEncodeError piping VN stdout through Tee-Object | Windows default cp1252 codec | `sys.stdout.reconfigure(encoding="utf-8")` at top of smoke script — **pending** |
| 9 | User pasted Anthropic key in chat | Treated AI chat as private | Security warning, key rotation, memory rule not to store keys |

---

### What's Pending (Active Threads)

**Immediate (next 15-20 min of user work)**:
1. Top up Anthropic at `console.anthropic.com/settings/billing` → $5 min
2. Revert `.env`:
   ```
   LLM_PROVIDER=anthropic
   LLM_MODEL=claude-sonnet-4-5
   LLM_MODEL_CHEAP=claude-haiku-4-5-20251001
   ```
   Comment out Gemini block + keep Ollama block commented as fallback
3. Reply to Codex with the pivot message (provided in turn just before this compression)

**Codex execution (next 30-50 min)**:
- Log D-016 in `DECISIONS_LOG.md`; mark D-015 status `Superseded by D-016`
- Apply Windows UTF-8 reconfigure to `scripts/smoke_research.py`
- Run `uv run python scripts/smoke_research.py 2>&1 | Tee-Object -FilePath smoke_anthropic.log` at FULL scope
- Verify smoke: ≥5 sources with Vietnamese summaries, ≥1 fact each
- Update manifests (`SKILLS.md` S02-S04 → 🟢, `HANDOFF.md`, `CURRENT_STATE.md` W2 milestone checked)
- Output audit per Section 1-10 schema

**Owner review (5 min)**: Paste audit to Claude for verification

**Post-W2 (next chat)**: Open new chat for W3 / S05 Outline Generator

---

### Specific Next Steps (Numbered)

1. **User**: Top up Anthropic $5 (5 min)
2. **User**: Edit `.env` per pivot template (2 min)
3. **User**: Paste Codex pivot message into existing Codex chat (instant)
4. **Codex**: Log D-016 + supersede D-015 (5 min)
5. **Codex**: Apply Windows UTF-8 fix to smoke script (2 min)
6. **Codex**: Run full-scope smoke with Anthropic (3-5 min)
7. **Codex**: Update manifests if smoke passes (10 min)
8. **Codex**: Produce audit (10 min)
9. **User**: Paste audit to Claude (instant)
10. **Claude**: Verify audit Honesty Checklist (12 items), §5 deviations, smoke output quality, manifest consistency
11. **If PASS**: W2 done. Open new chat for W3 / S05.

Default assumption for new session: **W2 needs final audit verification then closes**.

---

### User Preferences (Persist Across Sessions)

- **Workflow**: Delegate implementation to Codex via structured prompts; Claude designs + reviews audits. `[BLOCKED]` honest-stop pattern is the safety mechanism user values.
- **Doc language**: English-primary (D-008). Vietnamese only inside prompt content + sample outputs.
- **Cost philosophy**: Started cost-conscious (free tier strategy); D-016 accepts paid Anthropic for dev. Lesson learned: time-cost > money-cost when free-tier debugging takes hours.
- **Format**: Tables and visual widgets for comparing options; concise bullets over paragraphs; code blocks only when actually code.
- **Honesty over polish**: User pushes back on AI hiding deviations or fabricating. Codex prompts always include Honesty Checklist.
- **No emojis except status indicators** (🟢🟡🔴 in SKILLS.md).
- **Decision-making**: Reads recommendations, picks path explicitly, expects execution. Don't ask "what do you think" repeatedly — give recommendation + reasoning.
- **Time priority**: Stated explicitly in W2 — "TỐI ƯU HÓA THỜi GIAN DEV". Feedback loop speed > perfect free-tier compliance.

---

### Continuity Notes (Do NOT Re-Discover)

1. **Google API key format migration is real and ongoing**. Google rolled out new `AQ.Ab8R...` format to user's account ~1 week before this session (early June 2026). `langchain-google-genai` does not support this format. June 19, 2026 deadline for old standard keys. **Do NOT suggest Gemini as dev provider for this user** — their account is migrated.
2. **Ollama qwen2.5:7b on user's hardware**: ~30s per LLM call. For 20+ sequential calls = 10+ minutes. Not viable for tight dev iteration. Branch stays in `client.py` as documented offline alternative.
3. **`langchain-anthropic==1.4.5` API**: Constructor uses `model_name` (not `model`), `max_tokens_to_sample` (not `max_tokens`), `api_key: SecretStr` (not `str`). Pinned to prevent regression.
4. **Codex is reliable when prompt enforces honesty**. All 3 audits this session had perfect honesty checklists. `[BLOCKED]` format prevents fabricated proceed.
5. **All unit tests pass under mocks (6/6)**. Live smoke is the only remaining W2 gate.
6. **Anthropic account billing was set up correctly** — only credits missing. Top-up is credit-card transaction, no account setup needed.
7. **User on Windows + PowerShell**. Bash commands need translation. `.env` encoding verified clean (no BOM, first 3 bytes are `# `).
8. **Map-reduce uses `asyncio.gather` but Ollama serializes inference** — this is why Ollama was slow. Cloud providers (Anthropic/OpenAI) truly parallelize.
9. **Diagnostic instrumentation in research.py + smoke_research.py is still active** — owner may want to remove verbose `print()` calls after W2 closes and rely on `logger.info` + LangSmith trace only.
10. **`scripts/smoke_research.py` Windows UTF-8 fix not yet applied** — needed before any smoke can run cleanly on Windows.

---

### Files Currently Mid-Edit

| File | Status when compressed |
|---|---|
| `scripts/smoke_research.py` | Diagnostic `astream()` ready; needs `sys.stdout.reconfigure(encoding="utf-8")` at top |
| `.env` | Currently `LLM_PROVIDER=gemini` + invalid `AQ.` key; needs revert to Anthropic |
| `DECISIONS_LOG.md` | D-015 status still "Active"; needs update to "Superseded by D-016" + D-016 entry |
| `SKILLS.md` | S02-S04 at 🟡 In Progress; will move to 🟢 if smoke passes |
| `HANDOFF.md` | Reflects last Codex session (diagnostic ready); needs overwrite after next smoke |
| `CURRENT_STATE.md` | Shows B-001 (Ollama timeout) + B-002 (Anthropic billing); both resolve after Anthropic top-up |

---

### Open Questions (Not Resolved)

| ID | Question | Target |
|---|---|---|
| Q1 | Promote S02-S04 after dev smoke, or wait for W2 eval set too? | W2 close |
| Q2 | Disable LangSmith tracing locally until key verified valid? | W2 close |
| Q3 | Remove diagnostic prints from research.py after W2? | W2 close |
| Q4 | Free tier limit: 3 generations/month vs 3/week? | W6 |
| Q5 | Cache research results across users? | W5 |
| Q6 | Pricing: $9/mo vs $19/mo starter? | W6 |
| Q7 | Fact-check threshold user-configurable in paid tier? | W5 |
| Q8 | Domain + brand identity | W6 |
| Q9 | Vietnamese-specific embedding model for source dedup? | W3 |

---

## Compression Quality Checklist

- [x] Could a new Claude session pick up from here? Yes — next steps numbered 1-11
- [x] All decisions in `DECISIONS_LOG.md` or here? Yes — D-001 to D-015 logged, D-016 pending log
- [x] Active Threads specific enough to resume in <5 min? Yes — user action 1-3 takes 15 min
- [ ] `HANDOFF.md` updated to reference this compression? **Owner: add a one-line note when receiving this file**
- [x] `CURRENT_STATE.md` consistent? Yes — B-001/B-002 still pending, matches "Pending" section here
- [x] Under 500 lines? Yes (~250 lines)

---

## Anti-Patterns to Avoid Next Compression

1. Don't paraphrase what's in `DECISIONS_LOG.md` / `SKILLS.md` / `RULES.md` — link instead
2. Don't include code blocks — reference paths
3. Don't write narrative — bullets and tables are denser
4. Don't compress prematurely (< 30 turns usually doesn't need it)
5. Don't compress without owner confirmation

---

*Memory of long conversations. Sloppy compression compounds across sessions.*