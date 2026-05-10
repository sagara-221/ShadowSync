# Constructionで行うこと

**作成日**: 2026-05-10  
**対象**: ShadowSync 子AI-DLC / 孫AI-DLC 全体  
**目的**: Inception完了後に、ConstructionのFunctional Design / Infrastructure Design / Code Generationで具体化する作業を整理する。

---

## 1. 現状サマリ

親AI-DLCの想定仕様は、子AI-DLCのInception結果に合わせて更新済み。主な反映内容は以下。

- 構造化データはLLM処理せず、LambdaでDynamoDBスキーマへ直接マッピングする。
- スクリーンショット画像のみ、S3 ObjectCreatedを契機にBedrock Nova Lite等で日本語キャプションを生成する。
- ss-tool向けPresigned URL取得はIoT Core Request/Responseを使う。
- Chrome拡張向けPresigned URL取得はLambda Function URL + Cognito IDプール由来IAM認証を使う。
- `daily-log` は初期版ではDynamoDB正規化済みアクティビティを主入力とする。
- `digital-twin` は `data-accumulation` が構築するS3 Vectors / Bedrock Knowledge Base系RAG I/Fに依存する。

Constructionでは `data-accumulation` を先に具体化し、`daily-log` と `digital-twin` が参照する読み取りI/Fを順に確定する。

---

## 2. 最優先で行うこと

以下の課題の詳細質問は `aidlc-docs/construction-design-qa-backlog.md` に分離して管理する。

| 優先度 | Constructionで行うこと | 目的 | 対応 |
|---|---|---|---|
| 高 | DynamoDB共通スキーマとロガー送信スキーマの固定 | logger実装とdata-accumulation実装の接続契約を固定する | `data-accumulation` Functional Design冒頭でJSON Schema相当の契約を確定し、logger実装が参照する |
| 高 | ss-tool Presigned URL Request/Response仕様の固定 | 画像アップロード経路を実装可能にする | request/response topic、correlation_id、エラー形式、URL有効期限、S3 key生成責務を確定 |
| 高 | `user_id` 信頼境界の統一 | ユーザー別データ分離を設計へ反映する | X.509証明書属性、Cognito identity、ペイロード内user_idの照合ルールを明文化 |
| 高 | RAG用データ同期方式の決定 | digital-twinの検索I/Fを確定する | DynamoDB Streams、EventBridgeバッチ、または専用同期Lambdaのどれを採用するか決める |

---

## 3. サブシステム別に行うこと

### 3.1 logger/osapi

**現状**: Functional Design完了、Code Generation待ち。

**Constructionで行うこと**:
- `event_type` と `data` フィールドを `data-accumulation` の最新スキーマに固定する必要がある。
- オフラインキュー再送時の重複送信とDynamoDB冪等性の関係が未確定。

**Constructionで確認すること**:
- `window_changed`, `audio_session_changed`, `periodic_snapshot` のサンプルペイロードをdata-accumulation側と一致させる。
- timestampとevent_id生成責務を確認する。

### 3.2 logger/ss-tool

**現状**: Application Design完了、Functional Design待ち。

**Constructionで行うこと**:
- Presigned URL取得方式はIoT Core Request/Responseに更新されたため、ss-tool側のS3Uploader設計へ具体的に反映する必要がある。
- S3アップロード成功後、MQTTメタデータ送信失敗時の再送・整合性が未確定。
- ローカル容量上限到達時の削除ポリシーと監査ログ粒度を詰める必要がある。

**Constructionで確認すること**:
- `correlation_id`、タイムアウト、リトライ、エラーレスポンス形式。
- S3 keyをクライアントが指定するのか、Presigned URL Lambdaが採番するのか。
- 画像アップロードとメタデータ送信の順序保証。

### 3.3 logger/chrome-extension

**現状**: Inception完了、Code Generation待ち。

