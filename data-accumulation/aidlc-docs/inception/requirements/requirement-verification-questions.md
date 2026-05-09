# Requirements Verification Questions

**Purpose**: Clarify and validate requirements for the data-accumulation system

**Instructions**: Please answer each question by filling in the [Answer]: tag with the letter of your choice (A, B, C, D, X). For option X (Other), please provide your custom response after the [Answer]: tag.

---

## Question 1: IoT Core認証方式の詳細
intent.mdでは「X.509証明書認証（ローカルアプリ用）」と「Cognito IDプール（Chrome拡張用）」の両方を構築すると記載されていますが、以下の詳細を確認させてください。

**1.1: X.509証明書の管理方法**
ローカルアプリ（data-logger）用のX.509証明書はどのように管理しますか？

A) AWS IoT Coreで証明書を生成し、手動でローカルアプリに配布する
B) AWS IoT Coreで証明書を生成し、自動プロビジョニング機能を使用する
C) 既存の証明書管理システムを使用する
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

**1.2: Cognito IDプールの認証フロー**
Chrome拡張機能からのアクセスでは、どのような認証フローを想定していますか？

A) Cognito User Poolsでユーザー認証後、IDプールで一時的なAWS認証情報を取得
B) 匿名認証（Unauthenticated identities）を使用
C) 外部IDプロバイダー（Google, Facebook等）と連携
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 2: データスキーマと正規化の詳細

**2.1: 共通スキーマの構造**
「意味のある共通形式に変換・正規化」とありますが、共通スキーマにはどのような情報を含めますか？

A) タイムスタンプ、user_id、アクティビティタイプ、コンテキスト（抽出された意味情報）のみ
B) 上記に加えて、元データへの参照（S3パス等）と信頼度スコア
C) 上記に加えて、タグ、カテゴリ、関連エンティティ（プロジェクト名、ファイル名等）
D) まだ決定していない（推奨構造を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

**2.2: 画像データの処理方針**
スクリーンショット画像からの情報抽出について、どの程度の処理を行いますか？

A) OCRでテキスト抽出のみ
B) OCR + Bedrockでの画像解析（何をしているかの推測）
C) OCR + 画像解析 + オブジェクト検出（アプリケーションUI要素の識別）
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: X(Bedrockの軽量モデルAmazon Nova Liteなどによりその画面で具体的にどのような作業をしているのかを抽出)

---

## Question 3: データストレージ戦略

**3.1: DynamoDBのテーブル設計**
マルチテナント構成でのDynamoDBテーブル設計について、どのアプローチを採用しますか？

A) 単一テーブル設計（user_idをパーティションキー、timestamp等をソートキー）
B) ユーザーごとに個別テーブルを作成
C) アクティビティタイプごとにテーブルを分割（各テーブルでuser_idをパーティションキー）
D) まだ決定していない（推奨設計を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

**3.2: RAG検索用Knowledge Baseの構成**
Amazon Bedrock Knowledge Baseの構成について教えてください。

A) 全ユーザーのデータを単一のKnowledge Baseに格納（メタデータでフィルタリング）
B) ユーザーごとに個別のKnowledge Baseを作成
C) Knowledge Baseは使用せず、DynamoDBとOpenSearch等で独自に実装
D) まだ決定していない（推奨構成を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: X(料金をかけたくないのでS3 Vectorの機能を使ってユーザーごとにKnowledgeベースを作る)

---

## Question 4: データパイプラインのアーキテクチャ

**4.1: メッセージ処理の非同期パターン**
IoT Coreから受信したメッセージの処理フローはどのようにしますか？

A) IoT Core → Lambda（直接処理）→ DynamoDB/S3
B) IoT Core → EventBridge → Lambda → DynamoDB/S3
C) IoT Core → Kinesis Data Streams → Lambda → DynamoDB/S3
D) IoT Core → SQS → Lambda → DynamoDB/S3
E) まだ決定していない（推奨アーキテクチャを提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: X(Cでも良いが、AWS Glueなどを噛ませた方がよければそちらにして欲しい。料金があまりかからない構成にしたい)

**4.2: Bedrock呼び出しの処理方式**
Bedrockでの意味抽出処理は同期・非同期どちらで行いますか？

