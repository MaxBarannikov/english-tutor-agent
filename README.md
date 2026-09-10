# AI English Tutor Agent

A conversational English tutor that lets you practise speaking naturally with an
AI and get structured, CEFR-aware feedback on your mistakes — without letting the
corrections take over the conversation.

## Key features

* 🎙️ **Local speech-to-text** — faster-whisper runs on your machine; no API key,
  no per-minute cost, audio never leaves the laptop
* 🤖 **Conversational agent built with LangGraph** — stateful graph, checkpointed
  to SQLite, survives a process restart
* 🔌 **LangChain as the model layer** — one interface over Claude, OpenAI and
  local Ollama models, with `with_structured_output` returning validated Pydantic
  corrections; switching provider is one env var
* 🧠 **Parallel response & analysis** — `responder` and `analyzer` are two
  independent LLM calls issued concurrently; neither is asked to do the other's job
* 📊 **CEFR-aware corrections** — the learner's level is passed into both prompts
  and changes behaviour: at A2 the analyzer ignores subtle article and aspect
  errors, at C1 it flags them
* 🎯 **Severity gating** — every `critical` correction is shown, at most one
  `minor` per turn, `style` findings are held for the end-of-session report
* 💬 **Chainlit UI** with an end-of-session summary (`/summary`)
* 💰 **Cost visibility** — tokens and USD are reported under every reply
* 📈 **Eval suite** with precision / recall / F0.5 per error type and a separate
  false-positive rate on correct sentences

## Architecture

```mermaid
flowchart TD
    mic["🎙 Speech"] --> asr["faster-whisper<br/>local, on-device"]
    asr --> draft["Draft<br/>Send · Discard"]
    draft --> start(["START"])
    typed["⌨️ Typed message"] --> start

    start -->|route_turn| responder["responder<br/>temp 0.7 · CEFR-aware<br/>never mentions mistakes"]
    start -->|route_turn| analyzer["analyzer<br/>temp 0 · structured output<br/>only the last message"]
    start -.->|end_session| summary["summary<br/>session report"]

    responder -->|reply| merge["merge<br/>severity gating in Python"]
    analyzer -->|corrections| merge
    merge --> out["Reply, then corrections"]
    summary --> report["End-of-session report"]

    classDef io fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b
    classDef node fill:#ecfdf5,stroke:#059669,color:#064e3b
    classDef sink fill:#fff7ed,stroke:#ea580c,color:#7c2d12
    class mic,typed,asr,draft io
    class responder,analyzer,merge,summary node
    class out,report sink
```

Two concerns run in parallel and never mix:

- **responder** plays a conversation partner. It never mentions mistakes, keeps
  vocabulary at the learner's CEFR level, and always ends with a question so the
  dialogue does not stall.
- **analyzer** looks only at the learner's last message and returns structured
  corrections via `with_structured_output`. It never produces conversational text.

They write to different state keys (`reply` and `corrections`), which is what
makes running them concurrently safe — a shared key without a reducer would raise
`InvalidUpdateError`.

The **merge** node assembles the turn — reply first, corrections second — and
applies severity gating. Over-correcting makes learners stop talking, so *how
much* is surfaced is decided in Python, not asked of the model.

**summary** is a separate branch taken when the session ends, so no reply is
generated for a turn the learner never wrote.

## Tech stack

Every dependency earns its place; nothing is here to pad the list.

| | | |
|---|---|---|
| **Orchestration** | LangGraph | Typed state, a conditional entry point, two concurrent branches, and a SQLite checkpointer that survives a restart |
| **Model layer** | langchain-core | One `BaseChatModel` interface over three providers, `with_structured_output`, message types, per-turn usage callbacks |
| **Providers** | langchain-anthropic · langchain-openai · langchain-ollama | Claude, GPT or a local model — chosen by env var, wired in one file |
| **Speech** | faster-whisper | On-device transcription: no API key, no per-minute cost, audio never leaves the machine |
| **Data** | Pydantic v2 · pydantic-settings | Every structure crossing a boundary is a validated model; config comes from `.env`, never from code |
| **UI** | Chainlit | Chat surface with mic input, a CEFR level selector, and the cost line under each reply |
| **Language** | Python 3.12 | `type` aliases, `TypedDict` state, modern unions — no `typing.List`, no `Optional`, no `Any` |
| **Tooling** | uv · ruff · mypy strict · pytest | `make check` runs the lot; tests mock at the LLM boundary and never hit the network |

Deliberately absent: no vector store, no RAG, no agent framework on top of the
graph, and none of LangChain's legacy `LLMChain` / `initialize_agent` APIs. The
problem does not need them.

## Why this project?

It is a portfolio project, built to show what separates a production-oriented LLM
application from a chatbot wrapper: stateful graph workflows, parallel calls with
reducer-safe state, structured outputs everywhere, deterministic business logic
kept out of the prompt, measured quality, visible cost, and local speech
processing.

## Quickstart

```bash
uv sync
cp .env.example .env      # set TUTOR_PROVIDER and the provider's API key
uv run chainlit run app.py
```

Set your CEFR level in the chat settings — it changes both prompts, not just the
wording. Send `/summary` to end the session and get the report.

## Voice input

Press the mic and speak. The recording is transcribed locally with faster-whisper
and comes back as a draft:

