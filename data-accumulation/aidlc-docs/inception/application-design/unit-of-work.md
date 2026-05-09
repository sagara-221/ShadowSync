# Unit of Work Definition

## Overview
このドキュメントは、ShadowSyncデータ蓄積システムを実装可能なユニット（作業単位）に分解した定義です。

**分解方針**:
- Lambda関数を個別ユニットとして扱う
- データ取り込みと処理を統合ユニットとして管理
- 機能別CloudFormationスタック構成
- 関数別ディレクトリ構造でコード配置

---

## Unit Definitions

### Unit 1: Ingestion & Processing
**目的**: データ取り込みから初期処理までの統合パイプライン

**責任範囲**:
- IoT Coreによるメッセージ受信（3つのロガーツールから）
- Kinesis Data Streamsによるストリーミング処理
- Lambda Routerによるメッセージルーティング
- Lambda Structuredによる構造化データの直接DynamoDB保存
- Lambda Screenshot Metaによるスクリーンショットメタデータ保存

**含まれるコンポーネント**:
- AWS IoT Core（MQTTブローカー、トピック、ルール）
- Kinesis Data Streams
- Lambda: Router
- Lambda: Structured
- Lambda: Screenshot Meta
- IoT Rules Engine

**CloudFormationスタック**: `iac/ingestion-processing.yaml`

**コード配置**:
```
lambda/
  ├── router/
  │   ├── handler.py
  │   ├── requirements.txt
  │   └── tests/
  ├── structured/
  │   ├── handler.py
  │   ├── requirements.txt
  │   └── tests/
  └── screenshot-meta/
      ├── handler.py
      ├── requirements.txt
      └── tests/
```

**環境変数**:
- `KINESIS_STREAM_NAME`: Kinesisストリーム名
- `DYNAMODB_TABLE_NAME`: DynamoDBテーブル名
- `LOG_LEVEL`: ログレベル（INFO, DEBUG, ERROR）

**IAMロール**:
- IoT Core → Kinesis書き込み権限
- Lambda Router → Kinesis読み取り、Lambda呼び出し権限
- Lambda Structured → DynamoDB書き込み権限
- Lambda Screenshot Meta → DynamoDB書き込み権限

---

### Unit 2: Lambda Bedrock (AI Processing)
**目的**: スクリーンショット画像の日本語キャプション生成

**責任範囲**:
- S3 ObjectCreatedイベントの受信
- S3から画像取得
- Amazon Bedrock Nova Liteによるキャプション生成
- DynamoDBレコードの更新（caption_ja追加）

**含まれるコンポーネント**:
- Lambda: Bedrock
- S3 Event Notification設定
- Amazon Bedrock統合

**CloudFormationスタック**: `iac/ai-processing.yaml`

**コード配置**:
```
lambda/
  └── bedrock/
      ├── handler.py
      ├── requirements.txt
      └── tests/
```

**環境変数**:
- `BEDROCK_MODEL_ID`: Bedrockモデル識別子（amazon.nova-lite-v1:0）
- `DYNAMODB_TABLE_NAME`: DynamoDBテーブル名
- `S3_BUCKET_NAME`: スクリーンショットバケット名
- `LOG_LEVEL`: ログレベル

**IAMロール**:
- Lambda Bedrock → S3読み取り、Bedrock呼び出し、DynamoDB更新権限

**処理フロー**:
1. S3 ObjectCreated イベント受信
2. イベントからS3オブジェクトキー抽出
3. S3から画像取得（WebP形式）
4. Bedrockで日本語キャプション生成
5. DynamoDBレコード検索（s3_object_keyで）
6. caption_jaフィールドを更新

---

### Unit 3: Lambda Presigned URL (API)
**目的**: スクリーンショット画像アップロード用のPresigned URL生成

**責任範囲**:
- API Gateway HTTP APIエンドポイント提供
- Cognito認証統合
- S3 Presigned URL生成
- ユーザーID検証とアクセス制御

**含まれるコンポーネント**:
- API Gateway HTTP API
- Lambda: Presigned URL
- Cognito統合（オーソライザー）

**CloudFormationスタック**: `iac/api.yaml`

