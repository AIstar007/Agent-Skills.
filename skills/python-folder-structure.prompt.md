# Clean Architecture — Python AI Agent Project Scaffold

## Description
Generates a clean architecture folder structure for Python AI agent projects using FastAPI, following strict layer separation (Domain → Application → Infrastructure → Presentation) with a dedicated agent layer. Use when scaffolding a new project or restructuring an existing one.

## Trigger Phrases
- "scaffold clean architecture"
- "create project structure"
- "new agent project"
- "clean architecture folder structure"
- "set up clean architecture"

---

## Architecture Overview

```
Dependency Rule: Outer layers depend inward. Inner layers NEVER import from outer layers.

  ┌─────────────────────────────────────────────────┐
  │                 Presentation                     │  ← FastAPI routes, deps, exception handlers
  ├─────────────────────────────────────────────────┤
  │                 Application                      │  ← Use cases, workflows, orchestration
  ├─────────────────────────────────────────────────┤
  │           Agents  /  Agent Factory               │  ← AI agent definitions, configs, prompts
  ├─────────────────────────────────────────────────┤
  │                Infrastructure                    │  ← External clients, DB, cache, MCP, HTTP
  ├─────────────────────────────────────────────────┤
  │                   Domain                         │  ← Models, interfaces (ABCs), domain services
  ├─────────────────────────────────────────────────┤
  │            Shared  /  Settings  /  Config        │  ← Cross-cutting: utils, tools, env config
  └─────────────────────────────────────────────────┘
```

---

## Full Folder Structure

When the user asks to scaffold a project, create **exactly** this structure. Replace `{project_name}` and `{agent_name}` with actual names. Every directory must contain an `__init__.py`.

```
{project_root}/
├── Dockerfile
├── pyproject.toml
├── README.md
├── .gitignore
├── .python-version
│
└── app/
    ├── __init__.py
    ├── main.py                          # FastAPI app factory + lifespan
    ├── cli.py                           # CLI entry point (optional)
    │
    ├── domain/                          # ── INNERMOST LAYER ──
    │   ├── __init__.py
    │   ├── exceptions.py                # Domain-specific exceptions
    │   │
    │   ├── models/                      # Pure data models (dataclasses / Pydantic)
    │   │   ├── __init__.py
    │   │   ├── agent.py                 # AgentConfig model
    │   │   ├── session.py               # ConversationSession, state models
    │   │   ├── {entity}.py              # One file per domain entity
    │   │   └── ...
    │   │
    │   ├── interfaces/                  # Abstract base classes (ports)
    │   │   ├── __init__.py
    │   │   ├── agent_base.py            # BaseAgent ABC (initialize / run / close)
    │   │   ├── repositories.py          # Repository ABCs
    │   │   └── session_store.py         # SessionStore ABC
    │   │
    │   └── services/                    # Domain services (pure business logic)
    │       ├── __init__.py
    │       └── {service_name}.py        # e.g. booking_service.py, timatic.py
    │
    ├── application/                     # ── USE CASE / ORCHESTRATION LAYER ──
    │   ├── __init__.py
    │   │
    │   ├── use_cases/                   # One class per use case
    │   │   ├── __init__.py
    │   │   └── {feature}_use_case.py    # e.g. document_check_use_case.py
    │   │
    │   └── workflows/                   # Multi-step workflow orchestration
    │       ├── __init__.py
    │       └── {workflow_name}/
    │           ├── __init__.py
    │           ├── workflow.py           # Workflow definition / DAG
    │           ├── shared.py             # Shared workflow state
    │           └── executors/            # Individual step executors
    │               ├── __init__.py
    │               └── {step_name}.py
    │
    ├── agents/                          # ── AI AGENT DEFINITIONS ──
    │   ├── __init__.py
    │   │
    │   └── {agent_name}/                # One folder per agent
    │       ├── __init__.py
    │       ├── agent.py                 # Agent class extending BaseAgent
    │       ├── config.yaml              # Agent config (model, tools, prompts, sub_agents)
    │       ├── tools.py                 # Agent-local tool functions (optional)
    │       ├── prompts/
    │       │   └── system.jinja         # System prompt template (Jinja2)
    │       └── sub_agents/              # Nested sub-agents (optional)
    │           └── {sub_agent_name}/
    │               ├── __init__.py
    │               ├── agent.py
    │               ├── config.yaml
    │               └── prompts/
    │                   └── system.jinja
    │
    ├── agent_factory/                   # ── AGENT LIFECYCLE MANAGEMENT ──
    │   ├── __init__.py
    │   ├── factory.py                   # Discovers, resolves deps, builds & caches agents
    │   ├── loader.py                    # Config discovery & YAML loading
    │   ├── models.py                    # ResolvedAgentDeps, ResolvedSubAgentDeps
    │   └── shared_tool_registry.py      # Cross-agent shared tool resolution
    │
    ├── infrastructure/                  # ── ADAPTERS / EXTERNAL WORLD ──
    │   ├── __init__.py
    │   │
    │   ├── external/                    # Third-party API clients
    │   │   ├── __init__.py
    │   │   └── {provider}/              # One folder per external system
    │   │       ├── __init__.py
    │   │       ├── client.py            # HTTP client implementation
    │   │       ├── models.py            # Request/response DTOs
    │   │       └── enums.py             # Provider-specific enums (optional)
    │   │
    │   ├── http/                        # Shared HTTP infrastructure
    │   │   ├── __init__.py
    │   │   ├── base_client.py           # Base async HTTP client
    │   │   └── exceptions.py            # HTTP-layer exceptions
    │   │
    │   ├── persistence/                 # Database adapters
    │   │   ├── __init__.py
    │   │   └── relational/
    │   │       ├── __init__.py
    │   │       └── base.py              # Concrete repository implementations
    │   │
    │   ├── session/                     # Session store adapters
    │   │   ├── __init__.py
    │   │   └── redis_store.py           # Redis-backed SessionStore impl
    │   │
    │   ├── mcp/                         # MCP server integration
    │   │   ├── __init__.py
    │   │   └── registry.py              # MCP tool registry
    │   │
    │   └── services/                    # Infrastructure service registry (DI root)
    │       ├── __init__.py
    │       └── registry.py              # ServiceRegistry: wires infra → domain ports
    │
    ├── presentation/                    # ── API / ENTRY POINT LAYER ──
    │   ├── __init__.py
    │   ├── deps.py                      # Dependency providers (use case factories)
    │   ├── exceptions.py                # Global exception handlers
    │   │
    │   └── v1/                          # Versioned API routes
    │       ├── __init__.py
    │       ├── router.py                # Aggregates all sub-routers
    │       ├── health.py                # Health check route
    │       └── {feature}/               # Feature-specific routes
    │           ├── __init__.py
    │           └── routes.py
    │
    ├── settings/                        # ── APP CONFIGURATION ──
    │   ├── __init__.py
    │   ├── settings.py                  # Config loader (JSON merge, validation)
    │   └── models.py                    # Pydantic settings models (AppConfig, etc.)
    │
    ├── config/                          # ── STATIC CONFIGURATION FILES ──
    │   ├── mcp_servers.json             # MCP server definitions
    │   ├── constants/                   # Domain-specific constants
    │   │   └── {provider}_constant.py
    │   └── environments/                # Per-environment JSON configs
    │       ├── environment.json         # Active config (symlinked or copied)
    │       ├── environment.dev1.json
    │       ├── environment.qa.json
    │       ├── environment.staging.json
    │       ├── environment.prod.json
    │       └── ...
    │
    └── shared/                          # ── CROSS-CUTTING CONCERNS ──
        ├── __init__.py
        ├── execution_context.py         # Immutable per-request context dataclass
        │
        ├── tools/                       # Shared agent tools (FunctionTool instances)
        │   ├── __init__.py
        │   └── {tool_name}.py
        │
        └── utils/                       # Pure utility functions
            ├── __init__.py
            ├── auth.py
            ├── logging.py
            ├── datetime_utils.py
            └── ...
```

