# Implementation Plan

## 1. Goal

本ドキュメントは、`backend_spec.md`、`frontend_spec.md`、`frontend_api_mock.json` を実装に落とすための段階的な開発計画を定義する。目的は、長期タスクを coding agent に安全に委譲できる粒度へ分解し、人間レビューを milestone ごとに挟めるようにすることである。

## 2. Planning Principles

- 各 phase で「何を作るか」と同時に「何をまだ作らないか」を明記する
- 早い段階で typed schema と mock contract を固める
- 早い段階で end-to-end の最小経路を通す
- governance と QA は後付けではなく、早期から骨格を入れる
- human review は phase 完了時ではなく、破壊的変更の前に入れる

## 3. Phase Plan

### Phase 0. Workspace Bootstrap

目的:
実装の土台となる repo 構成、開発環境、基本 lint / format / test 動線を整える。

含むもの:

- backend / frontend のディレクトリ雛形
- package / dependency 管理の初期化
- 共通 lint / format / type-check 設定
- CI の最小セットアップ

含まないもの:

- business logic
- graph orchestration
- 本番接続

レビュー観点:

- ディレクトリ責務が明確か
- backend / frontend の開発開始が詰まらないか

### Phase 1. Schema and Contract Foundation

目的:
state schema、API projection、artifact contract、review contract を先に固定する。

含むもの:

- `GraphState` 系 Pydantic モデルの初版
- tool I/O 用 schema 定義
- frontend projection schema
- `frontend_api_mock.json` と整合する API contract

含まないもの:

- 実際の tool 実装
- graph 実行

成果物:

- schema モジュール
- JSON Schema / OpenAPI の初版
- mock との整合テスト

レビュー観点:

- schema が仕様とズレていないか
- `Field(description=...)` が主要項目に入っているか
- frontend projection が backend 生 state に依存しすぎていないか

### Phase 2. Artifact and State Infrastructure

目的:
artifact registry、state persistence、State Manager の read-path を成立させる。

含むもの:

- artifact registry
- state persistence layer
- `query_state_view` の MVP
- summary / projection の deterministic 実装
- working memory / episodic memory 更新骨格

含まないもの:

- LLM 実行本体
- policy tool の高度化

成果物:

- `query_state_view`
- artifact CRUD
- summary view 生成

レビュー観点:

- State Manager が recommendation を返していないか
- caller identity が正しく注入されるか
- memory 更新責務が state update 経路に収まっているか

### Phase 3. Supervisor and Governance Skeleton

目的:
Supervisor loop と governance tool 群の最小動線を作る。

含むもの:

- `supervisor_node`
- `dispatch_router_node`
- `state_update_node`
- `evaluate_routing_policy`
- `evaluate_phase_gate`
- `evaluate_advisor_consult_trigger`
- `classify_execution_risk`
- `evaluate_execution_readiness`
- override request / validation の最小実装

含まないもの:

- SQL / Python の実 execution
- 高度な UI

成果物:

- policy tool 群
- decision log 動線
- justified override の最小 enforcement

レビュー観点:

- Supervisor が無条件 override できないか
- mandatory consult が deterministic に発火するか
- policy version が decision に残るか

### Phase 4. Specialist Execution MVP

目的:
SQL / Python / QA の最小実行経路を通す。

含むもの:

- `emit_execution_plan`
- `decide_execution_plan`
- `run_sql`
- `run_python`
- `run_qa_stage`
- `evaluate_qa_gates`
- `mark_artifact_quality_stage`
- provisional mode 記録

含まないもの:

- 高度な因果分析
- 完全な observability ダッシュボード

成果物:

- plan -> approval routing -> execute -> artifact 保存 -> QA gate の経路

レビュー観点:

- low-risk delegated approval と high-risk supervisor review が分かれているか
- QA stage ごとに block / skip が動くか
- provisional artifact が final に混ざらないか

### Phase 5. Frontend Workspace MVP

目的:
project workspace、review center、artifact workspace を mock 駆動で実装する。

含むもの:

- project home
- conversation screen
- review center
- artifact workspace
- timeline / governance summary

含まないもの:

- 高度な collaboration
- 本格的な chart annotation

成果物:

- frontend MVP
- backend projection API との接続

レビュー観点:

- pending review が見落とされにくいか
- draft / provisional / final が明確に分かるか
- override / advisor / governance 情報が過不足なく見えるか

### Phase 6. End-to-End Hardening

目的:
observability、error handling、security、tenant isolation を強化する。

含むもの:

- metrics / tracing
- retry / degraded mode policy
- `check_access_scope`
- tenant / access enforcement
- policy store の versioning

成果物:

- observability 指標
- access / policy hardening

レビュー観点:

- failure 時の縮退運転が仕様通りか
- sensitive data が mask されるか
- override や provisional mode が監査できるか

## 4. Recommended Delivery Slices

coding agent に渡すタスクは以下のような単位に切る。

- `GraphState` と関連 Pydantic モデルを実装する
- `query_state_view` の deterministic projection を実装する
- artifact registry と load / store API を実装する
- Supervisor policy tool 3 点を実装する
- execution risk / readiness 判定 tool を実装する
- override request / validation tool を実装する
- QA stage 実行と gate 判定を実装する
- frontend の review center を mock ベースで実装する
- artifact workspace を mock ベースで実装する

各 slice は 1 turn でレビュー可能な大きさに抑える。

## 5. Human Review Gates

人間レビューを最低限入れるタイミング:

1. schema 初版完成後
2. State Manager read-path MVP 完成後
3. governance / override skeleton 完成後
4. first end-to-end execution 経路完成後
5. frontend MVP 完成後
6. security / tenant isolation hardening 前

## 6. Exit Criteria

各 phase は少なくとも以下を満たしたら次へ進める。

- 明示された成果物が揃っている
- 対応する review checklist を通過している
- 次 phase の前提が実装済みである
- mock / schema / implementation の間で重大不整合がない
