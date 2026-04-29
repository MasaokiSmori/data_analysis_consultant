# backend/state

GraphState 永続化と LangGraph checkpoint。[`../../backend_spec.md`](../../backend_spec.md) §14、[`../../tech_stack.md`](../../tech_stack.md) §1.4 参照。

## ファイル(予定)

- `store.py` — GraphState の load / save
- `checkpoint.py` — LangGraph checkpoint backend(Postgres 同居)
- `state_version.py` — state_version の単調増加管理
- `summary_cache.py` — SummaryCacheEntry 管理(`backend_spec.md §7.3`)
- `mutations.py` — state mutation primitives(append-only / supersede / etc.)
- `migrations/` — Alembic migration スクリプト

## 規約

- artifact 本体は state に乗せない(storage_uri のみ)
- write-path は **`state_update_node` 経由のみ**(`backend_spec.md §14.3`)
- read-path は `query_state_view` tool 経由(全 agent から)
- checkpoint backend は `langgraph-checkpoint-postgres`(同じ DB に同居)
- transaction boundary は service 層で管理

## state_version の使い方

- write のたびに +1
- summary cache の invalidation key として使う
- ToolCallRecord に `state_version_at_call` を記録(再現性のため)

## 注意点

- 並列 specialist worker からの結果を merge するときは、必ず `state_update_node` 1 箇所に集約
- artifact_id 採番は中央(artifact_store_tool 側)で一意化、client 側生成禁止
