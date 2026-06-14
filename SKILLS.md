# Skills Catalog

> Index of all agent skills planned and implemented for `langgraph-writer`. Each skill is a single, named, testable capability — usually realized as one LangGraph node plus its prompt, optional tools, and eval criteria.
>
> When you build a skill, you implement: (1) a node function, (2) a prompt template, (3) optional tool wrappers, (4) unit + integration tests, (5) at least one eval metric. Skills are the project's vocabulary — refer to them by ID in commits, PRs, and discussions.

---

## Status Legend

| Symbol | Meaning |
|---|---|
| 🔴 | **Planned** — no code yet |
| 🟡 | **In Progress** — partial implementation, tests missing |
| 🟢 | **Implemented** — node + tests + at least one eval pass |
| 🔵 | **Optimized** — benchmarked, prompt iterated ≥2 times, hits target metric |
| ⚫ | **Deprecated** — replaced or removed (keep entry for history) |

---

## Skill Index

| ID | Name | Status | Phase | Owner of node | Depends on |
|---|---|---|---|---|---|
| S01 | Topic Planner | 🔴 | W1 | `nodes/planner.py` | — |
| S02 | Query Generator | 🟡 | W2 | `nodes/research.py` (sub-step) | S01 |
| S03 | Web Researcher | 🟡 | W2 | `nodes/research.py` | S02 |
| S04 | Source Summarizer | 🟡 | W2 | `nodes/research.py` (sub-step) | S03 |
| S05 | Outline Generator | 🔴 | W3 | `nodes/outline.py` | S04 |
| S06 | Content Writer | 🔴 | W3 | `nodes/writer.py` | S05 |
| S07 | Fact Checker | 🔴 | W4 | `nodes/fact_check.py` | S06 |
| S08 | Style Editor | 🔴 | post-MVP | `nodes/editor.py` | S06 |
| S09 | SEO Optimizer | 🔴 | post-MVP | `nodes/seo.py` | S06 |
| S10 | Image Suggester | 🔴 | post-MVP | `nodes/images.py` | S06 |
| S11 | Citation Formatter | 🔴 | post-MVP | `nodes/citations.py` | S07 |

---

## Core Skills (MVP)

### S01 — Topic Planner

**Status**: 🔴 Planned · **Phase**: Week 1

**Purpose**: Analyze the user's raw topic input, identify research angles, target audience, and the rhetorical stance the article should take. Acts as the strategic brief for downstream agents.

**Inputs** (from state):
- `topic: str` — raw user input, e.g., `"Tác động của AI đến thị trường lao động Việt Nam"`
- `tone: str` — one of `trang trọng | thân mật | học thuật`
- `target_length: int` — word count

**Outputs** (written to state):
- `plan: str` — markdown, 3-5 bullet research directions

**LLM**: Claude Sonnet 4.5, `temperature=0.5` (slight creativity, mostly deterministic).

**Tools**: None — pure LLM call.

**Prompt**: `PLANNER_SYSTEM` in `app/prompts/templates.py`.

**Eval metrics**:
- *Coverage*: does the plan name at least 3 distinct angles? (LLM judge)
- *Vietnamese fluency*: human review on a 5-pt scale
- *Specificity*: does the plan reference the actual topic, not generic boilerplate?

**Acceptance criteria**: Produces a 3-5 bullet plan, in Vietnamese, addressing the specific topic given. Coverage ≥3 on 10 test topics.

---

### S02 — Query Generator

**Status**: 🟡 In Progress · **Phase**: Week 2

**Purpose**: Convert the plan into 3-5 diverse search queries suitable for Tavily. Mixes Vietnamese and English queries when English sources are likely better (e.g., academic, statistical).

**Inputs**: `plan: str`, `topic: str`

**Outputs**: `queries: list[str]` (3-5 items)

**LLM**: Claude Sonnet 4.5, `temperature=0.7` (encourage diversity).

**Tools**: Pydantic structured output for the query list.

**Prompt**: `QUERY_GEN_SYSTEM` in templates.py.

**Eval metrics**:
- *Diversity*: cosine distance between query embeddings should average >0.4
- *Hit rate*: % of queries returning ≥3 Tavily results
- *Bilingual ratio*: at least 1 Vietnamese + 1 English query when applicable

