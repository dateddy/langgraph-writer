# Project Context — Immutable Foundation

> **Read this file when**: you are a new Claude/Codex session and need to understand WHAT this project is, WHY it exists, and WHAT is locked. This file changes rarely (maybe 2-3 times across the project lifetime). For real-time state, read `CURRENT_STATE.md`. For decision history, read `DECISIONS_LOG.md`.
>
> **Mutability**: Updates require explicit user approval. If you find yourself wanting to change this file, stop and ask first.

---

## Identity

**Project name**: `langgraph-writer`

**One-line description**: A multi-agent Vietnamese content writing pipeline that produces fact-checked, sourced articles in under 5 minutes.

**Why it exists**:
1. Solo developer's portfolio + recruiter impression piece (CS junior, AI Engineering track)
2. Validate a SaaS hypothesis: Vietnamese content creators will pay for AI writing that's actually good in Vietnamese
3. Vehicle for learning production-grade LangGraph, harness engineering, and MLOps

---

## Goals (Concrete and Measurable)

The project succeeds if, by end of Week 7:

| # | Goal | Measurement |
|---|---|---|
| G1 | End-to-end pipeline works | 800-word Vietnamese article generated in <5 min |
| G2 | Fact-check is non-trivial | Catches ≥60% of injected fabrications on 20-article test set |
| G3 | Eval harness exists | LangSmith dashboard shows traces for every run; RAGAS metrics computed automatically |
| G4 | Product is live | Public URL with auth, 3 free generations/user, Stripe sandbox transactions work |
| G5 | Portfolio quality | Public GitHub with README + architecture diagram + demo GIF + technical writeup |
| G6 | User-validated | At least 5 external users successfully generate articles without help |

Goals are **immutable**. Trading off a goal requires a logged decision in `DECISIONS_LOG.md`.

---

## Constraints (Hard Limits)

| Constraint | Value | Why |
|---|---|---|
| Operating cost | <$50/month | Solo dev, no funding |
| Build time | 7 weeks, ~15 hrs/week (~105 hrs total) | Summer break window |
| Team size | 1 (solo) | No collaborators in MVP |
| Target audience | Vietnamese-speaking content creators | The moat — VN-first |
| Output language | Vietnamese only | See ADR-003 in DECISIONS_LOG.md |

---

## Tech Stack (Locked)

These choices are decided. Changing any requires a logged decision.

### Backend
- **Runtime**: Python 3.12
- **Package manager**: uv
- **Orchestration**: LangGraph 0.2+
- **LLM framework**: LangChain 0.3+
- **LLM provider**: Anthropic Claude (primary), OpenAI (fallback only)
- **Models**: Claude Sonnet 4.5 (heavy nodes), Claude Haiku (cheap nodes like planner)
- **HTTP**: FastAPI + sse-starlette
- **Validation**: Pydantic v2

### Tools & Data
- **Web search**: Tavily
- **HTTP client**: httpx (async)
- **Web parsing**: BeautifulSoup4
- **DB + Auth**: Supabase (Postgres + Auth + RLS)

### Frontend
- **Framework**: Next.js 14 (App Router)
- **Deploy**: Vercel

### Observability & Quality
- **Tracing**: LangSmith
- **Evaluation**: RAGAS + custom LLM judges
- **Testing**: pytest, pytest-asyncio
- **Linting**: ruff
- **Type checking**: mypy

### Payments
- **Stripe** (subscription model)

### Forbidden libraries
- `requests`, `aiohttp` (use `httpx`)
- `flask` (use `FastAPI`)
- `openai` SDK directly (go through LangChain)
- Legacy `langchain.agents` API (use LangGraph)

---

## Scope — Locked at Week 0

### In Scope (MVP)
1. Pipeline: `planner → research → outline → writer → fact-check` with rewrite loop
2. Vietnamese output (only)
3. FastAPI backend with SSE streaming
4. Next.js editor UI
5. Supabase auth + per-user history
6. Stripe subscription (free + $9/mo paid tier)
7. LangSmith tracing
8. RAGAS eval harness
9. Public GitHub repo with full docs

