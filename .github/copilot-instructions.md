# Copilot Instructions — TMCP Agent Project Management (v2.0 Refactored)

## Quick Summary
- **Status**: ✅ All 34 backend tests passing + ✅ 3/3 frontend E2E tests passing
- **Architecture**: Refactored to Domain-Driven Design (backend) + Feature-Slice Architecture (frontend)
- **Stack**: FastAPI + LangGraph + SQLModel (backend) | React 19 + Vite + Playwright (frontend)
- **Agent Workflows**: LangGraph with AsyncSqliteSaver checkpointing; router → context_retriever → branching nodes
- **Primary Goal**: Classify incoming request → generate `ai_spec_doc`, `antigravity_prompt`, `subtask_list` for persistence & UI display

## Where to Look (High-Signal Files)

**Backend (Post-Refactor)**
- Entry & Startup: [backend/app/main.py](backend/app/main.py) — FastAPI app, lifespan handlers, CORS setup
- Agent Orchestration: [backend/app/services/agent_service.py](backend/app/services/agent_service.py) — `AgentService.run_workflow()` (async)
- Data Models: [backend/app/models/__init__.py](backend/app/models/__init__.py) — Task, Project, Workspace, Subtask, TaskLog
- Agent Graph: [backend/app/agents/graph.py](backend/app/agents/graph.py) — `build_graph()`, `AgentState`, `route_task()`
- Agent Nodes: [backend/app/agents/nodes.py](backend/app/agents/nodes.py) — router, context_retriever, pm_agent, tech_lead, fast_track_agent
- Agent Tools: [backend/app/agents/tools.py](backend/app/agents/tools.py) — `get_project_overview()`, `read_local_source_code()`
- MCP Integration: [backend/app/agents/mcp_tools.py](backend/app/agents/mcp_tools.py) — MCP filesystem tools wrapper
- API Endpoints: [backend/app/api/v1/endpoints/](backend/app/api/v1/endpoints/) — workspaces.py, projects.py, tasks.py, filesystem.py

**Frontend (Feature-Slice)**
- API Client: [frontend/src/services/api.ts](frontend/src/services/api.ts) — All fetch calls (API_BASE_URL: `http://127.0.0.1:8000`)
- Router: [frontend/src/app/router.tsx](frontend/src/app/router.tsx) — React Router v7 routes
- Main Layout: [frontend/src/app/App.tsx](frontend/src/app/App.tsx) — App wrapper
- Task Modal: [frontend/src/components/TaskModal.tsx](frontend/src/components/TaskModal.tsx) — Task details, live logs, results display
- Dashboard: [frontend/src/features/dashboard/Dashboard.tsx](frontend/src/features/dashboard/Dashboard.tsx) — Workspaces & projects overview
- Kanban: [frontend/src/features/projects/ProjectKanban.tsx](frontend/src/features/projects/ProjectKanban.tsx) — Task board with drag-drop
- Types: [frontend/src/types/index.ts](frontend/src/types/index.ts) — TypeScript interfaces
- E2E Tests: [frontend/tests/](frontend/tests/) — Playwright test suites (3/3 ✅)
- Playwright Config: [frontend/playwright.config.ts](frontend/playwright.config.ts) — `baseURL: http://localhost:3001`

## Architecture (Post-Refactor v2.0)

