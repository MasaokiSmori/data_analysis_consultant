# First Agent Briefs

## 1. How To Use

以下は、客先 clone 後に最初に使いやすい brief のたたき台である。必要に応じて顧客固有事情だけ埋めて使う。

原則:

- 1 brief = 1 task
- scope を広げない
- spec を変える task と実装 task を混ぜない

## 2. Brief: T0.1 Backend Project Skeleton

```text
## Task: T0.1 Backend project skeleton

### 目的
backend 実装の最小土台を整え、以後の schema / service / graph 実装が同じ Python toolchain 上で安全に進められるようにする。

### 完成定義(Acceptance Criteria)
- [ ] `backend/pyproject.toml` が作成されている
- [ ] `backend/.python-version` が作成されている
- [ ] `backend/ruff.toml` と `backend/mypy.ini` がある
- [ ] `backend/conftest.py` が最小構成で存在する
- [ ] `uv run ruff check backend` が通る
- [ ] `uv run mypy backend` が通る
- [ ] `uv run pytest backend` が空テストでも起動する

### 範囲
触ってよい:
- backend/pyproject.toml
- backend/.python-version
- backend/ruff.toml
- backend/mypy.ini
- backend/conftest.py
- backend/README.md

触ってはいけない:
- backend_spec.md
- frontend_spec.md
- system_design.md
- implementation_plan.md
- frontend/

### 関連仕様
- implementation_plan.md `T0.1`
- tech_stack.md §1.1
- system_design.md §3
- milestone_review_checklist.md の global checks

### 注意事項
- Python は 3.12+ 前提
- Pydantic v2 strict mode を前提に依存関係を選ぶ
- 将来の LangGraph 導入を阻害しない構成にする
- まだ本番コードは書かず、toolchain bootstrap に限定する

### テスト要件
- pytest が起動すること
- ruff と mypy の最小 pass
- LLM モックは不要

### 期待アウトプット
- backend 側 bootstrap ファイル群
- 1 PR

### 完了報告に含めるべき情報
- 作成 / 更新したファイル一覧
- 実行したコマンド
- 通過したチェック
- 未確定の依存や保留事項
```

## 3. Brief: T0.2 Frontend Project Skeleton

```text
## Task: T0.2 Frontend project skeleton

### 目的
frontend 実装の最小土台を整え、以後の workspace UI と generated contract 利用が同じ TypeScript toolchain で進められるようにする。

### 完成定義(Acceptance Criteria)
- [ ] `frontend/package.json` が作成されている
- [ ] `frontend/next.config.ts` が作成されている
- [ ] `frontend/tsconfig.json` が strict mode で設定されている
- [ ] `frontend/biome.json` が作成されている
- [ ] Next.js app skeleton が最小構成で存在する
- [ ] `pnpm lint` と `pnpm typecheck` の土台がある

### 範囲
触ってよい:
- frontend/package.json
- frontend/next.config.ts
- frontend/tsconfig.json
- frontend/biome.json
- frontend/app/*
- frontend/README.md

触ってはいけない:
- backend/
- backend_spec.md
- frontend_spec.md
- system_design.md
- implementation_plan.md

### 関連仕様
- implementation_plan.md `T0.2`
- tech_stack.md §1.3
- system_design.md §4
- frontend_spec.md §19

### 注意事項
- TypeScript strict を前提にする
- generated contract 利用を前提とした構成にする
- backend API proxy は作らない
- page は薄く、feature module 分割を前提にする

### テスト要件
- typecheck が起動すること
- 最小 lint が通ること
- 実 backend 連携は不要

### 期待アウトプット
- frontend 側 bootstrap ファイル群
- 1 PR

### 完了報告に含めるべき情報
- 作成 / 更新したファイル一覧
- 実行したコマンド
- 通過したチェック
- 未確定の依存や保留事項
```

## 4. Brief: T0.3 Local Dev Infra

```text
## Task: T0.3 Local dev infra

### 目的
backend / frontend / Postgres をローカルで並行起動できる基盤を作り、以後の task が共通の実行環境を前提にできるようにする。

### 完成定義(Acceptance Criteria)
- [ ] `docker-compose.yml` がある
- [ ] `.env.example` がある
- [ ] `Makefile` に最小起動 target がある
- [ ] Postgres コンテナの起動条件が明記されている
- [ ] backend / frontend が後続 task から接続できる前提が README にある

### 範囲
触ってよい:
- docker-compose.yml
- .env.example
- Makefile
- README.md

触ってはいけない:
- backend_spec.md
- frontend_spec.md
- system_design.md
- implementation_plan.md

### 関連仕様
- implementation_plan.md `T0.3`
- tech_stack.md §1.4
- system_design.md §2

### 注意事項
- pgvector 前提を崩さない
- dev 用であり、本番 deploy 設計は含めない
- command 名は後続 task で再利用しやすいものにする

### テスト要件
- `docker compose config` が通ること
- `make` target の説明があること
- 実運用の migration はまだ不要

### 期待アウトプット
- local infra bootstrap ファイル群
- 1 PR

### 完了報告に含めるべき情報
- 作成 / 更新したファイル一覧
- 実行したコマンド
- 通過したチェック
- 未確定の依存や保留事項
```
