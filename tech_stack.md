# Tech Stack

> 本ドキュメントは本プロダクトで採用する技術スタックの **正本** である。判断が変わった場合はここを更新し、commit message に判断理由を残すこと。
> 仕様書 (`backend_spec.md` / `frontend_spec.md`) は **ロジック** の正本、本書は **物理選定** の正本。

## 1. 確定事項

### 1.1 Backend

| 観点 | 採用 | 補足 |
|---|---|---|
| 言語 | Python 3.12+ | LangGraph 公式サポート最新 |
| Web framework | FastAPI | async ネイティブ、OpenAPI 自動生成 |
| ASGI server | uvicorn | production は gunicorn + uvicorn workers |
| ORM | SQLAlchemy 2.x (async) | Pydantic との親和性 |
| Migration | Alembic | |
| Multi-agent orchestration | LangGraph | バージョンは pyproject で固定 |
| Pydantic | v2 (strict mode) | |
| Lint / Format | ruff | format + lint 一体 |
| Type checker | mypy (strict) | |
| Test | pytest + pytest-asyncio | LLM モックは `implementation_plan.md §15.2` |
| Dependency 管理 | uv | poetry より高速、Python 自体の管理も可 |
| Logging | structlog | JSON ログ |

### 1.2 LLM Provider

| 観点 | 採用 |
|---|---|
| 抽象化 | adapter pattern (`backend/llm/`) |
| MVP デフォルト | Anthropic 直 |
| その他対応 adapter | Vertex AI Anthropic, AWS Bedrock Anthropic, Ollama (ローカル) |
| 切替単位 | テナント単位設定 |
| 切替決定タイミング | onboarding 時(`docs/customer/onboarding_checklist.md §4`) |

### 1.2.1 Prompt Asset Format

| 観点 | 採用 | 補足 |
|---|---|---|
| Prompt の物理形式 | Python module (`backend/prompts/*.py`) | version 管理、型補完、reuse を優先 |
| Prompt 本文の持ち方 | 定数文字列 + 組み立て関数 | 小さな差分を関数化しやすい |
| Prompt version | module 内定数で明示 | `PROMPT_VERSION = "x.y.z"` |
| template engine | 不採用 (MVP) | Jinja での過剰抽象化を避ける |

ベースルール:

- prompt は `backend/prompts/` 配下の `.py` とする
- specialist ごとに module を分ける
- version は prompt module 側で明示し、`AgentInvocationRecord` に記録する
- prompt を YAML / TOML / markdown へ分散しない

### 1.3 Frontend

| 観点 | 採用 | 補足 |
|---|---|---|
| Framework | Next.js (App Router) | React 19+ |
| 言語 | TypeScript (strict) | |
| UI library | shadcn/ui | tailwind ベース、ベンダーロックなし |
| 状態管理(server) | Tanstack Query | API キャッシュ・再検証 |
| 状態管理(client) | Zustand | UI ローカル状態 |
| 型契約生成 | openapi-typescript-codegen | backend OpenAPI から生成 |
| Real-time | SSE | LangGraph `astream_events` を直接配信 |
| Form | react-hook-form + zod | |
| Test | Vitest + React Testing Library + Playwright (E2E) | |
| Lint | eslint + biome | |
| Dependency 管理 | pnpm | |

### 1.4 Storage / Infra

| 観点 | 採用 | 補足 |
|---|---|---|
| 状態永続化 | PostgreSQL 16+ | Cloud SQL (GCP) |
| Graph checkpoint | 同じ Postgres | `langgraph-checkpoint-postgres` |
| Vector store | pgvector | 同じ Postgres、metric registry / past findings |
| Artifact store | adapter 化 | デフォルト GCS、S3 / Azure Blob 対応 |
| Python 実行 sandbox | E2B (default) + Docker (on-prem) | `backend/tools/python_runtime.py` で抽象化 |
| Cloud provider (MVP) | **GCP** | Cloud Run + Cloud SQL + GCS + Secret Manager |
| Cloud provider 抽象化 | adapter 化 | AWS / Azure 対応 |

### 1.5 Auth

| 観点 | 採用 | 補足 |
|---|---|---|
| End-user 認証 | Auth.js (NextAuth) | Postgres adapter で同居 |
| FE ↔ BE トークン | JWT (Auth.js から発行) | FastAPI dependency で検証 |
| テナント解決 | JWT claim → FastAPI dependency | |
| BE ↔ 外部サービス | GCP Service Account / Secret Manager | |
| BE ↔ DWH | 顧客提供 credential、Secret Manager 経由 | |

### 1.6 Observability

| 観点 | 採用 | 補足 |
|---|---|---|
| LLM トレース | Langfuse | OSS、self-host 可 |
| アプリケーション計測 | OpenTelemetry | Cloud Trace 等にも export 可 |
| ログ | structlog | JSON ログ、Cloud Logging |
| エラー追跡 | Sentry | |
| メトリクス | Prometheus exporter + Cloud Monitoring | |

