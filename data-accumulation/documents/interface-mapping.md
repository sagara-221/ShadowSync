# インターフェースマッピング定義

**作成日**: 2026-05-10  
**バージョン**: 1.0  
**ステータス**: 承認済み  

---

## 概要

親AI-DLCが定義したMQTTインターフェース（`data-accumulation-interface.md`）と、data-accumulationサブシステムの内部DynamoDBスキーマの間のマッピングを定義します。

Lambda関数がインターフェース変換レイヤーとして機能し、外部インターフェースの安定性と内部スキーマの柔軟性を両立します。

---

## 設計原則

1. **外部インターフェースの安定性**: 親AI-DLC定義に完全準拠
2. **内部スキーマの柔軟性**: 必要に応じて明示的な命名を使用
3. **Lambda変換レイヤー**: 外部と内部の差異を吸収

---

## トピックマッピング

### 親AI-DLC定義（MQTT）

```
shadowsync/logs/{user_id}/{device_id}/chrome       ← Chrome拡張
shadowsync/logs/{user_id}/{device_id}/osapi        ← OS APIロガー
shadowsync/logs/{user_id}/{device_id}/screenshot   ← スクリーンショットロガー
```

### data-accumulation実装（IoT Rule）

```sql
SELECT *, topic(5) as topic_suffix
FROM 'shadowsync/logs/+/+/+'
WHERE topic(5) IN ('chrome', 'osapi', 'screenshot')
```

**マッピング**:
- トピック: 親定義と完全一致
- ルーティング: `topic_suffix`で識別

---

## logger_typeマッピング

| 親AI-DLC定義（MQTT） | data-accumulation実装（DynamoDB） | 変換 |
|---------------------|--------------------------------|------|
| `"chrome"` | `"chrome"` | そのまま |
| `"osapi"` | `"osapi"` | そのまま |
| `"screenshot"` | `"screenshot"` | そのまま |

**変換**: 不要（完全一致）

---

## ペイロードフィールドマッピング

### MQTTペイロード（親AI-DLC定義）

```json
{
  "user_id": "user-123",
  "device_id": "device-123",
  "timestamp": "2026-05-10T12:00:00Z",
  "logger_type": "chrome",
  "event_type": "page_navigation",
  "data": {
    "url": "https://github.com/...",
    "title": "GitHub"
  }
}
```

### DynamoDBスキーマ（data-accumulation実装）

```json
{
  "user_id": "user-123",
  "timestamp_event_id": "2026-05-10T12:00:00Z#abc-123",
  "timestamp": "2026-05-10T12:00:00Z",
  "device_id": "device-123",
  "logger_type": "chrome",
  "event_type": "page_navigation",
  "activity_data": {
    "url": "https://github.com/...",
    "title": "GitHub"
  }
}
```

### フィールドマッピング表

| MQTTフィールド | DynamoDBフィールド | 変換 | 備考 |
|--------------|------------------|------|------|
| `user_id` | `user_id` | そのまま | パーティションキー |
| `timestamp` | `timestamp` | そのまま | タイムスタンプ |
| - | `timestamp_event_id` | 生成 | ソートキー（`timestamp#uuid`） |
| `device_id` | `device_id` | そのまま | デバイス識別子 |
| `logger_type` | `logger_type` | そのまま | ロガータイプ |
| `event_type` | `event_type` | そのまま | イベントタイプ |
| `data` | `activity_data` | **フィールド名変換** | イベント固有データ |

---

## Lambda変換ロジック

### 実装例（Python）

