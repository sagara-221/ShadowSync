# Unit of Work to Story Mapping

## Overview
このドキュメントは、要件定義書の機能要件（FR）を各ユニットにマッピングし、すべての要件がカバーされていることを検証します。

**Note**: このプロジェクトはUser Storiesステージをスキップしたため、要件定義書（requirements.md）の機能要件を直接マッピングします。

---

## Functional Requirements Coverage

### FR-1: データ取り込みパイプライン

#### FR-1.1: IoT Coreメッセージ受信
**Assigned Unit**: Ingestion & Processing  
**Components**:
- AWS IoT Core (MQTTブローカー)
- IoT Rules Engine
- トピック設定（3つのロガーツール対応）

**Implementation Details**:
- Chrome拡張: MQTT over WebSockets (port 443) + Cognito認証
- OS APIロガー: MQTTS + X.509認証
- スクリーンショットロガー: MQTTS + X.509認証

**Status**: ✅ Covered

---

#### FR-1.2: 認証と認可
**Assigned Unit**: Authentication  
**Components**:
- Cognito User Pool (Chrome拡張用)
- Cognito ID Pool (一時認証情報)
- X.509証明書管理 (ローカルアプリ用)
- IoT Coreポリシー (マルチテナント分離)

**Implementation Details**:
- 自動プロビジョニング
- user_idベースのトピックフィルタリング
- 厳格なIAMポリシー

**Status**: ✅ Covered

---

#### FR-1.3: メッセージルーティング
**Assigned Unit**: Ingestion & Processing  
**Components**:
- Kinesis Data Streams
- Lambda Router
- Lambda Structured
- Lambda Screenshot Meta

**Implementation Details**:
- logger_type + event_typeによる分岐
- 構造化データ: 直接DynamoDB保存
- スクリーンショットメタデータ: DynamoDB保存（キャプションは後処理）

**Status**: ✅ Covered

---

### FR-2: 画像アップロード基盤

#### FR-2.1: Presigned URL生成
**Assigned Unit**: Lambda Presigned URL  
**Components**:
- API Gateway HTTP API
- Lambda Presigned URL
- Cognito統合

**Implementation Details**:
- POST /api/v1/upload/presigned-url
- ユーザーID検証
- S3 Presigned URL生成

**Status**: ✅ Covered

---

#### FR-2.2: Presigned URLセキュリティ
**Assigned Unit**: Lambda Presigned URL  
**Components**:
- IAMポリシー (user_idベースのS3プレフィックス制限)
- URL有効期限設定
- アップロード後検証 (Lambda Bedrock)

**Implementation Details**:
- 有効期限: 1-24時間
- user_idベースのアクセス制御
- ファイルサイズ・タイプ検証

**Status**: ✅ Covered

---

#### FR-2.3: S3ストレージ構成
**Assigned Unit**: Storage  
**Components**:
- S3 Bucket: Raw Screenshots
- S3 Bucket: Vectors
- ライフサイクルポリシー
- 暗号化設定

**Implementation Details**:
- パス: `raw/screenshots/{user_id}/{device_id}/YYYY/MM/DD/HH-mm-ss.webp`
- 画像形式: WebP
- 暗号化: SSE-S3
- ライフサイクル: 90日後Glacier移行

**Status**: ✅ Covered

---

### FR-3: データ変換と正規化

#### FR-3.1: 構造化データの直接マッピング
**Assigned Unit**: Ingestion & Processing  
**Components**:
- Lambda Structured
- Lambda Screenshot Meta

**Implementation Details**:
- Chrome拡張: ブラウザアクティビティ → DynamoDB
- OS APIロガー: ウィンドウ/オーディオ/スナップショット → DynamoDB
- スクリーンショットロガー: メタデータのみ → DynamoDB

**Status**: ✅ Covered

---

#### FR-3.2: 画像キャプション生成（S3イベントトリガー）
**Assigned Unit**: Lambda Bedrock  
**Components**:
- Lambda Bedrock
- Amazon Bedrock Nova Lite
- S3 Event Notification

**Implementation Details**:
- S3 ObjectCreated イベントトリガー
- 画像取得 → Bedrock呼び出し → 日本語キャプション生成
- DynamoDBレコード更新 (caption_ja追加)

**Status**: ✅ Covered

---

