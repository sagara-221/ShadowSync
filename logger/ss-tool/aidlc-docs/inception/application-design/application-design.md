# Application Design — ss-tool (Screenshot Logger)

## 1. 設計概要

本ドキュメントはスクリーンショット取得ロガーツール (ss-tool) のアプリケーション設計を統合的にまとめたものである。

### 設計方針
- **アーキテクチャ**: イベント駆動アーキテクチャ（内部EventBusによる非同期パイプライン）
- **言語/ランタイム**: Python + pywin32
- **実行形態**: Windowsサービス (バックグラウンド常駐)
- **オフラインキュー**: SQLite永続化
- **S3認証**: Presigned URL方式（バックエンドAPI経由）
- **ウィンドウ検知**: WinEventHook (EVENT_SYSTEM_FOREGROUND)
- **容量制御**: 最古優先削除ポリシー
- **ログ**: Python標準logging（ファイルローテーション）

## 2. コンポーネント構成

### コンポーネント一覧（11コンポーネント）

| # | Component | 責務 | カテゴリ |
|---|---|---|---|
| 1 | ScreenCapture | アクティブウィンドウ撮影+WebP変換 | キャプチャ層 |
| 2 | WindowMonitor | WinEventHookによるウィンドウ切替検知 | イベントソース |
| 3 | PeriodicScheduler | 定期撮影タイマー（5分間隔） | イベントソース |
| 4 | EventBus | 内部イベント仲介（queue.Queue） | インフラ層 |
| 5 | QueueManager | SQLiteベースの永続キュー管理 | データ層 |
| 6 | S3Uploader | Presigned URL取得+S3 PUT | 通信層 |
| 7 | MqttPublisher | MQTTS経由メタデータ送信 | 通信層 |
| 8 | StorageManager | ローカルストレージ容量監視+最古削除 | データ層 |
| 9 | ConfigManager | 設定ファイル(JSON/YAML)読み込み | インフラ層 |
| 10 | ServiceHost | Windowsサービスライフサイクル管理 | インフラ層 |
| 11 | Logger | Python logging（ファイルローテーション） | インフラ層 |

### サービスレイヤー（4サービス）

| # | Service | 責務 |
|---|---|---|
| 1 | CaptureService | 撮影オーケストレーション (トリガー受信→キャプチャ→キュー登録) |
| 2 | UploadService | 送信オーケストレーション (キュー取得→S3→MQTT→クリーンアップ) |
| 3 | StorageService | ストレージ監視 (容量チェック→最古削除) |
| 4 | MainOrchestrator | 全体制御 (初期化→起動→停止) |

## 3. イベントフロー

```
[WindowMonitor] ──WindowChanged──> [EventBus] ──CaptureRequested──> [CaptureService]
[PeriodicScheduler] ──PeriodicTick──> [EventBus] ──CaptureRequested──> [CaptureService]

[CaptureService] ──CaptureCompleted──> [EventBus] ──> [UploadService]

[UploadService] ──UploadCompleted──> [EventBus] ──> [StorageService] (cleanup)
```

## 4. キー設計判断

| 判断事項 | 選択 | 根拠 |
|---|---|---|
| S3認証方式 | Presigned URL | クライアント側にIAMクレデンシャルを持たない。バックエンド側で権限制御が可能。 |
| アーキテクチャ | イベント駆動 | 撮影→キュー→送信の非同期パイプラインに最適。コンポーネントの疎結合性を確保。 |
| キュー永続化 | SQLite | トランザクション安全、Python標準搭載、クエリ対応。プロセス再起動後も復元可能。 |
| ウィンドウ検知 | WinEventHook | イベント駆動でCPU効率が良い。ポーリングより低リソース消費。 |
| 容量制御 | 最古優先削除 | 撮影を止めず最新データを優先。ロギングの継続性を確保。 |
| ログ | Python logging | ファイルローテーション付き。十分な運用性を確保しつつシンプル。 |

## 5. 外部インターフェース

| Service | Protocol | Auth | Direction |
|---|---|---|---|
| AWS IoT Core | MQTTS (port 8883) | X.509 証明書 | Outbound (Publish) |
| Presigned URL API | HTTPS | API依存 | Outbound (GET) |
| Amazon S3 | HTTPS (PUT) | Presigned URL | Outbound (Upload) |

## 6. 詳細参照

- **コンポーネント定義**: [components.md](components.md)
- **メソッドシグネチャ**: [component-methods.md](component-methods.md)
- **サービス定義**: [services.md](services.md)
- **依存関係**: [component-dependency.md](component-dependency.md)
- **データインターフェース**: [data-accumulation-interface.md](../../../../aidlc-docs/inception/application-design/data-accumulation-interface.md) (親AI-DLC参照)
