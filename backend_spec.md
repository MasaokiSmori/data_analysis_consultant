# Backend Specification

## 1. Overview

本プロダクトは、顧客社内データベースと周辺メタデータを用いてデータ分析コンサルティング業務を遂行する、LangGraph ベースのマルチエージェントオーケストレーション基盤である。`Supervisor` が全体 PM として複数の `Specialist` を統括し、課題整理、指標定義、データ理解、分析実行、品質保証、示唆導出、提案作成までを一貫して進める。

本システムは単発入力を即時に分析へ変換することを前提としない。案件の初期段階では要件、ゴール、制約、期待アウトプットが曖昧であるため、分析開始前および主要フェーズの節目でユーザーと対話し、中間生成物を確認しながら進行する。

## 2. Design Principles

- `Supervisor-first orchestration`
  task の作成、割り当て、再割り当て、差し戻し、完了判定は `Supervisor` が行う。
- `No direct specialist-to-specialist messaging`
  Specialist 同士は互いの存在を知るが、直接依頼・指示・相談してはならない。
- `Parallel-first execution`
  依存関係がない task は並列実行する。
- `Quality over shortest path`
  最短経路より分析品質、妥当性、説明可能性を優先する。
- `Human-in-the-loop by default`
  要件が曖昧な段階では execution を始めず、対話と中間確認を優先する。
- `Phase-gated project flow`
  分析はフェーズ分割し、必要に応じて user review を挟む。
- `Structured recommendations`
  Specialist は task 完了時または実行不能時に、理由付きの複数候補 recommendation を Supervisor に返せる。
- `State compression`
  生 state を広く渡さず、State Manager が compact context を返す。
- `Restricted read-path`
  `query_state_view` は read-only だが、agent ごとに許可された view のみ返す。
- `State Manager as read-path API`
  State Manager は state 正本管理者であり、read-path service / API として要約を返す。write-path は `state_update_node` に限定する。graph には State Manager 本体を node として埋め込まず、service を利用する node を置く。
- `Typed and deterministic state access`
  `query_state_view` は自由文ではなく型付き request のみを受け付ける。State Manager は registered view に対する deterministic な projection / filter / field selection のみを行い、ranking、recommendation、priority judgement を返さない。
- `Plan before execute`
  SQL / Python / Causal / QA は runtime 起動前に `execution_plan` を作成し、approval routing を通過した後にのみ実行する。
- `Justified override over unconditional override`
  Supervisor の override は常に policy より優先されるのではなく、policy の適用が不適切である理由、override しない場合の不利益、override 後の緩和策を明示した `justified override` のみ有効とする。根拠不十分な Supervisor 判断は policy を上書きしない。
- `Opt-in prior knowledge`
  過去知見は prompt に自動注入せず、必要時のみ tool 経由で取得し、現在案件の成果物と混同しない。
- `Reproducibility by design`
  artifact は snapshot に紐づけ、invocation は prompt version / model 設定 / seed を記録する。
- `Hallucination containment`
  SQL / Python / narrative claim には事前検証、事後サニティチェック、出典引用を強制する。
- `Human-review-only reasoning traces`
  node / agent の判断要約は人間レビュー用 trace として別保存してよいが、GraphState の正本には含めず、他 agent の入力や再計画の根拠に再注入してはならない。

## 3. Architecture and Phases

### 3.1 Layers

1. `Orchestration Layer`
   LangGraph の状態遷移、node 実行、並列制御。
2. `Coordination Layer`
   `Supervisor`、`State Manager service`、`Advisor` による意思決定、state 管理、助言。
3. `Execution Layer`
   各 Specialist による実務 execution。
4. `Tool / Runtime Layer`
   SQL、Python、metadata、artifact storage。
5. `Interaction Layer`
   チャット UI、フェーズ確認 UI、中間生成物表示、承認・差し戻し操作。

### 3.2 Human Interaction Model

以下ではユーザー対話または確認を優先する。

- 初期要件が曖昧で、分析目的が固定されていない
- 指標定義や対象期間に複数解釈がある
- 分析方針に大きなトレードオフがある
- 中間生成物の解釈がビジネス文脈依存である
- 最終示唆や提案が意思決定に直接影響する

### 3.3 Phase Model

1. `Requirement Alignment`
2. `Scoping and Metric Definition`
3. `Data Understanding`
4. `Analytical Execution`
5. `Validation and QA`
6. `Insight and Recommendation`
7. `Delivery and Iteration`

必要に応じて前フェーズに戻れる。

## 4. Agent Topology

### 4.1 Agent List

- `Supervisor`
- `State Manager`
- `Advisor`
- `Scoping Specialist`
- `Metric Definition Specialist`
- `SQL Specialist`
- `Python Specialist`
- `Data Validation / QA Specialist`
- `Causal / Experimentation Specialist`
- `Insight Specialist`
- `Business Recommendation / Reporting Specialist`

### 4.2 Team Knowledge

全 agent は `team_directory` により他 agent の存在と役割を知る。ただし、それは協働のためのドメイン知識であり、直接通信の許可ではない。

```json
{
  "team_directory": [
    {
      "agent_name": "sql_specialist",
      "role": "Handles SQL-based data exploration, aggregation, and validation support"
    }
  ]
}
```

## 5. Agent Responsibilities

### 5.1 Coordination Agents

- `Supervisor`
  役割: 全体 PM。task 分解、依存整理、並列判断、品質担保、最終統合。
  やること: dispatch、recommendation 採否、Advisor 相談、QA 差し込み、完了判定、補助コンポーネントの裁定結果に対する承認または override。
  やらないこと: SQL / Python の直接実行、専門成果物の代行、定型ルール判定の毎回手作業での再実施。
  備考: Supervisor は「全部を直接判断する人」ではなく最終裁定者とする。routing、phase gate、plan preflight、execution readiness 判定は補助コンポーネントで前処理し、Supervisor は高リスク・例外・override 判断に集中する。

- `State Manager`
  役割: state 正本管理と compact context の提供を担う read-path service。
  やること: task / artifact / decision / recommendation の整合管理、registered view の deterministic な projection / filter / field selection、テンプレート化された事実列挙 summary 生成、`query_state_view` 応答、summary cache 管理、query schema 検証。
  やらないこと: 最終判断、分析妥当性判断、提案判断、重要度付け、優先順位付け、次アクション提案、root-cause explanation。

- `Advisor`
  役割: Supervisor の壁打ち相手。
  やること: 代替案提示、リスク指摘、批判的レビュー。
  やらないこと: 自律 task 開始、直接依頼、最終判断。
  備考: read 権限は Supervisor と対等、write / dispatch 権限は持たない。自由相談は許可するが、少なくとも `high-risk task`、同一 plan の差し戻し 2 回以上、phase 移行前では mandatory consult を行う。

### 5.2 Specialist Agents

- `Scoping Specialist`
  課題定義、論点分解、目的整理、分析計画立案。

- `Metric Definition Specialist`
  KPI / 指標定義、分母分子、除外条件、定義衝突解消。

- `SQL Specialist`
  SQL による基礎理解、高度集計、指標算出、中間 dataset 作成。Plan-Execute 対象。

- `Python Specialist`
  EDA、統計、可視化、機械学習、シミュレーション。Plan-Execute 対象。

- `Data Validation / QA Specialist`
  データ品質、集計妥当性、定義整合、解釈飛躍レビュー。Plan-Execute 対象だが軽量 plan は review checklist。内部的には stage-based に運用し、必要に応じて `data_quality_review`、`analytical_consistency_review`、`narrative_claim_review` を段階的に実施する。

- `Causal / Experimentation Specialist`
  AB テスト、施策評価、因果推論。Plan-Execute 対象。

- `Insight Specialist`
  validated artifact の解釈、示唆整理、原因仮説構造化。

- `Business Recommendation / Reporting Specialist`
  示唆の提案化、優先順位付け、報告構成整理。

## 6. Task Routing and Execution Rules

### 6.1 Dispatch Rules

- すべての task は `Supervisor` が各 Specialist に割り当てる。
- Specialist は他 Specialist に直接 task を発行できない。
- Specialist は task 完了時または実行不能時に、0 件以上 3 件以下の推薦候補を返せる。
- Supervisor は recommendation を `accept`、`reject`、`modify`、`defer` できる。

### 6.2 Parallelism Rules

- 依存関係がない task は並列実行可能。
- 同一 artifact を競合更新する task は並列化しない。
- 品質向上のための重複レビューや別経路検証は並列化してよい。

### 6.3 Recommendation Contract

recommendation は単一候補ではなく `recommendation_set` として返す。

```json
{
  "source_task_id": "task_034",
  "source_agent": "sql_specialist",
  "recommendation_candidates": [
    {
      "candidate_agent": "python_specialist",
      "reason": "Further modeling is needed to evaluate non-linear effects",
      "expected_output": "Model comparison and feature importance summary",
      "priority": "high",
      "rank": 1
    },
    {
      "candidate_agent": "insight_specialist",
      "reason": "Current findings may already be interpretable",
      "expected_output": "Narrative synthesis of findings",
      "priority": "medium",
      "rank": 2
    }
  ]
}
```

Supervisor は 1 件採用、複数採用、全 reject のいずれも可能。

### 6.4 Plan-Execute Contract

SQL / Python / Causal / QA は以下の流れを取る。

1. Supervisor が task を dispatch する
2. Specialist が必要なら `query_state_view` で関連 context を取得する
3. Specialist が `emit_execution_plan` で `execution_plan` artifact を作成する
4. `state_update_node` が task を `awaiting_plan_approval_tasks` に移し、`plan_status = "pending_approval"` にする
5. `Plan Reviewer` と `Execution Readiness Policy` が plan の形式・安全性・リスク帯を評価する
6. 低リスク plan は policy に基づく delegated approval component が `decide_execution_plan` を実行してよい
7. 中高リスク plan、policy 不一致 plan、override 必要 plan は Supervisor が `decide_execution_plan` を裁定する
8. `approved` の場合のみ task は `active_tasks` に移る
9. 実行完了後、Specialist は `plan_vs_actual_diff` artifact を生成する

QA Specialist への適用:

- QA は stage ごとに独立した `execution_plan` を emit する
- `data_quality_review` が block または `rework_required` の場合、後続 stage の plan は emit しない
- `analytical_consistency_review` と `narrative_claim_review` は前 stage の結果に応じて emit / skip を決める

`execution_plan` には少なくとも以下を含む。

- task_id と目的
- 使用予定 tool
- 参照予定 table / column
- 想定出力 artifact
- 主要ステップ
- 概略コスト
- リスクと代替案

plan approval のベースルール:

- allowlist 逸脱、未解決の metric ambiguity、高コスト、破壊的実行、因果識別戦略を含む plan は Supervisor 承認必須
- 既知テンプレートに一致し、低コストかつ read-only な plan は delegated approval component による裁定を許可できる
- delegated approval component が裁定した場合も、その根拠 policy と評価結果を decision log に残す
- Supervisor は `justified override` の条件を満たす場合にのみ delegated decision を override できる
- 依拠する metric definition artifact

### 6.5 Risk Classification and Delegated Approval

`Execution Readiness Policy` は少なくとも以下の観点で risk を分類する。

- `execution_cost`
  低コスト / 中コスト / 高コスト
- `destructiveness`
  read-only か、外部副作用や state mutation を伴うか
- `definition_stability`
  metric definition や対象期間が安定しているか
- `methodological_risk`
  因果推論、統計的仮定、複雑モデリングを含むか
- `business_impact`
  user-facing deliverable や重要判断に直結するか
- `data_sensitivity`
  機微データや tenant 境界をまたぐ参照可能性があるか