### Out of Scope (Explicit Non-Goals)
- ❌ English-language output pipeline
- ❌ Native mobile apps
- ❌ Voice / TTS
- ❌ AI image generation
- ❌ Real-time collaboration on a doc
- ❌ Built-in plagiarism detection
- ❌ White-label / multi-tenant
- ❌ On-prem / self-hosted
- ❌ CMS webhook integrations (WordPress, etc.)
- ❌ Chrome extension
- ❌ Fine-tuning custom models

---

## Coding Conventions

See `RULES.md` for the complete enforced ruleset. Key invariants (these never change):

- One node = one async function = one file in `app/graph/nodes/`
- All prompts live in `app/prompts/templates.py` — no exceptions
- Vietnamese prompts stay in Vietnamese — never translated
- State updates: return only changed fields, never the full state
- Every external I/O call must be `async/await`

---

## Glossary

Project-specific terms used throughout the codebase and docs.

| Term | Meaning |
|---|---|
| **Skill (S01-S11)** | A single agent capability — see `SKILLS.md` for the full catalog |
| **Node** | A LangGraph node = one async function implementing a skill |
| **State** | The `WriterState` TypedDict that flows through the graph |
| **Draft** | The article output produced by Writer (S06) before fact-check |
| **Fact-check loop** | Conditional edge from Fact Checker (S07) back to Writer (S06) when score < 0.7 |
| **Skill ID** | Reference identifier like `S03`, used in commits, PRs, and discussions |
| **Harness** | The evaluation infrastructure (LangSmith + RAGAS + custom judges) |
| **Drift** | Any deviation from initial scope — must be logged in `DECISIONS_LOG.md` |
| **Map-reduce summarization** | Pattern used in S04 where each source is summarized individually (map) then combined (reduce) |

---

## Key Invariants

Things that must **always** be true. If any of these become false, the project has deviated from its identity and requires a major pivot decision.

1. **Vietnamese is the moat.** Every feature must serve Vietnamese output quality.
2. **The harness exists.** Eval infrastructure is not optional, not "later" — it's day-1.
3. **The pipeline is a graph, not a script.** Linear scripts that "look like" agents are forbidden.
4. **Prompts are versioned.** Iteration history is preserved, not overwritten.
5. **Decisions are logged.** No oral tradition — if it's not in `DECISIONS_LOG.md`, it didn't happen.

---

## File Map

Where to find what:

| File | Purpose | Mutability |
|---|---|---|
| `context.md` | This file. Identity, goals, constraints, tech stack. | Rarely changes |
| `CURRENT_STATE.md` | Real-time dashboard: active task, blockers, open questions, milestones | Updated frequently |
| `DECISIONS_LOG.md` | Append-only history of all decisions and drifts | Append-only |
| `HANDOFF.md` | End-of-session bridge between Claude/Codex sessions | Overwritten each session |
| `compress.md` | Compressed summary when a chat gets too long | Overwritten each compression |
| `RULES.md` | Codex GPT coding rules (folder, style, async, prompts) | Rarely changes |
| `SKILLS.md` | Catalog of all 11 agent skills with status | Updated as skills progress |
| `SETUP_GUIDE.md` | Step-by-step setup commands and file scaffolding | Rarely changes |

---

## Reading Protocol for New Sessions

When you (Claude/Codex) start a fresh session on this project:

1. **First**: Read `HANDOFF.md` — what was the previous session doing
2. **If references to "we decided X"**: Check `DECISIONS_LOG.md` for the reasoning
3. **If references to "we're working on Y"**: Check `CURRENT_STATE.md` for live state
4. **If foundational confusion**: Read this file (`context.md`)
5. **Before writing any code**: Read `RULES.md`

---

*Last reviewed: Week 0. Next mandatory review: end of Week 4 (mid-project) or when a logged decision contradicts something here.*