A) 同期処理（リアルタイムで結果を返す）
B) 非同期処理（バッチ処理で後から処理）
C) ハイブリッド（重要度や緊急度に応じて使い分け）
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 5: Presigned URL発行の仕組み

**5.1: Presigned URL発行のエンドポイント**
画像アップロード用のPresigned URLはどのように発行しますか？

A) API Gateway + Lambda でHTTP APIを提供
B) IoT Coreのカスタムトピックでリクエスト/レスポンス
C) AppSync（GraphQL）経由で発行
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: X(料金がかからない、かつ構成がシンプルなものを提案してください)

**5.2: Presigned URLの有効期限とセキュリティ**
Presigned URLの有効期限とアクセス制御はどうしますか？

A) 短時間（5-15分）の有効期限、user_id別のS3プレフィックスで分離
B) 長時間（1-24時間）の有効期限、アップロード後にLambdaで検証
C) 中程度（30分-1時間）の有効期限、S3バケットポリシーで厳密に制御
D) まだ決定していない（推奨設定を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 6: エラーハンドリングとリトライ

**6.1: データ処理失敗時の対応**
Bedrock呼び出しやデータ変換が失敗した場合、どのように対応しますか？

A) DLQ（Dead Letter Queue）に送信し、手動で再処理
B) 自動リトライ（指数バックオフ）+ 最終的にDLQへ
C) エラーログのみ記録し、元データは保持（後で再処理可能）
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 7: IaCツールの選択

**7.1: インフラストラクチャのコード化**
IaCツールはどれを使用しますか？

A) Terraform
B) AWS SAM
C) AWS CDK
D) CloudFormation（直接）
E) まだ決定していない（推奨ツールを提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: D

**7.2: IaCのディレクトリ構造**
IaCコードはどこに配置しますか？

A) ワークスペースルートに `infrastructure/` または `terraform/` ディレクトリを作成
B) ワークスペースルートに `iac/` ディレクトリを作成
C) 各サービスごとにディレクトリを分けて配置（例: `lambda/`, `iot/` 等）
D) まだ決定していない（推奨構造を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: X(iac/フォルダを作成し、その中で各サービスごとにディレクトリを分けてください)

---

## Question 8: 開発環境とテスト戦略

**8.1: ローカル開発環境**
開発時のローカルテスト環境はどうしますか？

A) LocalStack等を使用してAWSサービスをローカルでエミュレート
B) AWS上に開発用環境を構築（本番と分離）
C) ユニットテストのみローカル、統合テストはAWS上で実行
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

**8.2: テストの範囲**
どの程度のテストカバレッジを目指しますか？

A) ユニットテストのみ（Lambda関数の個別テスト）
B) ユニットテスト + 統合テスト（サービス間連携のテスト）
C) ユニットテスト + 統合テスト + E2Eテスト（実際のIoT Coreからのフロー全体）
D) まだ決定していない（推奨範囲を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 9: モニタリングとロギング

**9.1: ログ管理**
アプリケーションログはどのように管理しますか？

A) CloudWatch Logsのみ
B) CloudWatch Logs + 構造化ログ（JSON形式）
C) CloudWatch Logs + 外部ログ管理サービス（Datadog, Splunk等）
D) まだ決定していない（推奨方式を提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

**9.2: アラートとメトリクス**
どのようなメトリクスを監視し、アラートを設定しますか？

A) 基本的なメトリクスのみ（Lambda実行エラー、DynamoDB書き込み失敗等）
B) 基本 + ビジネスメトリクス（処理件数、Bedrock呼び出し成功率等）
C) 基本 + ビジネス + コストメトリクス（Bedrock利用料金、S3ストレージコスト等）
D) まだ決定していない（推奨メトリクスを提案してほしい）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10: セキュリティ拡張機能
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)
B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 11: Property-Based Testing拡張機能
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)
B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)
C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Instructions for Completion

1. Review each question carefully
2. Fill in [Answer]: with your chosen letter (A, B, C, D, E, or X)
3. For option X, provide detailed explanation after the [Answer]: tag
4. Save this file after completing all answers
5. Notify the AI that answers are ready for review

---
