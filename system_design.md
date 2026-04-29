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

### 3.2.1 State Manager Service

責務:

- state 正本への read-path
- deterministic projection / filter / field selection
- summary view 生成
- working memory projection 生成
- summary cache
- caller identity に基づく view / field access 判定

配置例:

- `backend/services/state_manager.py`

原則:

- State Manager 本体は graph node ではなく service として実装する
- graph からは `state_manager_refresh_node` などの利用 node を通じて呼ぶ
- `query_state_view` も同じ service を叩く

### 3.2.2 State Persistence Design

実装初期の永続化方針は、Pydantic の入れ子構造をそのまま 1 行へ押し込むのではなく、主要ドメインごとに正規化し、再構成は service / repository 層で行う。

基本方針:

- `projects` 1 テーブル
  project-level metadata、phase、tenant、top-level status
- `tasks` 1 テーブル
  `task_queue` / `active_tasks` / `completed_tasks` は別 collection ではなく `status` 列で表現する
- `artifacts` 1 テーブル
  metadata を保持し、本体は artifact store に置く
- `decisions` 1 テーブル
  `SupervisorDecision` と関連 advisor note 参照
- `qa_reviews` 1 テーブル
  stage-based QA の結果
- `agent_invocations` 1 テーブル
  invocation metadata、tool call 参照
- `tool_calls` 1 テーブル
  tool execution metadata

JSONB を許容する領域:

- `project_memory`
- `decision_memory`
- `working_memory`
- `agent_episodic_memories`
- summary cache

原則:

- append-heavy かつ bounded / projection-oriented な構造は JSONB を許容する
- cross-record search / filter / join が必要なものは正規化する
- `agent_episodic_memories` は初期実装では project 単位 JSONB で持ってよい
- artifact body は DB に入れず object storage を正本とする
- Pydantic schema と SQLAlchemy ORM は分離する
- Alembic migration 命名は `0001_initial.py`, `0002_<topic>_<verb>.py` を採用する

非採用:

- SQLModel を source of truth にする設計
- GraphState 全体を 1 JSON blob だけで持つ設計
- queue ごとに task 専用テーブルを分ける設計

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

### 3.7 Human Review Trace Layer

責務:

- node / agent ごとの `Human Review Reasoning Trace` 出力
- redact 済み reasoning summary の永続化
- project / run / task / node 単位での trace 検索
- GraphState や agent input への再注入禁止の担保

配置例:

- `backend/observability/reasoning_trace.py`
- `backend/observability/redaction.py`

原則:

- trace は state 正本ではなく observability artifact として扱う
- 保存形式は JSONL を推奨する
- `supervisor_node`, `advisor_consult_node`, `phase_gate_node`, `user_review_gate_node`, `state_manager_refresh_node`, `error_triage_node` は必須出力対象
- specialist worker は Plan-Execute 対象から優先的に対応する

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

## 5. Contract Generation Design

backend と frontend の型契約は、手書き二重管理ではなく生成パイプラインで同期する。

### 5.1 Source of Truth

- backend の API request / response schema は `backend/schemas/*.py` の Pydantic を正本とする
- frontend は generated な `frontend/lib/contracts.ts` のみを消費し、手書き型を持たない
- SSE event は HTTP OpenAPI と別経路で `backend/schemas/events.py` を正本とする

### 5.2 Generation Pipeline

```text
backend/schemas/*.py            (Pydantic, source of truth)
   ↓ FastAPI が自動生成
backend OpenAPI document        (/openapi.json)
   ↓ openapi-typescript-codegen
frontend/lib/contracts.ts       (TypeScript contracts)
   ↓ import
frontend feature modules
```

SSE event は別パイプラインとする。

```text
backend/schemas/events.py
   ↓ export_event_schema.py
event JSON Schema
   ↓ json-schema-to-typescript
frontend/lib/events.ts
```

### 5.3 Backend Contract Rules

- 全 API endpoint の request / response は Pydantic モデルで定義する
- 主要 field には `Field(description=...)` を付ける
- Literal / Enum は top-level に集約する
- response model は `response_model=` を明示し、暗黙的 dict を避ける
- tool I/O は `backend/schemas/tools.py` に定義し、frontend に見せる projection は `backend/schemas/api.py` に別定義する

### 5.4 Frontend Contract Rules

- `frontend/lib/contracts.ts` は generated file とし、手動編集禁止
- backend schema 変更後の PR で必ず再生成する
- generated diff は source diff と同じ PR でレビューする
- React Query や feature module は generated 型を直接使う

### 5.5 Generated File Policy

- MVP では generated file を commit に含める
- 変更が必要なら generated file ではなく backend schema を修正する
- OpenAPI version は app version と同期する
- 型 rename は破壊的変更候補として扱い、versioning 判断を行う

### 5.6 Developer Flow

backend の API 変更時:

1. `backend/schemas/*.py` を編集する
2. schema unit test を通す
3. `make codegen` で OpenAPI 出力と TS 再生成を行う
4. frontend 側の型エラーを解消する
5. backend / frontend / generated diff を一つの PR に含める

このフロー違反、特に generated TypeScript の手編集はレビューで reject する。

## 6. Key Runtime Boundaries

### 6.1 State Manager Boundary

- State Manager は recommendation を返さない
- typed query のみ受ける
- caller identity は runtime から inject される
- State Manager 本体は service であり、graph node はその利用者である

### 6.2 Supervisor Boundary

- 最終裁定者
- policy tool の結果を受けて判断する
- unconditional override はできない

### 6.3 QA Boundary

- QA は stage 単位で実行する
- stage 間の進行は gate で制御する
- draft と final を混同しない

### 6.4 Frontend Boundary

- backend 生 state は持たない
- projection を表示する
- masked / unavailable を補完しない

## 7. Data Flow

### 7.1 Execution Path

1. API request
2. service layer
3. graph node
4. policy tool
5. execution tool
6. artifact store
7. state update
8. summary refresh via State Manager service
9. projection API
10. frontend render

### 7.2 Review Path

1. backend creates `UserReviewRequest`
2. review API exposes projection
3. frontend review center renders request
4. user submits action
5. backend records `UserInteractionRecord`
6. supervisor replans if needed

## 8. Implementation Notes

### 8.1 What Should Be Pure Functions

- risk classification
- phase gate evaluation
- advisor trigger evaluation
- QA gate evaluation
- access scope checks

### 8.2 What Should Persist

- GraphState
- artifacts
- decisions
- QA reviews
- working memory
- episodic memory
- user reviews

### 8.3 What Should Stay Replaceable

- LLM provider
- SQL runtime
- Python runtime
- policy store backend
- artifact store backend

## 9. Non-Goals for MVP

- multi-tenant collaboration UI
- rich annotation on charts
- automatic PMO agent
- full causal experimentation suite
- autonomous long-running retry orchestration without human review