delegated approval の許可条件:

- read-only である
- allowlist 内の参照に限定される
- definition ambiguity が未解決でない
- 高感度データを含まない
- 既知テンプレートまたは事前定義 policy に一致する
- 高リスクな business impact を持たない

delegated approval の禁止条件:

- destructive / write 系 execution
- causal / experimentation を伴う execution
- high-cost plan
- metric ambiguity 未解決
- user-facing narrative の根拠を直接生成する重要 plan
- tenant / project 境界に関わる参照

### 6.6 Override Policy

Supervisor の override は、policy を無効化する特権ではなく、例外申請として扱う。override を成立させるには、少なくとも以下を満たす。

- `override_target`
  どの policy / evaluator / delegated decision を上書きするかを明示する
- `override_reason_type`
  `missing_context`, `policy_gap`, `exceptional_business_urgency`, `false_positive_risk_block`, `human_instruction_priority` のいずれかを指定する
- `override_rationale`
  policy の既定判断が今回不適切である理由を説明する
- `expected_risk_if_not_overridden`
  override しない場合の不利益を説明する
- `expected_risk_if_overridden`
  override 後に増えるリスクを説明する
- `mitigation_plan`
  追加 QA、追加 review、user clarification などの緩和策を提示する
- `override_scope`
  `task`, `phase`, `project` のいずれに適用するかを明示する

override のベースルール:

- rationale が弱い、または mitigation がない override は不成立とする
- `high-risk` override は mandatory advisor consult 後でなければ成立しない
- override は `soft_override`, `temporary_override`, `hard_override` のいずれかとして記録する
- override は policy を恒久的に無効化せず、今回の適用範囲に限定する
- override された policy は後続の continuous improvement 対象としてレビュー候補に登録する

## 7. Context, State Access, and Cache

### 7.1 Why State Manager Exists

生 state を広く渡すと、context window 圧迫、不必要な履歴への引きずり、一貫性崩壊が起きやすい。このため State Manager service が state の窓口となり、用途ごとの compact context を返す。

### 7.2 State Access Policy

State Manager は自由文の相談相手ではなく、型付き request に対して deterministic に応答する read-path service として扱う。問い合わせは原則 `view_name + scope + filters + fields + response_format` で構成し、schema に一致しない要求や許可されていない view / filter / field は reject する。

`query_state_view` の caller identity は tool runtime layer が authenticated caller から自動 inject する。State Manager はこの injected identity を用いて view 権限と field 権限を判定する。caller が自由に `requesting_agent` を偽装することはできない。

- `Supervisor` は全 summary view と必要時の drill-down view を読める
- `Advisor` は `Supervisor` と同等の read 権限を持つ
- `Specialist` は task 遂行に必要な compact view のみ読める
- Specialist 同士の直接通信は禁止し、read-path は常に State Manager 経由とする
- `query_state_view` は `ToolCallRecord` に記録し、過剰呼び出しを監視する
- State Manager は ranking、recommendation、priority judgement、next-action suggestion を返さない
- 自然言語返却が必要な場合も、事前定義テンプレートによる事実列挙に限定する
- graph node は State Manager service を呼び出す利用者であり、State Manager service 自体ではない

役割別の基本 view 権限:

- `Scoping`: `project_snapshot`, `user_alignment_summary`, `pending_user_review_summary`
- `Metric Definition`: metric / schema 関連 `artifact_index`、定義関連 `decision_context`
- `SQL`, `Python`, `Causal`: task-scoped `artifact_index`, `qa_status_summary`, `decision_context`
- `QA`: `artifact_index`, `qa_status_summary`, `decision_context`, task-scoped `recommendation_digest`
- `Insight`: validated artifact 向け `artifact_index`, `qa_status_summary`, `project_snapshot`
- `Business Recommendation`: `project_snapshot`, approved artifact 向け `artifact_index`, `user_alignment_summary`

例外的に未許可 view が必要な場合、Supervisor が task 単位で一時許可できる。その履歴は `SupervisorDecision` と `ToolCallRecord` に残す。

State Manager のベースルール:

- registered view だけを返す
- filter key は view ごとの許可リストからのみ受け付ける
- fields は view ごとの許可リストからのみ受け付ける
- 欠損値や未確定情報は補完推論せず、そのまま欠損として返す
- `important`, `recommended`, `should`, `best`, `risky` などの judgement 語を返却に含めない

### 7.3 Summary Cache Policy

- State Manager は `query_state_view` 応答をキャッシュしてよい
- cache key は少なくとも `state_version`, `view_name`, `scope`, `requesting_agent`, `summary_prompt_version`
- 同一 key には再要約せず cache を返す
- 無効化は TTL より `state_version` 変化を優先する
- cache は global state 本体ではなく、State Manager 専用 summary store に置く

### 7.4 Memory Model

本システムは agent の内的な長期記憶に依存せず、外部化された memory を正本として扱う。memory は少なくとも以下の 4 層に分ける。

- `Project Memory`
  案件全体の恒久的前提。用語、目的、制約、合意済み定義、対象範囲を保持する。
- `Decision Memory`
  重要判断とその理由。却下案、override、advisor consult、review 結果を保持する。
- `Working Memory`
  現在の phase / task 群に対する短中期の実務記憶。`project memory`, `decision memory` と同等の主要 memory 層として扱う。
- `Agent Episodic Memory`
  各 agent が自分の直近の作業連続性を保つための bounded short-term memory。正本ではない。

`Working Memory` は少なくとも以下を持つ。

- current phase の主目的
- 直近重要論点
- 現在の block 項目
- provisional 前提
- 次 gate で確認すべき事項
- 現 phase で重点的に参照すべき artifact / decision

`Agent Episodic Memory` のベースルール:

- agent ごとに直近 `n=2` 件まで保持する
- raw 会話全文や未圧縮 reasoning は保持しない
- 保持するのは、直近で参照した重要 artifact、直近の block / failure、直近の state query 要約、直近の review / correction などの構造化要約に限定する
- 正式な前提や長期判断根拠としては使わず、必要な内容は `project memory`, `decision memory`, `working memory` に昇格してから使う
- State Manager と state update 経路の管理下に置き、agent の私有長期メモにはしない

更新責務:

- `working memory` は `phase_gate_node`, `user_review_gate_node`, `state_manager_refresh_node` のタイミングで再生成または更新する
- `agent episodic memory` は `state_update_node` が invocation 結果を受けて圧縮更新し、agent ごとに直近 `n=2` 件を保つ
- stale な `working memory` は phase 変更、重要 decision、user review 応答、重大 QA finding の発生時に invalid とみなし再計算する

## 8. Core Schemas

### 8.1 Task Object

```json
{
  "task_id": "task_001",
  "assigned_agent": "sql_specialist",
  "objective": "Understand the monthly trend of active users and conversion",
  "business_context": "The client wants to understand the recent slowdown",
  "phase": "data_understanding",
  "input_context": [],
  "input_artifacts": [],
  "constraints": [],
  "expected_output": [],
  "priority": "high",
  "status": "queued",
  "dependencies": [],
  "output_artifacts": [],
  "recommendations": [],
  "review_notes": [],
  "blocking_reason": null
}
```

### 8.2 Artifact Contract

```json
{
  "artifact_id": "artifact_017",
  "artifact_type": "metric_definition_sheet",
  "producer_agent": "metric_definition_specialist",
  "phase": "scoping_and_metric_definition",
  "summary": "Draft metric definitions for active user and conversion",
  "storage_uri": "artifact://project_x/metric_definition_v1.json",
  "qa_required": true,
  "qa_status": "unreviewed",
  "ui_renderer": "document",
  "visibility": "user_review",
  "data_snapshot_id": "snapshot_2026_04_24T03_00Z",
  "producer_invocation_id": "invocation_0042"
}
```

`visibility` は `internal`, `user_review`, `final_delivery` を取りうる。過去知見は visibility を増やさず、`reference_summary` により補助情報として扱う。

### 8.3 User Review Request / Response

```json
{
  "review_id": "review_005",
  "phase": "scoping_and_metric_definition",
  "artifact_ids": ["artifact_017"],
  "prompt_to_user": "Please confirm whether these metric definitions match your business interpretation.",
  "allowed_actions": ["approve", "request_changes", "clarify", "pause"],
  "blocking_scope": "phase",
  "deadline": null
}
```

```json
{
  "review_id": "review_005",
  "action": "request_changes",
  "comments": "Active user should exclude internal staff accounts.",
  "requested_changes": ["Revise active user definition"],
  "approved_scope": []
}
```

## 9. Tooling Model

### 9.1 Shared Knowledge Sources

- `table.csv`
- `datamart spec directory`

### 9.2 Tooling Principles

- tool は単なる便利機能ではなく、責務分離と安全性の境界として扱う
- すべての tool は原則 `strict JSON I/O` を持つ
- 重い出力は state に直接埋め込まず、`artifact_id + summary + storage_uri` を返す
- tool failure は自然言語例外ではなく構造化エラーで返す
- runtime tool は権限を限定し、execution 責務を持つ agent のみに開放する
- `query_state_view` は compact context を返す read-path API として扱う
- `query_state_view` は typed query に対して deterministic に応答し、judge しない

### 9.3 Common Tool Sets

tool は以下の 4 群に分類する。

1. `Shared reference tools`
   `query_state_view`, `load_artifact`, `list_tables`, `get_table_summary`, `get_datamart_spec`
2. `Execution tools`
   `run_sql`, `run_python`, `validate_sql`, `estimate_sql_cost`, `run_data_sanity_check`
3. `Governance tools`
   `emit_execution_plan`, `decide_execution_plan`, `classify_execution_risk`, `evaluate_execution_readiness`, `evaluate_routing_policy`, `evaluate_phase_gate`, `evaluate_advisor_consult_trigger`, `submit_override_request`, `validate_override_request`, `get_artifact_lineage`
4. `QA governance tools`
   `run_qa_stage`, `evaluate_qa_gates`, `mark_artifact_quality_stage`
5. `Knowledge retrieval tools`
   `search_metric_registry`, `search_past_findings`
6. `Fallback and access tools`
   `record_provisional_mode`, `resolve_user_wait_policy`, `check_access_scope`

### 9.4 Required Tools

- `list_tables`
- `get_table_summary`
- `get_datamart_spec`
- `run_sql`
- `run_python`
- `store_artifact`
- `load_artifact`
- `validate_sql`
- `estimate_sql_cost`
- `run_data_sanity_check`
- `search_metric_registry`
- `search_past_findings`
- `get_artifact_lineage`
- `query_state_view`
- `emit_execution_plan`
- `decide_execution_plan`
- `classify_execution_risk`
- `evaluate_execution_readiness`
- `evaluate_routing_policy`
- `evaluate_phase_gate`
- `evaluate_advisor_consult_trigger`
- `submit_override_request`
- `validate_override_request`
- `run_qa_stage`
- `evaluate_qa_gates`
- `mark_artifact_quality_stage`
- `record_provisional_mode`
- `resolve_user_wait_policy`
- `check_access_scope`

補足:

- `search_metric_registry` は過去定義を pull-only で検索する
- `search_past_findings` は過去 insight / pitfall を検索し、取得結果は `reference_summary` として扱う
- `query_state_view` は許可 view のみ指定可能
- `query_state_view` は自由文 query を受け付けない
- `emit_execution_plan` は artifact 作成のみで、state 遷移は `state_update_node` が行う
- `decide_execution_plan` は Supervisor に加え、policy 上許可された delegated approval component から呼べる
- `decide_execution_plan` による override は `justified override` payload を必須とする
- `classify_execution_risk` と `evaluate_execution_readiness` は policy version を返す
- `evaluate_routing_policy`, `evaluate_phase_gate`, `evaluate_advisor_consult_trigger` は pure-function policy tool として `supervisor_node` 内から呼ばれる
- `run_qa_stage` は単一の大きな QA ではなく stage 単位で実行する
- `check_access_scope` は tenant / role / artifact / field レベルの可否を返す

