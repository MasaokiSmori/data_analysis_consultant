# Implementation Plan

## 1. Goal

本ドキュメントは、`backend_spec.md` / `frontend_spec.md` / `system_design.md` で定義されたシステムを、coding agent または開発者が実際にコードに落とせる粒度に分解した実装計画である。

各 phase は milestone (`milestone_review_checklist.md`) に対応する。phase 内の task は coding agent への単一依頼として渡せる粒度を目標とし、依頼時は `task_brief_template.md` から brief を composing する。

## 2. Planning Principles

- 各 phase で「何を作るか」と「何をまだ作らないか」を明記する
- 早い段階で typed schema と mock contract を固める
- 早い段階で end-to-end の最小経路を通す
- governance と QA は後付けではなく、phase 1 から骨格を入れる
- human review は phase 完了時ではなく、破壊的変更の前に挟む
- task は依存関係を明示し、並列実行可能な単位に切る
- **1 task = 1 PR を原則** とする

## 3. How to Read This Plan

- 各 phase は `Goals` / `含まないもの` / `Tasks 表` / `Notable tasks (深掘り)` / `Exit criteria` で構成
- task ID は `T<phase>.<seq>` (例: T1.3)
- task 表の `Deps` 列で前提タスクを示す
- **同じ Deps を持つ task 群は並列着手可能**
- `task_brief_template.md` を使って task 表の 1 行から brief を composing する
- Notable tasks は実装難度が高い task の補足説明(全 task は深掘りしない)

## 4. Task Sizing

| ラベル | 規模感 |
|---|---|
| **S** | 1 日以下、1-2 ファイル、設計判断不要 |
| **M** | 2-5 日、複数ファイル、仕様 section の理解必要 |
| **L** | 1-2 週間、複数コンポーネント、設計メモを先に書く |
| **XL** | 着手前にさらに分解する |

## 5. Phase Overview

| Phase | Focus | Tasks | 期間目安 | Milestone |
|---|---|---|---|---|
| 0 | Tooling Bootstrap | 7 | 1-2 週間 | — |
| 1 | Schema & Contracts | 12 | 2-3 週間 | A |
| 2 | State & Artifacts | 15 | 3-4 週間 | B |
| 3 | Supervisor & Governance | 21 | 3-4 週間 | C |
| 4 | Specialist Execution MVP | 22 | 5-7 週間 | D |
| 5 | Frontend MVP | 22 | 5-7 週間 | E |
| 6 | Hardening | 13 | 3-4 週間 | F |

並列化機会:

- Phase 1 完了後、Phase 2 と Phase 5 (frontend MVP の MSW 駆動部分) は並列着手可能
- Phase 3 内では LLM provider 系 (T3.1-T3.5) と policy 系 (T3.6-T3.12) が並列
- Phase 4 内では SQL 系 (T4.2-T4.6) と Python 系 (T4.7-T4.8) と prompt 系 (T4.10 以降) が並列

---

## 6. Phase 0: Tooling Bootstrap

### 6.1 Status

Workspace Bootstrap (ディレクトリ・ドキュメント整備) は完了済み。本 phase では実コード基盤を整える。

### 6.2 Goals

- Python / TypeScript / DB / CI の最小可動セットアップ
- "hello" レベルの API + frontend が通信できる状態に到達
- codegen pipeline が drift 検知できる状態

### 6.3 含まないもの

- ビジネスロジック
- LangGraph 統合
- 本番接続
- 認証(`(auth)` route group の枠だけ Phase 5 で扱う)

### 6.4 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T0.1 | Backend project skeleton (uv + pyproject + ruff + mypy + pytest) | `backend/pyproject.toml`, `backend/.python-version`, `backend/ruff.toml`, `backend/mypy.ini`, `backend/conftest.py` | — | M |
| T0.2 | Frontend project skeleton (pnpm + Next.js 15 + shadcn/ui + biome + Vitest) | `frontend/package.json`, `frontend/next.config.ts`, `frontend/tsconfig.json`, `frontend/biome.json` | — | M |
| T0.3 | Local dev infra (docker-compose + Postgres+pgvector) | `docker-compose.yml`, `.env.example`, `Makefile` | — | S |
| T0.4 | CI minimal (lint / typecheck / unit / codegen drift) | `.github/workflows/ci.yml` | T0.1, T0.2 | M |
| T0.5 | Codegen pipeline scaffolding | `backend/scripts/export_openapi.py`, `frontend/lib/contracts.ts` (placeholder), `Makefile` `codegen` target | T0.1, T0.2 | M |
| T0.6 | Pre-commit hooks (ruff format, biome check) | `.pre-commit-config.yaml` | T0.1, T0.2 | S |
| T0.7 | Hello-world FastAPI endpoint + Next.js fetch demo | `backend/api/main.py`, `frontend/app/page.tsx` | T0.1, T0.2, T0.3 | S |

### 6.5 Notable tasks

#### T0.5 Codegen pipeline scaffolding

`system_design.md §5` の contract generation pipeline を動く状態にする。MVP では空の OpenAPI から空の TS 型を生成できれば良い。

