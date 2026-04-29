# backend/policies

pure-function policy module 群。[`../../backend_spec.md`](../../backend_spec.md) §13.4 および [`../../system_design.md`](../../system_design.md) §3.3 参照。

## ファイル(予定)

- `routing.py` — Routing Policy Engine: routing 提案
- `phase_gate.py` — Phase Gate Evaluator: フェーズ遷移可否判定
- `execution_readiness.py` — Plan Reviewer + Execution Readiness Policy: plan の risk 分類 + 承認分岐
- `risk_classification.py` — `classify_execution_risk` 実装
- `advisor_consult.py` — Advisor Consultation Trigger: mandatory consult 判定
- `qa_rulebook.py` — QA gate / checklist evaluator
- `override.py` — justified override の validation

## 規約

**絶対に守るべきルール:**

- I/O・副作用なし(DB / LLM / 外部サービス呼び出し禁止)
- 入出力は strict Pydantic schema
- `POLICY_VERSION` 定数で version 化(SemVer 推奨)
- 同一入力に対して必ず同一出力を返す(deterministic)
- override されても version は decision に記録される
- unit test 100% カバレッジ目標(pure function なので落とせない)

## 呼び出し元

policy module は主に以下から呼ばれる。

- `supervisor_node`(routing / phase_gate / advisor_consult)
- `phase_gate_node`(phase_gate)
- specialist worker subgraph(execution_readiness は plan emit 後に呼ばれる)
- governance service(API 経由の手動評価)

policy module 自身が他 policy を呼ぶことは可能(pure 同士なので副作用は発生しない)。

## バージョニング

policy 変更時は `POLICY_VERSION` を必ず bump する。decision_log に記録される version を見れば、どの policy で判定されたかが事後追跡できる。

policy 本体の物理形式は MVP では Python module で十分。将来 OPA (Rego) 等に切り出す可能性はあるが、現状は Python が運用しやすい。
