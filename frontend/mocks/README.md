# frontend/mocks

MSW (Mock Service Worker) handlers。backend が動かない状態でも frontend 開発・テストを可能にする。

## ディレクトリ(予定)

```text
mocks/
├── browser.ts                  # 開発時 worker setup
├── server.ts                   # test 時 server setup
├── handlers/
│   ├── index.ts                # 全 handler を集約
│   ├── auth.ts
│   ├── projects.ts
│   ├── conversation.ts         # SSE handler 含む
│   ├── reviews.ts
│   ├── artifacts.ts
│   ├── timeline.ts
│   ├── governance.ts
│   └── deliverables.ts
└── data/
    ├── projects.json           # frontend_api_mock.json から派生
    ├── reviews.json
    └── ...
```

## 規約

- レスポンスは [`../../frontend_api_mock.json`](../../frontend_api_mock.json) を起点とする
- backend が動かない状態でも frontend を開発可能にする
- 開発時のみ ON、production build には含めない([`browser.ts`](.) は dev mode のみ起動)
- API 契約変更時は `../lib/contracts.ts` 再生成 → mock data も追従
- SSE は MSW の event-stream モード or カスタム ReadableStream

## SSE モック

`handlers/conversation.ts` 内で `/projects/{id}/stream` の SSE handler を提供。

- 開発時は scripted event sequence を返す(graph 進行の再現)
- test 時は test 固有の event を inject 可能

## 開発フロー

1. backend が固まる前: MSW で frontend を進める
2. backend 接続が可能になったら: 環境変数で MSW を OFF
3. 統合 test では MSW を必須(実 backend を立ち上げない)
