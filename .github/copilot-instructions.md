# Copilot Instructions — TMCP Agent Project Management

## Quick summary
- Full-stack AI-driven project management: FastAPI backend + LangGraph agent workflows; React (Vite) frontend.  
- Primary goal for agents: classify an incoming request, generate a spec (`ai_spec_doc`), produce an `antigravity_prompt` and a `subtask_list` that the UI persists and displays.

## Where to look (high signal files)
- Backend entry & orchestration: `backend/main.py` (startup, `run_agent_workflow`) 🔧
- Data model & statuses: `backend/models.py` (Task/TaskStatus/Subtask/TaskLog) 📦
- Agent graph and routing: `backend/agents/graph.py` (entry = `router`, then `context_retriever` for ALL types) 🧭
- Node implementations & LLM config: `backend/agents/nodes.py` (router, pm_agent, tech_lead, fast_track_agent) 🧠
- MCP filesystem integration: `backend/agents/mcp_tools.py` (MCP server wrapper for LLM tool-calling) 🔌
- Local code-reading tools: `backend/agents/tools.py` (`read_local_source_code`, `get_project_overview`, `get_directory_tree`) 🔍
- Frontend API client: `frontend/services/api.ts` (API_BASE_URL = `127.0.0.1:8000`) 🌐
- Frontend types/UI: `frontend/types.ts`, `frontend/components/TaskModal.tsx` ✨

## How the agent workflow works (concise)
1. Client creates a Task (POST `/tasks/`).  
2. Mark Task status → `Agent_Processing` (PATCH `/tasks/{id}/status?status=Agent_Processing`) to start `run_agent_workflow`.  
3. Graph flow — ALL task types receive project context:  
   `router` → `context_retriever` → conditional branch:
   - Feature/Bug → `pm_agent` → `tech_lead` → END
   - Chore/Research → `fast_track_agent` → END
4. `context_retriever` runs for **every** task type — reads project overview (directory tree + key files like README, package.json, pyproject.toml) and optionally pattern-matched source files based on the request.
5. Node outputs the keys main.py expects: `ai_spec_doc`, `antigravity_prompt`, `subtask_list` — these are written to `Task` and `Subtask` rows and appended to `TaskLog` entries. (See `run_agent_workflow` in `backend/main.py`.)
6. `fast_track_agent` now also returns `ai_spec_doc` (previously missing — Research/Chore tasks had "No spec generated yet" in the UI).

## LLMs, mocks, and deterministic testing
- Default LLM: `ChatOllama` configured via env vars in `backend/agents/nodes.py`:
  - `OLLAMA_MODEL` (default: `qwen3:4b-instruct-2507-q4_K_M`)
  - `OLLAMA_BASE_URL` (default: `http://localhost:11434`)
  - These can also be set in `backend/.env`.
- Deterministic / unit-test mode: set `MOCK_LLM=true` in env — nodes return stable mock outputs.  
- Some test helpers set `OPENAI_API_KEY` to a dummy value (see `backend/verify_setup.py`).

## MCP Filesystem Integration
- `backend/agents/mcp_tools.py` provides `mcp_filesystem_tools(allowed_directories)` — an async context manager that starts a `@modelcontextprotocol/server-filesystem` MCP server via `npx`.
- Yields LangChain-compatible tools: `read_file`, `read_multiple_files`, `list_directory`, `directory_tree`, `search_files`, `get_file_info`.
- Used by `fast_track_agent_node` (async) to let the LLM explore project files on demand for Research/Chore tasks.
- Falls back gracefully if `npx` is not found or MCP packages are not installed.
- Dependencies: `mcp[cli]`, `langchain-mcp-adapters` (in `backend/pyproject.toml`).
- Requires Node.js/npm for `npx` to work.

## Important tools / constraints agents should know
- `get_project_overview` (in `backend/agents/tools.py`): lightweight — reads directory tree (max 3 levels) + key project config/doc files (README, package.json, pyproject.toml, Dockerfile, etc.). Used by `context_retriever_node` for initial context.
- `read_local_source_code` (in `backend/agents/tools.py`): heavier — reads file contents matching a glob pattern or file list; ignores `.git`, `node_modules`, `.venv`, binary extensions; caps at ~50 files and ~100k chars per file. Used for targeted deep reads.
- `get_directory_tree` (in `backend/agents/tools.py`): just the directory tree, no file contents.
- LangGraph checkpointing: uses `AsyncSqliteSaver` and persists to `checkpoints.db` (project root). `main.py` contains a workaround (patching `is_alive`) for a known langgraph sqlite bug.