#### FR-3.3: 共通スキーマ定義（DynamoDBスキーマ）
**Assigned Unit**: Storage  
**Components**:
- DynamoDB Table: shadowsync-activities
- GSI定義 (3つ)

**Implementation Details**:
- パーティションキー: user_id
- ソートキー: timestamp_event_id
- 共通属性: timestamp, device_id, logger_type, event_type, activity_data
- GSI-1: logger_type別クエリ
- GSI-2: event_type別クエリ
- GSI-3: device_id別クエリ

**Status**: ✅ Covered

---

### FR-4: データストレージ

#### FR-4.1: DynamoDBテーブル設計
**Assigned Unit**: Storage  
**Components**:
- DynamoDB Table
- GSI設定
- キャパシティモード設定

**Implementation Details**:
- 単一テーブル設計
- オンデマンドキャパシティ
- 保管時の暗号化

**Status**: ✅ Covered

---

#### FR-4.2: Amazon Bedrock Knowledge BaseによるRAG検索
**Assigned Unit**: Storage  
**Components**:
- S3 Bucket: Vectors
- Bedrock Knowledge Base設定 (将来実装)

**Implementation Details**:
- S3 Vectorsをバックエンドとして使用
- DynamoDBデータをナレッジベースに同期
- ユーザー固有のナレッジベース

**Status**: ✅ Covered (Infrastructure準備完了、同期ロジックは将来実装)

---

#### FR-4.3: 生データ保持
**Assigned Unit**: Storage  
**Components**:
- S3 Bucket: Raw Screenshots
- ライフサイクルポリシー

**Implementation Details**:
- 元のログと画像を保存
- 90日後にS3 Glacierへアーカイブ

**Status**: ✅ Covered

---

### FR-5: エラーハンドリングと信頼性

#### FR-5.1: リトライメカニズム
**Assigned Units**: All Lambda units  
**Components**:
- Lambda retry設定
- Kinesis retry設定
- Bedrock retry設定

**Implementation Details**:
- Lambda: 指数バックオフで2回リトライ
- Kinesis: 設定可能なリトライ回数
- Bedrock: スロットリング時のリトライ

**Status**: ✅ Covered

---

#### FR-5.2: デッドレターキュー（DLQ）
**Assigned Units**: Ingestion & Processing, Lambda Bedrock  
**Components**:
- SQS DLQ
- CloudWatchアラーム

**Implementation Details**:
- すべてのリトライ失敗後にDLQへ
- DLQ深度のCloudWatchアラーム
- 手動/自動再処理

**Status**: ✅ Covered

---

#### FR-5.3: データ整合性
**Assigned Units**: All Lambda units  
**Components**:
- Lambda冪等性処理
- DynamoDBトランザクション
- CloudWatch Logs

**Implementation Details**:
- 重複メッセージ処理の保証
- 必要に応じてトランザクション使用
- すべての処理ステップをログ記録

**Status**: ✅ Covered

---

## Non-Functional Requirements Coverage

### NFR-1: パフォーマンス
**Assigned Units**: Ingestion & Processing, Lambda Bedrock  
**Implementation**:
- Kinesis Data Streamsのオートスケーリング
- Lambda同時実行数設定
- DynamoDBオンデマンドキャパシティ

**Status**: ✅ Covered

---

### NFR-2: セキュリティ
**Assigned Units**: Authentication, All units  
**Implementation**:
- X.509証明書認証
- Cognito User Pool認証
- 最小権限IAMポリシー
- TLS 1.2以上
- 保管時の暗号化（S3, DynamoDB, Kinesis）
- マルチテナント分離

**Status**: ✅ Covered

---

### NFR-3: コスト最適化
**Assigned Units**: All units  
**Implementation**:
- サーバーレスアーキテクチャ
- DynamoDBオンデマンドモード
- S3ライフサイクルポリシー
- Bedrock使用最小化（スクリーンショットのみ）
- Amazon Nova Lite使用

**Status**: ✅ Covered

---

### NFR-4: 保守性
**Assigned Units**: All units  
**Implementation**:
- CloudFormation IaC
- 機能ベースのスタック構造
- 関数別ディレクトリ構造
- Lambda Layer共有ライブラリ
- 環境変数による設定管理

**Status**: ✅ Covered

---

### NFR-5: テスト容易性
**Assigned Units**: All units  
**Implementation**:
- ユニットテスト（各Lambda関数）
- 統合テスト（サービス間相互作用）
- E2Eテスト（完全なデータフロー）
- テスト環境分離

