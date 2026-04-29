# backend/tests

pytest 構成。[`../../implementation_plan.md`](../../implementation_plan.md) §15 参照。

## ディレクトリ構成(予定)

```text
tests/
├── conftest.py                # global fixture
├── unit/
│   ├── schemas/
│   ├── policies/
│   └── memory/
├── service/
│   ├── projects/
│   ├── reviews/
│   └── governance/
├── tool/
│   ├── governance/
│   ├── qa/
│   └── runtime/
├── graph/
│   ├── nodes/
│   └── subgraphs/
├── api/
│   └── e2e/
└── fixtures/
    ├── llm/                   # 録画 LLM レスポンス(redacted)
    ├── catalog/               # mock スキーマカタログ
    └── sql/                   # 期待 SQL サンプル
```

## 共通 fixture (`conftest.py`)

- `db_session`: testcontainers Postgres + transaction rollback
- `fake_llm`: deterministic LLM stub
- `fake_artifact_store`: in-memory artifact store
- `fake_dwh`: 期待結果を返す DWH stub
- `fake_sandbox`: Python sandbox stub

## 規約

- 実 LLM を叩くテストは `@pytest.mark.live_llm` でマーク、PR では走らない
- 実 DWH を叩くテストは作らない(stub のみ)
- testcontainers で Postgres 起動(pgvector 拡張込み: `pgvector/pgvector:pg16`)
- LLM 録画 fixture は redact 済みのみ commit

## マーカー

- `unit`: 副作用なし、高速
- `service`: DB あり
- `graph`: LLM stub あり
- `slow`: 5 秒以上
- `live_llm`: 実 LLM(nightly のみ)
- `integration`: 複数 service にまたがる

CI 設定例:
- PR: `pytest -m "not live_llm and not slow"`
- Nightly: `pytest -m "not live_llm" --slow`
- Weekly: `pytest -m live_llm`
