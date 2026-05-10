# 要求仕様書: Daily Log System

## 1. 意図分析サマリー

- **ユーザーリクエスト**: ShadowSync の `daily-log` 子AI-DLCについて、Inception フェーズを開始し、日誌作成システムの要件を明確化する。
- **リクエスト種別**: New Project
- **スコープ見積もり**: 単一子ユニット (`daily-log`) の新規バッチシステム
- **複雑度見積もり**: Moderate
- **プロジェクト状態**: Greenfield
- **主な外部依存**: `data-accumulation` の DynamoDB、Amazon Bedrock 等の LLM、Notion API、AWS Lambda、EventBridge

## 2. システム概要

`daily-log` は、個人活動ログ自動集計・活用システム ShadowSync のアウトプットを担うバッチシステムである。

本システムは、`data-accumulation` が保存した正規化済みアクティビティデータから対象日分のログを取得し、日報を生成して Notion に出力する。初期版では単一ユーザーを対象とするが、設計上はユーザーごとの Notion 設定と `user_id` 分離を前提にし、将来的なマルチテナント拡張を阻害しないこと。

## 3. スコープ

### 3.1 対象範囲

- DynamoDB に保存された正規化済みアクティビティの取得
- ユーザーごとの対象日判定
- 対象日のログ抽出
- LLM によるタイムライン分類、トピック抽出、要約文生成
- Notion API による日報ページ作成
- EventBridge を起点とした自動実行
- ユーザーごとの処理分離
- 構造化ログ出力
- Security Baseline と Property-Based Testing ルールを考慮した設計

### 3.2 対象外

- クライアントからの生データ受信
- リアルタイムの意味抽出
- チャットボットや対話型 UI
- 初期版における S3 元データ参照や S3 Vector 検索の利用
- 初期 Inception 時点での IaC 詳細設計
- 初期 Inception 時点での詳細テスト計画

## 4. 機能要件

### FR-1. 日報生成対象

- 初期版では単一ユーザーを対象に日報を生成する。
- 実装・データモデル・設定構造は、将来的に複数ユーザーを扱えるよう `user_id` を明示的に保持する。
- ユーザーごとの Notion Database ID と Integration Token を設定できる設計とする。

### FR-2. 対象日の判定

- 1日分のログ範囲はユーザーごとのタイムゾーン設定に従って判定する。
- 対象日の開始・終了時刻を UTC に変換し、DynamoDB から検索可能な形式にする。
- 日付境界を越えた後、自動的に前日または完了済み対象日の日報を生成する。

### FR-3. データ取得

- `data-accumulation` が管理する DynamoDB の正規化済みアクティビティを主データソースとする。
- 初期版では S3 の元データ参照や S3 Vector 等の検索用データは必須としない。
- `user_id` と対象日時範囲でデータを絞り込む。
- 他ユーザーのデータが混入しないよう、取得条件には必ず `user_id` を含める。

### FR-4. 日報生成

- 初期の日報には以下を含める。
  - 時系列タイムライン
  - 1日の要約
  - 主要トピック
- 将来的に日報項目をカスタムできるよう、出力セクションを設定可能な構造にする。
- 日報生成では、ルールベースで対象日のログを抽出した後、LLM でタイムライン分類、トピック抽出、要約文生成を行う。

### FR-5. Notion 連携

- ユーザーごとに Notion Database ID と Integration Token を設定する。
- Notion Integration Token は安全に保管し、アプリケーションログへ出力しない。
- 対象日の Notion 日報が既に存在する場合でも、新しいページを作成し、古いページは残す。
- Notion ページには、対象日、`user_id`、生成日時、日報本文を含める。

### FR-6. 自動実行

- EventBridge を起点に、対象日の境界を越えた後に日報生成を自動実行する。
- 将来的な複数ユーザー対応では、ユーザーごとのジョブを分散起動する構成を優先する。
- ユーザー数やログ量が増え、単一 Lambda で処理しきれない場合は、ユーザー単位の Lambda 実行に分割する。

### FR-7. エラーハンドリング

- Notion API、データ取得、LLM 呼び出し等で失敗した場合は、エラーログを残し、日報生成は失敗扱いで終了する。
- エラーログには原因調査に必要な情報を含めるが、Notion token、個人情報、LLM 入力全文などの秘匿情報は出力しない。
- 将来の再実行機能に備え、失敗した `user_id`、対象日、処理ステップ、エラー種別を記録できる構造にする。

## 5. 非機能要件

### NFR-1. 実行環境

- 主な実行環境は AWS Lambda とする。
- Lambda の主な実装言語は Python とする。
- EventBridge を定期実行の起点として利用する。

### NFR-2. 性能・スケーラビリティ

- 初期版は単一ユーザーの1日分ログを処理できることを優先する。
- 将来的な複数ユーザー対応では、ユーザー単位で処理を分割できる構造にする。
- Lambda の 15 分制限を超える可能性がある処理は、ユーザー単位または日付単位に分割可能にする。

### NFR-3. セキュリティ

