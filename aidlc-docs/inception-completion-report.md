# ShadowSync Inceptionフェーズ終了レポート

**作成日**: 2026-05-10  
**対象**: ShadowSync 親AI-DLCおよび子AI-DLC Inception成果物  
**提出目的**: ハッカソン「人をダメにするサービス」審査用資料  
**テーマ適合方針**: 人が日々の記録、振り返り、報告、過去行動の検索を自分で頑張らなくてもよい状態を作る。

---

## 1. システム概要

ShadowSyncは、PC上の活動ログを自動的に集め、日報生成とデジタルツイン対話へつなげるサービスである。

ユーザーは、作業履歴を覚える、日報を書く、昨日何をしていたか探す、過去の行動傾向を整理する、といった面倒な作業をシステムに任せられる。ハッカソンテーマである「人をダメにするサービス」に対して、本プロダクトは「記録する努力」と「思い出す努力」を減らす方向で設計している。

全体は以下の4領域で構成する。

| 領域 | 役割 |
|---|---|
| Logger | OSアクティビティ、Chrome閲覧情報、スクリーンショットを自動収集する |
| Data Accumulation | 収集データをDynamoDB/S3等へ蓄積し、検索・日報生成に使える形へ整える |
| Daily Log | 蓄積データから日報を自動生成し、Notionへ出力する |
| Digital Twin | 過去活動をRAG検索し、ユーザー自身の行動履歴に基づく対話を提供する |

---

## 2. ビジネス意図

### 2.1 Intent

日々の活動を人が手で記録しなくても、作業ログ、閲覧ログ、スクリーンショットをもとに自動で整理し、必要なときに日報や対話形式で取り出せるようにする。

### 2.2 解決する面倒

- 作業時間や作業内容を後から思い出す負担
- 日報・週報を書くために履歴をかき集める負担
- 過去の調査、閲覧、判断の経緯を探す負担
- 自分の行動傾向を分析するためにログを整理する負担

### 2.3 テーマ適合性

本サービスは、ユーザーの「記録する」「整理する」「思い出す」「報告する」を自動化対象にする。人の怠慢を肯定し、むしろ怠慢を成立させるために、裏側でログ収集とAI活用の仕組みを整える点がテーマに合っている。

---

## 3. 親AI-DLCのInception成果

親AI-DLCは、システム全体のコード実装ではなく、全体Intentの整理、ユニット分解、子AI-DLCへの委譲、親子間インターフェースの整合を担当した。

### 3.1 親AI-DLCで実施したこと

| ステージ | 結果 | 成果物 |
|---|---|---|
| Workspace Detection | 完了 | [aidlc-state.md](aidlc-state.md) |
| Requirements Analysis | 完了 | [requirements.md](inception/requirements/requirements.md) |
| Workflow Planning | 完了 | [execution-plan.md](inception/plans/execution-plan.md) |
| Units Planning | 完了 | [unit-of-work-plan.md](inception/plans/unit-of-work-plan.md) |
| Units Generation | 完了 | [unit-of-work.md](inception/application-design/unit-of-work.md), [unit-of-work-dependency.md](inception/application-design/unit-of-work-dependency.md), [unit-of-work-story-map.md](inception/application-design/unit-of-work-story-map.md) |

### 3.2 親AI-DLCの主な判断

- 親AI-DLCはアプリケーションコードを持たない。
- 実装責務は子AI-DLCへ分割し、各子AI-DLCが独立してInceptionからConstructionへ進める。
- ロガー群、データ蓄積、日報生成、デジタルツインを独立ユニットとして扱う。
- 親子間のデータ連携仕様は親AI-DLCで集約し、子AI-DLCの結果に合わせて更新する。
- 構造化ログはLambdaでDynamoDBスキーマへ直接マッピングし、LLM利用は主にスクリーンショットの日本語キャプション生成へ限定する。

---

## 4. ユニット分解

親AI-DLCでは、以下の単位にシステムを分割した。

```text
ShadowSync
├── logger/
│   ├── osapi/
│   ├── chrome-extension/
│   └── ss-tool/
├── data-accumulation/
├── daily-log/
└── digital-twin/
```

