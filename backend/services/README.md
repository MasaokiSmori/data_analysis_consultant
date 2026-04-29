# backend/services

API ↔ graph / state / tool の橋渡し service 層。stateful、DI で組む。

## ファイル(予定)

- `projects.py` — プロジェクト CRUD + 起動
- `conversation.py` — message 永続化 + graph dispatch
- `reviews.py` — user review 受付・配信
- `artifacts.py` — artifact 取得・lineage
- `governance.py` — risk / readiness / override projection
- `tenants.py` — テナント resolution、テナント設定
- `secrets.py` — Secret Manager adapter
- `auth.py` — auth provider 抽象化
- `errors.py` — エラー正規化
- `delegated_approval.py` — delegated approval component(`backend_spec.md §6.5`)

## 規約

- DB session は dependency で受ける(直接 import 禁止)
- LLM provider は dependency で受ける
- artifact store は adapter 経由(`tools/artifacts.py` を経由)
- 例外は `errors.py` の正規化型で投げ、API 層で HTTP status へ写像
- service は state を直接書かない、必ず `state_update_node` 経由

## 抽象化境界

[`../../tech_stack.md`](../../tech_stack.md) §5 の抽象化境界に従う。`secrets.py`, `auth.py` 等は cloud provider 切替に対応する。