**コード配置**:
```
lambda/
  └── presigned-url/
      ├── handler.py
      ├── requirements.txt
      └── tests/
```

**環境変数**:
- `S3_BUCKET_NAME`: スクリーンショットバケット名
- `PRESIGNED_URL_EXPIRATION`: URL有効期限（秒）
- `LOG_LEVEL`: ログレベル

**IAMロール**:
- Lambda Presigned URL → S3 PutObject権限（user_id制限付き）

**APIエンドポイント**:
- `POST /api/v1/upload/presigned-url`
- 認証: Cognito User Pool
- リクエスト: `{ "file_name": "screenshot.webp", "content_type": "image/webp", "file_size": 1048576 }`
- レスポンス: `{ "presigned_url": "https://...", "s3_key": "...", "expires_at": "..." }`

---

### Unit 4: Storage
**目的**: データ永続化とベクトルストレージ

**責任範囲**:
- DynamoDBテーブル管理（共通スキーマ）
- S3バケット管理（スクリーンショット、ベクトル）
- GSI（Global Secondary Index）設定
- ライフサイクルポリシー設定

**含まれるコンポーネント**:
- DynamoDB Table: `shadowsync-activities`
- S3 Bucket: Raw Screenshots
- S3 Bucket: Vectors (RAG用)
- GSI定義（3つ）

**CloudFormationスタック**: `iac/storage.yaml`

**DynamoDBテーブル設計**:
- **テーブル名**: `shadowsync-activities`
- **パーティションキー**: `user_id` (String)
- **ソートキー**: `timestamp_event_id` (String)
- **キャパシティモード**: オンデマンド
- **暗号化**: 有効

**GSI定義**:
1. **GSI-1**: `user_id` + `logger_type#timestamp` - ロガータイプ別クエリ
2. **GSI-2**: `user_id` + `event_type#timestamp` - イベントタイプ別クエリ
3. **GSI-3**: `device_id` + `timestamp` - デバイス別クエリ

**S3バケット構成**:
1. **Raw Screenshots Bucket**:
   - パス: `raw/screenshots/{user_id}/{device_id}/YYYY/MM/DD/HH-mm-ss.webp`
   - 暗号化: SSE-S3
   - ライフサイクル: 90日後Glacier移行
   - イベント通知: ObjectCreated → Lambda Bedrock

2. **Vectors Bucket**:
   - 用途: Bedrock Knowledge Base用ベクトルストレージ
   - 暗号化: SSE-S3

**IAMポリシー**:
- マルチテナント分離（user_idベースのパス制限）
- 最小権限の原則

---

### Unit 5: Authentication
**目的**: 認証・認可基盤の提供

**責任範囲**:
- Cognito User Pool管理（Chrome拡張用）
- Cognito ID Pool管理（一時認証情報）
- X.509証明書管理（ローカルアプリ用）
- IoT Coreポリシー設定（トピックアクセス制御）

**含まれるコンポーネント**:
- Amazon Cognito User Pool
- Amazon Cognito ID Pool
- AWS IoT Core証明書プロビジョニング
- IoT Coreポリシー

**CloudFormationスタック**: `iac/authentication.yaml`

**Cognito User Pool設定**:
- ユーザー名/パスワード認証
- MFA: オプション
- パスワードポリシー: 強力
- アプリクライアント: Chrome拡張用

**Cognito ID Pool設定**:
- 認証プロバイダー: Cognito User Pool
- 未認証アクセス: 無効
- IAMロール: 認証済みユーザー用

**X.509証明書管理**:
- 自動プロビジョニング: 有効
- 証明書ローテーション: 手動
- 証明書失効リスト: 設定