- backend に "echo" エンドポイント 1 つ用意
- `export_openapi.py` で `openapi.json` を生成
- `pnpm exec openapi-typescript` で `frontend/lib/contracts.ts` 生成
- `Makefile` に `make codegen` target
- CI で codegen 後の diff が空であることを確認(drift 防止)

### 6.6 Exit criteria

- `make dev` で backend + frontend + Postgres が起動
- `make test` で空テストが pass
- `make codegen` で型生成が成功
- CI が lint / typecheck / unit / codegen drift check を pass
- `frontend/app/page.tsx` から `backend/api/main.py` の `/echo` を叩いて表示できる

---

## 7. Phase 1: Schema & Contracts

### 7.1 Goals

`backend_spec.md §14.2` の Pydantic 全モデルと API / tool / event schema を実装。frontend が期待する mock との整合を取る。

### 7.2 含まないもの

- 永続化(モデル定義のみ。DB 接続は Phase 2)
- LangGraph 実行
- LLM 呼び出し
- service 層

### 7.3 Tasks

| ID | Task | Files | Deps | Size | Spec ref |
|---|---|---|---|---|---|
| T1.1 | Literals + Enums (`AgentName`, `PhaseName`, `RiskLevel`, `QualityStage`, `DecisionBasis`, etc.) | `backend/schemas/state.py` (Literals 部) | T0.1 | S | `backend_spec.md §14.2 (lines 1547-1592)` |
| T1.2 | AuditFields, OpenQuestion, TeamMember, ProjectContext | `backend/schemas/state.py` | T1.1 | S | §14.2 |
| T1.3 | TaskRecord 関連 (`TaskInputBundle`, `TaskOutputBundle`, `TaskRecord`) | `backend/schemas/state.py` | T1.1 | M | §14.2 |
| T1.4 | ArtifactRef + ToolCallRecord + AgentInvocation 系 | `backend/schemas/state.py` | T1.1 | M | §14.2 |
| T1.5 | Decision / Recommendation / Advisor / QA / UserReview 系 | `backend/schemas/state.py` | T1.1 | M | §14.2 |
| T1.6 | Memory 系 (`WorkingMemory`, `AgentEpisodicMemoryEntry`) | `backend/schemas/state.py` | T1.1 | S | §14.2, §7.4 |
| T1.7 | Cache / Phase / Final / Runtime / Catalog / SummaryViews | `backend/schemas/state.py` | T1.1 | S | §14.2 |
| T1.8 | GraphState top-level | `backend/schemas/state.py` | T1.2-T1.7 | S | §14.2 |
| T1.9 | Tool I/O schema (15 priority tools) | `backend/schemas/tools.py` | T1.1 | L | `backend_spec.md §9.7-§9.8` |
| T1.10 | API request/response schema (per `frontend_spec.md §11`) | `backend/schemas/api.py` | T1.1, T1.4 | M | `frontend_spec.md §11` |
| T1.11 | SSE event schema (discriminated union) | `backend/schemas/events.py` | T1.1 | M | `frontend_spec.md §12.2.1` |
| T1.12 | Mock alignment test (`frontend_api_mock.json` ↔ schema) | `backend/tests/unit/schemas/test_mock_alignment.py` | T1.10 | S | — |

### 7.4 Notable tasks

#### T1.9 Tool I/O schema

`backend_spec.md §9.8` の 15 priority tool すべての strict I/O Pydantic モデル。

- ファイル分割: 1 tool 1 セクション、または domain でグループ化(state_views, governance, qa, sql, python)
- 入力モデルと出力モデルのペア
- error envelope は共通 base class
- 各モデルに `Field(description=...)` 必須(OpenAPI 反映のため)
- example payloads は `backend/tests/fixtures/tool_examples/` に配置
- caller identity 系 field は schema に含めない(runtime 注入のため)

#### T1.10 API schema

`frontend_spec.md §11` 全エンドポイントの request / response モデル。

- projection モデル(frontend が受ける形)を別途定義(生 GraphState 露出禁止)
- `frontend_api_mock.json` の各 path に対応
- Tanstack Query が消費するため、null / optional の扱いを明示

#### T1.12 Mock alignment

`frontend_api_mock.json` の各 entry が `backend/schemas/api.py` の対応モデルで parse できることを assert。

- 失敗したら即 schema を修正(mock を変えない)
- 新 endpoint を追加する場合、mock も更新(`frontend_api_mock.json` 側で追加)

### 7.5 Exit criteria

- すべての Pydantic モデルが mypy strict で pass
- mock alignment test が pass
- OpenAPI 生成が成功し、frontend codegen で型が出力される
- `milestone_review_checklist.md` Milestone A の全項目クリア

---

## 8. Phase 2: State & Artifacts

### 8.1 Goals

State Manager の read-path、artifact registry、memory 更新を成立させる。LLM 実行はまだ含めない。

### 8.2 含まないもの

- LLM 実行
- policy module(Phase 3)
- specialist worker(Phase 4)
- 認証(Phase 6)

