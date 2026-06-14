# Codex GPT — Project Rules

> You are coding for **`langgraph-writer`**, a multi-agent Vietnamese content writing pipeline. Follow every rule in this document. When a rule conflicts with a request, surface the conflict and ask — do not silently ignore the rule.

---

## 1. Project Context

- **What**: Multi-agent pipeline that produces fact-checked Vietnamese articles from a topic input.
- **Pipeline**: `planner → research → outline → writer → fact-check` (with conditional rewrite loop).
- **Owner**: Solo developer, CS junior. Timeline: 7 weeks. Target: portfolio + SaaS launch.
- **End user output language**: Vietnamese. **Code/comments/docs language**: English.

---

## 2. Locked Tech Stack

Stay within this stack. If you need something outside it, flag the gap and propose — do not introduce silently.

| Layer | Tool | Version |
|---|---|---|
| Runtime | Python | 3.12 |
| Package manager | uv | latest |
| Orchestration | LangGraph | 0.2+ |
| LLM framework | LangChain | 0.3+ |
| LLM provider | Anthropic Claude (primary), OpenAI (fallback) | — |
| HTTP framework | FastAPI | latest |
| Streaming | sse-starlette | latest |
| Validation | Pydantic | v2 only |
| HTTP client | httpx | latest (async) |
| Web parsing | BeautifulSoup4 | latest |
| Search tool | Tavily | latest |
| Observability | LangSmith | latest |
| Evaluation | RAGAS | latest |
| Testing | pytest, pytest-asyncio | latest |
| Linter/Formatter | ruff | latest |
| Type checker | mypy | latest |

**Forbidden libraries**: `requests` (use httpx), `aiohttp` (use httpx), `flask` (use FastAPI), `langchain.agents` legacy API (use LangGraph), `openai` SDK directly (go through LangChain wrappers).

---

## 3. File & Folder Conventions

```
backend/app/
├── main.py              # FastAPI entrypoint ONLY — no business logic
├── config.py            # pydantic-settings Settings class ONLY
├── api/routes.py        # HTTP routes. Thin — delegate to graph immediately.
├── graph/
│   ├── state.py         # WriterState TypedDict ONLY
│   ├── builder.py       # build_writer_graph() ONLY
│   ├── edges.py         # Conditional routing functions ONLY
│   └── nodes/           # One file per node. File name == node function name.
├── prompts/templates.py # All prompts. No prompt text lives outside this file.
├── tools/               # External tool wrappers (Tavily, scraper).
├── llm/client.py        # get_llm() factory ONLY.
├── models/schemas.py    # API request/response schemas (Pydantic).
└── utils/               # Logger, streaming helpers, pure utilities.
```

**Rules**:
- Never create a file outside this tree without proposing the location first.
- One node per file in `nodes/`. File `nodes/writer.py` exports `writer_node`. No more.
- `main.py` and `config.py` are append-only — do not refactor their structure without asking.
- Tests mirror source tree: `tests/unit/graph/nodes/test_writer.py` for `app/graph/nodes/writer.py`.

---

## 4. Python Code Conventions

### Style
- Format: `ruff format` — code must pass `ruff check` before commit.
- Line length: 100.
- Quotes: double quotes for strings, single for dict keys when natural.
- Imports: stdlib → third-party → local, blank line between groups, sorted alphabetically within groups.
- F-strings only. Never `%` or `.format()`.
- Trailing commas in multi-line function calls and lists.

### Type hints (mandatory)
- Every function signature has full type hints, including `-> None` for void.
- Use `from __future__ import annotations` at the top of every module that uses forward refs.
- Prefer `list[T]`, `dict[K, V]`, `T | None` over `List`, `Dict`, `Optional` (Python 3.10+).
- Use `Annotated[type, ...]` for Pydantic fields and LangGraph reducers.

### Async
- Every function that makes an LLM call, HTTP call, or DB call MUST be `async def`.
- Inside an `async` function, every I/O call is `await`-ed. No sync I/O in async paths.
- Never wrap sync code in `asyncio.to_thread` without explaining why in a comment.

### Error handling
- Never use bare `except:`. Catch specific exception types.
- Never use `except Exception:` at the top level of a node — let it propagate to the graph error handler.
- Inside tools/external calls: catch the specific provider error, log, then re-raise as a domain exception from `app/utils/exceptions.py`.

### Logging
- Use `app.utils.logger.get_logger(__name__)` — never `print()`.
- Log levels: DEBUG for verbose, INFO for milestones, WARNING for degraded, ERROR for failures.
- Never log API keys, full prompts containing user PII, or raw LLM outputs >500 chars.

---

## 5. LangGraph Conventions

### Node functions
```python
async def writer_node(state: WriterState) -> dict:
    """One-line purpose. Vietnamese context if relevant."""
    # 1. Read needed fields from state (do not mutate state)
    # 2. Call LLM/tool
    # 3. Return ONLY the fields you update
    return {"draft": result}
```

- Always `async def`.
- First arg is `state: WriterState` (or specific subset).
- Return a `dict` containing only updated fields — NOT the full state.
- No side effects beyond LLM/tool calls and logging.
- Never call another node from inside a node — use edges.

### State updates
- For list fields that accumulate (e.g., `sources`): declare with `Annotated[list[T], add]` in `state.py`.
- For scalar/dict fields: plain types — LangGraph overwrites on update.
- Never modify state in-place. Always return a new dict.

### Edges
- Linear edges go directly in `builder.py`.
- Conditional edges: define routing function in `edges.py`, return string literals matching the routing map keys.
- Always include a checkpointer:
  - Dev: `MemorySaver()`
  - Prod: `SqliteSaver.from_conn_string("checkpoints.sqlite")` or Postgres equivalent.

---

## 6. Prompt Engineering Conventions