| ユニット | 責務 | Inception成果物 |
|---|---|---|
| `logger/osapi` | Windows上のアクティブウィンドウ、プロセス、オーディオセッション等を取得する | [requirements](../logger/osapi/aidlc-docs/inception/requirements/requirements.md), [execution-plan](../logger/osapi/aidlc-docs/inception/plans/execution-plan.md) |
| `logger/chrome-extension` | ChromeのURL、ページタイトル、必要に応じたHTMLスニペットを取得する | [requirements](../logger/chrome-extension/aidlc-docs/inception/requirements/requirements.md), [execution-plan](../logger/chrome-extension/aidlc-docs/inception/plans/execution-plan.md) |
| `logger/ss-tool` | アクティブウィンドウのスクリーンショットを取得し、S3へアップロードする | [requirements](../logger/ss-tool/aidlc-docs/inception/requirements/requirements.md), [application-design](../logger/ss-tool/aidlc-docs/inception/application-design/application-design.md) |
| `data-accumulation` | ログを受信・正規化し、DynamoDB/S3/S3 Vectors等へ保存する | [requirements](../data-accumulation/aidlc-docs/inception/requirements/requirements.md), [unit-of-work](../data-accumulation/aidlc-docs/inception/application-design/unit-of-work.md), [final-evaluation](../data-accumulation/aidlc-docs/inception-final-evaluation.md) |
| `daily-log` | 正規化済み活動データから日報を生成し、Notionへ出力する | [requirements](../daily-log/aidlc-docs/inception/requirements/requirements.md), [user-stories](../daily-log/aidlc-docs/inception/user-stories/stories.md), [application-design](../daily-log/aidlc-docs/inception/application-design/application-design.md) |
| `digital-twin` | 過去活動のRAG検索と、ユーザー自身のデジタルツイン対話を提供する | [requirements](../digital-twin/aidlc-docs/inception/requirements/requirements.md), [user-stories](../digital-twin/aidlc-docs/inception/user-stories/stories.md), [application-design](../digital-twin/aidlc-docs/inception/application-design/application-design.md), [unit-of-work](../digital-twin/aidlc-docs/inception/application-design/unit-of-work.md) |

ユニット間の依存関係は [unit-of-work-dependency.md](inception/application-design/unit-of-work-dependency.md) に整理した。

---

## 5. 親AI-DLCを用いた全体開発フロー

本プロジェクトでは、親AI-DLCと子AI-DLCを分けた再帰的な開発フローを採用した。

1. 親AI-DLCで全体Intentを整理する。
2. 親AI-DLCでユニット分解を行い、子AI-DLCのワークスペースを分ける。
3. 各子AI-DLCが独立してInceptionを実施する。
4. 子AI-DLCの成果物を親AI-DLCへフィードバックし、親子間インターフェースを同期する。
5. Constructionで扱う詳細設計事項を専用バックログへ整理する。
6. Constructionでは、各子AI-DLCが自分の責務範囲でFunctional Design、Infrastructure Design、Code Generationへ進む。

この流れにより、全体構想を親AI-DLCで保ちつつ、個別ユニットの具体設計は子AI-DLCに委譲できる。

---

## 6. 親子間インターフェースの整理

Inceptionの後半で、子AI-DLCの結果を反映して親AI-DLCの想定仕様を更新した。

主な確定事項は以下。

| 項目 | 最終方針 |
|---|---|
| OS APIログ | AWS IoT Core MQTTS + X.509で送信 |
| Chromeログ | MQTT over WebSockets + Cognito IDプールで送信 |
| ss-toolメタデータ | AWS IoT Core MQTTS + X.509で送信 |
| ss-tool画像アップロード | IoT Core Request/ResponseでPresigned URLを取得し、S3へPUT |
| Chrome画像アップロード | Lambda Function URL + Cognito IDプール由来IAM認証でPresigned URLを取得 |
| 構造化データ処理 | LambdaでDynamoDB共通スキーマへ直接マッピング |
| スクリーンショット処理 | S3 ObjectCreatedを契機にBedrock Nova Lite等で日本語キャプション生成 |
| 日報生成 | data-accumulationのDynamoDB正規化済み活動データを主入力にする |
| Digital Twin | data-accumulationが構築するS3 Vectors / Bedrock Knowledge Base系RAG I/Fに依存する |

詳細は [data-accumulation-interface.md](inception/application-design/data-accumulation-interface.md) にまとめた。

---

## 7. Constructionで行うこと

Inceptionで定義した全体Intent、ユニット境界、親子間インターフェースを前提に、Constructionでは以下を具体化する。