### 8.3 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T2.1 | DB connection (SQLAlchemy async + Alembic init) | `backend/state/db.py`, `backend/alembic/`, `backend/alembic.ini` | T0.3, T1.8 | M |
| T2.2 | Initial migration (全テーブル) | `backend/alembic/versions/0001_initial.py` | T2.1 | M |
| T2.3 | GraphState store (CRUD) | `backend/state/store.py` | T2.2 | M |
| T2.4 | LangGraph checkpoint backend | `backend/state/checkpoint.py` | T2.2 | M |
| T2.5 | state_version 単調増加 logic | `backend/state/state_version.py` | T2.3 | S |
| T2.6 | Artifact storage adapter (interface + GCS + local) | `backend/tools/artifact_store/` | T0.3 | L |
| T2.7 | `load_artifact` / `store_artifact` tools | `backend/tools/artifacts.py` | T2.6 | S |
| T2.8 | `get_artifact_lineage` tool | `backend/tools/artifacts.py` | T2.7 | S |
| T2.9 | `query_state_view` tool (caller identity 注入込み) | `backend/tools/state_views.py` | T2.3 | L |
| T2.10 | Deterministic SummaryViews 生成 | `backend/state/summary_views.py` | T2.3 | M |
| T2.11 | Working / Episodic / Project / Decision memory | `backend/memory/*.py` | T2.3 | M |
| T2.12 | Mock metadata catalog tools (`list_tables`, `get_table_summary`, `get_datamart_spec`) | `backend/tools/metadata.py` | T1.9 | M |
| T2.13 | State Manager service composition | `backend/services/state_manager.py` | T2.9, T2.10, T2.11 | M |
| T2.14 | `state_update_node` skeleton (write-path only) | `backend/graph/nodes/state_update.py` | T2.3 | M |
| T2.15 | `state_manager_refresh_node` skeleton | `backend/graph/nodes/state_manager_refresh.py` | T2.10 | S |

### 8.4 Notable tasks

#### T2.6 Artifact storage adapter

`tech_stack.md §1.4` の adapter 化を実装。

```python
class ArtifactStore(Protocol):
    async def put(self, artifact_id: str, data: bytes) -> str: ...  # storage_uri を返す
    async def get(self, storage_uri: str) -> bytes: ...
    async def signed_url(self, storage_uri: str, ttl: int) -> str: ...
```

- `GCSArtifactStore` (production)
- `LocalArtifactStore` (dev / test)
- artifact_id 採番は store 内で行う(client 側生成禁止)
- 大きい artifact は signed URL 経由(Phase 4 で必要)

#### T2.9 query_state_view tool

`backend_spec.md §7.2` および §9.8 を厳密実装。

- caller identity は FastAPI dependency または LangGraph runtime context から inject
- 自由文 query は受け付けない
- registered view + filters + fields の typed query のみ
- agent ごとの allowed view check (`team_directory.allowed_state_views`)
- response は `summary_text` (template-based) + structured data
- summary cache を `state_version` で invalidate

#### T2.11 Memory 4 層

`backend_spec.md §7.4` の bounded n=2 を含む実装。

- 更新主体は `state_update_node`(`backend/memory/README.md` 参照)
- `agent_episodic_memories` は `dict[agent_name, list[entry]]`、append 時に古いものを drop
- `working_memory` は phase gate / user review gate 通過時に regenerate
- 全 memory は GraphState の field として永続化

### 8.5 Exit criteria

- testcontainers Postgres でテストが pass
- `query_state_view` が typed query を deterministic に返す
- summary cache hit 率を計測できる(metric として export)
- `milestone_review_checklist.md` Milestone B の全項目クリア

---

## 9. Phase 3: Supervisor & Governance Skeleton

### 9.1 Goals

LLM 呼び出しを含む Supervisor loop と policy / governance tool 群の最小動線を作る。Specialist worker はまだ stub。

### 9.2 含まないもの

- 実 SQL / Python 実行(Phase 4)
- frontend(Phase 5)
- 高度な LLM provider(Vertex / Bedrock は Phase 6 以降)

