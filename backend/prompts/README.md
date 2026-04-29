# backend/prompts

agent prompt のテンプレート + 構築ロジック。[`../../backend_spec.md`](../../backend_spec.md) §10 参照。

## ファイル(予定)

- `common.py` — 全 agent 共通 prompt 要素(team_directory、direct messaging 禁止、recommendation format 等)
- `supervisor.py`
- `state_manager.py`
- `advisor.py`
- `scoping.py`
- `metric_definition.py`
- `sql.py`
- `python.py`
- `qa.py`
- `causal.py`
- `insight.py`
- `business_recommendation.py`
- `templates/` — テキストテンプレート(Jinja2 等を使う場合)

## 規約

- prompt は version 化する(`PROMPT_VERSION` 定数)
- prompt 変更は `AgentInvocationRecord.prompt_version` に記録される
- 過去案件知見の prompt 自動埋め込み禁止(opt-in pull のみ)
- 数値主張時の出典引用フォーマットを必ず含める
- 各 prompt は単体 test を持つ(text smoke test + structure test)

## 共通要素チェックリスト

`common.py` で以下を必ず提供:

- agent 自身の role
- team_directory
- direct messaging 禁止
- recommendation set フォーマット
- 品質基準
- 利用可能 tool
- 過去知見は pull のみ
- 数値主張の出典引用フォーマット
- `query_state_view` 利用ガイド(必要最小限)
- `emit_execution_plan` 必須(該当 agent のみ)

各 specialist prompt は `common.py` を base にして role-specific な部分のみ追加する。