### 9.5 Agent Tool Matrix

- `Supervisor`
  `query_state_view`, `load_artifact`, `get_artifact_lineage`, `decide_execution_plan`, `evaluate_routing_policy`, `evaluate_phase_gate`, `evaluate_advisor_consult_trigger`, `submit_override_request`, `validate_override_request`, `search_metric_registry`, `search_past_findings`
- `Advisor`
  `query_state_view`, `load_artifact`, `get_artifact_lineage`, `check_access_scope`, `search_metric_registry`, `search_past_findings`
- `Scoping Specialist`
  `query_state_view`, `load_artifact`, `list_tables`, `get_table_summary`, `get_datamart_spec`, `search_metric_registry`, `search_past_findings`
- `Metric Definition Specialist`
  `query_state_view`, `load_artifact`, `list_tables`, `get_table_summary`, `get_datamart_spec`, `search_metric_registry`
- `SQL Specialist`
  `query_state_view`, `load_artifact`, `list_tables`, `get_table_summary`, `get_datamart_spec`, `validate_sql`, `estimate_sql_cost`, `emit_execution_plan`, `classify_execution_risk`, `evaluate_execution_readiness`, `run_sql`
- `Python Specialist`
  `query_state_view`, `load_artifact`, `list_tables`, `get_table_summary`, `get_datamart_spec`, `emit_execution_plan`, `classify_execution_risk`, `evaluate_execution_readiness`, `run_python`
- `Data Validation / QA Specialist`
  `query_state_view`, `load_artifact`, `get_artifact_lineage`, `emit_execution_plan`, `validate_sql`, `estimate_sql_cost`, `run_sql`, `run_python`, `run_data_sanity_check`, `run_qa_stage`, `evaluate_qa_gates`, `mark_artifact_quality_stage`
- `Causal / Experimentation Specialist`
  `query_state_view`, `load_artifact`, `get_artifact_lineage`, `emit_execution_plan`, `classify_execution_risk`, `evaluate_execution_readiness`, `run_python`, 必要に応じて `run_sql`
- `Insight Specialist`
  `query_state_view`, `load_artifact`, `get_artifact_lineage`, `evaluate_qa_gates`, `search_past_findings`
- `Business Recommendation / Reporting Specialist`
  `query_state_view`, `load_artifact`, `evaluate_qa_gates`, `search_past_findings`
- `State Manager`
  orchestration state と artifact metadata のみを扱い、自ら `query_state_view` は呼ばない
- `delegated approval component`
  `decide_execution_plan`, `validate_override_request`

### 9.6 Tool Access Policy

- 全 agent は自分に許可された tool のみ使用できる
- metadata / artifact 読み取り系 tool は広く開放してよい
- `run_sql` / `run_python` は execution specialist のみに開放する
- `decide_execution_plan` は `Supervisor` と policy 上許可された delegated approval component のみが使用できる
- `evaluate_routing_policy`, `evaluate_phase_gate`, `evaluate_advisor_consult_trigger` は `Supervisor` または `supervisor_node` のみが使用できる
- `submit_override_request` は `Supervisor` のみが使用できる
- `validate_override_request` は `Supervisor` と delegated approval component が参照できる
- `run_qa_stage` は `Data Validation / QA Specialist` のみが使用できる
- `mark_artifact_quality_stage` は QA gate または user review gate を通じた遷移に限定する
- `record_provisional_mode` は Supervisor または system node が使用できる
- `check_access_scope` は access 判定専用であり、権限昇格を行わない
- plan-required task では、承認済み plan がない限り runtime tool 起動を禁止する
- `query_state_view` は task 遂行に必要な最小限の view / scope に限定する

### 9.7 Priority Tool Contracts

最初に厳密 I/O を固めるべき tool は以下の 15 個である。

1. `query_state_view`
2. `load_artifact`
3. `list_tables`
4. `get_table_summary`
5. `get_datamart_spec`
6. `emit_execution_plan`
7. `decide_execution_plan`
8. `classify_execution_risk`
9. `evaluate_execution_readiness`
10. `evaluate_routing_policy`
11. `evaluate_phase_gate`
12. `evaluate_advisor_consult_trigger`
13. `submit_override_request`
14. `validate_override_request`
15. `run_qa_stage`

`evaluate_qa_gates`, `validate_sql`, `run_sql`, `run_python`, `record_provisional_mode`, `resolve_user_wait_policy`, `check_access_scope`, `mark_artifact_quality_stage` は第 2 優先群として早期に定義する。

### 9.8 Tool I/O Contracts

#### `query_state_view`

目的:
State Manager から typed / deterministic な compact context を取得する。

入力:

```json
{
  "view_name": "artifact_index",
  "filters": {
    "qa_status": "approved"
  },
  "fields": ["artifact_id", "artifact_type", "qa_status", "phase"],
  "scope": {
    "task_id": "task_034"
  },
  "response_format": "json"
}
```

`requesting_agent` は tool runtime layer が authenticated caller から自動 inject するため、呼び出し側 payload には含めない。

出力:

```json
{
  "view_name": "artifact_index",
  "state_version": 42,
  "data": [
    {
      "artifact_id": "artifact_017",
      "artifact_type": "dataset",
      "qa_status": "approved",
      "phase": "validation_and_qa"
    }
  ],
  "summary_text": "1 approved artifact matched the requested filters.",
  "cache_hit": true
}
```

制約:

- 自由文 query は受け付けない
- `view_name` は registered view のみ指定可能
- `filters` と `fields` は view ごとの許可リストに従う
- `summary_text` はテンプレート化された事実列挙に限定し、recommendation や priority judgement を含めない

#### `load_artifact`

目的:
artifact の要約または本文参照情報を取得する。

入力:

```json
{
  "artifact_id": "artifact_017"
}
```

出力:

```json
{
  "artifact_id": "artifact_017",
  "artifact_type": "metric_definition_sheet",
  "summary": "Draft metric definitions for active user and conversion",
  "storage_uri": "artifact://project_x/metric_definition_v1.json"
}
```

#### `list_tables`

目的:
利用可能テーブル一覧を返す。

入力:

```json
{
  "keyword": "orders"
}
```

出力:

```json
{
  "tables": [
    {
      "table_name": "fact_orders",
      "summary": "Order-level transactional fact table"
    }
  ]
}
```

#### `get_table_summary`

目的:
指定テーブルの意味、粒度、注意点を返す。

入力:

```json
{
  "table_name": "fact_orders"
}
```

出力:

```json
{
  "table_name": "fact_orders",
  "grain": "one row per order",
  "summary": "Primary transactional table for completed and cancelled orders",
  "warnings": ["Contains internal test orders unless filtered"]
}
```

#### `get_datamart_spec`

目的:
仕様書 CSV の参照情報を返す。

入力:

```json
{
  "table_name": "fact_orders"
}
```

出力:

```json
{
  "table_name": "fact_orders",
  "artifact_id": "artifact_spec_001",
  "summary": "Spec includes key columns, update cadence, and known caveats",
  "storage_uri": "artifact://specs/fact_orders.csv"
}
```

#### `emit_execution_plan`

目的:
execution plan artifact を作成する。

入力:

```json
{
  "task_id": "task_034",
  "planned_tools": ["validate_sql", "run_sql"],
  "planned_tables": ["fact_orders"],
  "planned_columns": ["order_id", "created_at", "customer_id"],
  "expected_outputs": ["aggregated_dataset", "finding_memo"],
  "risk_notes": ["Potentially large scan over 24 months of data"]
}
```

出力:

```json
{
  "task_id": "task_034",
  "artifact_id": "artifact_plan_034",
  "summary": "Execution plan created and ready for approval routing"
}
```

#### `decide_execution_plan`

目的:
plan の実行可否を裁定する。裁定者は Supervisor、または policy 上許可された delegated approval component とする。

入力:

```json
{
  "task_id": "task_034",
  "execution_plan_artifact_id": "artifact_plan_034",
  "decision": "approve",
  "comment": "Plan is aligned with metric definition and cost estimate is acceptable",
  "decision_actor": "supervisor",
  "decision_basis": "manual_review",
  "policy_version": "approval_policy_v3",
  "override": null
}
```

出力:

```json
{
  "task_id": "task_034",
  "decision": "approve",
  "decision_id": "decision_012",
  "decision_actor": "supervisor",
  "policy_version": "approval_policy_v3"
}
```

override 付き入力例:

```json
{
  "task_id": "task_034",
  "execution_plan_artifact_id": "artifact_plan_034",
  "decision": "approve",
  "comment": "Temporary override due to urgent user-directed read-only clarification workflow",
  "decision_actor": "supervisor",
  "decision_basis": "justified_override",
  "policy_version": "approval_policy_v3",
  "override": {
    "override_target": "execution_readiness_policy",
    "override_level": "temporary_override",
    "override_reason_type": "human_instruction_priority",
    "override_rationale": "The user explicitly requested interim exploratory output before full metric lock.",
    "expected_risk_if_not_overridden": "Project stalls and user cannot validate direction.",
    "expected_risk_if_overridden": "Result may rely on provisional assumptions.",
    "mitigation_plan": ["mark output as provisional", "mandatory analytical_consistency_review before delivery"],
    "override_scope": "task"
  }
}
```

#### `classify_execution_risk`

目的:
execution plan を risk policy に基づき `low / medium / high` に分類する。

入力:

```json
{
  "task_id": "task_034",
  "execution_plan_artifact_id": "artifact_plan_034",
  "policy_version": "approval_policy_v3"
}
```

出力:

```json
{
  "task_id": "task_034",
  "risk_level": "medium",
  "policy_version": "approval_policy_v3",
  "factors": [
    {"name": "execution_cost", "level": "medium"},
    {"name": "destructiveness", "level": "low"},
    {"name": "business_impact", "level": "medium"}
  ]
}
```

#### `evaluate_execution_readiness`

目的:
execution plan が delegated approval 可能か、Supervisor 承認必須か、reject すべきかを返す。

入力:

```json
{
  "task_id": "task_034",
  "execution_plan_artifact_id": "artifact_plan_034",
  "risk_level": "medium",
  "policy_version": "approval_policy_v3"
}
```

出力:

```json
{
  "task_id": "task_034",
  "readiness_decision": "requires_supervisor",
  "policy_version": "approval_policy_v3",
  "reasons": ["business_impact_requires_manual_review"]
}
```

#### `evaluate_routing_policy`

目的:
Supervisor が dispatch 前に routing policy の既定提案を取得する。

入力:

```json
{
  "project_id": "project_acme_growth_001",
  "phase": "validation_and_qa",
  "open_question_ids": ["question_014"],
  "active_task_ids": ["task_061"]
}
```

出力:

```json
{
  "policy_version": "routing_policy_v2",
  "recommended_actions": [
    {
      "action": "dispatch_task",
      "assigned_agent": "metric_definition_specialist",
      "reason": "definition ambiguity blocks QA closure"
    }
  ]
}
```

#### `evaluate_phase_gate`

目的:
phase 遷移条件が満たされているかを deterministic に判定する。

入力:

```json
{
  "project_id": "project_acme_growth_001",
  "current_phase": "validation_and_qa"
}
```

出力:

```json
{
  "policy_version": "phase_gate_policy_v1",
  "gate_result": "blocked",
  "reasons": ["pending_user_review_exists", "analytical_consistency_review_not_complete"]
}
```

