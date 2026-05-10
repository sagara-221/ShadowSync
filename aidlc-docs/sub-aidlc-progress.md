# ShadowSync — 子AI-DLC 進捗管理

**最終更新**: 2026-05-10T13:50:00+09:00

---

## システム全体像

```
ShadowSync (親AI-DLC) ── 完了
├── logger/ (子AI-DLC: ロガーシステム)
│   ├── osapi/            (孫) ── Inception完了 → Construction途中
│   ├── ss-tool/          (孫) ── Inception完了 → Construction待ち
│   └── chrome-extension/ (孫) ── Inception完了 → Construction待ち
├── data-accumulation/    (子) ── 未着手
├── daily-log/            (子) ── 未着手
└── digital-twin/         (子) ── 未着手
```

---

## 進捗サマリ

| ユニット | フェーズ | 現在のステージ | 進捗率 | 備考 |
|---|---|---|:---:|---|
| **親 AI-DLC** | 完了 | — | 100% | Units Generation 完了。子AI-DLCへの指示書(intent.md)配布済み |
| **logger/osapi** | Construction | Code Generation 待ち | 60% | Inception完了、Functional Design完了 |
| **logger/ss-tool** | Construction | Functional Design 待ち | 40% | Inception完了（Application Design含む） |
| **logger/chrome-extension** | Construction | Code Generation 待ち | 30% | Inception完了。FD/NFRスキップで直接CGへ |
| **data-accumulation** | 未着手 | — | 0% | intent.md 配布済み |
| **daily-log** | 未着手 | — | 0% | intent.md 配布済み |
| **digital-twin** | 未着手 | — | 0% | intent.md 配布済み |

---

## 詳細ステータス

### 1. Logger — OS API ロガー (`logger/osapi/`)

**概要**: Windows上のアクティビティ（ウィンドウ切替、入力有無、オーディオセッション）を監視し、AWS IoT CoreへMQTTS送信するバックグラウンドエージェント。

| ステージ | 状態 | 成果物 |
|---|:---:|---|
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md`, `requirement-verification-questions.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| Application Design | ⏭️ スキップ | — |
| **Functional Design** | ✅ **完了** | `domain-entities.md`, `business-logic-model.md`, `business-rules.md` |
| Code Generation | ⏳ **次に実行** | `osapi-code-generation-plan.md` (計画のみ作成済み) |
| Build and Test | ⬚ 未着手 | — |

**技術スタック**: Python, psutil, pywin32, MQTTS (X.509証明書)  
**実行形態**: バックグラウンドプロセス

---

### 2. Logger — スクリーンショットロガー (`logger/ss-tool/`)

**概要**: アクティブウィンドウのスクリーンショットをWebP形式で撮影し、S3へアップロード + メタデータをMQTTS送信するWindowsサービス。

| ステージ | 状態 | 成果物 |
|---|:---:|---|
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md`, `requirement-verification-questions.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| **Application Design** | ✅ **完了** | `components.md`, `component-methods.md`, `services.md`, `component-dependency.md`, `application-design.md` |
| Functional Design | ⏳ **次に実行** | — |
| NFR Requirements | ⬚ 未着手 | — |
| NFR Design | ⬚ 未着手 | — |
| Code Generation | ⬚ 未着手 | — |
| Build and Test | ⬚ 未着手 | — |

**技術スタック**: Python, pywin32, SQLite, Presigned URL (S3), MQTTS (X.509証明書)  
**アーキテクチャ**: イベント駆動 (11コンポーネント + 4サービス)  
**実行形態**: Windowsサービス (pywin32)

---

### 3. Logger — Chrome拡張ロガー (`logger/chrome-extension/`)

**概要**: Chromeブラウザの閲覧履歴（URL、タイトル、HTMLスニペット）を取得し、MQTT over WebSocketsでAWS IoT Coreへ送信するChrome拡張機能。

| ステージ | 状態 | 成果物 |
|---|:---:|---|
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md`, `requirement-verification-questions.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| Application Design | ⏭️ スキップ | — |
| Functional Design | ⏭️ スキップ | — |
| NFR Requirements/Design | ⏭️ スキップ | — |
| Code Generation | ⏳ **次に実行** | — |
| Build and Test | ⬚ 未着手 | — |

**技術スタック**: JavaScript (Manifest V3), Service Worker, IndexedDB, MQTT over WebSockets  
**認証方式**: Amazon Cognito IDプール  

---

### 4. データ蓄積システム (`data-accumulation/`)

**概要**: IoT Coreで受信したログ + S3の画像を処理し、Bedrockで意味抽出・正規化してDynamoDB等に蓄積するAWSバックエンドシステム。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Inception 開始 | ⬚ **未着手** | — |

**技術スタック (予定)**: AWS IoT Core, Lambda, EventBridge, S3, DynamoDB, Bedrock  
**依存関係**: logger (osapi, ss-tool, chrome-extension) からのデータを受信する  
**注意事項**: ss-tool が Presigned URL 方式を採用したため、URL発行API (Lambda/API Gateway) の設計が本システムの責務に含まれる

---

### 5. 日誌作成システム (`daily-log/`)

**概要**: 蓄積データから日報を自動生成し、Notion APIでエクスポートするバッチシステム。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Inception 開始 | ⬚ **未着手** | — |

**技術スタック (予定)**: AWS Lambda, EventBridge, Bedrock, Notion API  
**依存関係**: data-accumulation のデータソース (DynamoDB等) に依存

---

### 6. デジタルツインシステム (`digital-twin/`)

**概要**: 過去の活動データを検索・回答するRAGベースのAIチャットボット。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Inception 開始 | ⬚ **未着手** | — |

**技術スタック (予定)**: Bedrock Knowledge Base, RAG, LLM (Claude)  
**依存関係**: data-accumulation のナレッジベース/ベクトルストアに依存

---

## ユニット間の依存関係

```
logger/osapi ──────────┐
logger/ss-tool ────────┤──▶ data-accumulation ──┬──▶ daily-log
logger/chrome-extension┘                        └──▶ digital-twin
```

| 上流 | 下流 | 依存内容 |
|---|---|---|
| logger/* | data-accumulation | MQTT/S3 経由のデータ送信 |
| data-accumulation | daily-log | DynamoDB からの日次データ取得 |
| data-accumulation | digital-twin | ナレッジベース/ベクトルストアの検索 |
| ss-tool | data-accumulation | **Presigned URL 発行API** (設計協調が必要) |

---

## 推奨される次のアクション

| 優先度 | アクション | 対象 |
|---|---|---|
| 🔴 高 | Code Generation の実行 | logger/osapi |
| 🟠 中 | Functional Design の実行 | logger/ss-tool |
| 🟠 中 | Code Generation の実行 | logger/chrome-extension |
| 🟡 低 | Inception の開始 | data-accumulation |
| ⚪ 後回し | Inception の開始 | daily-log (data-accumulationに依存) |
| ⚪ 後回し | Inception の開始 | digital-twin (data-accumulationに依存) |
