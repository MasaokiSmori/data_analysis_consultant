# backend/api

FastAPI router 群。エンドポイント定義 + バリデーション + service 呼び出しのみを担う。

## ファイル分割(予定)

- `projects.py`
- `conversation.py`
- `reviews.py`
- `artifacts.py`
- `timeline.py`
- `governance.py`
- `deliverables.py`
- `admin.py`
- `auth.py`
- `dependencies.py`(共通 dependency: 認証、テナント解決、DB session)

エンドポイント一覧は [`../../frontend_spec.md`](../../frontend_spec.md) §11 を参照。

## 規約

- ビジネスロジックは services 層、本層は dispatch のみ
- request / response は [`../schemas/`](../schemas/) の Pydantic モデルを必ず利用
- 認証 / テナント解決は FastAPI dependency で共通化(`dependencies.py`)
- `response_model=` を必ず指定し、暗黙的 dict 化を避ける
- error は HTTPException ではなく、`backend/services/errors.py` で正規化されたものを使う
