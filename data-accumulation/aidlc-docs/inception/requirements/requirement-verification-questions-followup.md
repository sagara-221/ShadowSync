# Requirements Verification Questions - Follow-up

**Purpose**: Clarify ambiguous or incomplete answers from the initial questionnaire

**Instructions**: Please answer each question by filling in the [Answer]: tag with the letter of your choice (A, B, C, D, X). For option X (Other), please provide your custom response after the [Answer]: tag.

---

## Follow-up 1: データパイプラインアーキテクチャの詳細

**元の回答**: Question 4.1で「Cでも良いが、AWS Glueなどを噛ませた方がよければそちらにして欲しい。料金があまりかからない構成にしたい」

AWS Glueは主にETL（Extract, Transform, Load）バッチ処理やデータカタログ管理に使用されるサービスで、リアルタイムストリーミング処理には通常使用されません。また、Glueは比較的コストが高いサービスです。

**推奨アーキテクチャ**:
- **オプションA（シンプル・低コスト）**: IoT Core → IoT Rules → Lambda → DynamoDB/S3
  - メリット: 最もシンプル、追加サービス不要、低コスト
  - デメリット: スケーラビリティに限界、バックプレッシャー対応が難しい
  
- **オプションB（スケーラブル・中コスト）**: IoT Core → IoT Rules → Kinesis Data Streams → Lambda → DynamoDB/S3
  - メリット: 高スループット対応、バッファリング、リトライ機能
  - デメリット: Kinesisのシャード料金が発生（$0.015/シャード/時間）

- **オプションC（バランス型・低コスト）**: IoT Core → IoT Rules → SQS → Lambda → DynamoDB/S3
  - メリット: 低コスト（最初の100万リクエスト無料）、バッファリング、DLQ対応
  - デメリット: Kinesisより順序保証が弱い

個人利用で料金を抑えたい場合、どのアーキテクチャを採用しますか？

A) オプションA（IoT Rules → Lambda直接）- 最もシンプルで低コスト
B) オプションB（Kinesis Data Streams経由）- スケーラビリティ重視
C) オプションC（SQS経由）- コストとスケーラビリティのバランス
D) その他の提案があれば教えてください

[Answer]: B

---

## Follow-up 2: Presigned URL発行エンドポイントの選択

**元の回答**: Question 5.1で「料金がかからない、かつ構成がシンプルなものを提案してください」

**推奨オプション**:
- **オプションA（最もシンプル・低コスト）**: Lambda Function URL
  - メリット: 追加サービス不要、HTTPSエンドポイント自動提供、最初の100万リクエスト無料
  - デメリット: 認証はIAM or なし（Cognito統合は手動実装が必要）
  
- **オプションB（認証統合・中コスト）**: API Gateway HTTP API + Lambda
  - メリット: Cognito統合が簡単、リクエスト制限・スロットリング機能
  - デメリット: API Gateway料金（$1.00/百万リクエスト）

- **オプションC（IoT統合・低コスト）**: IoT Core カスタムトピック（MQTT）
  - メリット: 既存のIoT接続を再利用、追加コストほぼなし
  - デメリット: Chrome拡張からはMQTT over WebSocketが必要

Chrome拡張とローカルアプリの両方から使用することを考慮すると、どれを選びますか？

A) Lambda Function URL（最もシンプル、認証は手動実装）
B) API Gateway HTTP API + Lambda（Cognito統合が簡単）
C) IoT Core カスタムトピック（既存接続を再利用）
D) その他の提案があれば教えてください

[Answer]: B

---

## Follow-up 3: Presigned URLの有効期限（Question 5.2の未回答）

Question 5.2が未回答でした。Presigned URLの有効期限とセキュリティ設定について教えてください。

A) 短時間（5-15分）の有効期限、user_id別のS3プレフィックスで分離
B) 長時間（1-24時間）の有効期限、アップロード後にLambdaで検証
C) 中程度（30分-1時間）の有効期限、S3バケットポリシーで厳密に制御
D) その他の設定があれば教えてください

[Answer]: B

---

## Follow-up 4: S3 Vectorの詳細確認

**元の回答**: Question 3.2で「料金をかけたくないのでS3 Vectorの機能を使ってユーザーごとにKnowledgeベースを作る」

確認させてください。以下のどちらを想定していますか？

A) Amazon Bedrock Knowledge Base with S3 as data source（ベクトルストアはOpenSearch Serverless or Aurora）
B) S3に直接ベクトルデータを保存し、独自の検索ロジックを実装
C) その他の方法があれば教えてください

**補足情報**:
- Bedrock Knowledge Baseを使用する場合、ベクトルストアとして以下が必要です：
  - OpenSearch Serverless（最低料金: 約$700/月）
  - Amazon Aurora PostgreSQL with pgvector（最低料金: 約$50/月）
  - Pinecone等の外部サービス
- S3に直接ベクトルを保存する場合、検索機能は自前で実装が必要です

料金を抑えたい場合、どのアプローチを採用しますか？

A) Bedrock Knowledge Base + Aurora PostgreSQL（最も統合が簡単、月約$50）
B) S3 + 独自検索実装（最も低コスト、実装が複雑）
C) 初期段階ではKnowledge Baseを使わず、DynamoDBのみで検索（最もシンプル）
D) その他の提案があれば教えてください

[Answer]: D(Amazon Bedrock Knowledge Basesとの直接統合
Bedrockでナレッジベース（RAG）を作る際の保存先としてS3 Vectorsをポンと指定するだけで、超低コストなRAG環境が構築できます。参考：https://aws.amazon.com/jp/s3/features/vectors/)

---

## Follow-up 5: IaCディレクトリ構造の詳細

**元の回答**: Question 7.2で「iac/フォルダを作成し、その中で各サービスごとにディレクトリを分けてください」

CloudFormationを使用する場合、以下のどの構造を想定していますか？

A) サービスごとに独立したスタック（例: iac/iot/, iac/lambda/, iac/dynamodb/）
B) 機能ごとにスタックを分割（例: iac/ingestion/, iac/processing/, iac/storage/）
C) 単一のネストされたスタック（iac/main.yaml + iac/nested/）
D) その他の構造があれば教えてください

[Answer]: B

---

## Instructions for Completion

1. Review each follow-up question carefully
2. Fill in [Answer]: with your chosen letter (A, B, C, D, or X)
3. For option X or D, provide detailed explanation after the [Answer]: tag
4. Save this file after completing all answers
5. Notify the AI that answers are ready for review

---
