# frontend/tests

Vitest + React Testing Library + Playwright。[`../../implementation_plan.md`](../../implementation_plan.md) §15.1 参照。

## ディレクトリ構成(予定)

```text
tests/
├── setup.ts                    # global setup(MSW server, jest-dom 等)
├── unit/                       # pure 関数・hook
├── component/                  # 単体 UI component
├── integration/                # feature module + MSW
└── e2e/                        # Playwright(staging を叩く)
```

## 規約

- 実 backend を起動しない unit / component / integration では MSW を使用
- e2e は Playwright で staging を叩く(本番ではない)
- snapshot test は最小限に(LLM 生成出力に対する snapshot は常に陳腐化する)
- 色だけのテスト・実装詳細のテストは避け、ユーザー視点の挙動を assert
- 認証が絡むテストは Auth.js の test helper or MSW で session を mock

## 重要テスト観点

[`../../milestone_review_checklist.md`](../../milestone_review_checklist.md) Milestone E に対応。

- pending review が UI 上で見落とされにくいか
- quality_stage / provisional / risk が視覚的に区別されているか
- override / advisor consult / governance 情報が表示されるか
- masked fields が補完されず unavailable として見えるか
- citation link が source artifact に正しく繋がるか

## E2E シナリオ(MVP)

- ログイン → project 作成 → conversation 開始 → review approve → deliverable 確認
- request_changes フロー
- pause / resume フロー
- override 表示確認
