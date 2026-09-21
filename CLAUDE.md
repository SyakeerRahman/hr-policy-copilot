# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

HR Policy Copilot: an Agentic RAG app (LangGraph + FastAPI + Pinecone + OpenAI + Tavily) for a fictional retailer, NovaRetail. It answers from a private HR knowledge base first and falls back to web search only when private evidence is weak. The UI is plain HTML/CSS/JS served by FastAPI.

## Commands

```powershell
pip install -r requirements.txt      # VS Code is set to use a conda env
python ingest_sample_kb.py           # embed data/sample_kb/* into Pinecone (run once per fresh index)
python run.py                        # dev server with reload, http://127.0.0.1:8080 (docs at /docs)
docker build -t hr-policy-copilot .
docker run -p 8080:8080 --env-file .env hr-policy-copilot
```

- No test suite, linter, or formatter is configured. `test.py` is a scratch script that only chunks the handbook and prints the chunk count (no API keys needed).
- Config comes from `.env` at the repo root (see `app/core/config.py`). The README mentions `.env.example`, but that file does not exist. Required keys: `OPENAI_API_KEY`, `TAVILY_API_KEY`, `PINECONE_API_KEY`. `ADMIN_API_KEY` guards uploads.

## Architecture

Request flow: `static/js/app.js` -> `POST /api/chat` ([app/api/routes.py](app/api/routes.py)) -> `ask()` in [app/rag/workflow.py](app/rag/workflow.py) -> audit row written to SQLite.

**The LangGraph graph** (`build_graph()` in `workflow.py`) is the core. State shape is `AgentState` in [app/rag/state.py](app/rag/state.py).

```
route_question -> direct_answer                                  (greetings / chat)
               -> retrieve_kb -> grade_kb -> generate_from_kb     (KB good)
                                          -> search_web -> grade_web -> generate_from_web (web good)
                                                                     -> rewrite_query -> retrieve_kb (retry)
                                                                     -> insufficient     (retries exhausted)
```

- Routing and grading use `llm().with_structured_output(<Pydantic model>, method="json_mode")`. JSON mode needs the literal word "JSON" plus an example in each prompt. Keep it there when editing prompts.
- Every node returns `trace: add_trace(state, "...")`. The trace goes back to the UI and into the audit table, so new nodes must append to it.
- Grading checks `state['question']` (the original). Retrieval and web search use `state['current_query']` (the rewritten one).
- `max_retries` (default 1) caps the rewrite loop. `top_k` (default 4) sets the retrieval count.
- `source_used` changes meaning while the graph runs: the router sets `kb`/`direct`, and the final nodes set `private_kb`, `web_search`, `direct`, or `insufficient_evidence`. The frontend reads the final value.

**Import-time side effects:** `app/main.py` runs `init_db()`, and `workflow.py` compiles the graph when imported. The LLM, Tavily client, embeddings, and Pinecone store are lazy singletons, so the app starts without keys and fails on the first `/api/chat` call instead.

**Vector store** ([app/rag/vectorstore.py](app/rag/vectorstore.py)): `ensure_index()` creates the serverless index (aws/us-east-1, cosine) if missing. **It deletes and recreates the index if the embedding dimension does not match `EMBEDDING_MODEL`.** Changing the embedding model therefore wipes all ingested vectors. Add new models to `EMBEDDING_DIMENSIONS`.

**Ingestion** ([app/services/ingestion.py](app/services/ingestion.py)): `.pdf`, `.txt`, `.md`, `.docx`, chunked at 900 chars with 120 overlap. `POST /api/ingest` needs the `X-Admin-Key` header, saves the file to `uploads/`, and upserts the chunks. There is no dedup, so re-ingesting a file creates duplicate vectors.

**Audit** ([app/services/audit.py](app/services/audit.py)): SQLite at `data/audit.db` (committed to git), table `query_audit`. No endpoint reads it.

## Other folders

- `Agentic-RAG-demo/`: the original notebook prototype, with its own `requirements.txt`. The app does not use it.
- `steps.md`: the build order used to create the project. `create_project.py`: the scaffold script.
- `docs/`: problem statement PDF and architecture diagram.