#### `evaluate_advisor_consult_trigger`

目的:
mandatory advisor consult 条件に該当するかを判定する。

入力:

```json
{
  "task_id": "task_066",
  "risk_level": "high",
  "plan_revision_count": 2,
  "is_phase_transition": false
}
```

出力:

```json
{
  "must_consult": true,
  "trigger_reasons": ["high_risk_task", "plan_revision_count_threshold"]
}
```

#### `submit_override_request`

目的:
Supervisor が justified override を構造化して申請する。

入力:

```json
{
  "decision_id": "decision_012",
  "override_target": "execution_readiness_policy",
  "override_level": "temporary_override",
  "override_reason_type": "human_instruction_priority",
  "override_rationale": "User explicitly requested provisional exploratory output.",
  "expected_risk_if_not_overridden": "Direction cannot be validated in time.",
  "expected_risk_if_overridden": "Output may rely on provisional assumptions.",
  "mitigation_plan": ["mark output as provisional", "mandatory analytical_consistency_review before delivery"],
  "override_scope": "task"
}
```

出力:

```json
{
  "override_request_id": "override_001",
  "status": "submitted"
}
```

#### `validate_override_request`

目的:
override request が justified override の成立条件を満たしているか検証する。

入力:

```json
{
  "override_request_id": "override_001",
  "policy_version": "approval_policy_v3"
}
```

出力:

```json
{
  "override_request_id": "override_001",
  "valid": true,
  "missing_fields": [],
  "requires_advisor_consult": true
}
```

#### `run_qa_stage`

目的:
指定 stage の QA を実行し、QA record を生成する。

入力:

```json
{
  "related_task_id": "task_052",
  "review_stage": "analytical_consistency_review",
  "artifact_ids": ["artifact_sql_034", "artifact_py_052"]
}
```

出力:

```json
{
  "qa_review_id": "qa_021",
  "review_stage": "analytical_consistency_review",
  "stage_status": "approved",
  "findings": []
}
```

#### `evaluate_qa_gates`

目的:
artifact が downstream へ進行可能かを gate ルールに基づき判定する。

入力:

```json
{
  "artifact_ids": ["artifact_py_052"],
  "target_stage": "business_recommendation"
}
```

出力:

```json
{
  "target_stage": "business_recommendation",
  "allowed": false,
  "missing_gates": ["narrative_claim_review"]
}
```

#### `mark_artifact_quality_stage`

目的:
artifact の quality stage を更新する。

入力:

```json
{
  "artifact_id": "artifact_py_052",
  "quality_stage": "QA-approved",
  "reason": "Passed analytical_consistency_review"
}
```

出力:

```json
{
  "artifact_id": "artifact_py_052",
  "quality_stage": "QA-approved"
}
```

#### `record_provisional_mode`

目的:
provisional / degraded mode の前提、欠落情報、再確認条件を記録する。

入力:

```json
{
  "task_id": "task_034",
  "mode": "provisional_continue",
  "missing_information": ["final metric definition"],
  "mitigation_plan": ["mark artifact as provisional", "request user review in next gate"]
}
```

出力:

```json
{
  "task_id": "task_034",
  "mode_record_id": "prov_009",
  "status": "recorded"
}
```

#### `resolve_user_wait_policy`

目的:
user 非応答時の `pause / remind / provisional_continue` を評価する。

入力:

```json
{
  "review_id": "review_005",
  "waiting_hours": 48,
  "blocking_scope": "phase"
}
```

出力:

```json
{
  "review_id": "review_005",
  "recommended_action": "remind",
  "reasons": ["timeout_threshold_crossed_but_project_not_high_risk"]
}
```

#### `check_access_scope`

目的:
agent が tenant / artifact / field にアクセス可能かを判定する。

入力:

```json
{
  "agent_name": "advisor",
  "artifact_id": "artifact_sql_034",
  "requested_fields": ["summary", "storage_uri"]
}
```

出力:

```json
{
  "allowed": true,
  "allowed_fields": ["summary", "storage_uri"],
  "denied_fields": []
}
```

#### `validate_sql`

目的:
SQL の静的妥当性を確認する。

入力:

```json
{
  "sql": "select customer_id, count(*) from fact_orders group by 1"
}
```

出力:

```json
{
  "valid": true,
  "errors": [],
  "warnings": []
}
```

#### `run_sql`

目的:
SQL を実行し dataset artifact を作成する。

入力:

```json
{
  "task_id": "task_034",
  "sql": "select customer_id, count(*) as orders from fact_orders group by 1"
}
```

出力:

```json
{
  "task_id": "task_034",
  "artifact_id": "artifact_sql_034",
  "summary": "Created aggregated orders-by-customer dataset",
  "storage_uri": "artifact://task_034/orders_by_customer.parquet"
}
```

#### `run_python`

目的:
Python analysis を実行し result artifact を作成する。

入力:

```json
{
  "task_id": "task_052",
  "entrypoint": "analyze_retention.py",
  "input_artifact_ids": ["artifact_sql_034"]
}
```

出力:

```json
{
  "task_id": "task_052",
  "artifact_id": "artifact_py_052",
  "summary": "Generated retention analysis charts and model summary",
  "storage_uri": "artifact://task_052/retention_analysis.json"
}
```

### 9.9 Tool Error Contract

すべての tool は失敗時に少なくとも以下の構造を返す。

```json
{
  "ok": false,
  "error_type": "schema_error",
  "message": "Unknown column: customerd_id"
}
```

## 10. Prompting Strategy

すべての agent prompt には少なくとも以下を含める。

- agent の role
- `team_directory`
- direct messaging 禁止
- recommendation フォーマット
- 品質基準
- 利用可能 tool
- 過去知見は prompt に事前埋め込みせず、必要時のみ pull すること
- 数値主張には artifact_id と該当領域の引用が必要であること
- `query_state_view` は task 遂行に必要な最小限に留めること
- `query_state_view` は typed query のみを使い、State Manager に recommendation や解釈を求めないこと
- execution specialist には `emit_execution_plan` 必須であること
- Supervisor は補助コンポーネントの裁定結果を参照しつつ、最終裁定と override に集中すること
- Supervisor は自由相談を維持しつつ、mandatory consult 条件では必ず Advisor に壁打ちすること
- QA Specialist は review stage ごとの判定基準を使い、未実施 stage があれば明示すること

## 11. Quality and Reproducibility Gates

### 11.1 Analytical Quality

- 指標定義が曖昧なまま集計を確定しない
- QA 未通過の重要 artifact を Insight / Reporting の根拠にしない
- 因果推論が必要な問いを単純集計で断定しない
- 必要に応じて追加分析、再検証、別経路レビューを行う

### 11.2 Hallucination Containment

- `validate_sql` を通過した SQL のみ `run_sql` を許可
- 重い SQL は `estimate_sql_cost` で事前見積
- 可能なら sample-first execution を行う
- dataset 生成 task は `run_data_sanity_check` を必須化
- QA は `metric_definition_sheet` との整合を必須レビューする
- narrative の数値主張には artifact_id と該当領域の引用を必須化
- 重要数値は必要に応じて SQL / Python 等の二重計算で突合する
- allowlist 外の table / column 参照は Supervisor 承認なしに行わない
- `known-value assertions` を sanity check に使えるようにする

### 11.3 QA Stage Rules

`Data Validation / QA Specialist` は単一の重いレビュー工程としてではなく、以下の stage を必要に応じて段階実行する。QA は stage ごとに独立した plan を持ち、各 stage の approve / reject / block を経て次 stage に進む。

1. `data_quality_review`
   行数、欠損、重複、粒度不整合、join 異常、期間欠落、期待値域逸脱を確認する。
2. `analytical_consistency_review`
   metric definition との整合、集計ロジック、再現性、plan と実行結果の差分、因果と相関の混同を確認する。
3. `narrative_claim_review`
   Insight / Reporting の数値主張の引用、解釈飛躍、根拠 artifact との整合を確認する。

QA のベースルール:

- dataset artifact は最低でも `data_quality_review` を通す
- Insight / Recommendation に使う artifact は `analytical_consistency_review` を通す
- user-facing narrative は `narrative_claim_review` を通す
- 各 stage は `approved`, `rework_required`, `not_applicable` のいずれかで記録する
- 未実施 stage がある場合、QA record に理由を残す
- 重大 issue を検出した stage は後続 stage を block してよい
- rule-based に判定できる項目は checklist と evaluator で先に処理し、QA Specialist は例外判断に集中する

downstream gate のベースルール:

- `Insight Specialist` は `data_quality_review` 完了済み artifact に対して `draft insight` を作成してよい
- user-facing な insight を確定するには、対応 artifact が `analytical_consistency_review` を通過している必要がある
- `Business Recommendation / Reporting Specialist` が user-facing recommendation を確定するには、対応 narrative が `narrative_claim_review` を通過している必要がある
- `draft` artifact は exploratory / provisional 用途に限定し、最終 deliverable の根拠にはできない
- `QA-approved` artifact のみが `final_delivery` visibility を持てる

### 11.4 Draft and Approved Artifact Policy

artifact の品質状態と利用可能範囲は明示的に分ける。

- `draft`
  exploratory / provisional。内部利用のみ
- `internal`
  チーム内参照可能だが、QA や user review を未完了の可能性がある
- `QA-approved`
  規定された QA gate を通過済み
- `user-reviewed`
  user review を経た中間成果物
- `final-delivery`
  最終提示可能な成果物

ベースルール:

- `draft` と `internal` は final decision の根拠にしない
- `provisional` な前提に依存する artifact は summary と visibility にその旨を明示する
- user-facing UI では `draft` / `QA-approved` / `final-delivery` を明確に区別する
- `TaskRecord.risk_level` は task の固定属性ではなく、最新に承認済みまたは審査中の execution plan に対する最新 risk 評価として扱う

### 11.5 Reproducibility

- dataset artifact は snapshot ID を持つ
- invocation は prompt version / model id / model params hash / random seed を記録する
- ML / 統計の乱数は固定する
- 同一 snapshot と設定で replay 可能であることを設計目標とする

## 12. Completion and User Review

### 12.1 Completion Criteria

少なくとも以下を満たす。

- 主要問いへの分析結果が揃っている
- 重要指標定義が明確化されている
- 必要な QA が完了している
- 示唆と提案がビジネス文脈に接続されている
- 主要 artifact が再参照可能である
- 必要な user review が完了している、または省略理由が decision log に残っている

### 12.2 Project Closure Policy

`project_context.user_signoff_required` を持つ。

- `true`: 明示的なユーザー承認が必要
- `false`: Supervisor が deliverable 完了と判断できるが、その理由を decision log に残す

### 12.3 User Review Gates

少なくとも以下のタイミングで user review 候補とする。

- Requirement Alignment 完了時
- Scoping / Metric Definition 完了時
- 初期データ理解完了時
- 主要分析結果出揃い時
- 最終示唆・提案提示前

UI には少なくとも以下を表示できるようにする。

- requirement summary
- project scope memo
- metric definition sheet
- data understanding summary
- exploratory findings summary
- QA issue summary
- insight memo
- recommendation outline

## 13. LangGraph Implementation Design

### 13.1 Graph Goals

- Supervisor 主導のハブ型制御
- Specialist 間 direct edge を作らない
- 並列 execution と再計画を自然に扱う
- state の肥大化を artifact 管理と summary view で抑える
- 失敗、差し戻し、追加検証を通常フローとして扱う

### 13.2 High-Level Graph Shape

