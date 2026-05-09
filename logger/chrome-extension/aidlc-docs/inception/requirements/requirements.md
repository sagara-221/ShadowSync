# ShadowSync Chrome Extension Logger - 要求仕様 (Requirements)

## 1. Intent Analysis Summary
- **User Request**: ShadowSync用のChrome拡張ロガーツールの作成（intent.md に基づく）
- **Request Type**: New Project (Greenfield)
- **Scope Estimate**: Single Component (Chrome Extension)
- **Complexity Estimate**: Simple to Moderate

## 2. 機能要件 (Functional Requirements)
1. **閲覧ログ取得トリガー**
   - ユーザーがブラウザでページ遷移を行った際（URLが変更されたタイミング）に、URLおよびページタイトルを取得する。
2. **HTMLスニペット取得**
   - オプションとしてHTMLスニペットの取得を可能とする。
   - パフォーマンスや通信量を考慮し、設定画面から取得のON/OFFを切り替えられるようにする。
3. **データ送信 (AWS IoT Core)**
   - 取得した情報を共通のJSONスキーマ（`data-accumulation-interface.md` に準拠）にまとめ、AWS IoT CoreへMQTT over WebSockets通信で送信する。
   - メッセージのサイズ上限（128KB）に留意する。
4. **オフライン対応**
   - ブラウザがオフラインで送信できない場合は、IndexedDB等のローカルストレージにログデータをキューイングする。
   - オンライン復帰時にキューを再送する仕組みを実装する。
5. **マルチテナント・認証情報設定 (Options Page)**
   - 拡張機能の設定画面（Options）を提供し、ユーザーが手動で以下の項目を設定・保存できるようにする。
     - `user_id`（テナント識別用）
     - `device_id`（デバイス識別用）
     - AWS Cognito設定値（Identity Pool ID、リージョン等）
     - プライバシー除外ドメイン設定（後述）
     - HTMLスニペット取得のON/OFF
6. **プライバシーと除外ドメイン設定**
   - ユーザーが設定画面で「ログ取得を除外するドメイン（例: 銀行サイト等）」のリストを管理できるようにし、該当ドメインでは情報の取得・送信を行わない。

## 3. 非機能要件 (Non-Functional Requirements)
1. **実行環境と制約**
   - Google Chrome ブラウザで動作すること。
   - Manifest V3の仕様に準拠すること（Service Workerを利用したバックグラウンド処理）。
2. **パフォーマンスへの影響最小化**
   - ブラウザのメモリや動作速度に悪影響を与えないよう、軽量な処理にとどめること。
3. **通信と認証方式**
   - 通信プロトコル: MQTT over WebSockets (ポート443)
   - 認証方式: Amazon Cognito IDプールを用いて一時クレデンシャルを取得して接続する（X.509デバイス証明書は同梱しない）。
4. **スコープ外 (Out of Scope)**
   - HTMLからの意味抽出や高度なテキスト解析は拡張機能側では行わない（すべてバックエンド側で行う）。

## 4. Extension Configuration
- **Security Baseline**: スキップ (Not Enforced)
- **Property-Based Testing**: 一部有効 (Partial - for pure functions and serialization round-trips)
