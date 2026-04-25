# System Design

## 1. Goal

本ドキュメントは、仕様を実装に落とす際のモジュール構成、責務分割、実行境界を定義する。`backend_spec.md` の論理設計を、実際のコードベース構成へ写像するための設計書である。

## 2. Repository Shape

推奨構成:

```text
data_analysis_consultant/
├── backend/
│   ├── api/
│   ├── graph/
│   ├── memory/
│   ├── policies/
│   ├── schemas/
│   ├── services/
│   ├── state/
│   ├── tools/
│   └── tests/
├── frontend/
│   ├── app/
│   ├── components/
│   ├── features/
│   ├── lib/
│   ├── mocks/
│   └── tests/
├── backend_spec.md
├── frontend_spec.md
├── frontend_api_mock.json
└── skills/
```

## 3. Backend Design

### 3.1 Schema Layer

責務:

- Pydantic state schema
- API request / response schema
- tool I/O schema
- JSON Schema / OpenAPI export

配置例:

- `backend/schemas/state.py`
- `backend/schemas/api.py`
- `backend/schemas/tools.py`

### 3.2 State and Memory Layer

責務:

- `GraphState` persistence
- project / decision / working / episodic memory の正本管理
- state versioning
- summary cache index

配置例:

- `backend/state/store.py`
- `backend/memory/working_memory.py`
- `backend/memory/episodic_memory.py`

### 3.3 Policy Layer

責務:

- deterministic policy evaluation
- pure-function reviewer / evaluator 群
- policy version 管理

配置例:

- `backend/policies/routing.py`
- `backend/policies/phase_gate.py`
- `backend/policies/execution_readiness.py`
- `backend/policies/advisor_consult.py`
- `backend/policies/qa_rulebook.py`

原則:

- policy layer は長期状態を持たない
- input / output は strict schema
- override は policy layer ではなく supervisor decision layer で扱う

### 3.4 Tool Layer

責務:

- metadata access
- artifact access
- governance tools
- execution tools
- access and fallback tools

配置例:

- `backend/tools/state_views.py`
- `backend/tools/artifacts.py`
- `backend/tools/governance.py`
- `backend/tools/sql_runtime.py`
- `backend/tools/python_runtime.py`
- `backend/tools/qa.py`
- `backend/tools/access.py`

### 3.5 Graph Layer

責務:

- LangGraph node 実装
- orchestration flow
- node-to-tool wiring

配置例:

- `backend/graph/nodes/intake.py`
- `backend/graph/nodes/supervisor.py`
- `backend/graph/nodes/state_update.py`
- `backend/graph/nodes/state_manager_refresh.py`
- `backend/graph/nodes/user_review_gate.py`
- `backend/graph/nodes/phase_gate.py`
- `backend/graph/nodes/error_triage.py`

### 3.6 Service Layer

責務:

- API から graph / state / tool をつなぐ
- projection API
- auth / tenant resolution

配置例:

- `backend/services/projects.py`
- `backend/services/reviews.py`
- `backend/services/artifacts.py`
- `backend/services/governance.py`

## 4. Frontend Design

### 4.1 App Shell

責務:

- workspace layout
- routing
- global notification

配置例:

- `frontend/app/projects/[projectId]/page.tsx`
- `frontend/components/layout/workspace-shell.tsx`

### 4.2 Feature Modules

責務:

- conversation
- reviews
- artifacts
- timeline
- governance
- deliverables

配置例:

- `frontend/features/conversation/`
- `frontend/features/reviews/`
- `frontend/features/artifacts/`
- `frontend/features/timeline/`
- `frontend/features/governance/`
- `frontend/features/deliverables/`

### 4.3 Shared UI and Data

責務:

- artifact badges
- quality stage indicators
- risk badges
- override banners
- API client

配置例:

- `frontend/components/ui/`
- `frontend/lib/api-client.ts`
- `frontend/lib/contracts.ts`

## 5. Key Runtime Boundaries

### 5.1 State Manager Boundary

- State Manager は recommendation を返さない
- typed query のみ受ける
- caller identity は runtime から inject される

### 5.2 Supervisor Boundary

- 最終裁定者
- policy tool の結果を受けて判断する
- unconditional override はできない

### 5.3 QA Boundary

- QA は stage 単位で実行する
- stage 間の進行は gate で制御する
- draft と final を混同しない

### 5.4 Frontend Boundary

- backend 生 state は持たない
- projection を表示する
- masked / unavailable を補完しない

## 6. Data Flow

### 6.1 Execution Path

1. API request
2. service layer
3. graph node
4. policy tool
5. execution tool
6. artifact store
7. state update
8. summary refresh
9. projection API
10. frontend render

### 6.2 Review Path

1. backend creates `UserReviewRequest`
2. review API exposes projection
3. frontend review center renders request
4. user submits action
5. backend records `UserInteractionRecord`
6. supervisor replans if needed

## 7. Implementation Notes

### 7.1 What Should Be Pure Functions

- risk classification
- phase gate evaluation
- advisor trigger evaluation
- QA gate evaluation
- access scope checks

### 7.2 What Should Persist

- GraphState
- artifacts
- decisions
- QA reviews
- working memory
- episodic memory
- user reviews

### 7.3 What Should Stay Replaceable

- LLM provider
- SQL runtime
- Python runtime
- policy store backend
- artifact store backend

## 8. Non-Goals for MVP

- multi-tenant collaboration UI
- rich annotation on charts
- automatic PMO agent
- full causal experimentation suite
- autonomous long-running retry orchestration without human review