### Storage
- **All prompts live in `app/prompts/templates.py`**. No exceptions.
- Each prompt is a module-level constant: `PLANNER_SYSTEM`, `WRITER_SYSTEM`, etc.
- User-facing template strings use Python `.format()` with named placeholders, e.g., `"Tone: {tone}"`.

### Language
- **Vietnamese prompts stay in Vietnamese**. Do not translate to English when refactoring or "improving readability".
- Vietnamese examples in few-shot prompts must use correct diacritics — never strip them.
- Tone instructions in Vietnamese use Vietnamese terms: `trang trọng`, `thân mật`, `học thuật` — not `formal`, `casual`, `academic`.

### Versioning
- When iterating a prompt, version it: `WRITER_SYSTEM_V2` next to `WRITER_SYSTEM_V1`. Do not overwrite.
- Update which version is active by changing the import in the node file, not by mutating the template.
- Remove deprecated versions only after 1 week of stability + approval.

### Structure
Every system prompt follows this order:
1. **Role** — who the agent is, 1 sentence.
2. **Task** — what it does, 1-2 sentences.
3. **Constraints** — hard rules as a bullet list.
4. **Output format** — exact format expected (JSON schema, markdown, plain text).
5. **Few-shot examples** — optional, 1-3 examples max.

---

## 7. Pydantic Conventions

- Pydantic v2 syntax: `model_config = ConfigDict(...)`, never `class Config:`.
- API contracts in `app/models/schemas.py`, graph state in `app/graph/state.py`. Do not mix.
- Use `Field()` for constraints: `Field(min_length=1, max_length=200)`.
- Use `Annotated[type, Field(...)]` when combining with other metadata.
- Validators: `@field_validator("field_name")` for single field, `@model_validator(mode="after")` for cross-field.

---

## 8. Forbidden Patterns

Never do any of the following. If asked to, refuse and explain why.

1. **Mixing sync and async** in the same call chain.
2. **Hardcoding prompts** inside node files. They go in `templates.py`.
3. **Translating Vietnamese prompts** to English under any pretext.
4. **Hardcoding model names**. Always read from `settings.llm_model`.
5. **Using `print()`** for logging. Use the logger.
6. **Catching `Exception`** broadly at node level.
7. **Committing `.env`** or any file containing real API keys.
8. **Adding a dependency** without `uv add` and updating `pyproject.toml`.
9. **Modifying `WriterState`** without updating every node that reads it.
10. **Bypassing the graph** to call LLM directly from a route. Routes call `graph.ainvoke()` or `graph.astream()` — nothing else.
11. **Using `requests`, `aiohttp`, or `urllib`**. Only `httpx`.
12. **Silent fallbacks** — if `LLM_PROVIDER=anthropic` and the call fails, raise. Do not auto-switch to OpenAI.

---

## 9. When the User Requests a New Node

Execute this checklist in order. Stop and ask if any step is ambiguous.

1. **Update `state.py`** if new fields are needed. Show the diff before applying.
2. **Add prompt template** to `templates.py` following the structure in §6.
3. **Create `nodes/{name}.py`** with single async function `{name}_node`.
4. **Wire into `builder.py`** with proper edges. Show the updated graph topology.
5. **Add unit test** in `tests/unit/graph/nodes/test_{name}.py` mocking the LLM.
6. **Add integration test** in `tests/integration/test_pipeline.py` (only if the node changes E2E flow).
7. **Update `SKILLS.md`**: change status from 🔴 to 🟡 (in progress) or 🟢 (done).
8. **Update `CONTEXT.md`** drift log if scope changed.

---

## 10. When the User Requests a Backend Endpoint

1. Add Pydantic request/response schemas in `app/models/schemas.py`.
2. Add route handler in `app/api/routes.py` — keep handler under 20 lines, delegate to graph.
3. Streaming endpoints return `EventSourceResponse` from `sse-starlette`.
4. Add OpenAPI docstring + example payload.
5. Add integration test hitting the actual endpoint with TestClient.

---

## 11. Testing Rules

- Every node has at least one unit test with LLM mocked.
- Use `pytest.fixture` for shared setup. Fixtures live in `tests/conftest.py`.
- Mock LLMs with `from langchain_core.language_models.fake import FakeListChatModel`.
- Async tests use `@pytest.mark.asyncio`.
- Eval tests (in `tests/evals/`) are NOT run in CI by default — they hit real LLMs. Run manually.

---

## 12. Git & Commit Conventions

- Branch naming: `feat/skill-id-description`, `fix/short-desc`, `refactor/scope`, e.g., `feat/s03-research-node`.
- Commit messages follow Conventional Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:`.
- One logical change per commit.
- Never commit `.env`, `__pycache__/`, `.venv/`, `checkpoints.sqlite`, or any file >1MB.

---

## 13. When in Doubt — Ask Before Doing

Stop and ask the user before:

- Adding any new top-level folder.
- Introducing a new external service (Redis, Celery, etc.).
- Changing `WriterState` schema (cascades to every node).
- Deviating from any "Never" rule in §8.
- Choosing between two architecturally different approaches.
- Touching `main.py`, `config.py`, or `pyproject.toml`.
- Writing more than ~150 lines of code in one go without a checkpoint.

**State assumptions explicitly when you make them.** Never silently pick between valid options.

---

## 14. Response Format Rules (for Codex GPT outputs)

When generating code in this project:
- Always output full file contents when creating a new file, never diffs alone.
- When editing an existing file, show the surrounding context (5 lines before/after).
- Lead with a one-line summary of what changed.
- End with a checklist of files touched.
- If you produced code that violates any rule above, flag it and explain the trade-off.

---

*Last updated: Week 0. Update this file when the team agrees on a new convention — not when a single situation calls for an exception.*
