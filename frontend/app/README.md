# frontend/app

Next.js App Router pages。

## ルーティング(MVP)

```text
app/
├── (auth)/
│   ├── login/page.tsx
│   └── callback/page.tsx
├── (workspace)/
│   ├── layout.tsx                          # workspace shell
│   ├── projects/
│   │   ├── page.tsx                        # project 一覧
│   │   └── [projectId]/
│   │       ├── layout.tsx                  # project nav
│   │       ├── page.tsx                    # project home
│   │       ├── conversation/page.tsx
│   │       ├── reviews/page.tsx
│   │       ├── artifacts/page.tsx
│   │       ├── timeline/page.tsx
│   │       └── deliverables/page.tsx
│   └── admin/page.tsx                      # super admin
├── api/                                    # Auth.js のみ。BE への proxy は厳禁
│   └── auth/[...nextauth]/route.ts
├── layout.tsx                              # root layout
└── page.tsx                                # landing
```

## 規約

- App Router の Server Component を活用するが、リアルタイム要素(SSE / 状態購読)は Client Component
- 認証は [`(auth)/`](.) 配下、workspace は [`(workspace)/`](.) 配下に分離
- backend API への proxy 化はしない(直接 fetch、Auth.js から取得した JWT を使う)
- 共通 layout は [`../components/layout/`](../components/) を使用
- 各 page は薄く、機能本体は [`../features/`](../features/) に置く

## メタデータ

各 page は `metadata` export を持ち、画面タイトル・説明を統一する。
