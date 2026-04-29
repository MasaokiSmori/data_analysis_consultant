# backend/memory

`../../backend_spec.md` §7.4 の memory model を実装。

## ファイル(予定)

- `working_memory.py` — phase gate / user review gate 通過時に再生成
- `episodic_memory.py` — agent ごと bounded n=2、state_update 経路で更新
- `project_memory.py` — 案件恒久前提(用語、目的、制約、合意済み定義)
- `decision_memory.py` — 重要判断の永続層
- `refresh.py` — 各 memory の更新タイミング集約

## 規約

- agent の私有長期メモにしない(常に state 経路で管理)
- state 経由でのみ更新する
- 直接 DB 編集禁止(必ず service / state_update_node を経由)
- agent_episodic_memory は bounded(n=2)、`refresh.py` で enforce

## 更新タイミング

| memory | 更新主体 | タイミング |
|---|---|---|
| project_memory | state_update_node | project 作成時、user review approve 時 |
| decision_memory | state_update_node | SupervisorDecision 生成時 |
| working_memory | state_manager_refresh_node | phase gate / user review gate 通過時 |
| episodic_memory | state_update_node | specialist invocation 完了時 |
