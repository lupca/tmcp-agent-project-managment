
# Copilot Instructions — TMCP Agent Project Management (2026, E2E-Ready)

## 1. Project Overview

- **Backend**: FastAPI + LangGraph + SQLModel (Domain-Driven Design)
- **Frontend**: React 19 + Vite + Playwright (Feature-Slice)
- **Agent**: LangGraph workflow, async DB, robust context retrieval, deterministic LLM mocks for test
- **Testing**: 34+ backend tests, 12 E2E Playwright tests (full CRUD, agent, multi-project, modal, race conditions)

## 2. Key Files & Structure

**Backend**
- `backend/app/main.py`: FastAPI app, CORS, lifespan
- `backend/app/services/agent_service.py`: AgentService, async workflow, WAL mode, short-lived sessions
- `backend/app/agents/graph.py`: LangGraph, AgentState, routing
- `backend/app/agents/nodes.py`: Node logic, context, LLM calls
- `backend/app/agents/tools.py`: Project overview, code reading
- `backend/app/agents/mcp_tools.py`: MCP integration (optional)
- `backend/app/api/v1/endpoints/`: All API endpoints (workspaces, projects, tasks, filesystem)

## 6. Dev & Test Commands

**Backend**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -e backend[dev]
.venv/bin/uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8123
MOCK_LLM=true .venv/bin/python -m pytest backend/tests/ -v
```
**Frontend**
```bash
cd frontend
npm install
npm run dev
npx tsc --noEmit
npx playwright test --reporter=list
```

## 7. Troubleshooting & Quirks

- **Modal reopens after close**: Use retry-based closeModal (see E2E tests)
- **Port conflict**: `lsof -i :8123 -i :3000 | xargs -r kill -9`
- **API 405**: Check router registration in `api.py`
- **Pydantic/TypeScript model drift**: Update both backend models and frontend types
- **MCP tools missing**: Install Node.js/npm, `mcp[cli]`, `langchain-mcp-adapters`
- **Stale checkpoints**: Delete `checkpoints.db`, restart backend

---

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

**Playwright Config**: `baseURL: http://localhost:3000` (Vite dev server port)

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
.venv/bin/uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8123

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

# Development (http://localhost:3000)
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
| Port conflict | `lsof -i :8123 -i :3000 \| xargs -r kill -9` |
| E2E tests timeout/fail | Ensure frontend dev server on 3000, playwright baseURL correct |
| Playwright tests use port 3000 | Update [frontend/playwright.config.ts](frontend/playwright.config.ts) baseURL to 3000 |
| API 405 Not Found | Check router registered in [backend/app/api/v1/api.py](backend/app/api/v1/api.py) |
| Stale checkpoints | Delete `checkpoints.db`, restart backend |
| Pydantic model conflicts | Update both [backend/app/models/__init__.py](backend/app/models/__init__.py) + [frontend/src/types/index.ts](frontend/src/types/index.ts) |
| MCP tools missing | Ensure Node.js/npm installed, `mcp[cli]` in backend/pyproject.toml |
| Fast-track agent async errors | Account for async node execution if adding nodes after it |