### 9.3 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T3.1 | LLM Provider Protocol | `backend/llm/provider.py` | T1.9 | M |
| T3.2 | Anthropic adapter | `backend/llm/anthropic.py` | T3.1 | M |
| T3.3 | Fake LLM Provider (testing) | `backend/llm/fake.py` | T3.1 | S |
| T3.4 | LLM provider factory + tenant resolution | `backend/llm/factory.py` | T3.2 | S |
| T3.5 | Common prompt builder | `backend/prompts/common.py` | — | M |
| T3.6 | Routing Policy module | `backend/policies/routing.py` | T1.5 | M |
| T3.7 | Phase Gate Policy module | `backend/policies/phase_gate.py` | T1.5 | M |
| T3.8 | Advisor Consultation Trigger module | `backend/policies/advisor_consult.py` | T1.5 | S |
| T3.9 | classify_execution_risk module | `backend/policies/risk_classification.py` | T1.5 | M |
| T3.10 | evaluate_execution_readiness module | `backend/policies/execution_readiness.py` | T3.9 | M |
| T3.11 | QA Rulebook module | `backend/policies/qa_rulebook.py` | T1.5 | M |
| T3.12 | Override validation module | `backend/policies/override.py` | T1.5 | M |
| T3.13 | submit / validate override request tools | `backend/tools/governance.py` | T3.12 | S |
| T3.14 | classify / evaluate execution_risk tool wiring | `backend/tools/governance.py` | T3.10 | S |
| T3.15 | supervisor_node | `backend/graph/nodes/supervisor.py` | T3.5, T3.6, T3.7, T3.8 | L |
| T3.16 | dispatch_router_node | `backend/graph/nodes/dispatch_router.py` | T3.15 | M |
| T3.17 | phase_gate_node + completion_check_node | `backend/graph/nodes/{phase_gate,completion_check}.py` | T3.7 | M |
| T3.18 | advisor_consult_node + error_triage_node | `backend/graph/nodes/{advisor_consult,error_triage}.py` | T3.5 | M |
| T3.19 | Graph builder + edges (skeleton) | `backend/graph/builder.py`, `backend/graph/edges.py` | T3.15-T3.18 | M |
| T3.20 | Supervisor prompt | `backend/prompts/supervisor.py` | T3.5 | M |
| T3.21 | Advisor prompt | `backend/prompts/advisor.py` | T3.5 | S |
| T3.22 | Human Review Reasoning Trace schema + writer | `backend/observability/reasoning_trace.py`, `backend/observability/redaction.py` | T3.1, T3.5 | M |
| T3.23 | Required node trace emission (`supervisor`, `advisor_consult`, `phase_gate`, `user_review_gate`, `state_manager_refresh`, `error_triage`) | `backend/graph/nodes/*.py` | T3.15-T3.22 | M |

### 9.4 Notable tasks

#### T3.1 LLM Provider Protocol

`backend/llm/README.md` 参照。

```python
class LLMProvider(Protocol):
    async def complete(
        self, *,
        system: str,
        messages: list[Message],
        model: str,
        params: ModelParams,
        tools: list[ToolSchema] | None = None,
        response_format: type[BaseModel] | None = None,
    ) -> LLMResponse: ...
    async def stream(...) -> AsyncIterator[StreamEvent]: ...
```

- `prompt_version`, `model_id`, `params_hash` は呼び出し側が `AgentInvocationRecord` に書き込む
- tracing は base wrapper で自動化(Langfuse / OTel span は Phase 6)
- error は構造化例外として provider 層で raise

#### T3.6-T3.12 Policy modules

すべて pure function。共通テンプレ:

```python
POLICY_VERSION = "1.0.0"

class PolicyInput(BaseModel): ...
class PolicyOutput(BaseModel): ...

def evaluate(input: PolicyInput) -> PolicyOutput:
    # I/O・LLM・DB を呼ばない
    ...
```

- unit test 100% カバレッジ
- `POLICY_VERSION` を decision_log に必ず記録
- 入力に対して deterministic

#### T3.15 supervisor_node

最大 task。`backend_spec.md §15` の loop を実装。

- 各 step は明示関数として分割(`evaluate_pending_plans`, `check_advisor_consult`, `decide_dispatch`, etc.)
- LLM 呼び出しは structured output(Pydantic `response_format`)で固定
- policy module の判定結果を必ず参照、override は justified_override 形式
- decision_log への書き込みは return value で行う(state mutation は state_update_node 経由)
- Human Review Reasoning Trace を出力するが、trace は state に戻さない

#### T3.22-T3.23 Human Review Reasoning Trace

`backend_spec.md §16.5` に従い、人間レビュー専用の reasoning summary を出力する。

- chain-of-thought の全文保存はしない
- `input_summary`, `key_observations`, `decision_summary`, `alternatives_considered`, `tool_calls_*`, `policy_checks_referenced`, `risks_noted`, `open_questions`, `confidence` を structured に出す
- 出力は GraphState に保存しない
- 出力は agent の次入力に使わない
- redaction を必須にする

### 9.5 Exit criteria

- end-to-end でも graph が起動する(specialist は stub)
- supervisor が Fake LLM 経由で routing を返す
- policy version が decision に記録される
- justified override 不成立で policy が勝つテストが pass
- Milestone C の全項目クリア

---

## 10. Phase 4: Specialist Execution MVP

### 10.1 Goals

SQL / Python / QA / Causal の実 execution を通し、artifact が生成され、QA gate を通過するまでの流れを成立させる。**最初の DWH adapter は BigQuery、最初の sandbox は E2B**(`tech_stack.md §1.4`)。

### 10.2 含まないもの

- 全 DWH 対応(Snowflake / Redshift は Phase 6 以降)
- 全 sandbox 対応(Docker / Modal は Phase 6 以降)
- 高度な observability dashboard
- frontend との連動(Phase 5 と並列だが互いに stub で進める)