## Shared helpers in nodes.py
- `_extract_json(content)`: Robustly extracts JSON from LLM responses (handles ```json blocks, preamble text, etc.).
- `_build_context_block(state)`: Assembles context from `workspace_global_prompt`, `project_context`, `tech_stack`, and `retrieved_code_context` into a formatted string for prompts. Truncates code context at 15k chars.

## AgentState fields (graph.py)
```python
task_id, raw_input, project_context, tech_stack, repo_path,
workspace_global_prompt,  # NEW — from Workspace.global_system_prompt
task_type, retrieved_code_context, ai_spec_doc, antigravity_prompt, subtask_list
```

## Conventions & quick references
- Status flow: Inbox → Agent_Processing → Ready_for_AI → Reviewing → Done. (See `TaskStatus` in `models.py`.)
- Node output contract (expected keys): `ai_spec_doc` (markdown string), `antigravity_prompt` (string), `subtask_list` (list of {title, target_files, step_order}).
- `fast_track_agent_node` is **async** (uses MCP tools); all other nodes are sync.
- All node prompts include `_build_context_block(state)` for project-aware generation.
- Prompts instruct the LLM to write in the same language as the user's request.
- To trigger a workflow manually (example):
  - POST `/tasks/` then PATCH `/tasks/{id}/status?status=Agent_Processing`
- Frontend assumes API at `http://127.0.0.1:8000` (change `frontend/services/api.ts` if different).

## How to extend the agent graph (example)
1. Implement a node fn in `backend/agents/nodes.py` (return dict with agreed keys). Use `_build_context_block(state)` for context and `_extract_json()` for parsing LLM JSON output.
2. Add the node in `backend/agents/graph.py` with `workflow.add_node("your_node", your_node_fn)`.  
3. Wire edges (`add_edge` / `add_conditional_edges`) and update `route_task` if classification changes.  
4. Add integration/unit tests under `backend/tests/` and update `frontend/types.ts` if the model changes.

## Dev / run / test cheat-sheet (macOS)
- Backend dev:  python -m venv .venv && .venv/bin/pip install -e backend[dev]  
- Start backend (recommended — run from repo root):  `.venv/bin/uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000`  
  - If your current directory is `backend/`, use: `.venv/bin/uvicorn main:app --reload --host 127.0.0.1 --port 8000`  
  - Or from repo root with explicit app-dir: `.venv/bin/uvicorn main:app --app-dir backend --reload --host 127.0.0.1 --port 8000`  
  - Quick workaround (not recommended long-term): `PYTHONPATH=.. .venv/bin/uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000`  
  
  Note: `ModuleNotFoundError: No module named 'backend'` appears when Python's import path doesn't include the repository root — use one of the commands above depending on your working directory.
- Frontend dev:  cd frontend && npm install && npm run dev  
- Backend tests:  `MOCK_LLM=true .venv/bin/python -m pytest backend/tests/ -v`
- Frontend E2E:  npm run test:e2e --prefix frontend  
- Fast deterministic tests: export MOCK_LLM=true
- Change LLM model: set `OLLAMA_MODEL=model_name` in `backend/.env` or env var

## Known quirks / pitfalls ⚠️
- There is no `requirements.txt` at repo root — use `backend/pyproject.toml` or `pip install -e backend[dev]`.  
- `checkpoints.db` holds LangGraph state; `main.py` patches `is_alive` due to a langgraph sqlite edge-case.  
- When changing persisted models, update both `backend/models.py` and `frontend/types.ts` + API client.
- `fast_track_agent_node` is async — if you add nodes after it in the graph, account for async execution.
- MCP filesystem tools require Node.js/npx installed on the host machine.
- The first MCP tool invocation may be slower due to npx downloading `@modelcontextprotocol/server-filesystem`.

## Where to add tests
- Unit & API tests: `backend/tests/` (use TestClient; background tasks are exercised in tests).  
- Agent node tests: `backend/tests/test_agent_nodes.py` (26 tests covering helpers, tools, mock nodes, graph routing).
- E2E / UI: `frontend/tests/` (Playwright).

---
If anything above is unclear or you want example PR templates/tests for adding a node or changing models, tell me which section to expand. ✅
