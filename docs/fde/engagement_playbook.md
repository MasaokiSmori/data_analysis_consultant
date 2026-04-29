# FDE Engagement Playbook

## 1. Purpose

本書は、本リポジトリを客先環境へ clone し、coding agent と人間レビューを組み合わせて実装を進めるための運用手引きである。

前提:

- `backend_spec.md` と `frontend_spec.md` が論理仕様の正本
- `system_design.md` が実装構成の正本
- `implementation_plan.md` が実装順序の正本
- 変更はコードからではなく、必要に応じて正本ドキュメントから更新する

## 2. Engagement Principles

- 小さく始める
  最初は Phase 0 / Phase 1 の小さな slice から着手する
- spec-first を守る
  実装中に見つかった重要差分は code 側だけで閉じず、必要なら spec に戻す
- 1 task = 1 brief = 1 review を守る
  agent に大きな塊を渡さない
- 共通核と顧客差分を分ける
  orchestration / governance / review / QA は共通化し、DWH / auth / provider / deploy は顧客差分として扱う
- provisional と final を混同しない
  早く動かすための暫定実装と、本採用の設計判断を分けて記録する

## 3. Roles

### 3.1 Human Lead

- 顧客要件の確認
- brief の作成
- review gate の通過判断
- spec 変更の要否判断

### 3.2 Coding Agent

- brief に沿った実装
- 指定範囲のテスト
- self-review
- 未定義点の明示

### 3.3 Customer Stakeholder

- DWH / auth / data access / security 制約の提供
- review / approve / clarify への応答
- customer-specific decisions の確定

## 4. Standard Engagement Flow

1. clone 後に `README.md`、`backend_spec.md`、`frontend_spec.md`、`system_design.md`、`implementation_plan.md` を確認する
2. 顧客差分を `tech_stack.md` と `docs/customer/onboarding_checklist.md` に沿って洗い出す
3. Phase 0 の最初の 3 task (`T0.1`, `T0.2`, `T0.3`) から brief を作る
4. coding agent に 1 slice だけ依頼する
5. `milestone_review_checklist.md` と brief の acceptance criteria で review する
6. 通過したら次の slice に進む
7. 仕様上のズレが出たら、実装を続ける前に spec 変更 task を分離する

## 5. What To Customize Per Customer

- DWH adapter
- 認証方式
- LLM provider
- artifact storage
- deploy topology
- tenant / access policy
- mandatory review の運用閾値

これらはまず `tech_stack.md` と spec へ差分を反映し、その後に実装へ進む。

## 6. What Not To Customize Early

- State Manager の基本原則
- justified override の概念
- QA stage の分離
- artifact quality stage
- project / decision / working memory の分離

これらは本アーキテクチャの共通核なので、PoC 初期に崩さない。

## 7. Escalation Rules

以下に当たる場合、実装 task を止めて spec / design review に切り替える。

- task brief で定義されていない新しい state field が必要
- API projection の shape を大きく変える必要がある
- access control の前提が変わる
- provisional と final の境界が崩れる
- policy override を実装都合で緩めたくなる

## 8. Success Pattern

よい進め方:

- 1-3 日で終わる slice を切る
- generated file や schema diff を PR で一緒にレビューする
- 顧客固有差分を早い段階で `tech_stack.md` に落とす
- backend と frontend を mock / contract ベースで疎結合に進める

悪い進め方:

- 「Phase 1 を全部作って」のように大きく依頼する
- spec 未更新のまま customer-specific rule をコードへ直書きする
- generated contracts を手で直す
- provisional 実装をそのまま production へ流す

## 9. Recommended Deliverables Per Week

Week 1:

- 顧客差分の整理
- Phase 0 完了

Week 2:

- Phase 1 完了
- mock alignment と codegen の成立

Week 3-4:

- Phase 2-3 の骨格
- State Manager service
- Supervisor / policy skeleton

Week 5+:

- specialist MVP
- frontend workspace
- hardening

## 10. Exit Condition For A Customer PoC

- 顧客の 1 ユースケースを end-to-end で通せる
- review / approval / provenance が UI 上で確認できる
- provisional / QA-approved / final の区別が崩れていない
- access / tenant / masking の前提が文書化されている