### 10.3 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T4.1 | emit_execution_plan / decide_execution_plan tools | `backend/tools/governance.py` | T3.13 | M |
| T4.2 | validate_sql tool (sqlglot ベース) | `backend/tools/sql_runtime.py` | T1.9 | M |
| T4.3 | DWH adapter Protocol + BigQuery adapter | `backend/tools/sql_runtime/` | T2.6 | L |
| T4.4 | estimate_sql_cost tool (BigQuery dry-run) | `backend/tools/sql_runtime.py` | T4.3 | M |
| T4.5 | run_sql tool (preflight + postflight) | `backend/tools/sql_runtime.py` | T4.2, T4.3, T4.4 | L |
| T4.6 | run_data_sanity_check tool | `backend/tools/sql_runtime.py` | T4.5 | M |
| T4.7 | Sandbox adapter Protocol + E2B adapter | `backend/tools/python_runtime/` | — | L |
| T4.8 | run_python tool (preflight + postflight) | `backend/tools/python_runtime.py` | T4.7 | L |
| T4.9 | specialist_worker_node (共通) | `backend/graph/nodes/specialist_worker.py` | T3.16 | L |
| T4.10 | SQL Specialist prompt + flow | `backend/prompts/sql.py` | T4.5, T4.9 | M |
| T4.11 | Python Specialist prompt + flow | `backend/prompts/python.py` | T4.8, T4.9 | M |
| T4.12 | run_qa_stage / evaluate_qa_gates / mark_artifact_quality_stage | `backend/tools/qa.py` | T3.11 | M |
| T4.13 | QA Specialist 3-stage prompt + flow | `backend/prompts/qa.py` | T4.12 | L |
| T4.14 | Scoping Specialist prompt | `backend/prompts/scoping.py` | T3.5 | M |
| T4.15 | Metric Definition Specialist prompt | `backend/prompts/metric_definition.py` | T3.5 | M |
| T4.16 | Causal Specialist prompt + flow | `backend/prompts/causal.py` | T4.8, T4.9 | M |
| T4.17 | Insight Specialist prompt | `backend/prompts/insight.py` | T3.5 | M |
| T4.18 | Business Recommendation prompt | `backend/prompts/business_recommendation.py` | T3.5 | M |
| T4.19 | search_metric_registry / search_past_findings (pgvector) | `backend/tools/knowledge.py` | T2.6 | L |
| T4.20 | record_provisional_mode / resolve_user_wait_policy | `backend/tools/fallback.py` | T1.9 | S |
| T4.21 | user_review_gate_node | `backend/graph/nodes/user_review_gate.py` | T3.19 | M |
| T4.22 | intake_node + requirement_clarification_node | `backend/graph/nodes/{intake,requirement_clarification}.py` | T3.19 | M |

### 10.4 Notable tasks

#### T4.5 run_sql with safety layer

`backend_spec.md §19.4` をそのまま実装。preflight 6 step + postflight 5 step + plan_vs_actual_diff 生成。tool 内部で物理強制し、prompt に頼らない。

Preflight:

1. `plan_status == "approved"` 確認 → 拒否時は tool_error
2. `validate_sql` 通過確認
3. plan 記載の table/column 範囲内か(allowlist 整合)
4. `estimate_sql_cost` 実行
5. 必要なら preview (LIMIT)
6. 本実行可否の最終判定

Postflight:

1. result schema 検証
2. sanity_artifact 生成
3. metric_definition_sheet 照合(該当 task)
4. data_snapshot_id 確定
5. plan_vs_actual_diff 生成

#### T4.7 Sandbox adapter

```python
class SandboxAdapter(Protocol):
    async def run(
        self, *,
        entrypoint: str,
        input_artifacts: list[ArtifactRef],
        timeout: int,
        seed: int,
    ) -> SandboxResult: ...
```

- E2B 実装は session per task 方式
- input artifact のダウンロード / 出力 artifact のアップロードは adapter 内
- random_seed 固定とログ記録
- libraries の version 記録
- timeout / OOM の扱いは構造化エラーで返す

#### T4.13 QA Specialist 3-stage

`backend_spec.md §11.3` 厳密実装。

- 3 stage それぞれ独立 plan(stage ごとに `emit_execution_plan` / `decide_execution_plan`)
- 上流 stage block 時は下流 plan を emit しない
- `stage_status` は `QAStageStatus` Literal で管理
- artifact_quality_stage の遷移は本フローからのみ可能

### 10.5 Exit criteria

- 1 案件(sample データ)を end-to-end で完了させられる
  - Scoping → Metric Definition → SQL → QA → Insight → Recommendation → Delivery
- すべての artifact が citation 付き
- Plan-Execute Contract が 4 種 specialist で機能する
- QA 3 stage の依存関係が enforce される
- Milestone D の全項目クリア

---

## 11. Phase 5: Frontend Workspace MVP

### 11.1 Goals

`frontend_spec.md §19` 優先度順に project workspace を実装。MSW で backend なしでも動く状態を経由してから backend 接続。

### 11.2 含まないもの

- 高度な collaboration
- chart annotation
- mobile 完全対応(`frontend_spec.md §16` の review/承認のみ確保)

