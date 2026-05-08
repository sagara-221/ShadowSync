# 開発インテント (Chrome拡張 ロガーツール)

**目的**: Chromeブラウザ上での閲覧履歴やDOM（HTML等）の情報を取得し、AWSバックエンド（IoT Core/MQTT想定）へ送信するChrome拡張機能の開発。

**要件**:
- 独立した拡張機能として動作すること。
- **認証と通信**: 拡張機能内にデバイス証明書を保持するセキュリティリスクを回避するため、「Cognito IDプールによる一時認証 + MQTT over WebSockets」の構成でIoT Coreへデータ送信を行うこと。
- データの取得頻度やトリガー条件については本AI-DLCにて最適設計を行うこと。
- インターフェースの詳細は、親AI-DLCで定義した `../../aidlc-docs/inception/application-design/data-accumulation-interface.md` を必ず参照・準拠すること。