```python
import json
import uuid
from typing import Dict, Any

def transform_mqtt_to_dynamodb(mqtt_payload: Dict[str, Any]) -> Dict[str, Any]:
    """
    MQTTペイロード（親AI-DLC定義）をDynamoDBスキーマに変換
    
    Args:
        mqtt_payload: IoT Coreから受信したMQTTペイロード
        
    Returns:
        DynamoDBに保存するアイテム
    """
    # timestamp_event_id生成（ソートキー）
    timestamp = mqtt_payload["timestamp"]
    event_id = str(uuid.uuid4())
    timestamp_event_id = f"{timestamp}#{event_id}"
    
    # DynamoDBスキーマに変換
    dynamodb_item = {
        "user_id": mqtt_payload["user_id"],
        "timestamp_event_id": timestamp_event_id,
        "timestamp": timestamp,
        "device_id": mqtt_payload["device_id"],
        "logger_type": mqtt_payload["logger_type"],
        "event_type": mqtt_payload["event_type"],
        "activity_data": mqtt_payload["data"]  # data → activity_data 変換
    }
    
    return dynamodb_item


def transform_dynamodb_to_mqtt(dynamodb_item: Dict[str, Any]) -> Dict[str, Any]:
    """
    DynamoDBアイテムをMQTTペイロード形式に変換（逆変換）
    
    Args:
        dynamodb_item: DynamoDBから取得したアイテム
        
    Returns:
        MQTTペイロード形式
    """
    mqtt_payload = {
        "user_id": dynamodb_item["user_id"],
        "device_id": dynamodb_item["device_id"],
        "timestamp": dynamodb_item["timestamp"],
        "logger_type": dynamodb_item["logger_type"],
        "event_type": dynamodb_item["event_type"],
        "data": dynamodb_item["activity_data"]  # activity_data → data 変換
    }
    
    return mqtt_payload
```

### Lambda Structured実装例

```python
import boto3
import json
from typing import Dict, Any

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['DYNAMODB_TABLE_NAME'])

def handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    """
    構造化データ処理Lambda
    
    Kinesisから受信したMQTTペイロードをDynamoDBに保存
    """
    for record in event['Records']:
        # Kinesisレコードからペイロード取得
        payload = json.loads(base64.b64decode(record['kinesis']['data']))
        
        # MQTTペイロードをDynamoDBスキーマに変換
        dynamodb_item = transform_mqtt_to_dynamodb(payload)
        
        # DynamoDBに保存
        table.put_item(Item=dynamodb_item)
        
        print(f"Saved item: {dynamodb_item['timestamp_event_id']}")
    
    return {'statusCode': 200, 'message': 'Success'}
```

---

## event_typeマッピング

### Chrome拡張（logger_type: "chrome"）

| MQTTペイロード | DynamoDBスキーマ | 変換 |
|--------------|----------------|------|
| `event_type: "page_navigation"` | `event_type: "page_navigation"` | そのまま |

### OS APIロガー（logger_type: "osapi"）

| MQTTペイロード | DynamoDBスキーマ | 変換 |
|--------------|----------------|------|
| `event_type: "window_changed"` | `event_type: "window_changed"` | そのまま |
| `event_type: "audio_session_changed"` | `event_type: "audio_session_changed"` | そのまま |
| `event_type: "periodic_snapshot"` | `event_type: "periodic_snapshot"` | そのまま |

### スクリーンショットロガー（logger_type: "screenshot"）

| MQTTペイロード | DynamoDBスキーマ | 変換 |
|--------------|----------------|------|
| `event_type: "periodic_ss"` | `event_type: "periodic_ss"` | そのまま |
| `event_type: "window_changed"` | `event_type: "window_changed"` | そのまま |

---

## activity_data構造マッピング

### Chrome拡張（logger_type: "chrome"）

**MQTTペイロード（`data`）**:
```json
{
  "url": "https://github.com/user/repo",
  "title": "GitHub - user/repo",
  "html_snippet": "<html>...</html>"
}
```

**DynamoDBスキーマ（`activity_data`）**:
```json
{
  "url": "https://github.com/user/repo",
  "title": "GitHub - user/repo",
  "html_snippet": "<html>...</html>"
}
```

**変換**: フィールド名のみ変換（`data` → `activity_data`）、内容はそのまま

### OS APIロガー（logger_type: "osapi"）

#### window_changed

**MQTTペイロード（`data`）**:
```json
{
  "window_title": "VS Code - project.py",
  "process_name": "Code.exe"
}
```

**DynamoDBスキーマ（`activity_data`）**:
```json
{
  "window_title": "VS Code - project.py",
  "process_name": "Code.exe"
}
```

#### audio_session_changed

**MQTTペイロード（`data`）**:
```json
{
  "audio_sessions": [
    {
      "session_id": "session-abc-123",
      "process_name": "Spotify.exe",
      "volume_level": 0.75
    }
  ]
}
```

**DynamoDBスキーマ（`activity_data`）**:
```json
{
  "audio_sessions": [
    {
      "session_id": "session-abc-123",
      "process_name": "Spotify.exe",
      "volume_level": 0.75
    }
  ]
}
```