```text
START
  -> intake_node
  -> requirement_clarification_node
  -> user_review_gate_node
  -> state_manager_refresh_node
  -> supervisor_node
      -> dispatch_router_node
          -> specialist_worker_subgraph (0..N parallel)
              -> state_update_node
              -> state_manager_refresh_node
              -> phase_gate_node
              -> user_review_gate_node
              -> supervisor_node
  -> completion_check_node
  -> END
```

Plan-Execute 対象 Specialist では `specialist_worker_subgraph` 内に plan emit -> approval routing -> execute の 1 往復が追加される。

### 13.3 Recommended Nodes

1. `intake_node`
2. `requirement_clarification_node`
3. `user_review_gate_node`
4. `state_manager_refresh_node`
   State Manager service を呼び出して summary views と working memory projection を更新する利用 node。State Manager 本体ではない。
5. `supervisor_node`
6. `dispatch_router_node`
7. `specialist_worker_node`
8. `state_update_node`
9. `phase_gate_node`
10. `completion_check_node`
11. `advisor_consult_node`
12. `error_triage_node`

`specialist_worker_node` は共通 node とし、prompt / tool / response schema / post-processing を切り替える。

State Manager service 内部責務は少なくとも以下に分ける。

- `State Registry`
  canonical state の保持と整合性管理のみを担当する
- `Query Validator`
  `query_state_view` の schema 妥当性、view / field / filter の許可判定を行う
- `State Projector`
  registered view への projection / filter / field selection を deterministic に行う
- `Template Summarizer`
  必要時のみテンプレートベースの事実列挙 summary を生成し、新しい解釈や評価語を加えない

### 13.4 Supervisor Load-Reduction Best Practices

以下は、Supervisor の責任そのものを放棄させるためではなく、余計な判断責任を減らし、本質的な PM 判断に集中させるための施策である。

1. `Routing Policy Engine`
   定型的な routing 判断は policy として先に機械化する。例として、metric 未確定時は `Metric Definition` を優先、dataset 生成後は sanity check を必須、high-risk task は QA 必須とする。Supervisor はゼロから毎回 routing を考えるのではなく、policy の提案を採用または override する。

2. `Phase Gate Evaluator`
   フェーズ遷移条件は evaluator が先に判定し、Supervisor はその結果を見て承認または override する。これにより、phase を進める / 戻すのたびに同じチェックを Supervisor が手作業で繰り返す負荷を減らす。

3. `Plan Reviewer`
   `execution_plan` は Supervisor が直接一次査読する前に、軽量な preflight review を通す。allowlist 逸脱、metric definition 未参照、コスト異常、想定出力不足などを自動検査し、Supervisor は形式確認ではなく妥当性判断に集中する。

4. `Execution Readiness Policy`
   `execution_plan` を低リスク / 中リスク / 高リスクに分類し、誰が裁定すべきかを決める。低リスク plan は delegated approval component による裁定を許可し、中高リスクや例外ケースのみ Supervisor に上げる。

5. `QA Rulebook and Checklist Evaluator`
   QA のうち deterministic に判定できる項目は rulebook と checklist evaluator で事前処理する。QA Specialist は全件を一から読むのではなく、rule hit、未解決論点、例外ケースの確認に集中する。

6. `Advisor Consultation Triggers`
   Advisor 相談は自由に行ってよいが、少なくとも `high-risk task`、同一 `execution_plan` の差し戻し 2 回以上、phase 移行前には mandatory consult を行う。mandatory consult の結果は採否にかかわらず decision log に残す。

7. `Prefer policy / evaluator / reviewer over new manager agents`
   Supervisor の負荷軽減は、まず rule layer、evaluator、lightweight reviewer で行う。安易に manager 系 agent を増やすと責務衝突が起きやすいため、まずは node / policy として実装し、それでも不足する場合のみ agent 化を検討する。

各補助コンポーネントは少なくとも以下を持つ。

- 入力 schema
- 出力 schema
- policy version または rule version
- override 可否
- decision log に残す項目

実装単位の原則:

- `Routing Policy Engine`, `Phase Gate Evaluator`, `Plan Reviewer`, `Execution Readiness Policy`, `QA Rulebook and Checklist Evaluator`, `Advisor Consultation Triggers` は独立 agent ではなく、pure-function policy tool または policy module として実装する
- これらは主に `supervisor_node` または gate node から呼ばれ、長期状態を持たない
- `Plan Reviewer` の preflight 判定は `classify_execution_risk` と `evaluate_execution_readiness` の組み合わせで具体化する

優先順位のベースルール:

- policy / evaluator / reviewer の既定判断が標準経路である
- Supervisor は `justified override` が成立した場合のみ既定判断を上書きできる
- 根拠不十分な Supervisor 判断は policy より優先されない
- override の成立条件を満たさない場合、既定判断を採用する

## 14. LangGraph State Design

### 14.1 State Separation Policy

state に保持するもの:

- task 進行状況
- decision / recommendation
- artifact metadata
- open questions
- quality status
- summary views

state に全文保持しないもの:

- 大きな SQL 結果
- DataFrame 本体
- notebook 本文全体
- 長い仕様書 CSV
- 冗長な中間 reasoning

### 14.2 Typed State Sketch

以下は実装用 Pydantic モデルの圧縮版設計例である。フィールドは全 agent の input / output、tool execution、user review、phase gate、plan approval、summary cache を追跡できる粒度を保持する。

この節の class 定義は可読性のため `Field(description=...)` を省略した圧縮表現である。実装時には、**public field すべてに `Field(description=...)` を付与することを必須とする。** これにより以下の利点がある。

- JSON Schema / OpenAPI 生成時に意味が失われにくい
- tool I/O と state schema の意図を実装者が誤解しにくい
- UI / admin console / observability 側で項目説明を再利用しやすい
- LLM に構造化出力を要求する際の補助コンテキストとしても使いやすい

特に `TaskRecord`, `ArtifactRef`, `SupervisorDecision`, `QAReviewRecord`, `UserReviewRequest`, `ToolCallRecord` の主要フィールドには description を付ける前提とする。
`WorkingMemory`, `AgentEpisodicMemoryEntry`, `HumanReviewReasoningTrace`, `GraphState` の主要フィールドにも description を付け、memory や trace の意味と寿命が分かるようにする。

description の記述ルール:

- field が何を表すかを 1 文で明示する
- enum / literal 系は「どういう判定でその値になるか」を添える
- ID 系は「どのスコープで一意か」を添える
- URI / artifact / memory 系は「正本がどこにあるか」を添える
- bool 系は `True` の意味を必ず書く
- list / dict 系は「何の集合か」「bounded か」を必要に応じて書く

