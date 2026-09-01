# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Chapter 09 companion notebook for a Manning "AI Agents" book: RAG **query transformation**
techniques (rewrite-retrieve-read, then multi-query generation) over Wikivoyage UK travel
pages, using LangChain 1.0 + Chroma + OpenAI.

Effectively a single deliverable: `09-question_transformations.ipynb`. `src/ch09/` is an
empty `uv init` stub — no library code lives there yet. `Untitled.ipynb` is a scratch pad.

## Running

Secrets come from `pass` via `secretspec` (see `secretspec.toml`), injected at launch:

```bash
secretspec run -P development --reason "ch09 notebook" -- uv run jupyter lab
```

`--reason` is mandatory (secretspec ≥0.19 `require_reason` policy); without it the command
fails with "Accessing secrets requires a reason", which looks like a broken config.

Two traps that cost time:

- `secretspec check` prints `✓ OPENAI_API_KEY` when the `pass` entry **exists but is empty**.
  Verify content, not existence: `pass show secretspec/ch09/development/OPENAI_API_KEY | head -1 | awk '{print length($0)}'`
  (valid key ≈164 chars, `sk-proj` prefix). Notebook symptom: `"OPENAI_API_KEY" in os.environ`
  is `True` but `len(...)` is `0`.
- Env is injected **once at lab launch**; kernels inherit from the lab process. After changing
  a secret, relaunch `jupyter lab` — a kernel restart keeps the stale value.

No test suite, no build. Lint/format: `uv run ruff check .` / `uv run ruff format .`.
In JupyterLab, Ctrl+S already runs ruff on the whole notebook (jupyterlab-code-formatter,
`formatOnSave`).

## Notebook architecture

Cells build one pipeline in order; running out of order breaks it.

1. **Ingestion** (cells 0–2): 21 Wikivoyage destination pages → `AsyncHtmlLoader` →
   `HTMLSectionSplitter` on `h1`/`h2` → keep only chunks that carry a `Header 2` key
   (the "granular" chunks) → `Chroma` collection `uk_granular`, `OpenAIEmbeddings`.
   The collection is **in-memory** (no `persist_directory`) and cell 1 calls
   `reset_collection()`, so re-running the ingestion re-embeds everything and costs
   OpenAI calls. Ingestion is the slow/expensive part — avoid restarting the kernel casually.
2. **Rewriter chain** (9.1.2): `rewriter_prompt | llm | StrOutputParser()`, turns a natural
   question into a vector-store query string.
3. **Rewrite-retrieve-read chain** (9.1.4): the rewriter feeds `context` while the raw
   question feeds `question`, both via `RunnablePassthrough`, into `rag_prompt | llm`.
4. **Multi-query** (9.2, in progress): `MultiQueryRetriever` with a pydantic output parser.

`USER_AGENT` is set before the LangChain imports on purpose — Wikimedia returns a
robot-policy stub when the UA does not name a contact, so that line must stay first.

## Dependency notes (LangChain 1.0 layout)

Versions are pinned in `pyproject.toml`; keep them pinned. Imports moved in 1.0:

- `MultiQueryRetriever` → `langchain_classic.retrievers.multi_query` (not `langchain.retrievers`)
- `HTMLSectionSplitter` → `langchain_text_splitters` (transitive dep, not listed in `pyproject.toml`)
- Document loaders → `langchain_community`, Chroma → `langchain_chroma`

Model in use: `gpt-5-nano` via `ChatOpenAI`.

## Notebook hygiene

`nbdime` and `jupytext` are installed. Outputs are committed (see git history), so diffs are
noisy — use `nbdiff` rather than `git diff` when reviewing notebook changes, and keep
output-only churn in separate `docs:` commits, as the existing history does.
