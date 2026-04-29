# Sample Engagement Fixture

## Purpose

この fixture は、Phase 3 以降の graph / specialist / frontend integration を通すための最小サンプル案件である。

用途:

- scoping 起点の end-to-end テスト
- mock metadata catalog の提供
- SQL / QA / insight の期待フロー確認
- frontend MSW / API projection の安定化

## Included Files

- `user_request.md`
  サンプルの顧客要求文
- `catalog_tables.csv`
  利用可能テーブル一覧
- `datamart_spec.md`
  簡易データマート仕様
- `expected_artifacts.md`
  最低限生成される想定 artifact

## Usage Rules

- 本 fixture は production 相当データを表現しない
- PII は含めない
- graph / API / frontend test で共通参照してよい
- 仕様変更によりサンプル案件の前提が変わった場合、関連テストと一緒に更新する

## Candidate Scenario

- テーマ: retention 低下の要因分析
- 主要指標: weekly retained users, reactivation rate
- 初回要求: 「先月から継続率が落ちている。セグメント別に原因を見たい」
