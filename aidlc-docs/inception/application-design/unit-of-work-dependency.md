# ユニット依存関係マトリクス (Unit Dependency Matrix)

| ユニット名 | 依存先 | 依存の性質 | インターフェース / プロトコル |
| :--- | :--- | :--- | :--- |
| `logger/osapi` | `data-accumulation` | OSアクティビティ、オーディオセッション、定期スナップショットの送信 | AWS IoT Core MQTTS (X.509) |
| `logger/ss-tool` | `data-accumulation` | スクリーンショットメタデータの送信、画像アップロードURL取得 | AWS IoT Core MQTTS (X.509), IoT Core Request/Response, S3 Presigned URL |
| `logger/chrome-extension` | `data-accumulation` | ブラウザアクティビティの送信、必要時の画像アップロードURL取得 | MQTT over WebSockets (Cognito ID Pool), Lambda Function URL (IAM), S3 Presigned URL |
| `data-accumulation` | - | ログ正規化、DynamoDB/S3/S3 Vectors管理、RAG用データ同期 | AWS IoT Core, Kinesis, Lambda, DynamoDB, S3, Bedrock, S3 Vectors / Bedrock Knowledge Base |
| `daily-log` | `data-accumulation` | 日報生成用の正規化済みアクティビティ読み取り | AWS SDK (DynamoDB中心。初期版ではS3/S3 Vectors依存を増やさない) |
| `digital-twin` | `data-accumulation` | RAG検索、根拠メタデータ取得 | Bedrock Knowledge Base / S3 Vectors検索I/F (詳細はConstructionで確定) |
