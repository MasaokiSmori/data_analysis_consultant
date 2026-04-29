# Frontend

Next.js (App Router) + React + TypeScript。詳細は [`../tech_stack.md`](../tech_stack.md) §1.3、[`../system_design.md`](../system_design.md) §4、[`../frontend_spec.md`](../frontend_spec.md) 参照。

## レイヤー構成

| ディレクトリ | 役割 | 関連仕様 |
|---|---|---|
| [`app/`](app/) | Next.js App Router pages | `../frontend_spec.md §7` |
| [`components/`](components/) | shadcn/ui primitives + layout + domain | `../frontend_spec.md §14` |
| [`features/`](features/) | feature module(conversation / reviews / 等) | `../frontend_spec.md §4` |
| [`lib/`](lib/) | API client / generated contracts / utility | `../system_design.md §5` |
| [`mocks/`](mocks/) | MSW handlers | `../frontend_api_mock.json` |
| [`tests/`](tests/) | Vitest / Playwright | `../implementation_plan.md §15.1` |

## 規約

- API 型は [`lib/contracts.ts`](lib/) のみ使用、手書き禁止([`../system_design.md`](../system_design.md) §5)
- backend 生 state を直接受けない、projection のみ([`../frontend_spec.md`](../frontend_spec.md) §11)
- リアルタイムは SSE
- masked / unavailable な field は表示時に補完しない([`../frontend_spec.md`](../frontend_spec.md) §10.2)
- draft / QA-approved / final-delivery は視覚的に区別する

## 着手順序

[`../implementation_plan.md`](../implementation_plan.md) Phase 5 (Frontend Workspace MVP)、および [`../frontend_spec.md`](../frontend_spec.md) §19 の優先度順。

1. project workspace shell
2. conversation screen
3. review center
4. artifact workspace
5. live phase / status display
6. deliverables screen
7. timeline / provenance screen