- Security Baseline 拡張ルールを有効化する。
- Notion Integration Token は AWS Secrets Manager 等のシークレット管理サービスで管理する。
- DynamoDB へのアクセスは最小権限に限定する。
- Lambda の IAM Role は必要な DynamoDB 読み取り、Secrets Manager 読み取り、Bedrock 呼び出し、CloudWatch Logs 出力に限定する。
- 保存データは暗号化されていることを前提とし、追加リソースを作成する場合も暗号化を必須とする。
- 通信は TLS を利用する。
- CloudWatch Logs に秘匿情報、認証情報、個人情報、LLM 入力全文を出力しない。
- `user_id` によるデータ分離をアプリケーションロジック上も必須とする。

### NFR-4. ログ・監視

- Lambda は構造化ログを出力する。
- ログには timestamp、log level、correlation ID または execution ID、`user_id`、対象日、処理ステップ、結果を含める。
- 秘匿情報はログに含めない。
- 将来的に失敗数、Notion API エラー、LLM 呼び出し失敗、処理時間を監視できるようにする。

### NFR-5. テスト

- 詳細なテスト計画は後続フェーズで具体化する。
- Property-Based Testing 拡張ルールを有効化する。
- 日付範囲判定、タイムゾーン変換、`user_id` 分離、日報データ変換、Notion 出力モデル生成などの純粋ロジックは PBT 対象として扱う。
- Python の PBT フレームワークとして Hypothesis を候補にする。

### NFR-6. IaC

- IaC の詳細選定は後回しとする。
- ただし、後続の Infrastructure Design では Lambda、EventBridge、Secrets Manager、IAM Role、CloudWatch Logs などの構成を明文化する。

## 6. 技術制約

- アプリケーションコードは `daily-log/` のワークスペースルート配下に配置し、`aidlc-docs/` 配下には置かない。
- ドキュメントは `daily-log/aidlc-docs/` 配下に配置する。
- `data-accumulation` への依存は AWS SDK 経由の読み取りに限定する。
- 初期版では DynamoDB の正規化済みアクティビティを主入力とし、S3 元データや検索用ベクトルデータへの依存を増やさない。
- Notion API と Bedrock 呼び出しは外部依存として失敗し得る前提で扱う。

## 7. データ要件

### 7.1 入力データ

- `user_id`
- 対象日
- ユーザーのタイムゾーン
- DynamoDB の正規化済みアクティビティ
- ユーザー別 Notion 設定
- Bedrock 利用設定

### 7.2 出力データ

- Notion 日報ページ
- 生成日時
- 対象日
- タイムライン
- 主要トピック
- 1日の要約
- 処理結果ログ

## 8. 受け入れ基準

- 指定ユーザーの対象日データのみを DynamoDB から取得できる。
- ユーザーのタイムゾーンに従って対象日の開始・終了を判定できる。
- 対象日のログからタイムライン、主要トピック、要約を生成できる。
- Notion に新しい日報ページを作成できる。
- 同じ対象日の日報が既に存在しても、既存ページを上書きせず新規ページを作成する。
- Notion Integration Token をログに出力しない。
- 他ユーザーのデータが混入しない。
- Lambda は Python で実装できる構成になっている。
- Security Baseline の該当ルールに違反しない設計になっている。
- PBT 対象となる純粋ロジックが後続設計で識別可能になっている。

## 9. 未決事項

- IaC ツールの最終選定
- 詳細なテスト範囲と CI 構成
- Notion ページの具体的なプロパティ設計
- LLM モデル選定とプロンプト設計
- 複数ユーザー対応時のユーザー一覧管理方法
- 失敗ジョブの再実行方式

## 10. 要件トレーサビリティ

| 質問 | 回答 | 要件への反映 |
|---|---|---|
| Q1 | C | 初期版は単一ユーザー対象 |
| Q2 | A | ユーザーごとのタイムゾーンで対象日判定 |
| Q3 | X | 対象日境界を越えた後に自動生成 |
| Q4 | A | DynamoDB の正規化済みアクティビティを主データソースにする |
| Q5 | X | 初期版はA相当、将来カスタム可能にする |
| Q6 + Follow-up | B | LLM で分類、トピック抽出、要約を行う |
| Q7 | A | ユーザーごとに Notion Database ID と Token を設定 |
| Q8 | B | 既存ページを残し、新規ページを作成 |
| Q9 | B | ユーザーごとのジョブ分散を優先 |
| Q10 | C | エラーログのみ残し、失敗扱い |
| Q11 | X | IaC は後回し |
| Q12 | A | Python Lambda |
| Q13 | X | テスト詳細は後回し |
| Q14 | A | Security Baseline を有効化 |
| Q15 | A | PBT を有効化 |

## 11. Extension Compliance

### Security Baseline

- **Status**: Enabled
- **Requirements Analysis 評価**: 要件上、Secrets Manager、IAM 最小権限、暗号化、構造化ログ、秘匿情報のログ除外、`user_id` 分離を明記した。
- **Blocking Findings**: なし
- **後続フェーズでの注意**: Infrastructure Design と Code Generation では、IAM wildcard、平文シークレット、秘匿情報ログ出力がないことを検証する。

### Property-Based Testing

- **Status**: Enabled
- **Requirements Analysis 評価**: 日付範囲判定、タイムゾーン変換、ユーザー分離、データ変換、Notion 出力モデル生成を PBT 候補として明記した。
- **Blocking Findings**: なし
- **後続フェーズでの注意**: Functional Design で Testable Properties を明示し、Code Generation で Hypothesis 等による PBT を具体化する。