```python
from __future__ import annotations
from datetime import datetime
from typing import Any, Literal
from pydantic import BaseModel, Field

AgentName = Literal[
    "supervisor", "state_manager", "advisor",
    "scoping_specialist", "metric_definition_specialist",
    "sql_specialist", "python_specialist",
    "data_validation_qa_specialist", "causal_experimentation_specialist",
    "insight_specialist", "business_recommendation_reporting_specialist",
]
PhaseName = Literal[
    "requirement_alignment", "scoping_and_metric_definition", "data_understanding",
    "analytical_execution", "validation_and_qa", "insight_and_recommendation",
    "delivery_and_iteration",
]
TaskStatus = Literal["draft", "queued", "running", "completed", "failed", "blocked", "cancelled"]
ArtifactQAStatus = Literal["unreviewed", "in_review", "approved", "rejected"]
Priority = Literal["low", "medium", "high", "critical"]
Visibility = Literal["internal", "user_review", "final_delivery"]
BlockingScope = Literal["task", "phase", "project"]
ReviewAction = Literal["approve", "request_changes", "clarify", "pause"]
ProjectStatus = Literal["draft", "active", "paused", "awaiting_user", "completed", "cancelled"]
InvocationStatus = Literal["started", "completed", "failed", "blocked", "cancelled"]
ToolCallStatus = Literal["started", "completed", "failed", "skipped"]
RiskLevel = Literal["low", "medium", "high"]
QAReviewStage = Literal[
    "data_quality_review",
    "analytical_consistency_review",
    "narrative_claim_review",
]
QAStageStatus = Literal["approved", "rework_required", "not_applicable"]
QualityStage = Literal["draft", "internal", "QA-approved", "user-reviewed", "final-delivery"]
DecisionBasis = Literal["manual_review", "delegated_policy", "justified_override"]
OverrideLevel = Literal["soft_override", "temporary_override", "hard_override"]
OverrideReasonType = Literal[
    "missing_context",
    "policy_gap",
    "exceptional_business_urgency",
    "false_positive_risk_block",
    "human_instruction_priority",
]
StateViewName = Literal[
    "project_snapshot", "active_task_summary", "artifact_index", "recommendation_digest",
    "qa_status_summary", "decision_context", "pending_user_review_summary",
    "user_alignment_summary", "phase_summary", "delivery_readiness_summary",
]
PlanStatus = Literal[
    "not_required", "draft", "pending_approval", "approved",
    "rejected", "revision_requested", "executed", "superseded",
]


class AuditFields(BaseModel):
    created_at: datetime | None = None
    updated_at: datetime | None = None
    created_by: str | None = None
    updated_by: str | None = None


class TeamMember(BaseModel):
    agent_name: AgentName
    role_summary: str
    can_dispatch_tasks: bool = False
    can_access_sql_runtime: bool = False
    can_access_python_runtime: bool = False
    can_access_metadata_tools: bool = True
    can_access_artifact_store: bool = True
    allowed_state_views: list[StateViewName] = Field(default_factory=list)


class ProjectContext(BaseModel):
    project_id: str
    title: str = ""
    client_name: str = ""
    requester_name: str = ""
    business_background: str = ""
    primary_goal: str = ""
    success_criteria: list[str] = Field(default_factory=list)
    constraints: list[str] = Field(default_factory=list)
    out_of_scope: list[str] = Field(default_factory=list)
    expected_deliverables: list[str] = Field(default_factory=list)
    target_decision_makers: list[str] = Field(default_factory=list)
    user_signoff_required: bool = True
    project_retry_ceiling: int = 20
    current_status: ProjectStatus = "draft"


class OpenQuestion(BaseModel):
    question_id: str
    question_text: str
    owner_agent: AgentName | None = None
    phase: PhaseName
    severity: Priority = "medium"
    resolved: bool = False
    resolution_note: str = ""


class WorkingMemory(BaseModel):
    phase_objective: str = ""
    focus_points: list[str] = Field(default_factory=list)
    blocked_items: list[str] = Field(default_factory=list)
    provisional_assumptions: list[str] = Field(default_factory=list)
    next_gate_checks: list[str] = Field(default_factory=list)
    priority_artifact_ids: list[str] = Field(default_factory=list)
    priority_decision_ids: list[str] = Field(default_factory=list)


class AgentEpisodicMemoryEntry(BaseModel):
    entry_id: str
    agent_name: AgentName
    summary: str
    recent_artifact_ids: list[str] = Field(default_factory=list)
    recent_decision_ids: list[str] = Field(default_factory=list)
    recent_query_summaries: list[str] = Field(default_factory=list)
    recent_failures_or_blocks: list[str] = Field(default_factory=list)
    recent_reviews_or_corrections: list[str] = Field(default_factory=list)
    created_at: datetime | None = None


class RecommendationCandidate(BaseModel):
    source_task_id: str
    source_agent: AgentName
    candidate_agent: AgentName
    reason: str
    expected_output: str
    priority: Priority = "medium"
    rank: int = 1
    dependency_ids: list[str] = Field(default_factory=list)
    additional_notes: str = ""


class RecommendationSet(BaseModel):
    recommendation_set_id: str
    source_task_id: str
    source_agent: AgentName
    candidates: list[RecommendationCandidate] = Field(default_factory=list)
    accepted: bool | None = None
    supervisor_comment: str = ""


class TaskInputBundle(BaseModel):
    input_context: list[str] = Field(default_factory=list)
    input_artifact_ids: list[str] = Field(default_factory=list)
    input_question_ids: list[str] = Field(default_factory=list)
    referenced_decision_ids: list[str] = Field(default_factory=list)
    referenced_user_review_ids: list[str] = Field(default_factory=list)


class TaskOutputBundle(BaseModel):
    output_artifact_ids: list[str] = Field(default_factory=list)
    generated_question_ids: list[str] = Field(default_factory=list)
    generated_recommendation_set_ids: list[str] = Field(default_factory=list)
    generated_review_ids: list[str] = Field(default_factory=list)


class TaskRecord(BaseModel):
    task_id: str
    phase: PhaseName
    assigned_agent: AgentName
    title: str
    objective: str
    business_context: str = ""
    constraints: list[str] = Field(default_factory=list)
    expected_output: list[str] = Field(default_factory=list)
    priority: Priority = "medium"
    status: TaskStatus = "draft"
    depends_on_task_ids: list[str] = Field(default_factory=list)
    blocks_task_ids: list[str] = Field(default_factory=list)
    input_bundle: TaskInputBundle = Field(default_factory=TaskInputBundle)
    output_bundle: TaskOutputBundle = Field(default_factory=TaskOutputBundle)
    blocking_reason: str | None = None
    failure_reason: str | None = None
    last_failure_kind: str | None = None
    retry_count: int = 0
    max_retries: int = 3
    allowed_table_ids: list[str] = Field(default_factory=list)
    allowed_column_ids: list[str] = Field(default_factory=list)
    plan_required: bool = False
    plan_status: PlanStatus = "not_required"
    risk_level: RiskLevel = "medium"
    provisional_allowed: bool = False
    execution_plan_artifact_id: str | None = None
    plan_approval_decision_id: str | None = None
    plan_revision_count: int = 0
    plan_vs_actual_diff_artifact_id: str | None = None
    review_notes: list[str] = Field(default_factory=list)
    supervisor_instructions: list[str] = Field(default_factory=list)
    completion_summary: str = ""
    audit: AuditFields = Field(default_factory=AuditFields)

# 実装時の記述例:
# task_id: str = Field(description="Stable task identifier within the project")
# risk_level: Literal["low", "medium", "high"] = Field(
#     default="medium",
#     description="Risk tier assigned by execution readiness policy"
# )
# provisional_allowed: bool = Field(
#     default=False,
#     description="Whether provisional execution is allowed under degraded mode"
# )


class ArtifactRef(BaseModel):
    artifact_id: str
    artifact_type: str
    producer_agent: AgentName
    producer_task_id: str
    producer_invocation_id: str | None = None
    phase: PhaseName
    title: str
    summary: str
    storage_uri: str
    format: str = ""
    schema_version: str = "1.0"
    qa_required: bool = True
    qa_status: ArtifactQAStatus = "unreviewed"
    ui_renderer: str = "document"
    visibility: Visibility = "internal"
    quality_stage: QualityStage = "draft"
    provisional: bool = False
    tags: list[str] = Field(default_factory=list)
    upstream_artifact_ids: list[str] = Field(default_factory=list)
    data_snapshot_id: str = ""
    sanity_artifact_id: str | None = None
    reference_summary: str = ""
    audit: AuditFields = Field(default_factory=AuditFields)


class ToolCallRecord(BaseModel):
    tool_call_id: str
    invocation_id: str
    agent_name: AgentName
    task_id: str
    tool_name: str
    status: ToolCallStatus = "started"
    request_summary: str = ""
    output_artifact_ids: list[str] = Field(default_factory=list)
    error_message: str = ""
    requested_state_view: StateViewName | None = None
    state_version_at_call: int | None = None
    started_at: datetime | None = None
    finished_at: datetime | None = None
    trace_id: str = ""
    span_id: str = ""
    parent_span_id: str = ""


class AgentIOEnvelope(BaseModel):
    summary_context: str = ""
    task_snapshot: dict[str, Any] = Field(default_factory=dict)
    accessible_artifact_ids: list[str] = Field(default_factory=list)
    accessible_question_ids: list[str] = Field(default_factory=list)
    accessible_decision_ids: list[str] = Field(default_factory=list)
    allowed_tools: list[str] = Field(default_factory=list)


class AgentResultEnvelope(BaseModel):
    status: InvocationStatus
    summary: str = ""
    output_artifact_ids: list[str] = Field(default_factory=list)
    recommendation_set_ids: list[str] = Field(default_factory=list)
    generated_question_ids: list[str] = Field(default_factory=list)
    issues: list[str] = Field(default_factory=list)
    qa_findings: list[str] = Field(default_factory=list)
    blocking_reason: str = ""
    failure_reason: str = ""


class AgentInvocationRecord(BaseModel):
    invocation_id: str
    task_id: str
    agent_name: AgentName
    phase: PhaseName
    node_name: str
    input_envelope: AgentIOEnvelope
    result_envelope: AgentResultEnvelope | None = None
    tool_call_ids: list[str] = Field(default_factory=list)
    prompt_version: str = ""
    model_id: str = ""
    model_params_hash: str = ""
    random_seed: int | None = None
    started_at: datetime | None = None
    finished_at: datetime | None = None
    trace_id: str = ""
    span_id: str = ""
    parent_span_id: str = ""


class HumanReviewReasoningTrace(BaseModel):
    trace_id: str
    project_id: str
    run_id: str
    task_id: str
    node_name: str
    agent_name: AgentName
    timestamp: datetime
    input_summary: str = ""
    key_observations: list[str] = Field(default_factory=list)
    decision_summary: str = ""
    alternatives_considered: list[str] = Field(default_factory=list)
    tool_calls_planned: list[str] = Field(default_factory=list)
    tool_calls_executed: list[str] = Field(default_factory=list)
    policy_checks_referenced: list[str] = Field(default_factory=list)
    risks_noted: list[str] = Field(default_factory=list)
    open_questions: list[str] = Field(default_factory=list)
    confidence: Literal["low", "medium", "high"] = "medium"
    visibility: Literal["internal_operator_only", "internal_admin_only"] = "internal_operator_only"
    redaction_applied: bool = True


class AdvisorNote(BaseModel):
    advisor_note_id: str
    related_decision_id: str
    consult_trigger: str = ""
    summary: str
    risks: list[str] = Field(default_factory=list)
    alternatives: list[str] = Field(default_factory=list)
    recommended_path: str = ""


class SupervisorDecision(BaseModel):
    decision_id: str
    phase: PhaseName
    rationale: str
    decision_actor: str = "supervisor"
    decision_basis: DecisionBasis = "manual_review"
    policy_version: str = ""
    override_target: str = ""
    override_level: OverrideLevel | None = None
    override_reason_type: OverrideReasonType | None = None
    mitigation_plan: list[str] = Field(default_factory=list)
    dispatch_task_ids: list[str] = Field(default_factory=list)
    accepted_recommendation_set_ids: list[str] = Field(default_factory=list)
    rejected_recommendation_set_ids: list[str] = Field(default_factory=list)
    approved_plan_artifact_ids: list[str] = Field(default_factory=list)
    rejected_plan_artifact_ids: list[str] = Field(default_factory=list)
    revision_requested_plan_artifact_ids: list[str] = Field(default_factory=list)
    advisor_note_ids: list[str] = Field(default_factory=list)
    user_review_required: bool = False
    requested_user_review_id: str | None = None
    phase_transition_to: PhaseName | None = None
    completion_candidate: bool = False
    audit: AuditFields = Field(default_factory=AuditFields)


class QAReviewRecord(BaseModel):
    qa_review_id: str
    reviewer_agent: AgentName = "data_validation_qa_specialist"
    related_task_id: str
    review_stage: QAReviewStage
    artifact_ids: list[str] = Field(default_factory=list)
    approved: bool | None = None
    stage_status: QAStageStatus = "approved"
    findings: list[str] = Field(default_factory=list)
    required_rework: list[str] = Field(default_factory=list)
    summary: str = ""
    audit: AuditFields = Field(default_factory=AuditFields)


class UserReviewRequest(BaseModel):
    review_id: str
    phase: PhaseName
    title: str
    artifact_ids: list[str] = Field(default_factory=list)
    prompt_to_user: str
    allowed_actions: list[ReviewAction] = Field(default_factory=list)
    blocking_scope: BlockingScope = "phase"
    requested_by_agent: AgentName = "supervisor"
    related_task_ids: list[str] = Field(default_factory=list)
    deadline: datetime | None = None
    status: Literal["pending", "responded", "expired", "cancelled"] = "pending"


class UserInteractionRecord(BaseModel):
    review_id: str
    action: ReviewAction
    comments: str = ""
    requested_changes: list[str] = Field(default_factory=list)
    approved_scope: list[str] = Field(default_factory=list)
    responded_at: datetime | None = None


class PhaseTransitionRecord(BaseModel):
    transition_id: str
    from_phase: PhaseName | None = None
    to_phase: PhaseName
    reason: str
    triggered_by: AgentName = "supervisor"
    related_decision_id: str | None = None
    requires_user_review: bool = False
    audit: AuditFields = Field(default_factory=AuditFields)


class SummaryCacheEntry(BaseModel):
    cache_key: str
    state_version: int
    view_name: StateViewName
    scope_hash: str = ""
    requesting_agent: AgentName
    summary_prompt_version: str = ""
    summary_text: str
    created_at: datetime | None = None


class RuntimeRegistry(BaseModel):
    sql_runtime_name: str = ""
    python_runtime_name: str = ""
    artifact_store_name: str = ""
    metadata_catalog_name: str = ""
    enabled_tools: list[str] = Field(default_factory=list)
    tracing_backend_name: str = ""
    tracing_export_endpoint: str = ""
    reasoning_trace_store_uri: str = ""
    approval_policy_store_uri: str = ""
    routing_policy_store_uri: str = ""
    phase_gate_policy_store_uri: str = ""
    qa_rulebook_store_uri: str = ""
    metric_registry_uri: str = ""
    past_findings_store_uri: str = ""
    summary_cache_store_uri: str = ""


class DataCatalogRefs(BaseModel):
    table_catalog_uri: str = ""
    datamart_spec_root_uri: str = ""
    metric_definition_root_uri: str = ""
    additional_reference_uris: list[str] = Field(default_factory=list)


class SummaryViews(BaseModel):
    project_snapshot: str = ""
    active_task_summary: str = ""
    recommendation_digest: str = ""
    qa_status_summary: str = ""
    user_alignment_summary: str = ""
    working_memory_summary: str = ""
    pending_user_review_summary: str = ""
    phase_summary: str = ""
    delivery_readiness_summary: str = ""


class FinalOutputRecord(BaseModel):
    output_id: str
    title: str
    artifact_ids: list[str] = Field(default_factory=list)
    summary: str = ""
    delivered_to_user: bool = False
    signed_off: bool = False
    audit: AuditFields = Field(default_factory=AuditFields)


class GraphState(BaseModel):
    project_context: ProjectContext
    team_directory: list[TeamMember] = Field(default_factory=list)
    state_version: int = 0
    current_phase: PhaseName = "requirement_alignment"
    phase_history: list[PhaseTransitionRecord] = Field(default_factory=list)
    draft_tasks: list[TaskRecord] = Field(default_factory=list)
    task_queue: list[TaskRecord] = Field(default_factory=list)
    awaiting_plan_approval_tasks: list[TaskRecord] = Field(default_factory=list)
    active_tasks: list[TaskRecord] = Field(default_factory=list)
    completed_tasks: list[TaskRecord] = Field(default_factory=list)
    failed_tasks: list[TaskRecord] = Field(default_factory=list)
    blocked_tasks: list[TaskRecord] = Field(default_factory=list)
    cancelled_tasks: list[TaskRecord] = Field(default_factory=list)
    artifacts: list[ArtifactRef] = Field(default_factory=list)
    recommendations: list[RecommendationSet] = Field(default_factory=list)
    qa_reviews: list[QAReviewRecord] = Field(default_factory=list)
    open_questions: list[OpenQuestion] = Field(default_factory=list)
    project_memory: list[str] = Field(default_factory=list)
    decision_memory: list[str] = Field(default_factory=list)
    working_memory: WorkingMemory = Field(default_factory=WorkingMemory)
    agent_episodic_memories: dict[AgentName, list[AgentEpisodicMemoryEntry]] = Field(default_factory=dict)
    supervisor_decisions: list[SupervisorDecision] = Field(default_factory=list)
    advisor_notes: list[AdvisorNote] = Field(default_factory=list)
    agent_invocations: list[AgentInvocationRecord] = Field(default_factory=list)
    tool_calls: list[ToolCallRecord] = Field(default_factory=list)
    data_catalog_refs: DataCatalogRefs = Field(default_factory=DataCatalogRefs)
    runtime_registry: RuntimeRegistry = Field(default_factory=RuntimeRegistry)
    pending_user_review: UserReviewRequest | None = None
    user_review_history: list[UserReviewRequest] = Field(default_factory=list)
    user_interactions: list[UserInteractionRecord] = Field(default_factory=list)
    final_outputs: list[FinalOutputRecord] = Field(default_factory=list)
    summary_views: SummaryViews = Field(default_factory=SummaryViews)
    summary_cache_index: list[SummaryCacheEntry] = Field(default_factory=list)
    error_log: list[str] = Field(default_factory=list)
    checkpoints: list[str] = Field(default_factory=list)
```