### 11.3 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T5.1 | Next.js shell + Auth.js setup | `frontend/app/(auth)/`, `frontend/app/api/auth/[...nextauth]/route.ts` | T0.2 | L |
| T5.2 | Tanstack Query setup + API client | `frontend/lib/{api-client,query-client}.ts` | T0.5 | M |
| T5.3 | SSE client wrapper | `frontend/lib/sse-client.ts` | T5.2 | M |
| T5.4 | MSW handler skeleton (per `frontend_spec.md §11`) | `frontend/mocks/handlers/`, `frontend/mocks/browser.ts`, `frontend/mocks/server.ts` | T0.5 | M |
| T5.5 | Workspace shell + project nav | `frontend/components/layout/workspace-shell.tsx`, `frontend/components/layout/project-nav.tsx` | T5.1 | M |
| T5.6 | Project home page | `frontend/app/(workspace)/projects/[projectId]/page.tsx`, `frontend/features/home/` | T5.5 | M |
| T5.7 | Pending review banner (global) | `frontend/components/layout/pending-review-banner.tsx` | T5.2 | M |
| T5.8 | Conversation feature module | `frontend/features/conversation/` | T5.3 | L |
| T5.9 | Review center + review form | `frontend/features/reviews/` | T5.7 | L |
| T5.10 | Artifact list view | `frontend/features/artifacts/list.tsx` | T5.2 | M |
| T5.11 | Artifact detail with extensible renderer registry | `frontend/features/artifacts/detail.tsx`, `frontend/features/artifacts/renderer/` | T5.10 | L |
| T5.12 | Document renderer | `frontend/features/artifacts/renderer/document.tsx` | T5.11 | S |
| T5.13 | Table preview renderer | `frontend/features/artifacts/renderer/table.tsx` | T5.11 | M |
| T5.14 | Chart gallery renderer | `frontend/features/artifacts/renderer/chart.tsx` | T5.11 | M |
| T5.15 | Issue list / report outline / json renderers | `frontend/features/artifacts/renderer/{issue,report,json}.tsx` | T5.11 | M |
| T5.16 | Artifact lineage view | `frontend/features/artifacts/lineage.tsx` | T5.11 | M |
| T5.17 | Timeline view | `frontend/features/timeline/` | T5.2 | M |
| T5.18 | Governance panel | `frontend/features/governance/` | T5.2 | M |
| T5.19 | Memory / focus panel | `frontend/features/governance/memory.tsx` | T5.18 | S |
| T5.20 | Deliverables view + sign-off | `frontend/features/deliverables/` | T5.2 | M |
| T5.21 | Quality stage / risk / phase / override / provisional badges | `frontend/components/domain/` | T5.5 | M |
| T5.22 | Citation links | `frontend/components/domain/citation-link.tsx` | T5.11 | S |

### 11.4 Notable tasks

#### T5.1 Next.js shell + Auth.js

- App Router で `(auth)` と `(workspace)` を route group で分離
- Auth.js を Postgres adapter で構成(Phase 2 の DB に同居)
- session 取得は server component で行い、client にトークン直接露出しない
- middleware で未認証は `/login` redirect

#### T5.3 SSE client wrapper

`frontend_spec.md §12.2.1` の SSE event を購読。

- EventSource API ラップ(再接続 + auth header injection)
- event type → callback dispatcher
- Zustand store に接続状態を反映
- project 切替で reconnect

#### T5.11 Extensible renderer registry

artifact_type に応じて renderer を dispatch する registry pattern。

```tsx
const renderers: Record<ArtifactType, Component<RendererProps>> = {
  document: DocumentRenderer,
  table_preview: TableRenderer,
  chart_gallery: ChartRenderer,
  issue_list: IssueListRenderer,
  report_outline: ReportRenderer,
  json: JsonRenderer,
};
```

新 artifact_type 追加時は registry に登録するだけで対応。

### 11.5 Exit criteria

- MSW のみで全 page が表示できる
- 実 backend との接続で同じ画面が動作する
- pending review が banner / conversation / review center の 3 箇所に同期表示される
- quality_stage / provisional / risk が視覚的に区別される
- citation link が source artifact + region に飛ぶ
- Milestone E の全項目クリア

---

## 12. Phase 6: Hardening

### 12.1 Goals

observability、failure handling、security、tenant isolation を強化し、本番運用に耐える状態にする。

### 12.2 Tasks

