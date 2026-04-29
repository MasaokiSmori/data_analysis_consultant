# frontend/features

機能単位のモジュール。[`../frontend_spec.md`](../frontend_spec.md) §4.1 の Core Areas に対応。

## モジュール(予定)

| モジュール | 役割 | 関連 spec |
|---|---|---|
| `conversation/` | chat thread + clarification + system message | `../frontend_spec.md §7.2` |
| `reviews/` | review center + review form | `../frontend_spec.md §7.4` |
| `artifacts/` | artifact workspace + renderer + lineage | `../frontend_spec.md §7.3` |
| `timeline/` | phase / decision / governance event timeline | `../frontend_spec.md §7.5` |
| `governance/` | risk / readiness / override / advisor consult panel | `../frontend_spec.md §10.1` |
| `deliverables/` | final outputs + sign-off | `../frontend_spec.md §7.6` |
| `home/` | project home page | `../frontend_spec.md §7.1` |

## 各 feature の内部構成

```text
features/<name>/
├── components/         # 内部 component
├── hooks/              # 内部 hook
├── api.ts              # Tanstack Query 呼び出し
├── store.ts            # Zustand store(必要時のみ)
├── types.ts            # feature 固有型(generated 型からの拡張のみ)
└── index.ts            # public API
```

## 規約

- API 呼び出しは Tanstack Query を使用([`../lib/api-client.ts`](../lib/) と generated 型を組み合わせる)
- ローカル UI 状態は Zustand store(必要時のみ、過剰に作らない)
- feature 間の直接 import は禁止(共通要素は [`../components/`](../components/) または [`../lib/`](../lib/) に切り出し)
- artifact renderer は extensible に作る(artifact_type ごとに renderer を登録)

## artifact renderer の拡張ポイント

`artifacts/components/renderer.tsx` は artifact_type で dispatch:

- `document`
- `table_preview`
- `chart_gallery`
- `issue_list`
- `report_outline`
- `json_view`

新 artifact_type 追加時は renderer の登録のみで対応する。
