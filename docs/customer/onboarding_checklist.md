# Customer Onboarding Checklist

> 顧客との契約後、テナントを立ち上げる前に必ず確認すべき事項のチェックリスト。
> 営業引き継ぎ + Customer Success + 実装担当が共有して使う。
> tenant 単位の設定はこのチェックを完了するまで kickoff しない。

## 1. 契約直後 (Day 0-3)

### 1.1 顧客プロフィール
- [ ] 顧客名
- [ ] テナント ID(社内採番)
- [ ] 主担当者(分析オーナー)— 名前、連絡先、決裁権限
- [ ] 副担当者
- [ ] 業界・業種
- [ ] 利用想定ユーザー数
- [ ] 想定案件数(月次)

### 1.2 初期分析テーマ
- [ ] 最初に取り組む分析テーマ(1-3 件)
- [ ] 各テーマの背景・期待成果物
- [ ] 各テーマの優先度
- [ ] 過去に同テーマで実施した分析の有無
- [ ] 期待リードタイム

## 2. データ環境 (Day 3-10)

### 2.1 DWH
- [ ] DWH 製品(BigQuery / Snowflake / Redshift / Databricks / その他)
- [ ] バージョン
- [ ] 接続情報の提供方法(Service Account / IAM Role / API Key)
- [ ] ネットワーク制約(IP allowlist / Private Service Connect 等)
- [ ] アクセス可能スキーマ・テーブル範囲
- [ ] PII / 機密データを含むテーブルの識別
- [ ] スキーマカタログ(`table.csv` 相当)の有無 / 提供形式
- [ ] データマート仕様書の有無 / 提供形式
- [ ] 主要指標定義の文書化状況
- [ ] データ更新頻度・SLA

### 2.2 データガバナンス
- [ ] 機密性レベルの判定(public / internal / confidential / pii)
- [ ] PII カラム一覧の確認
- [ ] tenant 越境禁止のスコープ
- [ ] データ越境(国・region)の制約
- [ ] サンプリング許可ポリシー(LIMIT 付き preview の可否)

## 3. クラウド環境 (Day 3-10)

### 3.1 顧客クラウド
- [ ] 利用クラウド(GCP / AWS / Azure / on-prem / hybrid)
- [ ] 弊社 SaaS から顧客環境への接続方法
- [ ] artifact 保存先(顧客クラウド内 / 弊社管理)
- [ ] Python sandbox の配置(顧客 VPC 内 / 弊社 E2B)
- [ ] artifact retention policy(保存期間)

### 3.2 デプロイ形態
- [ ] SaaS multi-tenant / on-prem / hybrid
- [ ] on-prem の場合、Kubernetes / Cloud Run on customer GCP / その他
- [ ] バックアップ・DR 要件

## 4. LLM プロバイダー (Day 5-12)

### 4.1 利用可能プロバイダー
- [ ] Anthropic 直 / Vertex AI / AWS Bedrock / ローカル LLM
- [ ] 規約上の制約(データを LLM provider に送って良いか)
- [ ] PII カラムを LLM に渡してよいか(or サンプル値マスキング要否)
- [ ] LLM トレース(Langfuse 等)に payload を残してよいか
- [ ] モデルバージョン pin 要否
- [ ] agent ごとのモデル割り当てカスタマイズ要否

### 4.2 コスト管理
- [ ] 月次コスト上限の有無
- [ ] alert 閾値
- [ ] 上限超過時の挙動(hard stop / soft warn)

## 5. 認証・認可 (Day 5-12)

### 5.1 ユーザー認証
- [ ] SSO の有無(Azure AD / Okta / Google Workspace / 自前)
- [ ] MFA 要件
- [ ] パスワードポリシー
- [ ] session lifetime
- [ ] domain restriction

### 5.2 ロール
- [ ] 想定ロール(viewer / analyst / admin など)
- [ ] artifact 機密性ごとの閲覧制御
- [ ] 削除権限の所在

## 6. ガバナンス・コンプライアンス (Day 7-15)

- [ ] 監査要件(SOC2 / ISO27001 / FISC / 3 省 2 ガイドライン等)
- [ ] ログ保持期間
- [ ] データ保持期間
- [ ] 削除依頼への対応 SLA
- [ ] backup / DR 要件
- [ ] override 履歴の社内承認フロー要否
- [ ] user review の社内承認フロー要否

## 7. 運用・サポート (Day 10-15)

- [ ] サポート窓口の連絡先
- [ ] エスカレーションパス
- [ ] 定期レビュー頻度(月次 / 四半期)
- [ ] SLA(uptime / response time)
- [ ] 障害通知方法

## 8. 技術ブートストラップ (Day 15-30)

- [ ] テナント DB 作成(Postgres schema)
- [ ] DWH 接続テスト(read-only サンプル query)
- [ ] LLM provider 接続テスト
- [ ] artifact store 接続テスト
- [ ] Secret Manager に credential 登録
- [ ] PoC 用初期案件 1 件投入
- [ ] 顧客レビュアーへの UI ハンズオン
- [ ] 運用監視ダッシュボードの共有

## 9. 引き継ぎドキュメント

onboarding 完了時に、以下を顧客向けナレッジベースに保管する。

- 接続情報サマリ(credential 自体は Secret Manager)
- スキーマカタログ初版
- 指標定義初版
- 担当者連絡先一覧
- 規約・コンプライアンス要件サマリ
- LLM provider 設定
- override / user review の社内ワークフロー定義

## 10. Red Flags

以下が確認できない場合、kickoff を保留する。

- DWH 接続情報の提供承諾
- PII カラムの識別合意
- LLM provider の選定
- データ越境ポリシーの確認
- 監査要件の確認
- 主担当者の決裁権限

## 11. 反復タイミング

以下のタイミングで本 checklist を見直す。

- 案件追加時(新分析テーマで前提が変わる可能性)
- 規制変更時
- DWH 切替時
- LLM provider 切替時
- 担当者交代時