実装時の description 例:

```python
class TeamMember(BaseModel):
    agent_name: AgentName = Field(
        description="Stable agent identifier used across dispatch, audit, and access control"
    )
    role_summary: str = Field(
        description="Short human-readable summary of the agent's responsibility in the orchestration"
    )
    allowed_state_views: list[StateViewName] = Field(
        default_factory=list,
        description="Registered read-only state views that this agent is allowed to request from the State Manager"
    )


class WorkingMemory(BaseModel):
    phase_objective: str = Field(
        description="Current phase objective that specialists should optimize for during this stage"
    )
    focus_points: list[str] = Field(
        default_factory=list,
        description="Top items that deserve immediate attention in the current phase"
    )
    provisional_assumptions: list[str] = Field(
        default_factory=list,
        description="Assumptions currently accepted for provisional progress and requiring later confirmation"
    )


class TaskRecord(BaseModel):
    task_id: str = Field(description="Stable task identifier unique within the project")
    plan_status: PlanStatus = Field(
        default="not_required",
        description="Current lifecycle status of the execution plan associated with this task"
    )
    risk_level: RiskLevel = Field(
        default="medium",
        description="Latest risk tier assigned to the task based on the most recent execution plan review"
    )
    provisional_allowed: bool = Field(
        default=False,
        description="Whether this task is allowed to proceed under degraded-mode provisional execution"
    )


class ArtifactRef(BaseModel):
    artifact_id: str = Field(description="Immutable artifact identifier unique within the project")
    storage_uri: str = Field(description="Canonical storage location of the artifact body outside GraphState")
    quality_stage: QualityStage = Field(
        default="draft",
        description="Current lifecycle stage that determines how widely this artifact may be used or shown"
    )


class HumanReviewReasoningTrace(BaseModel):
    trace_id: str = Field(description="Stable identifier for a human-review-only reasoning trace record")
    input_summary: str = Field(description="Redacted summary of the input context seen by the node or agent")
    decision_summary: str = Field(description="Short explanation of the main decision or conclusion reached")
    redaction_applied: bool = Field(
        default=True,
        description="True when sensitive content has been redacted before persistence for human review"
    )


class GraphState(BaseModel):
    state_version: int = Field(
        description="Monotonically increasing state version used for cache invalidation and audit ordering"
    )
    working_memory: WorkingMemory = Field(
        default_factory=WorkingMemory,
        description="Shared short-to-medium-term execution memory for the current phase"
    )
    agent_episodic_memories: dict[AgentName, list[AgentEpisodicMemoryEntry]] = Field(
        default_factory=dict,
        description="Per-agent bounded short-term memory entries; not a canonical long-term memory store"
    )
```

この state により、少なくとも以下を追跡できる。

- 各 Specialist が何を入力に仕事をしているか
- どの artifact を参照・生成したか
- QA の reject / rework 内容
- recommendation の採否理由
- pending な user review や plan approval
- 再現に必要な snapshot / prompt / model 設定
- narrative claim の根拠 artifact
- current phase で何を重視すべきかという `working memory`
- 各 agent の直近 2 件の bounded episodic memory

### 14.3 State Mutation Policy

- `specialist_worker_node`
  raw result を返すが state を直接編集しない
- `state_update_node`
  result を正規化して state に反映する
- `state_manager_refresh_node`
  State Manager service を呼び出して deterministic view と summary cache を更新する
- `supervisor_node`
  dispatch decision を返すが artifact 本文は保存しない
- `user_review_gate_node`
  `pending_user_review` と `user_interactions` を扱う

State Manager service 由来の処理では、ranking、recommendation、priority judgement をここで生成しない。

### 14.4 Pause and Resume

`user_review_gate_node` により review 待ちになると graph は checkpoint 可能な待機状態に入る。

- `approve`: 同 phase または次 phase へ進行
- `request_changes`: Supervisor が再計画
- `clarify`: clarification task を生成
- `pause`: 明示的 resume まで停止

`blocking_scope` は `task`, `phase`, `project` を取りうる。

## 15. Supervisor Control Loop

1. State Manager から最新 summary を受け取る
2. current phase と user alignment を確認する
3. task、open questions、QA 状況、recommendations、`awaiting_plan_approval_tasks` を確認する
4. pending plan があれば、まず `Plan Reviewer` と `Execution Readiness Policy` の結果を確認する
5. Supervisor 承認が必要な plan のみ `decide_execution_plan` で裁定する
6. `evaluate_advisor_consult_trigger` を用いて mandatory consult 条件を評価する
7. 自由相談が有益な場合、または mandatory consult 条件に該当する場合は Advisor に相談する
8. `evaluate_routing_policy` と `evaluate_phase_gate` の結果を参照して再計画する
9. 必要なら user review を要求する
10. 新規 task を 0 件以上生成する
11. 並列 dispatch する
12. 完了・失敗・blocked を受けて再計画する
13. 完了条件を満たせば終了する

Advisor 相談は最後の手段ではなく、判断の不確実性がある場面では繰り返し行ってよい。加えて、少なくとも `high-risk task`、同一 `execution_plan` の差し戻し 2 回以上、phase 移行前では mandatory consult を行う。Advisor note の採否に関わらず、Supervisor はその扱いを decision log に残す。

override を行う場合、Supervisor は `justified override` の成立条件を満たさなければならない。成立しない場合、policy / evaluator / reviewer の既定判断を採用する。

## 16. Observability and Monitoring

本システムは multi-agent orchestration であるため、単に結果が出たかだけでなく、どの agent がどの context を受け取り、どの tool を使い、どこで詰まったかを観測できる必要がある。

### 16.1 Observability Goals

- task routing の妥当性を後から評価できる
- tool failure、QA reject、plan reject の偏りを検知できる
- phase ごとのボトルネックを把握できる
- user review がどこで進行を止めているかを見られる
- artifact の lineage を辿って結論の根拠を監査できる

### 16.2 Required Signals

最低限、以下を観測対象とする。

- `Task lifecycle`
  queued, awaiting_plan_approval, running, blocked, failed, completed
- `Agent invocation`
  入力 context、使用 tool、所要時間、結果
- `Tool call`
  呼び出し回数、失敗率、平均時間、重い query の偏り
- `Plan review`
  approve / reject / request_revision の件数と理由
- `QA outcomes`
  reject 率、rework 回数、主要指摘カテゴリ
- `User review`
  review 待ち時間、差し戻し率、phase ごとの停止回数
- `Routing outcomes`
  recommendation 採用率、override 率、再計画回数
- `Governance override`
  justified override 成立率、override reason type 分布、override 後の再QA率
- `Memory refresh`
  working memory 更新頻度、episodic memory 利用状況、stale memory の発生

### 16.3 Suggested Metrics

- task completion latency
- phase completion latency
- plan approval latency
- QA rejection rate
- tool error rate
- retry count per task
- recommendation acceptance rate
- supervisor override rate
- justified override success rate
- mandatory consult trigger rate
- QA stage dwell time
- provisional execution rate
- working memory refresh frequency
- episodic memory hit rate
- stale memory incident count
- user review turnaround time
- cache hit rate for `query_state_view`

### 16.4 Traceability

少なくとも以下の関連を辿れることを必須とする。

- `task_id -> invocation_id -> tool_call_id`
- `task_id -> output artifact_ids`
- `artifact_id -> upstream_artifact_ids`
- `decision_id -> advisor_note_ids`
- `final output -> supporting artifact_ids`
- `narrative claim -> cited artifact_id`

### 16.5 Human Review Reasoning Trace

人間のレビュー、監査、prompt / policy 改善のために、各 node / agent は `Human Review Reasoning Trace` を外部ログとして出力してよい。

ベースルール:

- reasoning trace は `GraphState` の正本ではない
- reasoning trace は project / decision / working / episodic memory に昇格しない限り、agent の次入力に使わない
- reasoning trace は他 agent の input に注入しない
- reasoning trace は chain-of-thought 全文ではなく、構造化された判断要約とする
- reasoning trace は redact 前提とし、顧客データや秘匿情報をそのまま残さない
- reasoning trace の既定 visibility は `internal_operator_only` とする

必須出力 node:

- `supervisor_node`
- `advisor_consult_node`
- `phase_gate_node`
- `user_review_gate_node`
- `state_manager_refresh_node`
- `error_triage_node`

Plan-Execute 系での推奨出力 node:

- `specialist_worker_node` (`sql_specialist`, `python_specialist`, `causal_specialist`, `data_validation_qa_specialist`)

保存方針:

- reasoning trace は txt / json / jsonl / structured log のいずれでもよいが、実装では machine-readable な JSONL を推奨する
- 推奨保存先は `logs/reasoning/<project_id>/<run_id>/` または同等の object storage path とする
- `reasoning_trace_store_uri` は runtime registry で管理する

最小内容:

- `input_summary`
- `key_observations`
- `decision_summary`
- `alternatives_considered`
- `tool_calls_planned`
- `tool_calls_executed`
- `policy_checks_referenced`
- `risks_noted`
- `open_questions`
- `confidence`
- `redaction_applied`

## 17. Evaluation Framework

本システムは prompt や graph を改善できるよう、結果品質だけでなく process 品質も評価する。

### 17.1 Evaluation Levels

1. `Task-level evaluation`
   各 Specialist の単体成果物を評価する。
2. `Workflow-level evaluation`
   Supervisor の routing、phase 遷移、QA 差し込みを評価する。
3. `Project-level evaluation`
   最終 deliverable の妥当性、再現性、ユーザー満足度を評価する。

### 17.2 Evaluation Axes

- `Routing quality`
  適切な Specialist に適切な順番で task を振れたか
- `Execution quality`
  SQL / Python / Causal / QA の成果物が妥当か
