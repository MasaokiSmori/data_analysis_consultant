# backend/graph

LangGraph orchestration。[`../../backend_spec.md`](../../backend_spec.md) §13 の graph shape を実装する。

## ファイル(予定)

- `builder.py` — graph 構築のエントリポイント
- `edges.py` — 条件分岐ロジック
- `subgraphs/` — specialist worker subgraph
- `nodes/` — 各 node 実装(下記)

## Node 一覧

[`../../backend_spec.md`](../../backend_spec.md) §13.3 参照。1 node = 1 ファイル を原則とする。

- `intake.py`
- `requirement_clarification.py`
- `user_review_gate.py`
- `state_manager_refresh.py`
- `supervisor.py`
- `dispatch_router.py`
- `specialist_worker.py`(共通 worker、prompt と allowed tools を切り替え)
- `state_update.py`
- `phase_gate.py`
- `completion_check.py`
- `advisor_consult.py`
- `error_triage.py`

## 規約

- node は thin に保ち、重いロジックは [`../services/`](../services/) [`../tools/`](../tools/) [`../policies/`](../policies/) に逃がす
- node のシグネチャは `(state) -> dict[update]`
- side effect は service 経由のみ
- state mutation は `state_update.py` のみが行う
- streaming は `astream_events` 経由で SSE に乗せる(`../api/conversation.py` の `/stream` endpoint)
- node 内で LLM を直接呼ぶ場合も、`../llm/` の adapter 経由にする(直接 SDK 呼び出し禁止)

## Plan-Execute 対象 Specialist の流れ

SQL / Python / Causal / QA は specialist_worker_subgraph 内で plan emit → supervisor approval → execute の 1 往復が発生する。詳細は [`../../backend_spec.md`](../../backend_spec.md) §13.2、§17.4。
