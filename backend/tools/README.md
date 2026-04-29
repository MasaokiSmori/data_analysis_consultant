# backend/tools

[`../../backend_spec.md`](../../backend_spec.md) §9.8 の tool I/O contract に準拠した tool 実装。

## ファイル分割(予定)

| ファイル | 含む tool |
|---|---|
| `state_views.py` | `query_state_view` |
| `artifacts.py` | `load_artifact`, `store_artifact`, `get_artifact_lineage` |
| `metadata.py` | `list_tables`, `get_table_summary`, `get_datamart_spec` |
| `governance.py` | `emit_execution_plan`, `decide_execution_plan`, `classify_execution_risk`, `evaluate_execution_readiness`, `submit_override_request`, `validate_override_request` |
| `qa.py` | `run_qa_stage`, `evaluate_qa_gates`, `mark_artifact_quality_stage` |
| `sql_runtime.py` | `run_sql`, `validate_sql`, `estimate_sql_cost`, DWH adapter |
| `python_runtime.py` | `run_python`, sandbox adapter (E2B / Docker) |
| `access.py` | `check_access_scope` |
| `knowledge.py` | `search_metric_registry`, `search_past_findings` |
| `fallback.py` | `record_provisional_mode`, `resolve_user_wait_policy` |
| `runtime.py` | tool registry、caller identity injection、observability wrapping |

## 規約

- 入出力は strict JSON、エラーは [`../../backend_spec.md`](../../backend_spec.md) `tool_error_contract` に従う
- caller identity は runtime layer 側で inject、payload では受けない(`backend_spec.md §7.2`)
- 重い出力は artifact_id 化、本体は storage に保存
- すべての tool 呼び出しは `ToolCallRecord` に記録
- OpenTelemetry span を発行(`runtime.py` の wrapper で自動化)

## SQL/Python Safety Layer

`sql_runtime.py` と `python_runtime.py` は preflight / postflight を持つ。詳細は [`../../backend_spec.md`](../../backend_spec.md) §19.4。

- preflight: `plan_status == "approved"` 確認、`validate_sql`、`estimate_sql_cost`、allowlist 検査
- postflight: schema 比較、sanity_artifact 生成、`metric_definition_sheet` 照合、`plan_vs_actual_diff` 生成

これらは tool 層で **物理的に強制** する(prompt 任せにしない)。

## DWH adapter

`sql_runtime.py` 内に DWH adapter を持つ。

- `BigQueryAdapter`
- `SnowflakeAdapter`
- `RedshiftAdapter`
- `DatabricksAdapter`

選定はテナント設定([`../../docs/customer/onboarding_checklist.md`](../../docs/customer/onboarding_checklist.md) §2)に従う。

## Sandbox adapter

`python_runtime.py` 内に sandbox adapter を持つ。

- `E2BAdapter`(MVP デフォルト)
- `DockerAdapter`(on-prem)
- `ModalAdapter`(将来検討)

選定はテナント / プラン設定に従う。