- `Governance quality`
  plan gate、QA gate、user review gate が適切に機能したか
- `Insight quality`
  示唆が根拠 artifact に忠実か
- `Recommendation quality`
  提案がビジネス文脈に接続されているか
- `Operational efficiency`
  不要な往復、重複作業、過剰な tool 呼び出しがないか

### 17.3 Evaluation Artifacts

必要に応じて以下を評価用 artifact として保存する。

- routing review memo
- plan review memo
- QA scorecard
- final output review memo
- user feedback summary

### 17.4 Core Questions

少なくとも以下に答えられるようにする。

- Supervisor の routing は妥当だったか
- QA は本当に問題を検知したか
- execution plan gate は無駄な runtime 実行を減らしたか
- user review は差し戻しコストを下げたか
- final output は cited artifact に忠実か

## 18. Failure Handling and Recovery

multi-agent system では failure を例外ではなく通常運用として扱う。

### 18.1 Failure Classes

- `tool_error`
- `runtime_error`
- `schema_error`
- `dependency_error`
- `quality_rejection`
- `definition_ambiguity`
- `user_wait_timeout`
- `policy_violation`

### 18.2 Recovery Policy

- `tool_error`
  再試行、代替 tool、Supervisor への報告
- `runtime_error`
  入力修正または plan 改訂後の再実行
- `schema_error`
  state_update 失敗として記録し、差し戻し
- `dependency_error`
  blocked task として保持し、依存解消後に再投入
- `quality_rejection`
  QA 指摘付きで元担当へ再依頼
- `definition_ambiguity`
  Metric Definition または Scoping に戻す
- `user_wait_timeout`
  Supervisor が pause 継続か、暫定進行かを裁定する
- `policy_violation`
  実行停止、decision log 記録、必要なら user / admin に通知

### 18.3 Degraded Mode and Fallback Policy

正常系の前提が満たされない場合でも、常に即停止するのではなく縮退運転を選べるようにする。

- `provisional execution`
  指標定義や metadata に不確実性が残るが、探索的価値が高い場合に許可する。出力は必ず `draft` または `internal` とし、前提を明示する
- `pause`
  user 回答、権限解消、定義確定が不可欠な場合に選ぶ
- `human escalation`
  tenant 境界、重大な policy violation、高感度データ誤参照の恐れがある場合に選ぶ

縮退運転のベースルール:

- provisional 出力は final decision や final delivery の根拠にしない
- provisional 実行の前提、欠落情報、残存リスクを decision log に残す
- user 非応答時は timeout 後に `pause`, `remind`, `provisional continue` のいずれかを明示的に選ぶ
- `provisional continue` を選ぶ場合、次の user review または QA stage で必ず再確認する

### 18.4 Retry Policy

- 各 task は `retry_count` と `max_retries` を持つ
- 同一 failure が短期間に繰り返される場合、Supervisor は Advisor 相談を推奨される
- `project_retry_ceiling` を超える場合、案件全体を human review 対象とする

### 18.5 Loop Prevention

以下を無限ループの兆候として扱う。

- 同一 task の repeated revision
- recommendation の相互往復
- QA reject と rework の反復
- user review 待ちからの repeated reopen

検出時は Supervisor が continuation ではなく縮退運転、pause、または human escalation を判断する。

## 19. Artifact Lifecycle and Reuse

artifact は一時出力ではなく、監査・再利用・再現性の基盤として扱う。

### 19.1 Lifecycle Stages

- draft
- internal
- QA-approved
- user-reviewed
- final-delivery
- archived

### 19.2 Reuse Policy

- 現在案件の成果物と過去知見は混同しない
- 過去知見は `reference_summary` として補助的に参照する
- 再利用時は出典と元 artifact を辿れることを必須とする
- QA 未通過 artifact を再利用の根拠にしない

### 19.3 Versioning

- artifact は immutable な ID を持つ
- 改訂は新 artifact として保存する
- lineage で版のつながりを辿れるようにする

## 20. Security and Access Control

顧客社内データを扱うため、分析品質と同等にアクセス制御を重視する。

### 20.1 Access Principles

- 最小権限
- task 単位の allowlist
- read / execute / approve の責務分離
- 監査可能性

### 20.2 Required Controls

- table / column allowlist
- tenant 境界の分離
- tool 呼び出しの監査ログ
- sensitive artifact の visibility 制御
- user-facing 出力前の QA / review gate
- role ごとの artifact / field access 制御
- delegated approval component の approve 権限範囲制御

### 20.3 Sensitive Data Handling

- 個人情報や機密指標は必要最小限の agent にのみ開示する
- narrative や report では過剰な明細露出を避ける
- access policy 違反は `policy_violation` として扱う

### 20.4 Privilege, Visibility, and Tenant Isolation

- `Advisor` は原則として raw dataset 全文ではなく、必要最小限の artifact summary と関連 metadata を読む
- `Advisor` が raw artifact drill-down を行う場合は、その必要性と scope を decision log に残す
- `search_past_findings` と `search_metric_registry` は tenant / project scope を明示し、他 tenant の知見を混入させない
- `reference_summary` artifact は現在案件の artifact と区別できる visibility / tag を持つ
- delegated approval component は approval 権限のみを持ち、artifact 読み書きや dispatch 権限を持たない
- field-level に秘匿が必要なデータは State Manager でも summary 化せず、masked または inaccessible として扱う

## 21. Continuous Improvement Loop

本システムは静的な graph ではなく、運用しながら改善される前提で設計する。

### 21.1 Improvement Targets

- prompt version
- routing policy
- phase gate thresholds
- plan reviewer rules
- QA checklist
- tool schema
- risk classification policy
- override policy

### 21.2 Improvement Process

1. observability から問題傾向を検知する
2. evaluation artifact で原因を特定する
3. prompt / policy / tool を局所変更する
4. replay または比較評価で差分を確認する
5. 改善が確認できたもののみ本番へ反映する

### 21.3 Guardrails for Changes

- graph と prompt を同時に大きく変えすぎない
- 改善前後で比較可能な評価セットを持つ
- Supervisor の裁量を増やす変更は、必ず observability 強化とセットで行う
- policy / evaluator / reviewer の変更は version を付け、decision と invocation に紐づける

## 22. Appendix: Canonical Terms

以下は backend / frontend / 実装文書が共有する用語の正本である。新語を導入する場合は本付録を更新し、spec 本文側では意味を重複定義しすぎない。

### 22.1 Advisor

Supervisor の壁打ち相手として代替案・リスク・批判的レビューを返す coordination agent。task の自律実行や直接 dispatch は行わない。read 権限は Supervisor と対等。

### 22.2 artifact

分析プロセスで生成される中間または最終生成物。dataset、insight memo、execution plan、report outline 等を含む。`ArtifactRef` で metadata を管理し、本体は artifact store に保存される。

### 22.3 artifact lineage

ある artifact の上流 (`upstream_artifact_ids`) と下流の関係を辿れる DAG。`get_artifact_lineage` tool で取得する。

### 22.4 Advisor Consultation Trigger

Advisor への mandatory consult を発火させる条件評価。high-risk task、同一 plan の差し戻し 2 回以上、phase 移行前で発火する。policy module として実装する。

### 22.5 data_snapshot_id

artifact が依拠したデータ状態を再現するための識別子。warehouse の time-travel ID、table snapshot、抽出時刻付きハッシュ等を保持する。

### 22.6 delegated approval component

低リスクな execution_plan を Supervisor の代わりに承認できる policy 層コンポーネント。`evaluate_execution_readiness` の判定に基づき、許可された場合のみ起動する。

### 22.7 draft (artifact quality stage)

QA 未通過の探索的成果物。final delivery の根拠にしない。

### 22.8 execution_plan

SQL / Python / Causal / QA Specialist が runtime tool 起動前に emit する artifact。参照テーブル・カラム、想定出力、想定リスク、見積コストを含む。

### 22.9 Execution Readiness Policy

execution_plan の risk level を分類し、誰が承認すべきかを判定する policy module。`classify_execution_risk` と `evaluate_execution_readiness` の組み合わせで具体化する。

### 22.10 justified override

Supervisor が policy 既定判断を上書きする際に必要な構造化申請。`override_target`, `override_reason_type`, `rationale`, `expected_risk_if_not_overridden`, `expected_risk_if_overridden`, `mitigation_plan`, `override_scope` を必須とする。

### 22.11 Phase

分析プロジェクトの段階区分。`requirement_alignment`、`scoping_and_metric_definition`、`data_understanding`、`analytical_execution`、`validation_and_qa`、`insight_and_recommendation`、`delivery_and_iteration` の 7 種。

### 22.12 Phase Gate Evaluator

フェーズ遷移可否を deterministic に評価する policy module。Supervisor は結果を受けて承認または override する。

### 22.13 Plan-Execute Contract

SQL / Python / Causal / QA Specialist が必ず通る、plan emit → approval routing → execute の 3 段階フロー。

### 22.14 Plan Reviewer

execution_plan を Supervisor 承認前に preflight する policy 層コンポーネント。allowlist 逸脱、未参照 metric definition、コスト異常等を検査する。

### 22.15 policy module

pure-function として実装される判定ロジック。長期状態を持たず、入出力は strict schema、`POLICY_VERSION` 定数で version 化される。

### 22.16 project memory

案件全体の恒久的前提を保持する memory 層。用語、目的、制約、合意済み定義、対象範囲を含む。

### 22.17 provisional artifact

要件不確実性下で生成された暫定成果物。final delivery には使えない。`record_provisional_mode` tool で前提・欠落情報・mitigation を残す。

### 22.18 QA stage

QA Specialist が実施する 3 段階レビュー。`data_quality_review`、`analytical_consistency_review`、`narrative_claim_review`。各 stage は独立した execution_plan を持ち、上流が `rework_required` の場合、下流 plan は emit しない。

### 22.19 query_state_view

全 agent(State Manager 自身を除く)が State Manager に read-only で問い合わせる tool。registered view + filters + fields の typed query のみ受け付ける。caller identity は runtime layer が自動 inject する。

### 22.20 reference_summary

過去案件知見を現在案件の artifact と区別するための field。`search_metric_registry` / `search_past_findings` の取得結果に付与する。

### 22.21 retry_count / max_retries

TaskRecord の失敗ループ防止フィールド。`max_retries`(デフォルト 3)を超えた task は自動的に blocked 化され、Supervisor 判断に戻される。

### 22.22 Routing Policy Engine

Supervisor の dispatch 判断を支援する policy module。定型的な routing 提案を返す。Supervisor は採用または override する。

### 22.23 sanity artifact

dataset 生成 task が完了時に自動付随生成する artifact。行数、欠損率、値域、期待値域逸脱を記録する。`run_data_sanity_check` で生成する。

### 22.24 State Manager

state の正本管理 + read-path API を担う coordination agent。recommendation や judgement は返さない。「k8s に対する etcd と API server」の関係に近い。

### 22.25 state_version

GraphState の単調増加バージョン番号。summary cache の無効化 key として使う。

### 22.26 Supervisor

全 task の dispatch、再計画、完了判定を行う coordination agent。policy module の判定結果を踏まえて最終裁定を行う。`Supervisor first orchestration` 原則の中心。

### 22.27 TaskRecord

個別 task の runtime 表現。状態、入出力、`plan_status`、`risk_level`、`retry_count`、`allowed_table_ids` 等を保持する。

### 22.28 team_directory

全 agent が prompt に持つ「他 agent の存在と役割」のリファレンス。直接通信の許可ではない。

### 22.29 Working Memory

現 phase に関連する短中期実務記憶。phase gate と user review gate 通過時に再生成される。bounded で agent の私有長期メモにはしない。
