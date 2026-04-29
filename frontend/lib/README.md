# frontend/lib

共通 utility と契約。

## ファイル(予定)

- `api-client.ts` — fetch wrapper(認証 / error handling 付き)
- `contracts.ts` — backend OpenAPI から生成された型(**手書き禁止**)
- `events.ts` — SSE event 型(同じく生成、[`../system_design.md`](../system_design.md) §5.2)
- `sse-client.ts` — EventSource ラッパー(再接続、エラー処理)
- `query-client.ts` — Tanstack Query 共通設定
- `auth.ts` — Auth.js client side helpers
- `format.ts` — 日付・数値整形
- `logger.ts` — client-side logging
- `redact.ts` — UI 表示前の secret 除去(念のため)

## 規約

- `contracts.ts` と `events.ts` は **手動編集禁止**(generated)
- `api-client.ts` は generated 型を直接 wrap、独自型を増やさない
- error 形式は backend の正規化エラー([`../frontend_spec.md`](../frontend_spec.md) §11.7)に従う
- 認証 token は HttpOnly cookie で扱い、JS から直接 access しない

## SSE 接続管理

`sse-client.ts` は以下を扱う:

- 自動再接続(指数 backoff)
- 認証 token 切れ時の再 issue
- エラー時の callback
- 接続状態の Zustand store への反映

frontend 全体で SSE 接続を 1 つにまとめ、project 切替時に reconnect する。
