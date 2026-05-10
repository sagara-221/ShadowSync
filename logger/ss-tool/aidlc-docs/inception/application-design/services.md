# Services — ss-tool (Screenshot Logger)

## サービスレイヤー概要

本ツールはイベント駆動アーキテクチャを採用し、以下のサービスが内部イベントバスを介して非同期に連携する。

---

## 1. CaptureService（撮影オーケストレーション）

**責務**: 撮影トリガーの受信とScreenCaptureの実行、結果のキューイング

**オーケストレーションフロー**:
1. `CaptureRequested` イベントを受信（PeriodicScheduler または WindowMonitor から発行）
2. ScreenCapture.capture_active_window() を呼び出し
3. 画像をローカルストレージに一時保存
4. QueueManager.enqueue() でタスクを登録
5. `CaptureCompleted` イベントを発行

**購読イベント**: `CaptureRequested`
**発行イベント**: `CaptureCompleted`

---

## 2. UploadService（送信オーケストレーション）

**責務**: キューからタスクを取り出し、S3アップロードとMQTT送信を順次実行

**オーケストレーションフロー**:
1. `CaptureCompleted` イベントまたは定期的なキュー消化ループで起動
2. QueueManager.dequeue() で最古のpendingタスクを取得
3. S3Uploader.generate_s3_key() でS3キーを生成
4. S3Uploader.request_presigned_url() でPresigned URLを取得
5. S3Uploader.upload() で画像をアップロード
6. MqttPublisher.publish_metadata() でメタデータを送信
7. QueueManager.mark_status(task_id, SENT) でステータス更新
8. StorageManager.cleanup_sent() で送信済み画像を削除
9. `UploadCompleted` イベントを発行

**エラーハンドリング**:
- アップロード失敗時: QueueManager.mark_status(task_id, FAILED) → リトライキューに戻す
- ネットワーク障害時: バックオフ待機後にリトライ

**購読イベント**: `CaptureCompleted`
**発行イベント**: `UploadCompleted`, `TaskFailed`

---

## 3. StorageService（ストレージ監視）

**責務**: ストレージ使用量の定期監視と容量制御

**オーケストレーションフロー**:
1. 定期的（例: キャプチャ完了時）にStorageManager.is_over_limit() を確認
2. 容量超過時: QueueManager から最古タスクのパスを取得し、StorageManager.evict_oldest() で削除
3. QueueManager.delete_oldest() でキューレコードも同期削除

---

## 4. MainOrchestrator（全体制御）

**責務**: 全サービスおよびコンポーネントの初期化・起動・停止の統括

**初期化フロー**:
1. ConfigManager.load() で設定読み込み
2. Logger.setup() でログ設定
3. QueueManager を初期化（SQLite接続）
4. EventBus を作成
5. WindowMonitor, PeriodicScheduler を作成・EventBusに接続
6. CaptureService, UploadService, StorageService を作成
7. MqttPublisher.connect() でIoT Core接続
8. 各コンポーネント・サービスの start() を呼び出し

**停止フロー**:
1. WindowMonitor.stop(), PeriodicScheduler.stop()
2. EventBus.stop()（残存イベントのドレイン）
3. MqttPublisher.disconnect()
4. QueueManager のクローズ（SQLite接続解放）
