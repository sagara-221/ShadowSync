# ShadowSync — 子AI-DLC 進捗管理

**最終更新**: 2026-05-10T15:40:25+09:00

---

## システム全体像

```
ShadowSync (親AI-DLC) ── 完了
├── logger/ (子AI-DLC: ロガーシステム)
│   ├── osapi/            (孫) ── Inception完了 → Construction途中
│   ├── ss-tool/          (孫) ── Inception完了 → Construction待ち
│   └── chrome-extension/ (孫) ── Inception完了 → Construction待ち
├── data-accumulation/    (子) ── Inception完了 → Construction待ち
├── daily-log/            (子) ── Inception完了 → Construction待ち
└── digital-twin/         (子) ── Inception完了 → Construction待ち
```

---

## 進捗サマリ

| ユニット | フェーズ | 現在のステージ | 進捗率 | 備考 |
|---|---|---|:---:|---|
| **親 AI-DLC** | 完了 | — | 100% | Units Generation 完了。子AI-DLCへの指示書(intent.md)配布済み |
| **logger/osapi** | Construction | Code Generation 待ち | 60% | Inception完了、Functional Design完了 |
| **logger/ss-tool** | Construction | Functional Design 待ち | 40% | Inception完了（Application Design含む） |
| **logger/chrome-extension** | Construction | Code Generation 待ち | 30% | Inception完了。FD/NFRスキップで直接CGへ |
| **data-accumulation** | Inception完了 | Functional Design 待ち | 35% | 要件、Workflow、Units Planning/Generation完了。確認レポートで未確定事項を整理 |
| **daily-log** | Inception完了 | Construction開始待ち | 35% | 要件、User Stories、Application Design完了 |
| **digital-twin** | Inception完了 | Construction開始待ち | 35% | 要件、User Stories、Application Design、Units Generation完了 |

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

**概要**: IoT Coreで受信した構造化ログをLambdaでDynamoDBスキーマへ直接マッピングし、S3のスクリーンショット画像のみBedrockで日本語キャプション生成するAWSバックエンドシステム。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md`, `requirement-verification-questions*.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| User Stories | ⏭️ スキップ | Backend中心で明確なI/Fを優先 |
| Application Design | ⏭️ スキップ | Infrastructure-heavyとしてUnits生成へ進行 |
| Units Planning / Generation | ✅ 完了 | `unit-of-work*.md` |
| Functional Design | ⏳ **次に実行** | HTML保存戦略、RAG同期、Presigned URL詳細を確定 |

**技術スタック (予定)**: AWS IoT Core, Lambda, Kinesis, S3, DynamoDB, Bedrock, S3 Vectors / Bedrock Knowledge Base
**依存関係**: logger (osapi, ss-tool, chrome-extension) からのデータを受信する
**注意事項**: ss-tool向けは IoT Core Request/Response、Chrome拡張向けは Lambda Function URL + Cognito IDプール由来IAM認証でPresigned URLを発行する。Inception確認で旧API Gateway前提とステータス表記は同期済み。

---

### 5. 日誌作成システム (`daily-log/`)

**概要**: 蓄積データから日報を自動生成し、Notion APIでエクスポートするバッチシステム。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md`, `requirement-verification-questions*.md` |
| User Stories | ✅ 完了 | `personas.md`, `stories.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| Application Design | ✅ 完了 | `components.md`, `component-methods.md`, `services.md`, `application-design.md` |
| Functional Design | ⏳ **次に実行** | Notionプロパティ、対象日ルール、再実行方式を確定 |

**技術スタック (予定)**: AWS Lambda, EventBridge, Bedrock, Notion API
**依存関係**: data-accumulation のデータソース (DynamoDB等) に依存

---

### 6. デジタルツインシステム (`digital-twin/`)

**概要**: 過去の活動データを検索・回答するRAGベースのAIチャットボット。

| ステージ | 状態 | 備考 |
|---|:---:|---|
| intent.md 配布 | ✅ 完了 | 親AI-DLCから配布済み |
| Workspace Detection | ✅ 完了 | `aidlc-state.md` |
| Requirements Analysis | ✅ 完了 | `requirements.md` |
| User Stories | ✅ 完了 | `personas.md`, `stories.md` |
| Workflow Planning | ✅ 完了 | `execution-plan.md` |
| Application Design | ✅ 完了 | `components.md`, `component-methods.md`, `services.md`, `application-design.md` |
| Units Planning / Generation | ✅ 完了 | `unit-of-work*.md` |
| Functional Design | ⏳ **次に実行** | RAG検索I/F、会話履歴、認証フローを確定 |

**技術スタック (予定)**: Bedrock Knowledge Base / S3 Vectors, RAG, Bedrock Nova系モデル
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
| 🟠 中 | Functional Design の実行 | data-accumulation |
| 🟡 低 | Functional Design の実行 | daily-log (data-accumulationの読取I/Fに依存) |
| 🟡 低 | Functional Design の実行 | digital-twin (data-accumulationのRAG I/Fに依存) |

---

## Inception確認レポート

**作成済み**: `aidlc-docs/inception-confirmation-report.md`

## Construction準備 課題・懸念点整理

**作成済み**: `aidlc-docs/construction-readiness-issues.md`

### 確認結果サマリ

- logger各孫AI-DLC、data-accumulation、daily-log、digital-twin のInception成果物を確認済み。
- 重大なスコープ逸脱はなし。
- `data-accumulation` はInception完了。HTMLスニペット保存戦略、S3 Vectors/Knowledge Base同期方式が残課題。`aidlc-state.md` の現在ステージは同期済み。
- `daily-log` はInception完了。Notionプロパティ、対象日ルール、失敗ジョブ再実行方式を後続設計で確定する。
- `digital-twin` はInception完了。ドキュメントのステータス表記は承認済みに同期済み。RAG検索I/F具体化はConstructionで扱う。