**IoT Coreポリシー**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["iot:Connect"],
      "Resource": ["arn:aws:iot:*:*:client/${iot:Connection.Thing.ThingName}"]
    },
    {
      "Effect": "Allow",
      "Action": ["iot:Publish"],
      "Resource": ["arn:aws:iot:*:*:topic/shadowsync/logs/${iot:Connection.Thing.Attributes[user_id]}/*"]
    }
  ]
}
```

**マルチテナント分離**:
- トピックフィルタリング: user_idベース
- IAMポリシー: user_id変数使用
- データアクセス制御: DynamoDB/S3パス制限

---

## Code Organization Strategy

### Directory Structure
```
data-accumulation/
├── aidlc-docs/                    # AI-DLC documentation
│   ├── inception/
│   │   ├── requirements/
│   │   ├── plans/
│   │   └── application-design/
│   └── aidlc-state.md
├── lambda/                         # Lambda functions (function-based)
│   ├── router/
│   │   ├── handler.py
│   │   ├── requirements.txt
│   │   └── tests/
│   ├── structured/
│   │   ├── handler.py
│   │   ├── requirements.txt
│   │   └── tests/
│   ├── screenshot-meta/
│   │   ├── handler.py
│   │   ├── requirements.txt
│   │   └── tests/
│   ├── bedrock/
│   │   ├── handler.py
│   │   ├── requirements.txt
│   │   └── tests/
│   └── presigned-url/
│       ├── handler.py
│       ├── requirements.txt
│       └── tests/
├── layers/                         # Lambda Layers
│   └── common/
│       └── python/
│           ├── utils.py
│           ├── schema.py
│           └── aws_clients.py
├── iac/                            # Infrastructure as Code
│   ├── ingestion-processing.yaml
│   ├── ai-processing.yaml
│   ├── api.yaml
│   ├── storage.yaml
│   ├── authentication.yaml
│   └── params/
│       ├── dev.json
│       ├── staging.json
│       └── prod.json
├── tests/                          # Integration & E2E tests
│   ├── integration/
│   └── e2e/
├── scripts/                        # Deployment scripts
│   ├── deploy.sh
│   └── validate.sh
└── README.md
```

### Deployment Model
- **デプロイ単位**: ユニット単位（各CloudFormationスタック独立）
- **スタック依存関係**: CloudFormation Export/Import使用
- **環境管理**: 環境変数で切り替え
- **デプロイ順序**:
  1. Storage（基盤）
  2. Authentication（認証）
  3. Ingestion & Processing（データパイプライン）
  4. AI Processing（Bedrock）
  5. API（エンドポイント）

### Testing Strategy
- **ユニットテスト**: 各Lambda関数内にtestsディレクトリ
- **統合テスト**: `tests/integration/`でサービス間連携テスト
- **E2Eテスト**: `tests/e2e/`で完全なデータフローテスト
- **テスト範囲**: すべてのコンポーネント（Lambda、IoT Rules、DynamoDB操作）

---

## Unit Characteristics

| Unit | Type | Complexity | Dependencies | Deployment Priority |
|------|------|------------|--------------|---------------------|
| Storage | Infrastructure | Low | None | 1 (First) |
| Authentication | Infrastructure | Medium | None | 2 |
| Ingestion & Processing | Service | High | Storage, Authentication | 3 |
| Lambda Bedrock | Service | Medium | Storage | 4 |
| Lambda Presigned URL | Service | Low | Storage, Authentication | 5 |

**Complexity Factors**:
- **Low**: 単純なリソース定義、依存関係少ない
- **Medium**: 複数サービス統合、認証/認可ロジック
- **High**: 複雑なデータフロー、複数Lambda関数、イベント駆動

---

## Success Criteria

### Unit Completeness
- [ ] すべてのユニットに明確な責任範囲が定義されている
- [ ] すべてのユニットにCloudFormationスタックが割り当てられている
- [ ] すべてのLambda関数にコード配置場所が定義されている
- [ ] すべてのユニットにIAMロールが定義されている

### Code Organization
- [ ] ディレクトリ構造が明確に定義されている
- [ ] Lambda Layer戦略が定義されている
- [ ] テストコード配置が明確になっている
- [ ] デプロイスクリプト配置が定義されている

### Deployment Strategy
- [ ] デプロイ順序が明確になっている
- [ ] スタック間依存関係が定義されている
- [ ] 環境管理方法が定義されている
- [ ] ロールバック戦略が考慮されている

---

**Document Status**: Complete
**Next Step**: Generate unit dependency matrix and story mapping