**Acceptance criteria**: Produces 3-5 queries with diversity score >0.4, hit rate >80% on test set.

---

### S03 — Web Researcher

**Status**: 🟡 In Progress · **Phase**: Week 2

**Purpose**: Execute Tavily search for each query, fetch top URLs, extract clean readable text via scraper, deduplicate by URL.

**Inputs**: `queries: list[str]`

**Outputs** (appended via `Annotated[..., add]` reducer):
- `sources: list[SourceDoc]` where `SourceDoc = {url, title, content, score}`

**LLM**: None — pure tool calls.

**Tools**:
- `tools/tavily.py`: `tavily_search(query, max_results=5)` async wrapper
- `tools/scraper.py`: `fetch_clean_text(url)` using httpx + BeautifulSoup

**Eval metrics**:
- *Source relevance*: LLM-judged relevance score, target >0.7
- *Recency*: median publish date within last 12 months
- *Vietnamese ratio*: ≥30% of sources in Vietnamese when topic is VN-specific
- *Diversity of domains*: no single domain >40% of total sources

**Acceptance criteria**: Produces 8-15 unique sources per topic with relevance >0.7.

---

### S04 — Source Summarizer

**Status**: 🟡 In Progress · **Phase**: Week 2

**Purpose**: Run map-reduce summarization across the raw sources to extract atomic facts, statistics, quotes, and expert opinions. Tags each fact with its source URL for downstream citation.

**Inputs**: `sources: list[SourceDoc]` (raw content)

**Outputs**: enriched `sources` with `summary: str` and `facts: list[Fact]` per doc, where `Fact = {claim: str, source_idx: int}`

**LLM**: Claude Haiku (cheaper, map step) → Claude Sonnet (reduce step).

**Tools**: None — pure LLM map-reduce.

**Prompt**: `SUMMARIZE_MAP_SYSTEM`, `SUMMARIZE_REDUCE_SYSTEM` in templates.py.

**Eval metrics**:
- *Compression ratio*: summary length / source length, target 0.1-0.2
- *Fact preservation*: % of key facts in source retained (manual sample audit)
- *No hallucination*: 0 facts in summary that aren't in source (LLM judge + spot check)

**Acceptance criteria**: 80%+ key facts retained, 0% hallucinated facts on 5-source audit.

---

### S05 — Outline Generator

**Status**: 🔴 Planned · **Phase**: Week 3

**Purpose**: Produce a structured outline (4-6 sections, each with 2-4 key points) that flows logically from intro → body → conclusion.

**Inputs**: `topic`, `plan`, `sources` (with summaries)

**Outputs**: `outline: list[OutlineSection]` where `OutlineSection = {heading: str, key_points: list[str]}`

**LLM**: Claude Sonnet 4.5, `temperature=0.6`.

**Tools**: Pydantic structured output enforced.

**Prompt**: `OUTLINE_SYSTEM` in templates.py.

**Eval metrics**:
- *Structural correctness*: passes Pydantic schema validation
- *Logical flow*: LLM judge scores narrative coherence 1-5
- *Coverage*: outline references ≥60% of source facts

**Acceptance criteria**: Validates schema, flow score ≥4, coverage ≥60% on test set.

---

### S06 — Content Writer

**Status**: 🔴 Planned · **Phase**: Week 3

**Purpose**: Generate the full article in Vietnamese, section by section, grounded in the outline and supported by source summaries. This is the most expensive node — uses streaming output.

**Inputs**: `topic`, `tone`, `target_length`, `outline`, `sources`

**Outputs**: `draft: str` (markdown, Vietnamese)

**LLM**: Claude Sonnet 4.5, `temperature=0.7`, `streaming=True`.

**Tools**: None — single long LLM call (or section-by-section if length >1500 words).

**Prompt**: `WRITER_SYSTEM` in templates.py — emphasizes natural Vietnamese, avoids "translation-ese", forbids platitudes.

**Eval metrics**:
- *Length*: actual word count within ±20% of target
- *Vietnamese naturalness*: human or LLM judge on 5-pt scale (target ≥4)
- *Source grounding*: % of factual claims traceable to a source (target ≥80%)
- *Style consistency*: tone matches the requested register