**Status**: ✅ Covered

---

### NFR-6: 可観測性
**Assigned Units**: All units  
**Implementation**:
- CloudWatch Logs（構造化ログ、7日間保持）
- CloudWatch Metrics（基本メトリクス）
- CloudWatchアラーム（エラー、DLQ深度）

**Status**: ✅ Covered

---

## Unit Responsibility Matrix

| Unit | Functional Requirements | Non-Functional Requirements | Priority |
|------|------------------------|----------------------------|----------|
| **Ingestion & Processing** | FR-1.1, FR-1.3, FR-3.1, FR-5.1, FR-5.2, FR-5.3 | NFR-1, NFR-2, NFR-3, NFR-4, NFR-5, NFR-6 | High |
| **Lambda Bedrock** | FR-3.2, FR-5.1, FR-5.2, FR-5.3 | NFR-1, NFR-2, NFR-3, NFR-4, NFR-5, NFR-6 | High |
| **Lambda Presigned URL** | FR-2.1, FR-2.2, FR-5.3 | NFR-2, NFR-3, NFR-4, NFR-5, NFR-6 | Medium |
| **Storage** | FR-2.3, FR-3.3, FR-4.1, FR-4.2, FR-4.3 | NFR-2, NFR-3, NFR-4, NFR-6 | High |
| **Authentication** | FR-1.2 | NFR-2, NFR-3, NFR-4 | High |

**Priority Explanation**:
- **High**: システムのコア機能、データフロー必須
- **Medium**: 補助機能、段階的実装可能

---

## Coverage Verification

### Functional Requirements
- **Total FRs**: 13 (FR-1.1 ~ FR-5.3)
- **Covered**: 13
- **Coverage**: 100% ✅

### Non-Functional Requirements
- **Total NFRs**: 6 (NFR-1 ~ NFR-6)
- **Covered**: 6
- **Coverage**: 100% ✅

### Uncovered Requirements
**None** - すべての機能要件と非機能要件が5つのユニットにマッピングされています。

---

## Implementation Priority

### Phase 1: Foundation (Priority: Critical)
1. **Storage** - データ永続化基盤
2. **Authentication** - 認証・認可基盤

**Rationale**: すべてのサービスが依存する基盤

---

### Phase 2: Core Data Pipeline (Priority: High)
3. **Ingestion & Processing** - データ取り込みと初期処理

**Rationale**: システムのコア機能、データフロー開始点

---

### Phase 3: AI Processing (Priority: High)
4. **Lambda Bedrock** - 画像キャプション生成

**Rationale**: スクリーンショットの価値を高める重要機能

---

### Phase 4: API (Priority: Medium)
5. **Lambda Presigned URL** - 画像アップロードAPI

**Rationale**: スクリーンショットアップロードに必要だが、段階的実装可能

---

## Success Criteria

### Coverage Completeness
- [x] すべての機能要件がユニットにマッピングされている
- [x] すべての非機能要件がユニットにマッピングされている
- [x] カバレッジが100%である
- [x] 未カバー要件がゼロである

### Mapping Quality
- [x] 各ユニットの責任範囲が明確である
- [x] 要件の重複割り当てがない（または意図的である）
- [x] 実装優先度が定義されている
- [x] 依存関係が考慮されている

### Documentation
- [x] すべてのマッピングに実装詳細が記載されている
- [x] ステータスが明確に示されている
- [x] 将来実装項目が識別されている

---

## Notes

### Future Enhancements (Out of Scope)
以下の機能は現在のスコープ外ですが、将来的に追加される可能性があります：

1. **Bedrock Knowledge Base同期ロジック**
   - DynamoDBからS3 Vectorsへのデータ同期
   - ベクトル化とインデックス作成
   - RAGクエリインターフェース

2. **リアルタイム通知**
   - ユーザーアクティビティに関するプッシュ通知
   - SNS/SQS統合

3. **高度な分析**
   - 複雑な分析と可視化ダッシュボード
   - QuickSight統合

4. **データエクスポート機能**
   - ユーザーデータの一括エクスポート
   - CSV/JSON形式サポート

---

**Document Status**: Complete
**Coverage**: 100% (13/13 FRs, 6/6 NFRs)
**Next Step**: Proceed to Units Generation completion and CONSTRUCTION phase