| ID | Task | Files | Deps | Size |
|---|---|---|---|---|
| T6.1 | OpenTelemetry tracing 統合 | `backend/observability/otel.py` | All | M |
| T6.2 | Langfuse LLM trace integration | `backend/llm/provider.py` (wrapper 拡張) | T3.1 | M |
| T6.3 | Sentry error tracking | `backend/observability/sentry.py`, `frontend/lib/sentry.ts` | All | S |
| T6.4 | structlog JSON logging config | `backend/observability/logging.py` | All | S |
| T6.5 | Retry policy implementation (`max_retries` enforcement) | `backend/services/retry.py` | All | M |
| T6.6 | Degraded mode flow (provisional + pause + escalation) | `backend/services/degraded_mode.py` | All | M |
| T6.7 | check_access_scope tool | `backend/tools/access.py` | All | M |
| T6.8 | Tenant isolation (DB row-level) | migration + `backend/services/tenant_filter.py` | T2.2 | L |
| T6.9 | PII redaction layer | `backend/services/redaction.py` | T6.8 | M |
| T6.10 | Policy version registry | `backend/policies/registry.py` | All policies | S |
| T6.11 | Cost monitoring per tenant | `backend/observability/cost.py` | T6.2 | M |
| T6.12 | Health check / readiness endpoints | `backend/api/health.py` | All | S |
| T6.13 | Backup / restore runbook | `docs/operations/backup.md` | T2.2 | S |
| T6.14 | Reasoning trace retention / access policy | `docs/operations/reasoning_trace_policy.md`, `backend/observability/reasoning_trace.py` | T3.22 | S |

### 12.3 Notable tasks

#### T6.8 Tenant isolation

- 全テーブルに `tenant_id` カラム追加(migration)
- SQLAlchemy event listener で auto-filter
- API dependency で JWT claim → tenant_id resolution
- 別テナントのデータが漏洩しないことを assert する明示テスト
- pgvector の similarity 検索も tenant 内に閉じる

#### T6.6 Degraded mode

`backend_spec.md §18.3` 実装。

- `record_provisional_mode` で前提と mitigation を残す
- `resolve_user_wait_policy` で pause / remind / provisional_continue を判定
- provisional artifact が final delivery に混ざらないことを enforce
- frontend banner 表示と連動

### 12.4 Exit criteria

- Langfuse で LLM トレースが追える
- Cloud Trace で span chain が辿れる
- Sentry でエラー alert が動く
- 別テナントへのアクセスが access_error で reject される
- PII redaction を通過した payload に PII が残らないテストが pass
- Milestone F の全項目クリア

---

## 13. Cross-cutting Concerns

### 13.1 命名規約

- Python: `snake_case` (function, var), `PascalCase` (class), `UPPER_SNAKE` (const)
- TypeScript: `camelCase` (function, var), `PascalCase` (component, type), `UPPER_SNAKE` (const)
- DB column: `snake_case`
- API path: `kebab-case`
- Pydantic field: `snake_case`(alias で camelCase 公開はしない、generate 側で対応)

### 13.2 Logging 規約

- 全 log は JSON format(structlog)
- 必須 field: `trace_id`, `tenant_id`, `event`, `level`
- LLM 呼び出しは別途 Langfuse、application log には summary のみ
- secret や PII は logger で除去(`redaction.py` 経由)

### 13.3 Error 規約

- backend service 層は構造化例外を raise(`backend/services/errors.py`)
- API 層が HTTP status + error envelope に翻訳
- frontend は error envelope の `error_type` で UI 分岐
- tool error は `tool_error_contract` 準拠(`backend_spec.md §9.9`)

### 13.4 Migration 規約

- migration ID は連番(`0001_initial.py`, `0002_*`)
- 破壊的変更は別 PR、必ず "なぜ" を commit message に
- production migration は手動承認 step を CI に挟む

### 13.5 Feature flag

- MVP では feature flag 不要(1 production tenant 想定)
- 多テナント化時に LaunchDarkly か内製 feature_flags テーブルを検討

### 13.6 Documentation の責務

- 仕様変更は `*_spec.md` を更新、本 plan も追従
- 用語追加は `backend_spec.md` Appendix: Canonical Terms を更新
- 技術判断変更は `tech_stack.md` を更新
- これら 3 ファイルは正本、他は派生情報

---

## 14. Coding Agent Workflow

### 14.1 タスク受領

1. project lead が task 表から 1 task 選択(deps を満たすもの)
2. `task_brief_template.md` から brief を composing
3. agent に依頼

### 14.2 実装

1. agent は brief を読み、関連仕様を読む
2. 不明点があれば仕様で曖昧と判明した時点で **質問する(推測しない)**
3. 実装 + テスト
4. self-review(`milestone_review_checklist.md` 該当 milestone)

### 14.3 完了報告

完了報告には以下を含める:

- 変更ファイル一覧
- pytest / vitest 結果
- mypy / typescript 結果
- 仕様で曖昧だった箇所の独自判断(あれば)
- 関連する Mandatory Red Flags への自己評価

### 14.4 Review

- human reviewer が `milestone_review_checklist.md` で確認
- Mandatory Red Flags にヒットしたら差し戻し
- 必要な軽微修正があれば agent に return brief

### 14.5 Merge

- CI pass
- review approve
- merge to main
- 次 task に進む

---

## 15. Testing Strategy

本プロダクトのテスト戦略は implementation の一部として扱う。各 phase で「どこまで作るか」と同じくらい、「どこまで壊さず確認するか」を明示する。

### 15.1 Test Layers

backend:

| Type | Purpose | Tooling |
|---|---|---|
| Unit | pure 関数・schema・policy module | pytest |
| Service | service layer + DB | pytest + testcontainers (Postgres) |
| Tool | tool I/O 契約 | pytest |
| Graph | LangGraph node 単体・小規模グラフ実行 | pytest + pytest-asyncio |
| API | FastAPI endpoint | httpx + pytest |
| E2E | 全レイヤー通し | pytest + 軽量 LLM スタブ |

