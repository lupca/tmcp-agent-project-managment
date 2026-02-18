# Quick Start Guide - TMCP Agent Project Management

## Prerequisites
- Python 3.12+ with virtual environment already set up at `.venv/`
- Node.js 16+ with npm
- Both servers should be started for full functionality

## Running the Application

### 1. Start Backend Server (Terminal 1)
```bash
cd /Users/bodoi17/projects/tmcp-agent-project-managmet

# Activate venv if not already active
source .venv/bin/activate

# Start backend on port 8123
.venv/bin/uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8123
```

**Expected Output:**
```
INFO:     Uvicorn running on http://127.0.0.1:8123
INFO:     Application startup complete
```

**Available at:**
- API Docs: http://127.0.0.1:8123/docs
- OpenAPI Schema: http://127.0.0.1:8123/openapi.json

### 2. Start Frontend Server (Terminal 2)
```bash
cd /Users/bodoi17/projects/tmcp-agent-project-managmet/frontend

npm run dev
```

**Expected Output:**
```
  VITE v6.4.1  ready in 93 ms
  ➜  Local:   http://localhost:3000/
  ➜  Network: http://192.168.1.6:3000/
```

**Open in Browser:** http://localhost:3000/

## API Endpoints

### Workspaces
```bash
# Get all workspaces
curl http://127.0.0.1:8123/workspaces/

# Create workspace
curl -X POST http://127.0.0.1:8123/workspaces/ \
  -H "Content-Type: application/json" \
  -d '{"name":"My Workspace"}'
```

### Projects
```bash
# Get all projects
curl http://127.0.0.1:8123/projects/

# Create project
curl -X POST http://127.0.0.1:8123/projects/ \
  -H "Content-Type: application/json" \
  -d '{"name":"My Project","workspace_id":"<workspace_id>","repo_path":"/path/to/repo","tech_stack":"Python"}'
```

### Tasks
```bash
# Get all tasks
curl http://127.0.0.1:8123/tasks/

# Create task
curl -X POST http://127.0.0.1:8123/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title":"New Task","project_id":"<project_id>","type":"Feature","raw_input":"Do something"}'

# Update task status (triggers agent workflow)
curl -X PATCH "http://127.0.0.1:8123/tasks/{task_id}/status?status=Agent_Processing"

# Get task logs
curl http://127.0.0.1:8123/tasks/{task_id}/logs

# Get task subtasks
curl http://127.0.0.1:8123/tasks/{task_id}/subtasks
```

## Testing

### Run Backend Tests
```bash
cd /Users/bodoi17/projects/tmcp-agent-project-managmet

# Run with mock LLM for fast, deterministic testing
MOCK_LLM=true .venv/bin/python -m pytest backend/tests/ -v

# Run specific test
MOCK_LLM=true .venv/bin/python -m pytest backend/tests/test_api_workspaces.py -v
```

### Run Frontend E2E Tests
```bash
# Make sure both backend and frontend servers are running first
cd /Users/bodoi17/projects/tmcp-agent-project-managmet/frontend

npm run test:e2e

# Run specific test
npx playwright test tests/agent-workflow.spec.ts
```

### TypeScript Check
```bash
cd /Users/bodoi17/projects/tmcp-agent-project-managmet/frontend

npx tsc --noEmit  # Should output nothing if all types are correct
```

## Frontend Workflow

The application uses the following workflow:

1. **Dashboard** (http://localhost:3000/) - View all workspaces and projects
2. **Project View** (http://localhost:3000/projects/:projectId) - Kanban board for tasks by status
3. **Manager View** (http://localhost:3000/manager) - Bulk task management

### Creating a Workflow:
1. Create a Workspace (Dashboard → "New Workspace")
2. Create a Project (Dashboard → "New Project")
3. Create a Task (Project View → "Add Task")
4. Update Task Status to "Agent_Processing" to start the agent workflow
5. Monitor progress in TaskLogs
6. View AI-generated spec and subtasks when ready

## Database Files

- **SQLite Database**: `db/` directory (created on first run)
- **LangGraph Checkpoints**: `checkpoints.db` (created on agent initialization)

To reset:
```bash
rm -rf db/
rm checkpoints.db
```

## Environment Variables

Backend looks for these in `backend/.env` (optional):
```bash
OLLAMA_MODEL=qwen3:4b-instruct-2507-q4_K_M
OLLAMA_BASE_URL=http://localhost:11434
DATABASE_URL=sqlite:///db/tmcp.db
MOCK_LLM=false  # Set to true for mock responses
```

Frontend API is configured in `frontend/src/services/api.ts`:
```typescript
const API_BASE_URL = 'http://127.0.0.1:8123';
```

## Troubleshooting

### Port Already in Use
```bash
# Kill process on port 8123
lsof -i :8123 | awk 'NR>1 {print $2}' | xargs kill -9

# Kill process on port 3000
lsof -i :3000 | awk 'NR>1 {print $2}' | xargs kill -9
```

### Import Errors
Make sure you're running from the correct directory:
- Backend commands should run from project root: `/Users/bodoi17/projects/tmcp-agent-project-managmet/`
- Frontend commands should run from: `/Users/bodoi17/projects/tmcp-agent-project-managmet/frontend/`

### Database Issues
If you get database-related errors:
```bash
# Reset database
rm -rf db/ checkpoints.db

# Restart backend - it will recreate tables
```

### Dependencies Issues
```bash
# Backend
cd /Users/bodoi17/projects/tmcp-agent-project-managmet
.venv/bin/pip install -e backend[dev]

# Frontend
cd frontend
npm install
```

## Project Structure

```
tmcp-agent-project-managmet/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app
│   │   ├── api/                 # API endpoints
│   │   ├── core/                # Config & database
│   │   ├── services/            # Business logic
│   │   ├── agents/              # LangGraph workflows
│   │   └── models/              # Database models
│   ├── tests/                   # Tests (34 passing)
│   └── pyproject.toml
├── frontend/
│   ├── src/
│   │   ├── app/                 # Main app setup
│   │   ├── features/            # Feature components
│   │   ├── components/          # Shared UI
│   │   ├── services/            # API client
│   │   └── types/               # TypeScript definitions
│   ├── tests/                   # E2E tests
│   └── package.json
├── REFACTOR_FIXES.md           # Detailed fix report
└── README.md                    # Original docs
```

## Current Status

✅ **All tests passing** (34/34 backend tests)
✅ **Backend server running** on http://127.0.0.1:8123
✅ **Frontend server running** on http://localhost:3000
✅ **TypeScript compiling** without errors
✅ **APIs fully functional**

## Next Steps

1. Open http://localhost:3000 in your browser
2. Create a workspace and project
3. Add a task and trigger the agent workflow
4. Monitor the task logs and review generated specs

For more details, see `REFACTOR_FIXES.md` in the project root.