---

## Layer Rules & Conventions

### 1. Domain Layer (`app/domain/`)
- **NEVER** imports from `application`, `infrastructure`, `presentation`, or `agents`.
- Contains only: pure models, ABCs (interfaces), domain services, domain exceptions.
- Domain services accept interface types, never concrete implementations.
- Models are `dataclass(frozen=True)` or Pydantic `BaseModel`.

### 2. Application Layer (`app/application/`)
- Imports from `domain` only.
- **Use cases** follow a single `execute()` method pattern:
  ```python
  class SomeUseCase:
      def __init__(self, agent: BaseAgent, session_store: SessionStore):
          self._agent = agent
          self._session_store = session_store

      async def execute(self, ...) -> ResponseModel:
          # load state → call agent/service → persist state → return
  ```
- **Workflows** decompose multi-step processes into discrete `executors/`.

### 3. Agents Layer (`app/agents/`)
- Each agent lives in its own folder with `agent.py`, `config.yaml`, and `prompts/`.
- Agent classes extend `domain.interfaces.agent_base.BaseAgent`.
- `config.yaml` declares: model, tools, sub_agents, mcp_servers, prompt files.
- Agent-local tools go in `tools.py` inside the agent folder.
- Sub-agents nest inside `sub_agents/` with the same folder pattern.

### 4. Agent Factory (`app/agent_factory/`)
- Discovers all `config.yaml` files under `agents/`.
- Topologically sorts agents by `agent_tools` dependencies.
- Resolves: local tools → shared tools → MCP tools → sub-agents → prompts.
- Caches built agents as singletons in `_agent_cache`.

### 5. Infrastructure Layer (`app/infrastructure/`)
- Implements domain interfaces (repositories, session stores).
- `external/{provider}/` — one folder per third-party API with `client.py` + `models.py`.
- `http/base_client.py` — shared async HTTP base with retries, auth, error mapping.
- `services/registry.py` — `ServiceRegistry` acts as the composition root, wiring concrete adapters.
- `persistence/` — database adapter implementations.
- `session/` — session store implementations (Redis, etc.).
- `mcp/` — MCP tool server registry and lifecycle.

