# データ連携インターフェース定義 (Data Accumulation Interface)

各ロガーツール（osapi, chrome-extension, ss-tool）とデータ蓄積システム間の、IoT Core等を経由する通信仕様を定義します。

## 1. 通信プロトコルと認証方式
* **基本通信プロトコル**: MQTT (AWS IoT Core)
* **各ツールの認証・通信方式**:
  * **OS API / SSツール (ローカルアプリ)**: セキュリティの観点から、**X.509証明書**（デバイス証明書）を用いたMQTT(MQTTS)通信を標準とします。
  * **Chrome拡張機能**: **Amazon Cognito IDプール** を用いた一時クレデンシャル取得と、**MQTT over WebSockets (ポート443)** を用いた通信とします。
    * **【採用理由】**: Chromeブラウザ拡張機能内にX.509デバイス証明書（秘密鍵）を静的に保持することは、ソースコードやパッケージが解析されやすく、セキュリティリスクが非常に高いため推奨されません。Cognitoを用いてセキュアに一時的な認証情報を取得し、WebSocketsで通信するアーキテクチャがベストプラクティスとなります。

## 2. MQTT トピック設計
メッセージの種類や送信元に応じてルーティングしやすいよう、以下のトピック構造とします。

* `shadowsync/logs/{user_id}/{device_id}/osapi`
* `shadowsync/logs/{user_id}/{device_id}/chrome`
* `shadowsync/logs/{user_id}/{device_id}/screenshot`

*(※ `{user_id}` はユーザーを一意に識別するID。`{device_id}` はそのユーザーの各PCを識別するID。将来的な複数人利用（マルチテナント）を見据えアクセスを分離するために付与)*

## 3. 共通ペイロード (JSONスキーマ)
Bedrockでの抽出やDynamoDBへの保存を容易にするため、すべてのロガーは以下の「共通ヘッダ」を含めたJSONで送信します。

```json
{
  "user_id": "usr_123456",
  "device_id": "PC-001",
  "timestamp": "2026-05-08T23:30:00Z",
  "logger_type": "osapi | chrome | screenshot",
  "event_type": "window_changed | page_visited | periodic_ss",
  "data": {
    // 各ロガー固有のデータ（以下参照）
  }
}
```

### 3.1 各ロガーの固有データ (`data` プロパティ内) の構成案

**A. OS API (osapi)**
```json
"data": {
  "active_window_title": "初期仕様.md - Visual Studio Code",
    "process_name": "Code.exe",
  "interaction_type": "keyboard_input" // key, mouse, none 等
}
```

**B. Chrome拡張機能 (chrome-extension)**
```json
"data": {
  "url": "https://github.com/...",
  "page_title": "Repository - GitHub",
  "html_snippet": "<html>...</html>", // オプション(長文の場合は上限に注意)
  "scroll_depth_percent": 45
}
```

**C. スクリーンショット (ss-tool) の大容量データ通信設計**
* AWS IoT Coreのメッセージペイロード上限は **128KB** です。SS画像を直接MQTTペイロードに含めると上限を超過し、通信が遮断されるリスクがあります。
* そのため、SSツールは **「画像をS3に直接アップロード（Presigned URL等を利用）し、IoT CoreへはそのS3パス情報のみをMQTTで送信する」** 構成とします。

```json
"data": {
  "s3_object_key": "raw/screenshots/usr_123456/PC-001/2026/05/08/23-30-00.png",
  "screen_index": 0,
  "resolution": "1920x1080"
}
```
