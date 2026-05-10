# Component Dependencies — ss-tool (Screenshot Logger)

## 依存関係マトリクス

| Component | Depends On | Depended By |
|---|---|---|
| ServiceHost | MainOrchestrator | — (エントリポイント) |
| MainOrchestrator | ConfigManager, Logger, EventBus, WindowMonitor, PeriodicScheduler, CaptureService, UploadService, StorageService, MqttPublisher, QueueManager | ServiceHost |
| EventBus | — | WindowMonitor, PeriodicScheduler, CaptureService, UploadService |
| ConfigManager | — | MainOrchestrator, 全コンポーネント(設定値参照) |
| Logger | — | 全コンポーネント(ログ出力) |
| WindowMonitor | EventBus | MainOrchestrator |
| PeriodicScheduler | EventBus | MainOrchestrator |
| ScreenCapture | — | CaptureService |
| CaptureService | ScreenCapture, QueueManager, StorageManager, EventBus | MainOrchestrator |
| S3Uploader | ConfigManager | UploadService |
| MqttPublisher | ConfigManager | UploadService |
| QueueManager | — | CaptureService, UploadService, StorageService |
| StorageManager | QueueManager | CaptureService, StorageService |
| UploadService | QueueManager, S3Uploader, MqttPublisher, StorageManager, EventBus | MainOrchestrator |
| StorageService | StorageManager, QueueManager | MainOrchestrator |

## 通信パターン

### イベント駆動通信（EventBus経由）

```
WindowMonitor ──[WindowChanged]──> EventBus ──[CaptureRequested]──> CaptureService
PeriodicScheduler ──[PeriodicTick]──> EventBus ──[CaptureRequested]──> CaptureService
CaptureService ──[CaptureCompleted]──> EventBus ──> UploadService
UploadService ──[UploadCompleted]──> EventBus ──> StorageService (cleanup)
```

### 直接呼び出し（同期）

```
CaptureService --> ScreenCapture.capture_active_window()
CaptureService --> QueueManager.enqueue()
UploadService --> S3Uploader.request_presigned_url() / upload()
UploadService --> MqttPublisher.publish_metadata()
UploadService --> QueueManager.mark_status()
StorageService --> StorageManager.is_over_limit() / evict_oldest()
```

## データフロー図

```
+------------------+     +-----------------+
| WindowMonitor    |     | PeriodicScheduler|
| (WinEventHook)   |     | (5min timer)    |
+--------+---------+     +--------+--------+
         |                        |
         | WindowChanged          | PeriodicTick
         v                        v
+--------+------------------------+--------+
|              EventBus                     |
|         (Python queue.Queue)              |
+--------+----------+----------+-----------+
         |          |          |
         v          |          |
+--------+---------+|          |
| CaptureService   ||          |
| +-ScreenCapture   |          |
| +-QueueManager    |          |
| +-StorageManager  |          |
+--------+---------+          |
         |                     |
         | CaptureCompleted    |
         v                     v
+--------+--------------------+--------+
|            UploadService              |
| +-S3Uploader (Presigned URL + PUT)   |
| +-MqttPublisher (MQTTS)             |
| +-QueueManager (status update)       |
| +-StorageManager (cleanup)           |
+--------------------------------------+
         |
         | UploadCompleted
         v
+--------+---------+
| StorageService   |
| (capacity check) |
+------------------+
```

## 外部依存

| External Service | Protocol | Auth | Used By |
|---|---|---|---|
| AWS IoT Core | MQTTS (8883) | X.509 証明書 | MqttPublisher |
| Presigned URL API | HTTPS | (API依存) | S3Uploader |
| Amazon S3 | HTTPS (PUT) | Presigned URL | S3Uploader |
