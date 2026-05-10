# 作業単位計画

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - 作業単位計画  
**状態**: 完了  
**日付**: 2026-05-10

---

## 1. 目的

digital-twinを、後続のCONSTRUCTIONフェーズで扱いやすい作業単位へ分解する。今回の範囲はINCEPTIONのみであり、実装、詳細機能設計、テスト実装、インフラ構築は行わない。

作業単位は、ユーザーストーリー、アプリケーション設計、依存関係、チーム分担、デプロイ境界を踏まえて定義する。

---

## 2. 前提

- プロジェクト種別: Greenfield
- デスクトップクライアント: Windows対応のPython + PySide6を第一候補
- バックエンド: AWS Lambda中心のサーバーレスConversation API
- 通信方式: REST APIを基本にし、将来のストリーミング応答に備える
- RAG検索: data-accumulationのS3 Vectors連携をRetrieval Adapterに隠蔽
- 会話履歴: digital-twin専用DynamoDBテーブル
- 認証: Amazon Cognito User Pool
- セキュリティ: `user_id` 分離、Security Baseline、監査ログを必須制約とする
- PBT: 後続の機能設計へ不変条件候補を引き継ぐ

---

## 3. 生成予定成果物

承認後、以下を生成する。

- `aidlc-docs/inception/application-design/unit-of-work.md`
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md`

---

## 4. 実行チェックリスト

- [x] 承認済み要件を確認する。
- [x] 承認済みユーザーストーリーを確認する。
- [x] 承認済みアプリケーション設計を確認する。
- [x] 作業単位の分割方針を確定する。
- [x] 作業単位間の依存関係方針を確定する。
- [x] Greenfieldとしてのコード構成戦略を確定する。
- [x] Security BaselineとPBT引き継ぎ観点を作業単位へ割り当てる。
- [x] `unit-of-work.md` を生成する。
- [x] `unit-of-work-dependency.md` を生成する。
- [x] `unit-of-work-story-map.md` を生成する。
- [x] すべてのユーザーストーリーが作業単位へ割り当てられていることを確認する。
- [x] 作業単位の境界と依存関係を検証する。

---

## 5. 初期分割案

現時点では、以下の作業単位候補を想定する。

| 候補ID | 作業単位候補 | 主な対象 |
|---|---|---|
| DT-UOW-001 | Desktop Client | PySide6 UI、質問入力、回答表示、根拠表示、履歴表示 |
| DT-UOW-002 | Auth & User Context | Cognito認証、トークン処理、`user_id` 解決、認可境界 |
| DT-UOW-003 | Conversation API & Orchestrator | REST API、入力検証、会話処理統括、エラー応答 |
| DT-UOW-004 | Retrieval Adapter | data-accumulation/S3 Vectors連携、検索条件、`user_id` 分離 |
| DT-UOW-005 | Answer & Evidence | Bedrock Nova回答生成、根拠整形、根拠不足時応答 |
| DT-UOW-006 | Conversation History | 専用DynamoDB会話履歴、セッション取得、履歴文脈 |
| DT-UOW-007 | Audit & Security Validation | 監査イベント、ログサニタイズ、セキュリティ検証観点 |

この初期案は、回答内容に応じて変更する。

---

## 6. 確認質問

各質問の `[Answer]:` に選択肢の文字を記入してください。該当する選択肢がない場合は `X) その他` を選び、同じ行または直後に内容を書いてください。

### 質問1: 作業単位の分割基準
digital-twinの作業単位はどの粒度で分けますか？

A) アプリケーション設計のサービス境界に合わせる  
B) デスクトップ、API、検索、生成、履歴、監査の機能別に分ける  
C) フロントエンドとバックエンドの2単位に大きく分ける  
D) 最初はMVP単位を1つにまとめ、後で分割する  
X) その他

[Answer]:B

### 質問2: Desktop Clientの扱い
PySide6デスクトップアプリは独立した作業単位にしますか？

A) 独立した作業単位にする  
B) Auth Clientと合わせて「Desktop App」単位にする  
C) APIとの結合が強いためConversation API単位に含める  
D) 後続のUI設計で決める  
X) その他

[Answer]:B

### 質問3: AuthとUser Contextの境界
認証クライアント、Cognito連携、User Context Resolverはどう分けますか？

A) Desktop側Auth ClientとバックエンドUser Context Resolverを別作業単位にする  
B) 認証・認可として1つの作業単位にまとめる  
C) User Context ResolverはConversation API単位に含める  
D) Security Baseline設計で後から分ける  
X) その他

[Answer]:C

### 質問4: Retrieval Adapterの独立性
data-accumulation/S3 Vectors連携を扱うRetrieval Adapterは独立した作業単位にしますか？

A) 独立させる。data-accumulationとの境界が重要なため  
B) Conversation Orchestratorに含める  
C) Answer GeneratorとまとめてRAG回答単位にする  
D) 後続の機能設計で決める  
X) その他

[Answer]:A

### 質問5: Answer GeneratorとEvidence Formatterの分け方
回答生成と根拠整形はどのように扱いますか？

A) Answer GeneratorとEvidence Formatterを別作業単位にする  
B) 1つの「Answer & Evidence」作業単位にまとめる  
C) Conversation Orchestratorに含める  
D) 初期はまとめ、根拠表示が複雑化したら分離する  
X) その他

[Answer]:B

### 質問6: Conversation Historyの扱い
会話履歴保存は独立した作業単位にしますか？

A) 独立させる。専用DynamoDBと履歴文脈があるため  
B) Conversation API & Orchestratorに含める  
C) Desktop Clientの履歴表示とまとめる  
D) 後続のデータ設計で決める  
X) その他

[Answer]:A

### 質問7: 監査・セキュリティ検証の扱い
Audit Logger、ログサニタイズ、`user_id` 分離検証、PBT候補はどう扱いますか？

A) 横断作業単位として独立させる  
B) 各機能作業単位の受け入れ条件に分散させる  
C) 独立作業単位と各作業単位の受け入れ条件を併用する  
D) 後続のSecurity Baseline設計で扱う  
X) その他

[Answer]:C

### 質問8: 開発・所有の進め方
作業単位はどの開発体制を想定して分けますか？

A) 1人が順番に進める前提で、依存順に分ける  
B) 複数人または複数エージェントが並行できるよう、境界を明確に分ける  
C) MVPを最短で作るため、単位数を少なめにする  
D) 後続フェーズで決める  
X) その他

[Answer]:B

### 質問9: Greenfieldコード構成
後続でコードを作る場合、ディレクトリ構成はどの方針にしますか？

A) `desktop/`, `backend/`, `shared/`, `tests/`, `iac/` のように大きく分ける  
B) 作業単位ごとに `units/<unit-name>/` を作る  
C) AWS Lambda中心に `backend/functions/`, `backend/shared/`, `desktop/` とする  
D) まだ固定せず、作業単位文書には候補だけ残す  
X) その他

[Answer]:A

### 質問10: 作業単位生成の詳細度
生成する作業単位ドキュメントはどの詳細度にしますか？

A) 責務、含むコンポーネント、依存、対象ストーリーまで  
B) Aに加えて、想定ディレクトリ、主要API、テスト観点まで  
C) constructionに進む前提ではなく、候補単位として軽めにする  
D) 最小限の一覧だけにする  
X) その他

[Answer]:B

---

## 7. 承認ゲート

すべての `[Answer]:` が埋まった後、回答の曖昧さや矛盾を確認する。問題がなければ、この計画の承認を依頼し、承認後に作業単位成果物を生成する。

---

## 8. 回答分析

### 回答サマリー

- 質問1: B - デスクトップ、API、検索、生成、履歴、監査の機能別に分ける。
- 質問2: B - PySide6デスクトップアプリとAuth Clientを「Desktop App」単位にまとめる。
- 質問3: C - User Context ResolverはConversation API単位に含める。
- 質問4: A - Retrieval Adapterは独立作業単位にする。
- 質問5: B - Answer GeneratorとEvidence Formatterを「Answer & Evidence」単位にまとめる。
- 質問6: A - Conversation Historyは独立作業単位にする。
- 質問7: C - Audit & Security Validationは独立作業単位と各作業単位の受け入れ条件を併用する。
- 質問8: B - 複数人または複数エージェントが並行できるように境界を明確に分ける。
- 質問9: A - `desktop/`, `backend/`, `shared/`, `tests/`, `iac/` の大枠で構成する。
- 質問10: B - 責務、コンポーネント、依存、対象ストーリーに加え、想定ディレクトリ、主要API、テスト観点まで記載する。

### 曖昧さ・矛盾の確認

- すべての質問に回答済み。
- 選択肢外の回答なし。
- Desktop側Auth ClientはDesktop App単位、バックエンド側User Context ResolverはConversation API単位に含めるため、質問2と質問3は整合している。
- Retrieval、Answer & Evidence、Conversation History、Audit & Security Validationの境界は、アプリケーション設計の責務境界と整合している。
- 追加確認が必要な曖昧さはなし。

### 承認後の生成方針

承認後、以下の作業単位を中心に成果物を生成する。

| 作業単位ID | 作業単位 |
|---|---|
| DT-UOW-001 | Desktop App |
| DT-UOW-002 | Conversation API & Orchestrator |
| DT-UOW-003 | Retrieval Adapter |
| DT-UOW-004 | Answer & Evidence |
| DT-UOW-005 | Conversation History |
| DT-UOW-006 | Audit & Security Validation |