**Constructionで行うこと**:
- `page_title` ではなく `title` を使うなど、親インターフェース更新後のフィールド名へ合わせる必要がある。
- `html_snippet` のON/OFF、最大サイズ、切り詰め方式、除外ドメイン時の挙動がdata-accumulation側と連動する。
- Cognito ID Pool、MQTT over WebSockets、Options Page設定値の検証方法を決める必要がある。

**Constructionで確認すること**:
- 128KB上限を超えない送信制御。
- IndexedDB再送時の順序と重複対策。

### 3.4 data-accumulation

**現状**: Inception完了、Functional Design待ち。

**Constructionで行うこと**:
- HTMLスニペット保存戦略が未確定。DynamoDB 400KB制限を超える場合のS3退避、切り詰め、保存しない判断が必要。
- DynamoDBからS3 Vectors / Bedrock Knowledge Baseへの同期方式が未確定。
- Kinesisを初期から使うか、IoT Core → Lambda直接にするかのコスト・運用判断が残る。
- Security Baselineは無効だが、実質的に認証・認可・マルチテナント・S3/IAMを扱うため、セキュリティ設計レビューが必要。

**Constructionで確認すること**:
- DynamoDB key設計、GSI設計、日付範囲クエリ方式。
- `caption_ja` 更新時の既存レコード検索方法。`s3_object_key` のGSI有無または逆引き設計。
- Presigned URL Lambdaの入力検証、content_type、file_size、prefix制限。
- DLQ、リトライ、部分失敗時の再処理方式。

### 3.5 daily-log

**現状**: Inception完了、Functional Design待ち。

**Constructionで行うこと**:
- Notion Databaseの具体プロパティが未確定。
- 対象日判定の最終ルールが未確定。
- 失敗ジョブの再実行方式が未確定。
- LLMモデル、プロンプト入力契約、ログへ残さない情報の境界が未確定。

**Constructionで確認すること**:
- DynamoDBの日時範囲クエリと `user_id` 条件。
- タイムゾーン変換のPBT対象プロパティ。
- 同一対象日に複数ページを作る方針で、再実行時の識別方法。

### 3.6 digital-twin

**現状**: Inception完了、Functional Design待ち。

**Constructionで行うこと**:
- RAG検索I/Fがdata-accumulation側の後続設計に依存している。
- 根拠メタデータ形式が未確定。
- デスクトップアプリのCognito認証フロー詳細が未確定。
- Conversation API、会話履歴DynamoDB、監査ログの具体スキーマが未確定。
- Inception文書のステータス表記は承認済みに同期済み。以降はRAG検索I/FなどConstruction設計事項に集中する。

**Constructionで確認すること**:
- Retrieval Adapterの入出力契約。
- `user_id` フィルタが必ず検索条件に入ることをPBT/単体テストで担保する方針。
- 10秒以内応答目標に対する計測点とタイムアウト設計。

---

## 4. 推奨Construction順序

1. `data-accumulation` Functional Design
   - 共通スキーマ、Presigned URL、DynamoDB/S3/RAG同期契約を固定する。
2. `logger/ss-tool` Functional Design
   - Presigned URL Request/Responseとアップロード・メタデータ送信整合性を固定する。
3. `logger/osapi` / `logger/chrome-extension` Code Generation
   - 固定済みスキーマに合わせて実装する。
4. `daily-log` Functional Design
   - DynamoDB読取I/Fを前提に日報生成ロジックを設計する。
5. `digital-twin` Functional Design
   - data-accumulationのRAG検索I/Fを前提に対話・根拠表示を設計する。

---

## 5. Construction初期段階の完了条件

Construction初期段階で、最低限以下を満たすこと。

- 最新の親インターフェース文書を参照して、各ロガーのサンプルペイロードが一致している。
- data-accumulationのDynamoDBスキーマ、S3 key、Presigned URL、RAG同期方式がFunctional Designで確定している。
- `user_id` の信頼境界と照合ルールが全サブシステムで一致している。
- daily-logとdigital-twinが依存する読み取りI/Fが明文化されている。
- 子AI-DLCの `aidlc-state.md` が実態と一致している。