### Backend Directory Structure (Domain-Driven Design)
```
backend/
├── app/
│   ├── main.py                     # FastAPI app, startup/shutdown lifespan
│   ├── api/
│   │   ├── deps.py                 # Dependency injection (get_session)
│   │   └── v1/
│   │       ├── api.py              # Router aggregator
│   │       └── endpoints/
│   │           ├── workspaces.py   # Workspace CRUD + single GET ⭐
│   │           ├── projects.py     # Project CRUD + single GET ⭐
│   │           ├── tasks.py        # Task CRUD, status updates, agent launch
│   │           └── filesystem.py   # File operations
│   ├── core/
│   │   ├── config.py               # Settings with ConfigDict (Pydantic v2)
│   │   └── database.py             # SQLModel engine + create_db_and_tables()
│   ├── models/                     # SQLModel tables (Task, Project, Workspace, etc.)
│   ├── schemas/                    # Pydantic schemas (request/response)
│   ├── services/
│   │   └── agent_service.py        # AgentService (async), run_workflow()
│   └── agents/
│       ├── graph.py                # LangGraph StateGraph, build_graph(), route_task()
│       ├── nodes.py                # Node functions + helpers (_extract_json, _build_context_block)
│       ├── tools.py                # Local tools (get_project_overview, read_local_source_code)
│       └── mcp_tools.py            # MCP filesystem tools context manager
├── tests/
│   ├── conftest.py                 # Pytest fixtures (TestClient, Session)
│   ├── test_agent_nodes.py        # 26 node tests
│   ├── test_api_*.py              # 8 API endpoint tests
│   └── test_persistence.py         # LangGraph persistence test
└── pyproject.toml                  # Dependencies, [dev] extras
```

### Frontend Directory Structure (Feature-Slice)
```
frontend/
├── src/
│   ├── main.tsx                    # React 19 entry + React Router
│   ├── index.css                   # Tailwind + global styles (CDN-based)
│   ├── app/
│   │   ├── App.tsx                 # Main layout
│   │   └── router.tsx              # React Router v7.13 routes
│   ├── features/
│   │   ├── dashboard/              # Workspace/Project list & creation
│   │   ├── projects/               # ProjectKanban with drag-drop
│   │   └── manager/                # ManagerDashboard with stats
│   ├── components/                 # Shared UI (TaskModal, Icons, etc.)
│   ├── services/
│   │   └── api.ts                  # Fetch-based API client
│   ├── types/                      # TypeScript interfaces
│   └── lib/                        # Utilities
├── tests/
│   ├── agent-workflow.spec.ts     # ✅ PASS — Full workflow E2E
│   └── manager-flow.spec.ts       # ✅ PASS — Dashboard & modals
├── playwright.config.ts            # baseURL: http://localhost:3001 ⭐
└── package.json                    # Dependencies
```

## How the Agent Workflow Works (Concise)

1. **Create Task**: POST `/tasks/` → Task created with status `Inbox`
2. **Trigger Workflow**: PATCH `/tasks/{id}/status?status=Agent_Processing`
3. **Graph Execution** (in `agent_service.run_workflow()` async):
   - Graph nodes: `router` → `context_retriever` → conditional branching
   - **Feature/Bug**: `pm_agent` → `tech_lead` → END
   - **Chore/Research**: `fast_track_agent` → END
4. **Context Retrieval** (all task types):
   - `context_retriever_node` reads project overview (directory tree, README, package.json, pyproject.toml)
   - Optionally pattern-matches source files based on request
5. **LLM Generation**:
   - Nodes call LLM with context + instructions
   - Return: `ai_spec_doc` (markdown), `antigravity_prompt`, `subtask_list` (with step_order, target_files)
6. **Persist to Database**:
   - Update Task row with `ai_spec_doc`, `antigravity_prompt`
   - Create Subtask rows for each subtask
   - Create TaskLog entries for progress tracking
   - Status → `Ready_for_AI`
7. **Frontend Display**:
   - TaskModal shows live logs during Agent_Processing
   - Auto-displays antigravity_prompt & implementation plan on Ready_for_AI
   - Lists subtasks with step numbers

## API Endpoints (11 Total) ✅ UPDATED

### Workspaces
- `POST /workspaces/` — Create workspace
- `GET /workspaces/` — List all workspaces  
- `GET /workspaces/{workspace_id}` — Get single workspace ⭐ **NEW**

### Projects  
- `POST /projects/` — Create project
- `GET /projects/` — List all projects
- `GET /projects/{project_id}` — Get single project ⭐ **NEW**
- `PATCH /projects/{project_id}` — Update project

