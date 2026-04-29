# frontend/components

shadcn/ui primitives と shared layout / domain components。

## ディレクトリ構成(予定)

```text
components/
├── ui/                  # shadcn/ui generated primitives
│   ├── button.tsx
│   ├── dialog.tsx
│   ├── table.tsx
│   ├── badge.tsx
│   ├── tabs.tsx
│   └── ...
├── layout/              # workspace shell, navigation, drawer
│   ├── workspace-shell.tsx
│   ├── project-nav.tsx
│   ├── context-drawer.tsx
│   └── pending-review-banner.tsx
└── domain/              # domain-specific shared components
    ├── artifact-badge.tsx
    ├── quality-stage-indicator.tsx
    ├── risk-badge.tsx
    ├── override-banner.tsx
    ├── provisional-banner.tsx
    ├── phase-badge.tsx
    └── citation-link.tsx
```

## 規約

- `ui/` は shadcn CLI で生成、手動修正は最小限。upstream に追従可能な状態を保つ
- `layout/` と `domain/` は presentation のみ、データ取得は features 側に置く
- props は [`../lib/contracts.ts`](../lib/) の型を直接受ける(独自型を増やさない)
- アクセシビリティ要件は [`../frontend_spec.md`](../frontend_spec.md) §15 に従う
- 色だけで状態を伝えない(text label / icon を併用)

## 重要 component の役割

- **artifact-badge**: artifact_type を視覚化(dataset / insight_memo / execution_plan 等)
- **quality-stage-indicator**: draft / internal / QA-approved / user-reviewed / final-delivery を区別
- **risk-badge**: low / medium / high
- **override-banner**: justified override が記録されている場合に明示
- **provisional-banner**: provisional artifact であることを明示
- **citation-link**: narrative の数値主張から source artifact + region へのリンク
