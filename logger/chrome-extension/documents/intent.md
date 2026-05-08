# 子AI-DLC向け 開発指示書: Chrome拡張 ロガーツール

## 1. プロジェクトの目的と全体コンテキスト (Context)
* **概要**: 個人活動ログ自動集計・活用システム「ShadowSync」のクライアント側エージェントの一つ。
* **役割**: Chromeブラウザ上でのユーザーの閲覧履歴やWebページのコンテキスト（HTML等）を取得し、クラウド（AWS）へ送信する。

## 2. 機能要件 (Functional Requirements)
* **閲覧ログ取得**: ユーザーが閲覧しているページのURL、タイトル、および必要に応じてHTMLスニペット等を取得すること。
* **データ送信**: 取得した情報をJSON形式にまとめ、AWS IoT CoreへMQTT通信で送信すること。
* **オフライン対応**: ブラウザがオフラインの場合、拡張機能のローカルストレージ（IndexedDB等）にキューイングし、オンライン復帰時に再送する仕組みを実装すること。
* **マルチテナント対応**: 拡張機能の設定等で対象の `user_id` を保持できるようにし、クラウドへ送信するデータ（ペイロードやMQTTトピック）には必ず対象の `user_id` を付与して送信すること。

## 3. 非機能要件と技術的制約 (Non-Functional & Constraints)
* **実行環境**: Google Chrome ブラウザ。
* **技術制約**: Manifest V3の仕様に準拠すること（Service Workerでのバックグラウンド処理）。
* **運用要件**: ブラウザのパフォーマンス（メモリ・動作速度）に悪影響を与えないこと。

## 4. 外部インターフェースと境界 (Interfaces & Boundaries)
* **通信プロトコル**: AWS IoT Core (MQTT over WebSockets)
* **認証方式**: セキュリティリスク回避のため、Amazon Cognito IDプールを用いて一時的な認証情報を取得して接続すること（X.509証明書の同梱は禁止）。
* **スキーマ設計**: 詳細なJSONスキーマの設計は本AI-DLC自身で決定すること。
* **参照資料**: `../../aidlc-docs/inception/application-design/data-accumulation-interface.md`

## 5. スコープ外 (Out of Scope)
* HTMLからの意味抽出や高度なテキスト解析は行わない（すべてAWSバックエンド側で行う）。
