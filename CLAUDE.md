# CLAUDE.md

Guidance for AI assistants working in this repository.

## Project

**English AI Tutor** — a conversational agent that holds a natural dialogue
in English while correcting the learner's mistakes.

Two concerns run in parallel and must stay separated:

- **responder** — plays a conversation partner. Never mentions mistakes.
  Keeps vocabulary at the learner's CEFR level, always ends with a question
  so the dialogue does not stall.
- **analyzer** — inspects only the learner's last message and returns
  structured corrections. Never produces conversational text.

A merge node assembles the final output: reply first, corrections second.

This is a portfolio project. Code quality, readability and reproducibility
matter as much as functionality. Assume a senior engineer will read every file.

## Stack

- Python 3.12+
- LangGraph — orchestration, state, persistence
- langchain-core — model interface, `with_structured_output`
- Pydantic v2 — all data models
- uv — dependency and env management
- ruff — lint + format
- mypy (strict) — type checking
- pytest + pytest-asyncio — tests
- Chainlit — UI

Do not add a dependency without asking. Prefer the standard library.
Do not pull in the full `langchain` package; `langchain-core` plus the
provider package is enough.

## Layout

```
src/tutor/
    graph.py          # StateGraph assembly, compile, entry point
    state.py          # TutorState, reducers
    models.py         # Pydantic models (Correction, SessionSummary, ...)
    nodes/
        responder.py
        analyzer.py
        merge.py
        summary.py
    prompts/          # prompts as .md files, loaded at import
    llm.py            # model factory, single place where providers are wired
    config.py         # pydantic-settings, env vars
tests/
evals/
    dataset.jsonl     # labelled sentences
    run.py            # scoring script
```

One node per file. A node file exports exactly one function.

## Commands

```bash
uv sync                    # install
uv run chainlit run app.py # run UI
uv run pytest              # tests
uv run ruff check --fix .  # lint
uv run ruff format .       # format
uv run mypy src            # types
uv run python evals/run.py # eval suite
```

## Non-negotiable rules

1. **Structured output only.** Every node that extracts data uses
   `llm.with_structured_output(Model)`. Never parse model output with
   regex, `str.split`, or `json.loads` on raw text.
2. **Prompts live in `prompts/*.md`**, not inline in Python. They are
   content, they get iterated on, and they must be diffable.
3. **No secrets in code.** All configuration through `pydantic-settings`
   and `.env`. `.env.example` stays in sync.
4. **Type everything.** mypy strict must pass. No bare `dict`, no `Any`
   unless justified with a comment.
5. **No silent failures.** If the model returns something unusable, raise
   or log — never swallow and return a default.
6. **Tests must not call a real model.** Mock at the LLM boundary.

## Python standards

- Modern typing: `list[str]`, `str | None`, `Literal`, `TypedDict`.
  No `typing.List`, no `Optional`.
- Pydantic v2 for all data crossing a boundary. `Field(description=...)`
  on every field that goes into a structured-output schema — the model
  reads those descriptions, they are part of the prompt.
- `async def` for anything doing I/O. Use `ainvoke`, not `invoke`.
- Small pure functions where possible; keep LLM calls at the edges so the
  rest stays testable.
- Docstrings only where the *why* is non-obvious. Do not document what the
  signature already says.
- Comments explain reasoning, not mechanics. Delete commented-out code.
- No defensive `try/except Exception` wrappers around whole functions.
  Catch specific exceptions at specific call sites.

## Writing style for code

Aim for the shortest version that a reader understands on first pass.
Prefer boring, explicit code over clever code. If a function needs a
comment to explain its control flow, restructure it instead.

Do not add abstraction layers, base classes, or plugin systems for
hypothetical future needs. This project has one graph and four nodes.

## LLM engineering standards

- **Separation of concerns per call.** One call, one job. Never ask a
  single prompt to both converse and analyse.
- **Level-aware prompting.** The learner's CEFR level is passed into both
  prompts and changes behaviour: at A2 the analyzer ignores subtle article
  and aspect errors, at C1 it flags them.
- **Severity gating.** `Correction.severity` is `critical | minor | style`.
  `critical` always shown, at most one `minor` per turn, `style` only in
  the end-of-session summary. Over-correcting makes learners stop talking —
  this is a product requirement, not a nice-to-have.
- **Deterministic before probabilistic.** If a check can be done in Python
  (normalisation, exact match, length, level lookup), do it in Python.
- **Cost visibility.** Token usage and cost are tracked per turn via a
  callback and surfaced in the UI. Never ship a version where cost is
  invisible.
- **False positives are the main failure mode.** An analyzer that invents
  errors in correct sentences is worse than one that misses some. The eval
  set includes correct sentences specifically to measure this.
- Temperature: 0 for the analyzer, higher for the responder. Set it
  explicitly, never rely on defaults.

## LangGraph specifics

- State is a `TypedDict` in `state.py`. Message history uses
  `Annotated[list[AnyMessage], add_messages]`.
- **Parallel branches must not write to the same state key** unless that
  key has a reducer. `analyzer` and `responder` run concurrently — they
  write to `corrections` and `reply` respectively. Adding a shared key
  without a reducer raises `InvalidUpdateError`.
- Nodes return partial state updates (a dict with only changed keys),
  never the whole state.
- Persistence via a checkpointer; `thread_id` comes from the session.
  Verify the graph survives a process restart before calling a feature done.
- Keep routing logic in named functions, not lambdas, so it is testable
  and shows up in tracebacks.
- Update the mermaid diagram in `README.md` after any topology change.

## Testing

- Unit tests for node logic with a stubbed LLM returning fixed Pydantic
  objects. Fast, deterministic, no network.
- One integration test that runs the compiled graph end to end with fakes,
  asserting state transitions and that parallel writes merge correctly.
- Test severity gating explicitly — it is business logic, not prompting.
- No snapshot tests on model output. Model output is not stable.

## Evals

`evals/dataset.jsonl` holds labelled examples: sentences with known errors
plus correct sentences as negative controls.

`evals/run.py` reports, per error type:
precision, recall, F0.5, and false-positive rate on correct sentences.

Results go into the README as a table with the model name, date, and cost
per run. Update them when prompts change — stale numbers are worse than none.

## Git

- No type prefixes. A subject line is a short imperative phrase describing
  the change: `add faster-whisper for local transcription`, not
  `feat: add faster-whisper ...`.
- One logical change per commit. Do not mix a refactor with a feature.
- Commit messages describe intent, not the diff.
- Never commit `.env`, session databases, or eval output artefacts.

## Do not

- Do not use the deprecated `LLMChain`, `ConversationChain`, or
  `initialize_agent` APIs.
- Do not add memory/vector-store machinery — this project has no RAG.
- Do not print to stdout for logging; use the configured logger.
- Do not create README sections, badges, or scaffolding files that were
  not asked for.
- Do not leave TODO comments. Open an issue or implement it.

## Definition of done

A change is done when: mypy strict passes, ruff is clean, tests pass,
the graph diagram matches the code, and the README reflects any new
behaviour or metric.