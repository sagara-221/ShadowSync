# Component Methods — ss-tool (Screenshot Logger)

> **Note**: ビジネスルールの詳細は Functional Design (CONSTRUCTION) で定義する。本ドキュメントはメソッドシグネチャと高レベルの入出力のみ。

---

## 1. ScreenCapture

| Method | Input | Output | Purpose |
|---|---|---|---|
| `capture_active_window()` | — | `CaptureResult(image_bytes, metadata)` | アクティブウィンドウをキャプチャしWebPエンコード |
| `get_window_info()` | — | `WindowInfo(title, process_name, hwnd, rect)` | 現在のアクティブウィンドウ情報を取得 |

**型定義**:
- `CaptureResult`: `image_bytes: bytes`, `metadata: CaptureMetadata`
- `CaptureMetadata`: `timestamp: str`, `window_title: str`, `process_name: str`, `resolution: str`, `event_type: str`

## 2. WindowMonitor

| Method | Input | Output | Purpose |
|---|---|---|---|
| `start()` | — | — | WinEventHook監視を開始 |
| `stop()` | — | — | WinEventHook監視を停止 |
| `_on_foreground_change(hwnd)` | `hwnd: int` | — | フォアグラウンドウィンドウ変更コールバック（デバウンス内蔵） |

## 3. PeriodicScheduler

| Method | Input | Output | Purpose |
|---|---|---|---|
| `start(interval_seconds)` | `interval_seconds: int` | — | 定期タイマーを開始 |
| `stop()` | — | — | 定期タイマーを停止 |
| `_on_tick()` | — | — | タイマー発火時のコールバック |

## 4. EventBus

| Method | Input | Output | Purpose |
|---|---|---|---|
| `publish(event)` | `event: Event` | — | イベントをキューに追加 |
| `subscribe(event_type, handler)` | `event_type: str`, `handler: Callable` | — | イベントハンドラを登録 |
| `start()` | — | — | イベントディスパッチループを開始 |
| `stop()` | — | — | イベントディスパッチループを停止 |

**イベント型**: `CaptureRequested`, `CaptureCompleted`, `UploadCompleted`, `MetadataSent`, `TaskFailed`

## 5. QueueManager

| Method | Input | Output | Purpose |
|---|---|---|---|
| `enqueue(task)` | `task: UploadTask` | `task_id: str` | 新規タスクをキューに追加 |
| `dequeue()` | — | `Optional[UploadTask]` | 最古のpendingタスクを取得 |
| `mark_status(task_id, status)` | `task_id: str`, `status: TaskStatus` | — | タスクステータスを更新 |
| `get_pending_count()` | — | `int` | 未送信タスク数を返す |
| `get_total_size()` | — | `int` | 未送信画像の合計サイズ(bytes)を返す |
| `delete_completed()` | — | `int` | 送信完了タスクを削除し、削除件数を返す |
| `delete_oldest(count)` | `count: int` | `List[str]` | 最古のN件を削除し、削除された画像パスリストを返す |

**型定義**:
- `UploadTask`: `task_id: str`, `image_path: str`, `metadata: CaptureMetadata`, `status: TaskStatus`, `created_at: str`, `attempts: int`
- `TaskStatus`: `PENDING`, `UPLOADING`, `SENT`, `FAILED`

## 6. S3Uploader

| Method | Input | Output | Purpose |
|---|---|---|---|
| `request_presigned_url(s3_key)` | `s3_key: str` | `str` (presigned URL) | バックエンドAPIからPresigned URLを取得 |
| `upload(image_bytes, presigned_url)` | `image_bytes: bytes`, `presigned_url: str` | `bool` | Presigned URLへHTTPS PUTで画像アップロード |
| `generate_s3_key(metadata)` | `metadata: CaptureMetadata` | `str` | S3オブジェクトキーを生成 |

## 7. MqttPublisher

| Method | Input | Output | Purpose |
|---|---|---|---|
| `connect()` | — | — | IoT Core へ MQTTS 接続 |
| `disconnect()` | — | — | MQTTS 接続を切断 |
| `publish_metadata(topic, payload)` | `topic: str`, `payload: dict` | `bool` | メタデータJSONをパブリッシュ |
| `build_payload(metadata, s3_key)` | `metadata: CaptureMetadata`, `s3_key: str` | `dict` | 共通スキーマ準拠のJSONペイロードを構築 |
| `is_connected()` | — | `bool` | 接続状態を返す |

## 8. StorageManager

| Method | Input | Output | Purpose |
|---|---|---|---|
| `get_usage()` | — | `StorageUsage(used_bytes, limit_bytes, percent)` | 現在の使用量を返す |
| `is_over_limit()` | — | `bool` | 容量上限超過を判定 |
| `evict_oldest(target_bytes)` | `target_bytes: int` | `int` | 目標バイト数分の最古画像を削除し、削除件数を返す |
| `cleanup_sent(image_path)` | `image_path: str` | — | 送信完了済み画像を削除 |

## 9. ConfigManager

| Method | Input | Output | Purpose |
|---|---|---|---|
| `load(config_path)` | `config_path: str` | `AppConfig` | 設定ファイルを読み込み |
| `get(key)` | `key: str` | `Any` | 設定値を取得 |

**型定義**:
- `AppConfig`: `user_id: str`, `device_id: str`, `capture_interval_sec: int`, `storage_limit_bytes: int`, `iot_endpoint: str`, `cert_path: str`, `key_path: str`, `ca_path: str`, `s3_bucket: str`, `presigned_url_api: str`, `image_store_dir: str`, `db_path: str`, `log_dir: str`

## 10. ServiceHost

| Method | Input | Output | Purpose |
|---|---|---|---|
| `SvcDoRun()` | — | — | サービスメインエントリ |
| `SvcStop()` | — | — | サービス停止ハンドラ |
| `_start_components()` | — | — | 全コンポーネントを初期化・起動 |
| `_stop_components()` | — | — | 全コンポーネントを安全にシャットダウン |

## 11. Logger

| Method | Input | Output | Purpose |
|---|---|---|---|
| `setup(log_dir, level)` | `log_dir: str`, `level: str` | `logging.Logger` | ロガーを初期化（ファイルローテーション設定） |