### Tasks
- `POST /tasks/` — Create task + trigger agent  
- `GET /tasks/` — List tasks
- `GET /tasks/{task_id}` — Get single task
- `PATCH /tasks/{task_id}/status` — Update status, launch background workflow
- `GET /tasks/{task_id}/logs` — Stream task logs (TaskLog entries)
- `GET /tasks/{task_id}/subtasks` — Get generated subtasks

### Filesystem
- `GET /fs/directories?path={path}` — List directories

## LLMs, Mocks & Deterministic Testing

- **Default LLM**: `ChatOllama` configured via env vars in [backend/app/agents/nodes.py](backend/app/agents/nodes.py):
  - `OLLAMA_MODEL` (default: `qwen3:4b-instruct-2507-q4_K_M`)
  - `OLLAMA_BASE_URL` (default: `http://localhost:11434`)
- **Test Mode**: `MOCK_LLM=true` env var → nodes return stable mock outputs
- **Set In**: `backend/.env` or via shell export

## MCP Filesystem Integration

- `backend/app/agents/mcp_tools.py` — async context manager that starts MCP server via `npx`
- **Yields**: `read_file`, `read_multiple_files`, `list_directory`, `directory_tree`, `search_files`, `get_file_info`
- **Used by**: `fast_track_agent_node` (async) for intelligent file exploration
- **Requirements**: Node.js/npm for `npx`, `mcp[cli]` + `langchain-mcp-adapters` in `backend/pyproject.toml`
- **Graceful fallback**: Continues if MCP not available

## Agent Tools & Helpers

| Tool | Location | Purpose |
|------|----------|---------|
| `get_project_overview()` | [backend/app/agents/tools.py](backend/app/agents/tools.py) | Directory tree + key config files (README, package.json, pyproject.toml) |
| `read_local_source_code()` | [backend/app/agents/tools.py](backend/app/agents/tools.py) | Read file contents by glob/list; ~50 files, ~100k chars per file max |
| `get_directory_tree()` | [backend/app/agents/tools.py](backend/app/agents/tools.py) | Directory structure only |
| `_extract_json(content)` | [backend/app/agents/nodes.py](backend/app/agents/nodes.py) | Parse JSON from LLM responses |
| `_build_context_block(state)` | [backend/app/agents/nodes.py](backend/app/agents/nodes.py) | Format context for prompts (15k char limit) |

## AgentState Definition

```python
# backend/app/agents/graph.py
@dataclass
class AgentState(TypedDict, total=False):
    task_id: str
    raw_input: str
    task_type: str
    project_context: dict
    tech_stack: str
    repo_path: str
    workspace_global_prompt: str
    retrieved_code_context: str
    ai_spec_doc: str
    antigravity_prompt: str
    subtask_list: list
```

## Key Conventions

| Convention | Details |
|-----------|---------|
| **Status Flow** | Inbox → Agent_Processing → Ready_for_AI → Reviewing → Done |
| **Node Output** | `ai_spec_doc` (markdown), `antigravity_prompt` (str), `subtask_list` (list of dicts with step_order/title/target_files) |
| **Async Nodes** | Only `fast_track_agent_node` is async; all others are sync |
| **Context Building** | All nodes use `_build_context_block(state)` for LLM prompts |
| **Language** | LLM prompts instruct matching user request language |
| **API Base** | Frontend config: `http://127.0.0.1:8000` in `frontend/src/services/api.ts` |
| **Checkpointing** | `AsyncSqliteSaver` → `checkpoints.db` in project root |

## Testing Coverage (34 Backend + 3 Frontend E2E)

### Backend Tests ✅ 34/34 PASSING
```bash
MOCK_LLM=true python -m pytest backend/tests/ -v
```

**test_agent_nodes.py** (26 tests):
- Helpers: `_extract_json()`, `_build_context_block()`
- Nodes: router, context_retriever, pm_agent, tech_lead, fast_track_agent
- Graph routing: Feature/Bug → pm_agent; Research/Chore → fast_track_agent
- Tools: `get_project_overview()`, `get_directory_tree()`

