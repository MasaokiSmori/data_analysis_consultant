# Data Analysis Consultant

> 顧客社内データに対し、要件定義から提案までを一貫遂行する、マルチエージェント型 AI 分析プラットフォーム。

## このリポジトリについて

本リポジトリは `backend_spec.md` / `frontend_spec.md` で定義されたシステムを実装するための、ドキュメント・設計書・コードベースを格納する。現時点ではアプリケーション本体のコードは未着手だが、実装順序、レビュー基準、ディレクトリ責務、API 契約、mock は整備済みであり、このまま Phase 0 から実装に入れる。

## ドキュメント一覧

### 製品仕様(WHAT)
- [`backend_spec.md`](backend_spec.md) — multi-agent orchestration の論理仕様
- [`frontend_spec.md`](frontend_spec.md) — UI / UX 仕様
- [`frontend_api_mock.json`](frontend_api_mock.json) — frontend が期待する API ペイロード例

### 設計(HOW: 設計レイヤー)
- [`system_design.md`](system_design.md) — モジュール構成・責務分離
- [`tech_stack.md`](tech_stack.md) — 技術スタックの確定事項と未確定事項

### 開発実行(HOW: 実行レイヤー)
- [`implementation_plan.md`](implementation_plan.md) — Phase 別実装計画
- [`milestone_review_checklist.md`](milestone_review_checklist.md) — milestone レビュー観点
- [`task_brief_template.md`](task_brief_template.md) — coding agent 依頼テンプレ

### 顧客運用(WHO)
- [`docs/customer/onboarding_checklist.md`](docs/customer/onboarding_checklist.md) — テナント立ち上げ時の確認事項
- [`docs/customer/sales_pitch.md`](docs/customer/sales_pitch.md) — 営業補助資料

## ディレクトリ構成

| ディレクトリ | 役割 |
|---|---|
| [`backend/`](backend/) | Python + FastAPI 実装(ドキュメント骨格のみ) |
| [`frontend/`](frontend/) | Next.js + React 実装(ドキュメント骨格のみ) |
| [`skills/`](skills/) | Claude Code 用スキル定義 |

## 着手手順(coding agent 用)

新規開発者または coding agent が最初に読むべき順序:

1. `README.md`(本ファイル)— 全体像
2. `tech_stack.md` — 何を使うか
3. `system_design.md` — モジュール構成
4. `backend_spec.md` または `frontend_spec.md` — 該当範囲
5. `implementation_plan.md` — 自分の Phase
6. `task_brief_template.md` — タスク受領時の確認事項

## 最初の 3 タスク

実装開始時は、まず `implementation_plan.md` Phase 0 の次の 3 task から入る。

1. `T0.1` Backend project skeleton
2. `T0.2` Frontend project skeleton
3. `T0.3` Local dev infra

`T0.1` と `T0.2` は並列着手可能で、`T0.3` はその直後に着手しやすい。以後は各 task 行を 1 slice とみなし、`task_brief_template.md` で brief 化して進める。

## ドキュメント整理ルール

- top-level には「実装中に頻繁に開く文書」だけを置く
- 顧客向け、営業向け、運用向け文書は `docs/` 配下に寄せる
- 新しい md を増やす前に、既存文書の節追加で足りないかを確認する

## 開発状況

- [x] 仕様書整備
- [x] 設計書整備
- [x] 技術スタック確定
- [x] ディレクトリ骨格作成
- [ ] Tooling Bootstrap(`pyproject.toml` / `package.json` / CI)
- [ ] Phase 1: Schema and Contract Foundation
- [ ] 以降は `implementation_plan.md` 参照

## 用語の同期

用語の正本は `backend_spec.md` の Appendix: Canonical Terms とする。新語を導入したら、仕様本文だけで完結させず、必ず付録にも反映する。

## 製品名

仕様書群と sales_pitch では `[製品名]` を placeholder として使っている。正式名称が確定後、一括置換すること。
