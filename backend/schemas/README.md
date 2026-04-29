# backend/schemas

Pydantic schema の正本。[`../../backend_spec.md`](../../backend_spec.md) §14.2 および [`../../system_design.md`](../../system_design.md) §5.3 参照。

## ファイル(予定)

- `state.py` — GraphState 関連の全モデル(`backend_spec.md §14.2` の Typed State Sketch)
- `api.py` — API request / response モデル(`frontend_spec.md §11` 対応)
- `tools.py` — tool I/O モデル(`backend_spec.md §9.8`)
- `events.py` — SSE event の discriminated union
- `enums.py` — Literal 型の集約(必要に応じて分割)

## 規約

- Pydantic v2 strict mode
- Literal は top-level に集約(inline Literal 禁止)
- 主要 field に `Field(description=...)` を付与(OpenAPI 反映)
- `examples` を `Field` に含めて API docs を充実させる
- 手動編集のみ。本層は **コード生成元として正本**
- frontend/lib/contracts.ts は本層から自動生成される(`../../system_design.md` §5)

## 主要モデル一覧

`backend_spec.md §14.2` の以下を network-port 化する。

- AuditFields, TeamMember, ProjectContext
- OpenQuestion, WorkingMemory, AgentEpisodicMemoryEntry
- RecommendationCandidate, RecommendationSet
- TaskInputBundle, TaskOutputBundle, TaskRecord
- ArtifactRef
- ToolCallRecord
- AgentIOEnvelope, AgentResultEnvelope, AgentInvocationRecord
- AdvisorNote, SupervisorDecision
- QAReviewRecord
- UserReviewRequest, UserInteractionRecord
- PhaseTransitionRecord, SummaryCacheEntry
- RuntimeRegistry, DataCatalogRefs, SummaryViews
- FinalOutputRecord, GraphState

## 注意点

- `TaskRecord.risk_level` は固定属性ではなく **最新 plan の risk 評価**(`backend_spec.md §13.4` 注記)
- `ArtifactRef.data_snapshot_id` は dataset 系で必須
- `AgentInvocationRecord` は `prompt_version`, `model_id`, `random_seed` 等を必ず記録
