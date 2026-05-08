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
*   **責任**: IoT Core経由(MQTT想定)で受信したデータの処理・蓄積。Bedrockを用いた非構造化データ（画像・HTML等）からの情報抽出と共通スキーマ変換、およびDynamoDB等への保存。

### 3. 日誌作成システム (Daily Log System)
*   **パス**: `daily-log/`
*   **責任**: 蓄積されたデータソースを用いた日報の自動生成とNotionへのエクスポート。EventBridgeとLambdaによる定期実行。

### 4. デジタルツインシステム (Digital Twin System)
*   **パス**: `digital-twin/`
*   **責任**: ユーザーの過去の行動や傾向に対するチャットインターフェースの提供。RAGを活用したAI回答システムの構築。