> 🎙 **Draft:** I go to Rome yesterday with my sister
> _Send it, or type a corrected version._ **[Send] [Discard]**

Nothing reaches the tutor until you press **Send**, so you always see what was
recognised first. Typing anything discards the draft, so a stale one can never be
sent by surprise.

Chainlit offers no way to prefill the message composer, which is why the draft is
a message with buttons rather than editable text in the input box. To edit, type
the corrected version — the draft steps aside.

```bash
make asr    # download the model up front; otherwise the first dictation waits for it
```

`TUTOR_ASR_MODEL` picks the size (`tiny` … `large-v3-turbo`, default `small`).
On an M-series Mac, `small` transcribes ~4 s of speech in under two seconds and
is accurate enough for learner speech; move up to `medium` if your learners have
strong accents and you can spend the latency. `TUTOR_ASR_SAMPLE_RATE` must match
`sample_rate` under `[features.audio]` in `.chainlit/config.toml`.

## Providers

The provider is chosen with `TUTOR_PROVIDER`; `src/tutor/llm.py` is the only file
that knows about any of them.

| `TUTOR_PROVIDER` | Models | Credentials |
|---|---|---|
| `anthropic` | `claude-opus-5`, `claude-sonnet-5`, … | `ANTHROPIC_API_KEY` |
| `openai` | `gpt-…` | `OPENAI_API_KEY` |
| `ollama` | anything local, e.g. `qwen2.5:14b` | none; set `TUTOR_OLLAMA_BASE_URL` |

Models are configured per role (`TUTOR_RESPONDER_MODEL`, `TUTOR_ANALYZER_MODEL`,
`TUTOR_SUMMARY_MODEL`), so the analyzer — where precision matters — can run on a
stronger model than the conversation partner.

Temperature is set explicitly per role (analyzer `0`, responder `0.7`, summary
`0.2`). Claude models from Opus 4.7 onward reject sampling parameters outright,
so for those the factory omits temperature and the analyzer's determinism comes
from the prompt instead.

### Local models

Small local models are noticeably weaker at structured output and at *not*
inventing errors — see the eval numbers below. Nothing in the code papers over
this: a schema violation propagates instead of being swallowed. A 14B-class
instruct model is the practical floor for the analyzer.

## Cost

Token counts and cost are collected per turn by a callback and printed under
every reply. Anthropic list prices ship in `src/tutor/pricing.py`; for any other
model, supply prices via `TUTOR_PRICING_FILE` (JSON:
`{"<model id>": {"input": <usd per 1M>, "output": <usd per 1M>}}`). An unpriced
model logs a warning and still reports tokens — cost is never silently absent.

## Persistence

State is checkpointed to SQLite (`TUTOR_CHECKPOINT_DB`), keyed by the Chainlit
thread id. Verified across a process restart: the conversation, the CEFR level
and the accumulated corrections are restored from the checkpointer, not from
process memory.

## Evals

`evals/dataset.jsonl` holds labelled sentences per CEFR level, including
**correct sentences as negative controls**. False positives are the failure mode
that matters — an analyzer that invents errors in a correct sentence is worse
than one that misses some — so the false-positive rate is reported separately
from the per-type scores.

```bash
uv run python evals/run.py --concurrency 4
```

Latest run — `qwen2.5:7b-instruct-q4_K_M` via Ollama, 2026-08-30, 35 examples,
cost: n/a (local model):

| error type | P | R | F0.5 | support |
|---|---|---|---|---|
| agreement | 0.50 | 0.50 | 0.50 | 2 |
| article | 0.00 | 0.00 | 0.00 | 1 |
| aspect | 0.00 | 0.00 | 0.00 | 0 |
| grammar | 0.33 | 0.17 | 0.28 | 6 |
| preposition | 1.00 | 0.75 | 0.94 | 4 |
| tense | 0.50 | 0.71 | 0.53 | 7 |
| vocabulary | 0.00 | 0.00 | 0.00 | 1 |
| word_order | 0.50 | 1.00 | 0.56 | 1 |

False-positive rate on correct sentences: **0.08**.

This is a small local model on a small dataset — it is a baseline, not a
capability claim. Re-run and replace this table whenever the analyzer prompt or
model changes; stale numbers are worse than none.

## Commands

`make help` lists them; each target is a one-line `uv run …` if you prefer to run
it directly.

| | |
|---|---|
| `make install` | `uv sync` |
| `make run` | Chainlit UI |
| `make test` | test suite, no network |
| `make lint` / `make format` | ruff |
| `make typecheck` | mypy strict |
| `make check` | lint + types + tests + format check — what CI runs |
| `make evals` | eval suite (calls a real model) |
| `make asr` | pre-download the speech-recognition model |
| `make clean` | caches and the local session database |

## Layout

```
src/tutor/
    graph.py       StateGraph assembly, routing, checkpointer
    state.py       TutorState and its reducers
    models.py      Pydantic models — Correction, SessionSummary, …
    severity.py    the gating rule
    llm.py         model factory, the only provider-aware module
    pricing.py     token prices
    callbacks.py   per-turn usage accounting
    config.py      pydantic-settings
    asr.py         local speech recognition for voice input
    prompts/       responder.md, analyzer.md, summary.md — loaded at import
    nodes/         responder, analyzer, merge, summary
```
