# Backend

Python + FastAPI 実装。詳細は [`../tech_stack.md`](../tech_stack.md) および [`../system_design.md`](../system_design.md) §3 参照。

## レイヤー構成

| ディレクトリ | 役割 | 関連仕様 |
|---|---|---|
| [`api/`](api/) | FastAPI router | `../frontend_spec.md §11` |
| [`graph/`](graph/) | LangGraph orchestration | `../backend_spec.md §13` |
| [`llm/`](llm/) | LLM provider adapter | `../tech_stack.md §1.2` |
| [`memory/`](memory/) | working / episodic / project / decision memory | `../backend_spec.md §7.4` |
| [`policies/`](policies/) | pure-function policy module | `../backend_spec.md §13.4` |
| [`prompts/`](prompts/) | agent prompt 構築 | `../backend_spec.md §10` |
| [`schemas/`](schemas/) | Pydantic schema | `../backend_spec.md §14.2` |
| [`services/`](services/) | API ↔ graph/state/tool 間の service | `../system_design.md §3.6` |
| [`state/`](state/) | GraphState 永続化 + checkpoint | `../backend_spec.md §14` |
| [`tools/`](tools/) | tool 実装 | `../backend_spec.md §9.8` |
| [`tests/`](tests/) | pytest | `../implementation_plan.md §15` |

## 着手順序

[`../implementation_plan.md`](../implementation_plan.md) の Phase 1-6 を順に実装する。各 phase の入口で [`../milestone_review_checklist.md`](../milestone_review_checklist.md) を参照。

## 規約

- 仕様書は読み取り専用、変更が必要なら別タスクとして分離
- 用語は [`../backend_spec.md`](../backend_spec.md) Appendix: Canonical Terms に従う
- 触ってはいけない範囲は `task_brief_template.md` で都度定義
