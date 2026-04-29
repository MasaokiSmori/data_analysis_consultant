# Task Brief Template

> coding agent (Claude Code 等) に実装タスクを依頼する際の標準テンプレート。
> `implementation_plan.md` の task 表にある 1 行を 1 slice として agent に渡すときは、原則本テンプレに従う。

## 1. テンプレート

```text
## Task: <短いタイトル>

### 目的
<このタスクで実現したい価値を 1-2 文で。"なぜ" を必ず含める>

### 完成定義(Acceptance Criteria)
- [ ] <検証可能な条件 1>
- [ ] <検証可能な条件 2>
- [ ] テストが通ること(範囲は §テスト要件)

### 範囲
触ってよい:
- <ファイル / ディレクトリ>

触ってはいけない:
- <他 agent タスクとの境界>
- <レビュー必須の領域>
- 仕様書(*_spec.md)・設計書(system_design.md)・tech_stack.md は読み取り専用

### 関連仕様
- <backend_spec.md §X.Y>
- <system_design.md §X.Y>
- <tech_stack.md §X.Y>
- <frontend_spec.md §11.X>
- <その他参照すべきファイル>

### 注意事項
- <既知の落とし穴>
- <他タスクとの依存関係>
- <仕様で曖昧な点があれば、推測で進めず質問する>

### テスト要件
- <unit test を書く範囲>
- <integration test の有無>
- <LLM モック方針(implementation_plan.md §15.2 参照)>

### 期待アウトプット
- <生成 / 変更されるファイル>
- <PR の単位>

### 完了報告に含めるべき情報
- 変更ファイル一覧
- 通過したテスト一覧
- 既知の未解決事項
- レビュー観点として注意すべき点
- 仕様で未定義のため独自判断した箇所
```

## 2. 良い brief の特徴

- **完成定義が「動くか」ではなく「正しいか」を含む**
  - 例: 「endpoint が 200 を返す」ではなく「allowed_actions が `ReviewAction` Literal と一致する」
- **触ってはいけない範囲を明記**
  - 仕様改変、他 phase の依存層への侵入を防ぐ
- **関連仕様の section 番号まで指定**
  - agent が全文を読み直すのを防ぎ、token コストを抑える
- **不明点を「質問して止まる」ことを許可**
  - 推測で書き進めるよりも遥かに安全

## 3. 悪い brief の例(避けるべき)

- 「いい感じに実装して」
- 「テストも書いといて」(範囲不明)
- 「Phase 1 を全部やって」(粒度が大きすぎる)
- 「動けば OK」(完成定義の欠如)
- 「仕様書を読めば分かる」(参照箇所の指定なし)

## 4. レビュー観点との対応

`milestone_review_checklist.md` の対応 milestone セクションを brief に含めると、agent 自身が完了前にセルフレビューしやすくなる。可能なら以下を毎回付与する。

```text
### セルフレビュー観点
- milestone_review_checklist.md の Milestone X セクション
- §3 Mandatory Red Flags の該当項目
```

## 5. 例: Phase 1 タスクの brief

```text
## Task: GraphState Pydantic schema の実装

### 目的
backend_spec.md §14.2 の Typed State Sketch を Pydantic v2 モデルとして実装し、
以降の API / tool / graph 実装が安全に依存できる正本を確立する。

### 完成定義
- [ ] backend/schemas/state.py が backend_spec.md §14.2 の全モデルを実装している
- [ ] 主要 field に Field(description=...) が付与されている(milestone_review_checklist.md Milestone A)
- [ ] Literal は top-level に集約されている(RiskLevel, QualityStage 等)
- [ ] mypy strict が pass する
- [ ] ruff lint が pass する
- [ ] schema round-trip テスト(JSON serialize/deserialize)が通る

### 範囲
触ってよい:
- backend/schemas/state.py
- backend/tests/unit/schemas/test_state.py

触ってはいけない:
- backend_spec.md
- system_design.md
- backend/graph/, services/, tools/

### 関連仕様
- backend_spec.md §14.2(全文)
- backend_spec.md §14.1(state separation policy)
- backend_spec.md Appendix: Canonical Terms(用語確認)
- tech_stack.md §1.1(Pydantic v2 strict mode)
- milestone_review_checklist.md Milestone A

### 注意事項
- TaskRecord の risk_level は RiskLevel Literal を使う(inline Literal にしない)
- AgentEpisodicMemoryEntry は bounded n=2、ただしモデルに件数制約は持たせず service 層で enforce
- audit field は AuditFields でまとめる

### テスト要件
- 全モデルを 1 件ずつ default 値でインスタンス化できる
- 主要モデルの JSON serialize/deserialize が round-trip する
- LLM 呼び出しは含まない(LLM モック不要)

### 期待アウトプット
- backend/schemas/state.py(新規)
- backend/tests/unit/schemas/test_state.py(新規)
- 1 PR

### 完了報告
- 変更ファイル一覧
- pytest 結果
- mypy / ruff 結果
- 仕様で曖昧で独自判断した箇所(あれば)

### セルフレビュー観点
- milestone_review_checklist.md Milestone A
- §4 Mandatory Red Flags(特に「Literal が散らばっていないか」)
```

## 6. 一括依頼の禁止

「Phase 1 を全部やって」のような一括依頼は **禁止**。`implementation_plan.md` の task 表 1 行を 1 slice として分割する。

## 7. 反復の扱い

review で reject された場合、新しい brief を作るのではなく、元 brief に「修正依頼」セクションを追記して再依頼する。これにより、同じ task に対する判断履歴が一箇所に残る。