**Acceptance criteria**: Length within range, naturalness ≥4, grounding ≥80%, streaming works end-to-end.

---

### S07 — Fact Checker

**Status**: 🔴 Planned · **Phase**: Week 4

**Purpose**: Audit the draft against the source material. Extract each factual claim (number, date, name, quote) and verify it appears in or is supported by the sources. Produce a score and a list of issues; trigger a rewrite if score < threshold.

**Inputs**: `draft`, `sources`

**Outputs**:
- `fact_check_score: float` in [0.0, 1.0]
- `fact_check_issues: list[str]`
- `retry_count: int` (incremented)

**Conditional edge**: If `score < 0.7` AND `retry_count < 2` → loop to Writer. Else → END.

**LLM**: Claude Sonnet 4.5, `temperature=0.2` (deterministic, conservative).

**Tools**: None — LLM-only fact comparison.

**Prompt**: `FACT_CHECK_SYSTEM` in templates.py — instructs strict comparison, JSON output.

**Eval metrics**:
- *Recall*: % of injected fabrications caught (build a test set of 20 articles with planted errors)
- *Precision*: % of flagged "issues" that are actually issues (low false positive rate)
- *Score calibration*: does the score correlate with human-judged factuality?

**Acceptance criteria**: Recall ≥60% on injected-error test set, precision ≥70%.

---

## Extension Skills (Post-MVP)

### S08 — Style Editor

**Purpose**: Refine Vietnamese prose for naturalness. Removes "dịch máy" patterns (e.g., literal English sentence structures, awkward calques), tightens sentences, improves rhythm.

Triggers after S06 if a separate style-pass is enabled. Could also run as part of the rewrite loop in S07.

---

### S09 — SEO Optimizer

**Purpose**: Suggest title variants, meta description, H2/H3 placement, keyword density tuning. Optional final pass for users in the paid tier.

---

### S10 — Image Suggester

**Purpose**: Recommend stock image search terms or AI-generation prompts for each section. Does not generate images directly in MVP.

---

### S11 — Citation Formatter

**Purpose**: Convert `Fact{claim, source_idx}` references into inline citations (APA, MLA, or numeric superscripts) and a bibliography section.

---

## Skill Dependency Graph

```
        ┌─────────────┐
        │ S01 Planner │
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ S02 Query   │
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ S03 Research│
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ S04 Summarize│
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ S05 Outline │
        └──────┬──────┘
               ▼
        ┌─────────────┐
        │ S06 Writer  │◄─────────┐
        └──────┬──────┘          │
               ▼                 │ rewrite loop
        ┌─────────────┐          │
        │ S07 FactChk │──────────┘
        └──────┬──────┘
               ▼
              END

(Post-MVP)
S06 ─► S08 Style Editor
S06 ─► S09 SEO Optimizer
S06 ─► S10 Image Suggester
S07 ─► S11 Citation Formatter
```

---

## How to Add a New Skill

1. Pick the next sequential ID (S12, S13, ...).
2. Add a row to the Skill Index table at the top.
3. Add a full entry following the template below.
4. Update the Dependency Graph if it introduces new edges.
5. Open a tracking issue / PR referencing the skill ID.
6. Do **not** mark the skill 🟡 In Progress until the prompt template lands in `templates.py`.

---

## Skill Entry Template

```markdown
### S{NN} — {Name}

**Status**: 🔴 Planned · **Phase**: Week {N}

**Purpose**: One paragraph describing what this skill does and why it exists.

**Inputs**: (from state)
- `field_name: type` — description

**Outputs**: (written to state)
- `field_name: type` — description

**LLM**: Provider, model, temperature, streaming?

**Tools**: List of tools used, or "None".

**Prompt**: `CONSTANT_NAME` in `app/prompts/templates.py`.

**Eval metrics**:
- *Metric name*: target value

**Acceptance criteria**: Concrete pass/fail conditions.
```

---

*Last updated: Week 0. When a skill changes status, update both the Skill Index table and the full entry. Never delete a skill entry — mark it ⚫ Deprecated if removed.*