#### periodic_snapshot

**MQTTペイロード（`data`）**:
```json
{
  "is_active": true,
  "window_title": "Chrome - GitHub",
  "process_name": "chrome.exe",
  "audio_sessions": []
}
```

**DynamoDBスキーマ（`activity_data`）**:
```json
{
  "is_active": true,
  "window_title": "Chrome - GitHub",
  "process_name": "chrome.exe",
  "audio_sessions": []
}
```

### スクリーンショットロガー（logger_type: "screenshot"）

**MQTTペイロード（`data`）**:
```json
{
  "s3_object_key": "raw/screenshots/user-123/device-123/2026/05/10/12-00-00.webp",
  "screen_index": 0,
  "resolution": "1920x1080",
  "active_window_title": "VS Code - project.py",
  "process_name": "Code.exe"
}
```

**DynamoDBスキーマ（`activity_data`）**:
```json
{
  "s3_object_key": "raw/screenshots/user-123/device-123/2026/05/10/12-00-00.webp",
  "screen_index": 0,
  "resolution": "1920x1080",
  "active_window_title": "VS Code - project.py",
  "process_name": "Code.exe",
  "caption_ja": "VS Codeでプログラミングをしている画面"  // Bedrock処理後に追加
}
```

**変換**: フィールド名のみ変換、`caption_ja`はBedrock処理後に追加

---

## テスト戦略

### ユニットテスト

```python
def test_transform_mqtt_to_dynamodb():
    """MQTTペイロード → DynamoDBスキーマ変換のテスト"""
    mqtt_payload = {
        "user_id": "user-123",
        "device_id": "device-123",
        "timestamp": "2026-05-10T12:00:00Z",
        "logger_type": "chrome",
        "event_type": "page_navigation",
        "data": {
            "url": "https://github.com/...",
            "title": "GitHub"
        }
    }
    
    result = transform_mqtt_to_dynamodb(mqtt_payload)
    
    assert result["user_id"] == "user-123"
    assert result["device_id"] == "device-123"
    assert result["timestamp"] == "2026-05-10T12:00:00Z"
    assert result["logger_type"] == "chrome"
    assert result["event_type"] == "page_navigation"
    assert result["activity_data"] == mqtt_payload["data"]
    assert "timestamp_event_id" in result
    assert "#" in result["timestamp_event_id"]


def test_transform_dynamodb_to_mqtt():
    """DynamoDBスキーマ → MQTTペイロード変換のテスト"""
    dynamodb_item = {
        "user_id": "user-123",
        "timestamp_event_id": "2026-05-10T12:00:00Z#abc-123",
        "timestamp": "2026-05-10T12:00:00Z",
        "device_id": "device-123",
        "logger_type": "chrome",
        "event_type": "page_navigation",
        "activity_data": {
            "url": "https://github.com/...",
            "title": "GitHub"
        }
    }
    
    result = transform_dynamodb_to_mqtt(dynamodb_item)
    
    assert result["user_id"] == "user-123"
    assert result["device_id"] == "device-123"
    assert result["timestamp"] == "2026-05-10T12:00:00Z"
    assert result["logger_type"] == "chrome"
    assert result["event_type"] == "page_navigation"
    assert result["data"] == dynamodb_item["activity_data"]
    assert "timestamp_event_id" not in result
```

---

## まとめ

### 変換が必要なフィールド

| フィールド | 変換内容 | 理由 |
|-----------|---------|------|
| `data` → `activity_data` | フィールド名変換 | DynamoDB内部で明示的な命名を使用 |
| - → `timestamp_event_id` | 生成 | DynamoDBソートキー |

### 変換が不要なフィールド

- `user_id`
- `device_id`
- `timestamp`
- `logger_type`
- `event_type`
- `data`/`activity_data`の内容

### 設計の利点

1. **外部インターフェースの安定性**: ロガーツールは親定義に従うだけ
2. **内部スキーマの柔軟性**: DynamoDB内部で明示的な命名を維持
3. **変換の軽量性**: フィールド名変換のみ、パフォーマンス影響なし
4. **テスト容易性**: 変換ロジックが単純でテストしやすい

---

**作成者**: data-accumulation AI-DLC  
**承認者**: 親AI-DLC  
**ステータス**: 承認済み  
**バージョン**: 1.0
