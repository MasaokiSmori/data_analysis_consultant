---
name: data-analysis-consultant-specs
description: Use when editing specs, designs, mocks, or contracts in the data_analysis_consultant project — anything that touches agent behavior, governance, QA gates, plan-execute flow, persistence design, API projections, or terminology. Keeps backend_spec.md, frontend_spec.md, system_design.md, tech_stack.md, implementation_plan.md, and frontend_api_mock.json aligned, and prevents resurrecting consolidated files (glossary.md, api_endpoints.md, codegen.md, testing_strategy.md).
---

# Data Analysis Consultant Specs

Use this skill when editing or reviewing this project's specifications, designs, mocks, or contracts. 複数の正本ファイルが互いに参照しあう構造になっており、整合の崩れが本プロジェクトで最も頻発する failure mode。本 skill はその予防のためのもの。

## Core Files

正本ファイルとその責務:

- `../../backend_spec.md`
  multi-agent orchestration の論理仕様。agents、tools、governance、QA、override policy、state schema。`§22 Appendix: Canonical Terms` が用語正本。
- `../../frontend_spec.md`
  UI / UX 仕様。workspace shell、review UX、artifact presentation。`§11 Frontend-Backend Contracts`(`§11.5` が API endpoint 正本)。
- `../../system_design.md`
  モジュール構成と物理境界。`§3.2.2 State Persistence Design`、`§5 Contract Generation Design`、`§3.7 Human Review Trace Layer`。
- `../../tech_stack.md`
  物理選定の正本。LLM provider 抽象化、prompt 物理形式 (`§1.2.1`)、env vars (`§7`)、agent → モデル割り当て (`§4`)、abstraction boundaries (`§5`)。
- `../../implementation_plan.md`
  実装順序の正本。phase / task 分解、`§15 Testing Strategy`、`§15.2 LLM Mocking Policy`。
- `../../milestone_review_checklist.md`
  milestone レビュー観点と Mandatory Red Flags。
- `../../frontend_api_mock.json`
  frontend が期待する API ペイロード例。`frontend_spec.md §11` と整合させる。

## Change Category → Files

変更内容ごとに必ず触るファイル:

| 変更内容 | 必ず触るファイル | 注意 |
|---|---|---|
| 新 agent / agent 責務変更 | `backend_spec.md §4.1, §5, §22` | tool matrix と prompt も影響 |
| 新 tool / tool I/O 変更 | `backend_spec.md §9.4, §9.8, §22` | tool access policy 確認 |
| Plan-Execute / QA stage 変更 | `backend_spec.md §6.4, §11.3, §22` | UI gate 表示 (`frontend_spec.md`) も追従 |
| Routing / Policy module 変更 | `backend_spec.md §13.4` | `POLICY_VERSION` を bump |
| 新 API endpoint | `frontend_spec.md §11.5` + `frontend_api_mock.json` | projection model を別建て |
| UI 要素 / 画面追加 | `frontend_spec.md §7` (rough wireframe を更新) | shell との整合 |
| State / DB schema 変更 | `system_design.md §3.2.2` + `backend_spec.md §14` | Alembic migration 命名規約に従う |
| Contract / OpenAPI 変更 | `system_design.md §5` | codegen 再実行 |
| 物理選定 / cloud / provider 変更 | `tech_stack.md §1, §3, §5` | adapter 境界を維持 |
| Agent ごとの LLM 既定変更 | `tech_stack.md §4` | 役割の重み付けを破らない |
| 環境変数追加 | `tech_stack.md §7` + `.env.example` | secret は repo に書かない |
| 用語追加 / 改名 | `backend_spec.md §22` | spec 本文で重複定義しない |
| Phase / task 追加 | `implementation_plan.md` | deps と size を必ず埋める |
| Red flag / レビュー観点追加 | `milestone_review_checklist.md` | 既存 milestone と重複させない |

## Workflow

1. 変更要件から `Change Category → Files` 表で触るファイルを特定する。
2. 該当 spec を読み、現状を理解してから変更する(過去挙動の推測で書かない)。
3. backend governance / QA / risk / override / tool contracts が変わるなら、frontend 側 (`frontend_spec.md` および mock) の追従要否を必ず確認する。
4. frontend projection や画面が変わるなら、`frontend_api_mock.json` の対応 entry を同 PR で更新する。
5. 新概念の追加は `backend_spec.md §22 Appendix: Canonical Terms` を必ず併せて更新する。
6. 複数 spec を跨ぐ変更は、コミットを分けず 1 PR にまとめてレビュー可能にする。

## Guardrails

violation したら即差し戻すべき項目:

- Supervisor override を unconditional として記述しない。常に `justified override` 形式(`backend_spec.md §6.6`)。
- State Manager を recommendation engine / ranking 装置にしない。typed / deterministic な read-path のみ。
- State Manager 自身を thinking graph node として扱わない。service として定義し、graph node が呼び出す。
- `provisional` artifact を final delivery / user-facing recommendation に混ぜない。
- 用語を spec 本文だけに新規導入しない。`backend_spec.md §22` を必ず併せて更新する。
- API endpoint を `frontend_spec.md §11.5` 以外の場所で定義しない。
- 削除済みの top-level docs (`glossary.md`, `api_endpoints.md`, `codegen.md`, `testing_strategy.md`) を再作成しない。これらの内容は本 skill `Core Files` の consolidated 先にある。
- Pydantic と SQLAlchemy ORM を 1 source(SQLModel 等)に統合しない。`system_design.md §3.2.2` の方針通り分離する。
- Prompt を YAML / TOML / Markdown / Jinja に分散させない。`tech_stack.md §1.2.1` の通り Python module で持つ。
- frontend に backend 生 state を直接露出しない。`frontend_spec.md §11.3` の projection contract を経由する。

## Quick Checks

- 新 backend field が UI に影響するか?
  → `frontend_spec.md` と `frontend_api_mock.json` を併せて更新。
- 新 artifact quality state を追加したか?
  → `backend_spec.md §22` の用語、artifact display rule、mock を更新。
- 新 governance tool を追加したか?
  → timeline / status / governance panel の表示要否を `frontend_spec.md §7.5, §11.3` で確認。
- DB に保持するデータを追加したか?
  → `system_design.md §3.2.2` の方針(JSONB 許容領域 vs 正規化対象)に当てはめる。
- prompt や agent 挙動を変えたか?
  → `prompt_version` の bump 要否を `backend_spec.md §10` と整合させる。
- 用語 / role / artifact_type / event_type を新規導入したか?
  → top-level `Literal` の追加位置を `backend/schemas/state.py` または `events.py` に予約しているか確認(本 skill は schema 実装を直接触らないが、設計意図の整合は確認する)。

## Self-update Rule

本 skill 自体が陳腐化していたら更新する。具体的には:

- `Core Files` の構成が崩れた / 新ファイルが正本に昇格した
- consolidation 元のファイル名が変わった
- 新たな Mandatory Red Flag が `milestone_review_checklist.md` に追加された
- 新たな抽象化境界が `tech_stack.md §5` に追加された
- 新たな consolidation(複数 docs の単一正本化)が起きた

これらは本 skill の `Core Files` / `Change Category → Files` / `Guardrails` のいずれかに必ず反映する。
