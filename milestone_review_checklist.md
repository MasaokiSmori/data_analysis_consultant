# Milestone Review Checklist

## 1. Purpose

本チェックリストは、各実装 milestone で人間がレビューすべき観点をまとめたものである。coding agent の成果物を「動くか」だけでなく、「仕様から逸脱していないか」で確認するために使う。

## 2. Global Checks

すべての milestone で確認する。

- 仕様の新語や field が、関連ファイルすべてで整合しているか
- typed schema と mock contract がズレていないか
- draft / provisional / final の境界が崩れていないか
- State Manager が recommendation や ranking を返していないか
- Supervisor override が unconditional になっていないか
- access / tenant / masking の考慮が消えていないか

## 3. Milestone-Specific Checklists

### Milestone A. Schema Foundation

- `GraphState` の主要モデルが仕様に追随しているか
- `Field(description=...)` が主要 field に入っているか
- Literal / Enum が散らばらず top-level に整理されているか
- frontend projection schema が backend 生 state を露出していないか

### Milestone B. State Manager MVP

- `query_state_view` が typed / deterministic になっているか
- caller identity が runtime inject 前提で扱われているか
- allowed views / fields / filters が制御されているか
- `working memory` の summary が生成されるか
- episodic memory が agent の私有長期メモになっていないか

### Milestone C. Governance Skeleton

- `evaluate_routing_policy`, `evaluate_phase_gate`, `evaluate_advisor_consult_trigger` が pure-function 的に実装されているか
- `classify_execution_risk` と `evaluate_execution_readiness` が policy version を返すか
- override request に rationale / mitigation が必須になっているか
- 根拠不十分な Supervisor 判断が policy を上書きできないか
- required node の Human Review Reasoning Trace が出力され、state に混入していないか

### Milestone D. Execution MVP

- plan -> approval routing -> execute の経路が通るか
- low-risk delegated approval と high-risk supervisor review が分かれているか
- `TaskRecord.risk_level` が latest plan risk として扱われているか
- QA が stage 単位で動くか
- 前 stage の失敗時に後続 stage が止まるか

### Milestone E. Frontend MVP

- pending review が UI 上で見落とされにくいか
- quality stage / provisional / risk が視覚的に区別されているか
- override / advisor consult / governance 情報が表示されるか
- masked fields が補完されず unavailable として見えるか

### Milestone F. Hardening

- degraded mode / provisional mode が監査できるか
- `check_access_scope` が権限昇格なしに動くか
- tenant isolation が保たれているか
- policy store version が decision と結び付くか
- observability 指標が主要ボトルネックを捉えられるか
- reasoning trace の redaction / retention / internal visibility が守られているか

## 4. Mandatory Red Flags

以下が見つかった場合、その milestone は差し戻す。

- Supervisor が理由なしに policy を override できる
- State Manager が next action や recommendation を返している
- provisional artifact が final deliverable に混ざっている
- QA 未通過 artifact が user-facing recommendation の根拠になっている
- agent identity を caller payload から自由入力させている
- masked / inaccessible field を frontend で推測補完している
- Human Review Reasoning Trace が agent input や GraphState 正本に再注入されている

## 5. Suggested Review Ritual

各 milestone で以下の順にレビューする。

1. schema diff を確認する
2. mock / API payload を確認する
3. 最短 happy path を実行する
4. 代表的な failure / edge case を 1 つ通す
5. governance と visibility の崩れがないか確認する

## 6. Review Output Template

レビュー結果は少なくとも以下で記録する。

```text
Milestone:
Reviewer:
Date:

Pass/Fail:

Findings:
- 

Open Questions:
- 

Required Changes:
- 
```