### 1.7 CI/CD

| 観点 | 採用 |
|---|---|
| CI | GitHub Actions |
| Container registry | Artifact Registry (GCP) |
| Deploy target (MVP) | Cloud Run |
| IaC | Terraform |
| Secret 管理 | GCP Secret Manager |

## 2. 判断保留 / 顧客ごとに決める事項

| 観点 | 状況 | 決定タイミング |
|---|---|---|
| 顧客の DWH 製品 | 未定 | onboarding(`docs/customer/onboarding_checklist.md §2`) |
| 顧客の cloud provider | 未定 | onboarding(`docs/customer/onboarding_checklist.md §3`) |
| 顧客の LLM provider | 未定(規約・データ扱い方針依存) | onboarding(`docs/customer/onboarding_checklist.md §4`) |
| 顧客の artifact storage | cloud provider に従う | onboarding |
| デプロイ形態 | MVP は SaaS、エンタープライズで on-prem オプション | 個別契約 |

## 3. Deployment Topology

### 3.1 SaaS Multi-tenant 構成 (MVP デフォルト)

```text
[弊社 GCP プロジェクト]
  - Cloud Run: FastAPI backend
  - Cloud Run: Next.js frontend
  - Cloud SQL: PostgreSQL (state + checkpoint + pgvector)
  - GCS: Artifact store
  - Langfuse: LLM トレース
  - Secret Manager: 顧客ごとの DWH credential / LLM API key

[顧客環境]
  - DWH (BigQuery / Snowflake / Redshift / Databricks 等)
  - 必要に応じて顧客 VPC 内の sandbox

[外部 LLM]
  - Anthropic / Vertex AI / Bedrock(顧客選択)
```

### 3.2 On-prem 構成 (エンタープライズオプション)

全コンポーネントを顧客 VPC にデプロイ。`§1.4` の adapter 化により cloud-specific code を最小化することで対応する。

## 4. Agent ごとの LLM モデル既定

テナント設定で上書き可能。デフォルトは以下:

| Agent | デフォルトモデル(役割) | 理由 |
|---|---|---|
| Supervisor | 高性能モデル(Opus 系) | 最終裁定・dispatch |
| Advisor | 高性能モデル(Opus 系) | 批判的レビュー |
| Causal Specialist | 高性能モデル(Opus 系) | 識別戦略の妥当性判断 |
| Insight Specialist | 高性能モデル(Opus 系) | 深い解釈 |
| Scoping Specialist | バランス型(Sonnet 系) | 構造化 |
| Metric Definition Specialist | バランス型(Sonnet 系) | 定義精緻化 |
| SQL Specialist | バランス型(Sonnet 系) | コード生成 |
| Python Specialist | バランス型(Sonnet 系) | コード生成 |
| QA Specialist | バランス型(Sonnet 系) | レビュー |
| Business Recommendation Specialist | バランス型(Sonnet 系) | 提案文書 |
| State Manager | 軽量モデル(Haiku 系) | 確定的 projection |

具体モデル ID は実装着手時に最新世代を選定。本表は **役割の重み付け** が正本。

## 5. 抽象化境界(物理依存しないために)

以下の境界で adapter を立て、cloud / provider を差し替え可能にする。

- `backend/llm/` — LLM provider
- `backend/tools/python_runtime.py` — sandbox(E2B / Docker / Modal)
- `backend/tools/sql_runtime.py` — DWH(BigQuery / Snowflake / Redshift / Databricks)
- `backend/tools/artifacts.py` — artifact store(GCS / S3 / Azure Blob / local)
- `backend/services/secrets.py` — secret manager(GCP SM / AWS SM / Vault)
- `backend/services/auth.py` — auth provider(Auth.js / SAML / OIDC)

各 adapter は `backend/schemas/` で interface を Pydantic Protocol として定義する。

## 6. 用語

本書で使う技術用語は `backend_spec.md` Appendix: Canonical Terms を参照。

## 7. Environment Variables

Phase 0 の `.env.example` と実装時の env 命名は本節を正本とする。

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | backend の Postgres 接続 |
| `ANTHROPIC_API_KEY` | Anthropic provider key |
| `LANGFUSE_PUBLIC_KEY` | Langfuse public key |
| `LANGFUSE_SECRET_KEY` | Langfuse secret key |
| `LANGFUSE_HOST` | Langfuse host URL |
| `SENTRY_DSN` | backend / frontend error tracking |
| `E2B_API_KEY` | Python sandbox runtime |
| `GCS_BUCKET_ARTIFACTS` | artifact store bucket |
| `GCP_PROJECT_ID` | GCP project id |
| `NEXTAUTH_SECRET` | Auth.js secret |
| `NEXTAUTH_URL` | frontend base URL |
| `TENANT_ID_DEFAULT` | local dev 用 default tenant |

ルール:

- 新規 env を導入する場合は `.env.example` と本節を同時更新する
- customer-specific secret は README や task brief に直接書かない