| 領域 | Constructionで行うこと | 詳細 |
|---|---|---|
| data-accumulation | 共通スキーマ、DynamoDB/S3設計、Presigned URL、RAG同期方式をFunctional Design / Infrastructure Designで確定する | [Construction設計QAバックログ](construction-design-qa-backlog.md) |
| logger | data-accumulationの確定スキーマに合わせて、OS API、Chrome、スクリーンショット送信実装を進める | [Constructionで行うこと](construction-readiness-issues.md) |
| daily-log | DynamoDB読取I/F、日報生成タイミング、Notion出力仕様をFunctional Designで確定する | [Construction設計QAバックログ](construction-design-qa-backlog.md) |
| digital-twin | RAG検索I/F、根拠表示、会話履歴、認証フローをFunctional Designで確定する | [Construction設計QAバックログ](construction-design-qa-backlog.md) |

---

## 8. 子AI-DLC成果物リンク集

### 8.1 Logger

| 対象 | 状態 | リンク |
|---|---|---|
| OS API Logger | Inception完了、Construction途中 | [state](../logger/osapi/aidlc-docs/aidlc-state.md), [requirements](../logger/osapi/aidlc-docs/inception/requirements/requirements.md), [execution-plan](../logger/osapi/aidlc-docs/inception/plans/execution-plan.md) |
| Chrome Extension Logger | Inception完了、Construction待ち | [state](../logger/chrome-extension/aidlc-docs/aidlc-state.md), [requirements](../logger/chrome-extension/aidlc-docs/inception/requirements/requirements.md), [execution-plan](../logger/chrome-extension/aidlc-docs/inception/plans/execution-plan.md) |
| Screenshot Tool | Inception完了、Construction待ち | [state](../logger/ss-tool/aidlc-docs/aidlc-state.md), [requirements](../logger/ss-tool/aidlc-docs/inception/requirements/requirements.md), [application-design](../logger/ss-tool/aidlc-docs/inception/application-design/application-design.md) |

### 8.2 Backend / AI

| 対象 | 状態 | リンク |
|---|---|---|
| Data Accumulation | Inception完了、Functional Design待ち | [state](../data-accumulation/aidlc-docs/aidlc-state.md), [requirements](../data-accumulation/aidlc-docs/inception/requirements/requirements.md), [unit-of-work](../data-accumulation/aidlc-docs/inception/application-design/unit-of-work.md), [final-evaluation](../data-accumulation/aidlc-docs/inception-final-evaluation.md) |
| Daily Log | Inception完了、Construction待ち | [state](../daily-log/aidlc-docs/aidlc-state.md), [requirements](../daily-log/aidlc-docs/inception/requirements/requirements.md), [stories](../daily-log/aidlc-docs/inception/user-stories/stories.md), [application-design](../daily-log/aidlc-docs/inception/application-design/application-design.md) |
| Digital Twin | Inception完了、Construction待ち | [state](../digital-twin/aidlc-docs/aidlc-state.md), [requirements](../digital-twin/aidlc-docs/inception/requirements/requirements.md), [stories](../digital-twin/aidlc-docs/inception/user-stories/stories.md), [application-design](../digital-twin/aidlc-docs/inception/application-design/application-design.md), [unit-of-work](../digital-twin/aidlc-docs/inception/application-design/unit-of-work.md) |

---

## 9. 審査基準への対応

| 審査基準 | 対応内容 |
|---|---|
| ビジネス意図の明確さ | 「記録・振り返り・報告・検索を人が頑張らない」ことを中核Intentとして定義した |
| Unit分解の適切さ | Logger、Data Accumulation、Daily Log、Digital Twinへ責務分離し、さらにLoggerは3つの孫ユニットへ分解した |
| 創造性とテーマ適合性 | 人の活動ログを自動的に集め、日報と自分のデジタルツインへつなげることで、面倒な自己管理を肩代わりする |
| ドキュメント品質 | 親AI-DLC、子AI-DLC、終了レポート、Constructionバックログを分け、InceptionとConstructionの範囲を明確化した |

---

## 10. 終了判定

ShadowSyncの親AI-DLCおよび主要子AI-DLCのInceptionフェーズは完了している。

全体Intent、ユニット分解、親子間インターフェース、子AI-DLC成果物への導線が整理済みであり、Constructionで行う詳細設計事項は専用バックログへ整理済みである。よって、本プロダクトはInception成果物としてハッカソン提出可能な状態に到達した。
