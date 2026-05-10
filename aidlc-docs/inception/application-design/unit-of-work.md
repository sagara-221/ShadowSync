# ユニット定義 (Unit of Work Definitions)

本システム「個人活動ログ自動集計・活用システム (ShadowSync)」は、以下のユニット（子AI-DLCワークスペース）で構成されます。

## コード構成戦略とAI-DLC境界 (Code Organization & AI-DLC Boundaries)

本プロジェクトは、システム全体を管理する「親AI-DLC」と、各コンポーネントを独立して開発・テストする「子（孫）AI-DLC」による再帰的な構成をとります。親AI-DLCはアーキテクチャ設計および各ユニットへのインテント（要件）の受け渡しのみを行い、アプリケーションのコード実装は一切行いません。複数の人間が並列して開発できるよう、各ディレクトリは独立した環境として扱われます。

```text
(Workspace Root)
├── aidlc-docs/                # 親AI-DLC用のアーキテクチャドキュメント
├── logger/                    # [子ユニット] ロガーシステム (親)
│   ├── osapi/                 # [孫ユニット] OS APIログ取得
│   ├── chrome-extension/      # [孫ユニット] Chrome拡張ログ取得
│   └── ss-tool/               # [孫ユニット] スクリーンショット取得
├── data-accumulation/         # [子ユニット] データ蓄積システム
├── daily-log/                 # [子ユニット] 日誌作成システム
└── digital-twin/              # [子ユニット] デジタルツインシステム
```

## ユニット詳細

### 1. ロガーシステム (Logger System)
*   **パス**: `logger/` (孫要素: `osapi/`, `chrome-extension/`, `ss-tool/`)
*   **責任**: ローカルPC上でのユーザーアクティビティ監視、情報の取得、およびバックエンド（AWS IoT Core）への独立したデータ送信。各孫ツールは独立してデータを送信する構成。

### 2. データ蓄積システム (Data Accumulation System)
*   **パス**: `data-accumulation/`
*   **責任**: IoT Core経由(MQTT/MQTTS/MQTT over WebSockets)で受信したデータの処理・蓄積。構造化データ（OS API、Chrome URL/タイトル、オーディオセッション等）はLambdaでDynamoDBスキーマへ直接マッピングし、スクリーンショット画像はS3 ObjectCreatedを契機にBedrock Nova Lite等で日本語キャプションを生成する。ss-tool向けPresigned URLはIoT Core Request/Response、Chrome拡張向けPresigned URLはLambda Function URL + Cognito IDプール由来IAM認証で発行する。複数ユーザーが利用可能なマルチテナント構成とし、ユーザーごとの論理的なデータ分離とセキュアなアクセス制御を実現した上で、DynamoDB、S3、S3 Vectors / Bedrock Knowledge Base 等へ保存・管理する。

### 3. 日誌作成システム (Daily Log System)
*   **パス**: `daily-log/`
*   **責任**: `data-accumulation` が管理するDynamoDBの正規化済みアクティビティを主入力とした日報の自動生成とNotionへのエクスポート。初期版は単一ユーザーを優先しつつ、`user_id`、ユーザー別タイムゾーン、Notion設定を保持して将来の複数ユーザー対応を阻害しない。EventBridgeとLambdaによる定期実行。

### 4. デジタルツインシステム (Digital Twin System)
*   **パス**: `digital-twin/`
*   **責任**: ユーザーの過去の行動や傾向に対するデスクトップ対話インターフェースの提供。Cognito認証、Conversation API、`data-accumulation` が構築するS3 Vectors / Bedrock Knowledge BaseベースのRAG検索、Bedrock Nova系モデルによる根拠付きAI回答、会話履歴保存を扱う。活動ログの収集、正規化、ベクトル生成は対象外。