frontend:

| Type | Purpose | Tooling |
|---|---|---|
| Unit | pure 関数・hook | Vitest |
| Component | UI コンポーネント | Vitest + React Testing Library |
| Integration | feature module + mock API | Vitest + MSW |
| E2E | ユーザーフロー | Playwright |

### 15.2 LLM Mocking Policy

LLM 呼び出しのテストは以下の 3 層で扱う。

1. deterministic stub: prompt → 固定 response の dict 引き当て
2. recorded fixtures: 実 LLM レスポンスの記録再生
3. live LLM eval: nightly / weekly のみ

ベースルール:

- `LLMProvider` interface を抽象化する
- `FakeLLMProvider` を test fixture として用意する
- マッチしない prompt は silent fallback せず test failure にする
- recorded fixture は redact ルールを通してから保持する

### 15.3 Infra-Dependent Tests

- Postgres 系テストは sqlite で代替しない
- testcontainers で pgvector 付き Postgres を起動する
- migration は Alembic を通す
- SSE は FastAPI test client で順序付き assert を行う
- frontend mock は `frontend_api_mock.json` を起点に MSW で提供する

### 15.4 CI Test Stages

| Stage | Scope |
|---|---|
| pre-commit | ruff format / lint, eslint, biome |
| PR build | unit + service + tool + graph(stub LLM) + frontend unit + component + integration |
| Nightly | E2E + recorded LLM playback + eval |
| Weekly | live LLM eval + cost report |

### 15.5 Coverage Targets

- Pydantic schema: 100%
- policy module: 100%
- service layer: 80%+
- API: 80%+
- graph: 70%+
- tool: 80%+
- frontend: 70%+

### 15.6 Test Anti-Patterns

- LLM 呼び出しを mock せず unit test に含める
- Postgres 統合テストを sqlite で代替する
- snapshot test を盲信する
- `time.sleep` で flaky を誤魔化す
- test 内で実 DWH を叩く

### 15.7 Exceptional Test Cases

- justified override の invalid shape を網羅する
- plan-execute round-trip を integration test で通す
- tenant isolation を明示的に検証する
- PII masking 後の payload に機密が残らないことを検証する
- Human Review Reasoning Trace が state に混入せず、agent input に再利用されないことを検証する

---

## 16. Human Review Gates

`milestone_review_checklist.md` Milestone A-F に従う。

加えて、以下のタイミングで強制 review:

- 仕様破壊変更前(例: GraphState フィールド削除)
- 認証 / アクセス制御変更前
- Migration 適用前
- 新 LLM provider adapter 追加前
- production deploy 前

これらは `task_brief_template.md` の `触ってはいけない` で agent に明示する。

---

## 17. Dependency Map (concise)

phase 横断の主要依存:

```text
T0.1 (backend skeleton) ──┬→ T1.1 (Literals) ──┬→ T1.2-T1.7 (models) ─→ T1.8 (GraphState) ─┐
                          │                    │                                            │
T0.5 (codegen)            └→ T1.9 (tool I/O)   └→ T1.10 (API schema) ─→ T1.12 (mock align) │
                                                                                            ↓
                                                                                          T2.1 (DB)
                                                                                            │
                                                                                            ↓
                                                                                       T2.2-T2.15
                                                                                            │
                                              ┌──────────────────────┬───────────────────────┘
                                              ↓                      ↓
                                         T3.1 (LLM)              T3.6-T3.12 (policies)
                                              │                      │
                                              └──→ T3.15 (supervisor_node) ←┘
                                                          │
                                                          ↓
                                                     T3.19 (graph builder)
                                                          │
                          ┌───────────────────────────────┼─────────────────────────────────┐
                          ↓                               ↓                                 ↓
                     T4.1-T4.6 (SQL)                T4.7-T4.8 (Python)              T5.x (frontend MVP)
                          │                               │
                          └──────────────┬────────────────┘
                                         ↓
                                  T4.9-T4.22 (specialists)
                                         │
                                         ↓
                                       Phase 6
```

---

## 18. Exit Criteria for v1.0

- Phase 0-6 完了
- 1 案件サンプルを end-to-end で完走
- Langfuse でトレース可視化
- 1 customer-facing tenant が PoC として稼働
- `docs/customer/sales_pitch.md` の主要訴求(品質保証、再現性、監査性)が demo で見せられる
- `docs/customer/onboarding_checklist.md` の全項目に答えられる

## 19. Post v1.0 (Roadmap)

- Snowflake / Redshift / Databricks DWH adapter
- AWS / Azure cloud adapter
- Vertex AI / Bedrock LLM adapter
- on-prem deploy package
- multi-user collaboration (`frontend_spec.md §18` open questions)
- chart annotation
- mobile full UX
- automatic eval harness (`implementation_plan.md §15.2` 実運用)
- multi-project ベンチマーク(opt-in)
