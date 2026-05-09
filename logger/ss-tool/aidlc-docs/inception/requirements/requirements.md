# Requirements Document — ss-tool (Screenshot Logger)

## Intent Analysis Summary
- **User Request**: 個人活動ログ自動集計・活用システム「ShadowSync」のスクリーンショット取得ロガーツールの開発
- **Request Type**: New Project (子AI-DLC, Greenfield)
- **Scope Estimate**: Single Component (SSロガーツール)
- **Complexity Estimate**: Moderate (AWS IoT Core/S3連携、オフライン対応、Windowsサービス)

## 1. Functional Requirements

### FR-01: スクリーンショット撮影
- **定期撮影**: 5分間隔で定期的にアクティブウィンドウのスクリーンショットを撮影する
- **トリガー撮影**: アクティブウィンドウが切り替わったタイミングでも撮影を実行する
- **撮影対象**: フルスクリーンではなく、**アクティブウィンドウの内容のみ**をキャプチャする
- **画像形式**: WebP形式で保存する（高圧縮率・モダンフォーマット）

### FR-02: データ送信 — 画像本体 (S3)
- 撮影した画像本体はAmazon S3へ直接アップロードする
- S3オブジェクトキーの規則: `raw/screenshots/{user_id}/{device_id}/YYYY/MM/DD/HH-mm-ss.webp`
- アップロード認証方式はこの子AI-DLCのApplication Design段階で設計する（IoT Core Credential Provider or Presigned URL）

### FR-03: データ送信 — メタデータ (MQTT)
- S3のパスや撮影時のメタデータをJSONにまとめ、AWS IoT Core へ MQTTS で送信する
- MQTTトピック: `shadowsync/logs/{user_id}/{device_id}/screenshot`
- 認証方式: X.509デバイス証明書によるMQTTS通信
- ペイロードは親AI-DLCの共通スキーマに準拠:
  ```json
  {
    "user_id": "usr_123456",
    "device_id": "PC-001",
    "timestamp": "2026-05-08T23:30:00Z",
    "logger_type": "screenshot",
    "event_type": "periodic_ss | window_changed",
    "data": {
      "s3_object_key": "raw/screenshots/usr_123456/PC-001/2026/05/08/23-30-00.webp",
      "screen_index": 0,
      "resolution": "1920x1080",
      "active_window_title": "Visual Studio Code",
      "process_name": "Code.exe"
    }
  }
  ```

### FR-04: オフライン対応
- ネットワーク障害やAWSサービス障害時、画像とメタデータをローカルディスクに一時保存する
- ネットワーク復旧時に、未送信データを順次アップロード・送信する（キュー方式）
- 送信順序は撮影時刻順（FIFO）を保証する

### FR-05: マルチテナント対応
- `user_id` と `device_id` をローカル設定ファイル（JSON/YAML）で管理する
- クラウドへ送信するすべてのデータ（S3パス、MQTTペイロード）に `user_id` を必ず付与する

### FR-06: ストレージ管理
- 送信完了後のローカル画像は自動削除する
- オフライン一時保存の容量上限をユーザーが設定ファイルで指定できるようにする
- 容量上限到達時は最も古い未送信画像から削除するか、新規撮影を一時停止する（詳細は設計段階で決定）

## 2. Non-Functional Requirements

### NFR-01: 実行環境
- **OS**: Windows (ネイティブ)
- **言語/ランタイム**: Python
- **実行形態**: Windowsサービスとしてバックグラウンド常駐（pywin32/win32serviceutil 利用）

### NFR-02: パフォーマンス
- 5分ごとの定期撮影 + ウィンドウ切替時のイベント駆動撮影を並行処理
- ウィンドウ切替の短時間連続発生時は、デバウンス処理（例: 1〜2秒）を行いキャプチャの重複を防ぐ
- アクティブウィンドウのキャプチャは高速に完了し、ユーザーの操作を阻害しないこと

### NFR-03: リソース管理
- ローカルディスクの容量を圧迫しないよう、送信完了後に画像を速やかに削除する
- メモリ使用量を最小限に保つ（大量の画像をメモリに保持しない）
- CPU使用率を低く保ち、バックグラウンドプロセスとしてユーザー体験を損なわないこと

### NFR-04: 信頼性
- サービスの異常終了時に自動再起動を試みる
- 未送信キューの永続化（プロセス再起動後もキューを復元できること）

## 3. Technical Context

### 外部インターフェース
- **AWS IoT Core**: MQTTS (X.509証明書認証)
- **Amazon S3**: HTTPS (認証方式は設計段階で決定)
- **参照資料**: `../../aidlc-docs/inception/application-design/data-accumulation-interface.md`

### 設定ファイル
- `user_id`, `device_id` の保持
- 撮影間隔（デフォルト: 5分）
- ローカルストレージ容量上限
- AWS IoT Core エンドポイント
- 証明書パス
- S3バケット名

## 4. Scope Exclusions
- 画像に対するOCR（文字認識）や画像解析・タグ付けは一切行わない（バックエンド側の責務）
- GUI/ダッシュボードの提供は行わない（Windowsサービスとして動作）
- マルチモニターのフルスクリーン撮影は行わない（アクティブウィンドウのみ）

## 5. Extension Configuration
| Extension | Enabled | Mode | Decided At |
|---|---|---|---|
| Security Baseline | No | — | Requirements Analysis |
| Property-Based Testing | Yes | Partial (純粋関数 + シリアライゼーション往復のみ) | Requirements Analysis |