### 6. Presentation Layer (`app/presentation/`)
- FastAPI routers only. No business logic.
- `deps.py` — factory functions that compose use cases from `ServiceRegistry` + `agent_factory`.
- `v1/router.py` — aggregates all feature routers under a versioned prefix.
- Exception handlers translate domain exceptions to HTTP responses.

### 7. Settings & Config (`app/settings/`, `app/config/`)
- `settings/` — runtime config loading, validation, Pydantic models.
- `config/environments/` — static per-environment JSON files.
- `config/constants/` — domain-specific constant values.

### 8. Shared (`app/shared/`)
- `execution_context.py` — immutable `@dataclass(frozen=True)` passed through all layers.
- `tools/` — shared `FunctionTool` instances usable by any agent.
- `utils/` — pure utility functions (logging, auth helpers, datetime).

---

## Key Patterns

### Entry Point (`app/main.py`)
```python
@asynccontextmanager
async def lifespan(_: FastAPI):
    settings = get_settings()
    await service_registry.initialize(settings)
    await build_agents()
    try:
        yield
    finally:
        await close_agents()
        await service_registry.close()

def create_app() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    register_exception_handlers(app)
    app.include_router(api_v1_router, prefix="/api/v1")
    return app
```

### Dependency Injection Flow
```
Route handler
  → deps.py factory (composes use case with concrete adapters)
    → UseCase.__init__(agent, session_store, ...)
      → UseCase.execute()
        → agent.run(context, ...)
          → tools / sub-agents / MCP
```

### BaseAgent Lifecycle
```python
class BaseAgent(ABC):
    def __init__(self, deps: ResolvedAgentDeps) -> None: ...
    async def initialize(self) -> None: ...   # called once at startup
    async def run(self, context, **kwargs): ...  # called per request
    async def close(self) -> None: ...         # called at shutdown
```

### ExecutionContext
```python
@dataclass(frozen=True)
class ExecutionContext:
    session_id: str = ""
    token: str = ""
    correlation_id: str = ""
```

---

## Scaffold Instructions

When creating a new project with this architecture:

1. **Create the root files**: `Dockerfile`, `pyproject.toml`, `README.md`, `.gitignore`, `.python-version`.
2. **Create `app/main.py`** with the lifespan pattern above.
3. **Create domain layer first** — models, interfaces, exceptions. This is the stable core.
4. **Create infrastructure layer** — implement the domain interfaces with concrete adapters.
5. **Create `infrastructure/services/registry.py`** — the ServiceRegistry composition root.
6. **Create agents** — each in `agents/{name}/` with `agent.py`, `config.yaml`, `prompts/system.jinja`.
7. **Create `agent_factory/`** — factory, loader, models, shared tool registry.
8. **Create application layer** — use cases and workflows that orchestrate agents + domain services.
9. **Create presentation layer** — FastAPI routes, deps.py, exception handlers.
10. **Create settings & config** — environment JSON files, Pydantic config models.
11. **Create shared** — execution context, shared tools, utility functions.
12. **Add `__init__.py`** to every Python package directory.

---

## File Templates

### `__init__.py` (every package)
```python
```
(Empty file — just marks the directory as a Python package.)

### Agent `config.yaml`
```yaml
agent_name: {agent_name}
model: gemini-2.0-flash
instruction: prompts/system.jinja
tools:
  - tool_name_1
  - tool_name_2
sub_agents: []
agent_tools: []
mcp_servers: []
```

### Agent `prompts/system.jinja`
```jinja
You are a {{ agent_name }} agent.
{{ instruction_body }}
```

### Domain Model
```python
from __future__ import annotations
from dataclasses import dataclass

@dataclass(frozen=True)
class EntityName:
    field_one: str
    field_two: int
```

### Domain Interface (ABC)
```python
from __future__ import annotations
from abc import ABC, abstractmethod

class SomeRepository(ABC):
    @abstractmethod
    async def save(self, entity: Entity) -> None:
        raise NotImplementedError
```

### Use Case
```python
from __future__ import annotations

class FeatureUseCase:
    def __init__(self, agent: BaseAgent, session_store: SessionStore) -> None:
        self._agent = agent
        self._session_store = session_store

    async def execute(self, message: str, session_id: str, token: str) -> Response:
        session = await self._session_store.load(session_id)
        if session is None:
            session = ConversationSession()
        ctx = ExecutionContext(session_id=session_id, token=token)
        result = await self._agent.run(ctx, message=message, history=session.conversation_history)
        await self._session_store.save(session_id, session)
        return Response(session_id=session_id, response=result)
```

### Presentation deps.py
```python
from __future__ import annotations

async def get_feature_use_case() -> FeatureUseCase:
    agent = get_agent("{agent_name}")
    return FeatureUseCase(
        agent=agent,
        session_store=service_registry.session_store,
    )
```
