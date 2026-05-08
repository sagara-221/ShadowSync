# ユニット依存関係マトリクス (Unit Dependency Matrix)

| ユニット名 | 依存先 | 依存の性質 | インターフェース / プロトコル |
| :--- | :--- | :--- | :--- |
| `logger/*` (各孫ツール) | `data-accumulation` | データの送信先 | AWS IoT Core (MQTT) を想定 |
| `data-accumulation` | - | - | - |
| `daily-log` | `data-accumulation` | データの読み取り元 | AWS SDK (DynamoDB, S3 等) |
| `digital-twin` | `data-accumulation` | データの読み取り元 | AWS SDK / Bedrock Knowledge Base 等 |