**test_api_*.py** (8 tests):
- Workspace CRUD + GET single
- Project CRUD + GET single + PATCH update
- Task CRUD + status transitions + logs

**test_persistence.py** (1 test):
- AsyncSqliteSaver state persistence

### Frontend E2E Tests ✅ 3/3 PASSING
```bash
npm run test:e2e --prefix frontend
```

**Test Suites**:
1. **agent-workflow.spec.ts** — Full end-to-end workflow (create task → agent processes → results display)
2. **manager-flow.spec.ts** — Dashboard navigation + directory picker
3. Combined: ~2 minutes total

**Playwright Config**: `baseURL: http://localhost:3001` (Vite dev server port)

## How to Extend

### Adding a New Agent Node
1. Implement in [backend/app/agents/nodes.py](backend/app/agents/nodes.py):
   ```python
   def my_node(state: AgentState) -> dict:
       context = _build_context_block(state)
       # ... call LLM, process response
       return {
           "ai_spec_doc": "# Spec",
           "antigravity_prompt": "Do X",
           "subtask_list": [{"title": "...", "step_order": 1, "target_files": "..."}]
       }
   ```
2. Register in [backend/app/agents/graph.py](backend/app/agents/graph.py):
   ```python
   workflow.add_node("my_node", my_node)
   workflow.add_edge("prev_node", "my_node")
   ```
3. Add tests in [backend/tests/test_agent_nodes.py](backend/tests/test_agent_nodes.py)
4. Update frontend [frontend/src/types/index.ts](frontend/src/types/index.ts) if state changes

### Adding a New API Endpoint
1. Create function in [backend/app/api/v1/endpoints/](backend/app/api/v1/endpoints/):
   ```python
   @router.get("/path/{id}")
   def get_item(id: UUID, session: Session = Depends(deps.get_session)):
       item = session.get(Model, id)
       if not item:
           raise HTTPException(status_code=404)
       return item
   ```
2. Register in [backend/app/api/v1/api.py](backend/app/api/v1/api.py):
   ```python
   api_router.include_router(your_router, prefix="/path")
   ```
3. Update [frontend/src/services/api.ts](frontend/src/services/api.ts) with new client function
4. Add test in [backend/tests/](backend/tests/)

## Dev / Run / Test Commands

### Backend (from repo root)
```bash
# Setup
python -m venv .venv
source .venv/bin/activate
pip install -e backend[dev]

# Start
.venv/bin/uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8000

# Tests (deterministic)
MOCK_LLM=true .venv/bin/python -m pytest backend/tests/ -v

# Reset
rm -rf db/ checkpoints.db
```

### Frontend
```bash
# Setup
cd frontend
npm install

# Development (http://localhost:3001)
npm run dev

# Type check
npx tsc --noEmit

# E2E tests
npm run test:e2e
```

## Known Issues & Quirks ⚠️

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError: No module named 'backend'` | Run from repo root or use `backend.app.main` imports |
| Port conflict | `lsof -i :8000 -i :3001 \| xargs -r kill -9` |
| E2E tests timeout/fail | Ensure frontend dev server on 3001, playwright baseURL correct |
| Playwright tests use port 3000 | Update [frontend/playwright.config.ts](frontend/playwright.config.ts) baseURL to 3001 |
| API 405 Not Found | Check router registered in [backend/app/api/v1/api.py](backend/app/api/v1/api.py) |
| Stale checkpoints | Delete `checkpoints.db`, restart backend |
| Pydantic model conflicts | Update both [backend/app/models/__init__.py](backend/app/models/__init__.py) + [frontend/src/types/index.ts](frontend/src/types/index.ts) |
| MCP tools missing | Ensure Node.js/npm installed, `mcp[cli]` in backend/pyproject.toml |
| Fast-track agent async errors | Account for async node execution if adding nodes after it |
