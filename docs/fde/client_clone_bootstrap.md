# Client Clone Bootstrap

## 1. Goal

客先で本リポジトリを clone した直後に、人間と coding agent が迷わず最初の数 task に着手できるようにする。

## 2. First 30 Minutes

1. `README.md` を読む
2. `backend_spec.md` と `frontend_spec.md` の概要を確認する
3. `system_design.md` の repo shape と contract generation を確認する
4. `implementation_plan.md` の Phase 0 と Phase 1 を確認する
5. `docs/customer/onboarding_checklist.md` で顧客差分の空欄を洗い出す

## 3. Customer Facts To Fill In Before Coding

- 利用する DWH
- 認証方式
- 利用可能な LLM provider
- 顧客 cloud / network 制約
- artifact storage の制約
- sandbox 実行可否
- tenant / access policy の特記事項

これらが未定でも Phase 0 の多くは進められるが、未定であることは `tech_stack.md` に明示する。

## 4. Files To Update Before First Real Feature Work

- `tech_stack.md`
  顧客固有の採用方針や未確定事項
- 必要なら `backend_spec.md` / `frontend_spec.md`
  顧客要件により論理仕様が変わる場合のみ

注意:

- customer-specific 実装を、spec 未更新のままコードへ先に書かない
- 仕様変更と実装変更は別 task に分けてもよい

## 5. First Coding Tasks

最初の着手順:

1. `T0.1` Backend project skeleton
2. `T0.2` Frontend project skeleton
3. `T0.3` Local dev infra
4. `T0.5` Codegen pipeline scaffolding
5. `T1.1` Literals + Enums

`T0.1` と `T0.2` は並列着手可。

## 6. Recommended Review Rhythm

- 1 task 完了ごとに review
- Phase 完了時に milestone review
- customer-specific decision が入る前に human lead review
- access / auth / tenant が絡む変更は必ず別レビュー

## 7. Bootstrap DoD

以下が満たせれば、客先での実装 bootstrap は完了とみなしてよい。

- Phase 0 の brief が 3 本作られている
- 顧客差分が `tech_stack.md` で見える
- spec の未確定点と顧客未回答点が区別されている
- 最初の coding agent task が 1 つ実行可能な状態になっている

