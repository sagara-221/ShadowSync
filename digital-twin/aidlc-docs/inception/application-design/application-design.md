# アプリケーション設計書: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計  
**バージョン**: 1.0  
**日付**: 2026-05-10  
**ステータス**: 承認済み

---

## 1. 設計サマリー

digital-twinは、Windows対応のPythonデスクトップアプリとして提供し、AWS上のConversation APIを通じて、認証済みユーザーの活動ログをRAG検索し、Bedrock Nova系モデルで根拠付き回答を生成する。

デスクトップクライアントはPySide6を第一候補とする。バックエンドはAWS Lambda中心のサーバーレスAPIとし、通信はREST APIを基本にする。将来のストリーミング応答に備え、回答本文、根拠、状態、エラーを分離したレスポンスモデルを採用する。

---

## 2. 採用した主要判断

| 項目 | 判断 |
|---|---|
| デスクトップ技術 | Windows対応のPython + PySide6を第一候補 |
| バックエンド実行形態 | AWS Lambda中心のサーバーレスAPI |
| 通信方式 | REST APIを基本にし、将来のストリーミング応答に備える |
| 会話履歴保存 | digital-twin専用DynamoDBテーブル |
| RAG検索連携 | Retrieval Adapterに閉じ込める |
| 根拠表示責務 | バックエンドが標準モデルを返し、クライアントが表示調整する |
| セキュリティ責務 | User Context Resolverと各バックエンドサービスで一貫して強制 |
| 監査ログ | 認証・認可失敗、検索実行、回答生成、履歴保存失敗まで論理イベントを定義 |

---

## 3. システム構成

digital-twinは以下の主要領域で構成する。

- Desktop App Service
- Conversation Service
- Retrieval Service
- Answer Generation Service
- Evidence Service
- Conversation History Service
- Audit Service

Desktop App ServiceはローカルWindows環境で動作する。Conversation Service以降はAWS上で実行し、活動ログ検索、回答生成、履歴保存、監査を担当する。

---

## 4. コンポーネント構成

| コンポーネント | 責務 |
|---|---|
| Desktop Client | 質問入力、回答表示、根拠表示、履歴表示 |
| Auth Client | Cognitoログイン、トークン取得、セッション更新 |
| Conversation API | REST API受付、入力検証、応答返却 |
| User Context Resolver | トークン由来の `user_id` 確定 |
| Conversation Orchestrator | 検索、生成、根拠整形、履歴保存、監査の統括 |
| Retrieval Adapter | S3 Vectors連携の隠蔽と `user_id` 分離検索 |
| Answer Generator | Bedrock Nova系モデルによる回答生成 |
| Evidence Formatter | 検索結果から標準根拠モデルへの変換 |
| Conversation History Store | 専用DynamoDBへの会話履歴保存・取得 |
| Audit Logger | 監査イベント記録 |

詳細は `components.md` と `component-methods.md` に記載する。

---

## 5. 主要処理フロー

1. ユーザーがDesktop Clientで質問を入力する。
2. Auth ClientがCognitoアクセストークンを提供する。
3. Desktop ClientがConversation APIへ質問をPOSTする。
4. Conversation APIが入力を検証する。
5. User Context Resolverがトークンから `UserContext` を確定する。
6. Conversation Orchestratorが直近会話履歴を取得する。
7. Retrieval Adapterが `user_id` 分離条件付きで活動ログを検索する。
8. Answer Generatorが検索結果と会話文脈から回答を生成する。
9. Evidence Formatterが根拠表示モデルを生成する。
10. Conversation History Storeが質問、回答、根拠、処理結果を保存する。
11. Audit Loggerが検索、生成、失敗イベントを記録する。
12. Conversation APIが回答、根拠、状態をDesktop Clientへ返す。

---

## 6. API境界

初期API候補は以下とする。

| API | メソッド | 目的 |
|---|---|---|
| `/conversation/messages` | POST | 質問を送信し、回答と根拠を取得する |
| `/conversation/sessions` | GET | 自分の会話セッション一覧を取得する |
| `/conversation/sessions/{session_id}` | GET | 自分の指定セッション履歴を取得する |
| `/health` | GET | API疎通確認 |

すべてのユーザー固有APIはAuthorizationヘッダーを必須とする。`user_id` はクライアント入力ではなく、サーバー側で確定する。

---

## 7. データ境界

### 活動ログ

- data-accumulationが管理する。
- digital-twinは読み取り検索のみを行う。
- S3 Vectors連携の詳細はRetrieval Adapterに閉じる。

### 会話履歴

- digital-twin専用DynamoDBテーブルで管理する。
- 活動ログとは分離する。
- ユーザー別、セッション別、メッセージ別に取得できるよう設計する。

### 監査ログ

- 監査用ログ基盤に記録する。
- 認証トークン、全文プロンプト、検索結果全文、機微情報を不用意に記録しない。

---

## 8. セキュリティ設計

### 必須制約

- 認可に使う `user_id` はCognitoトークン由来のみとする。
- クライアント指定の `user_id` は無視する。
- RAG検索、会話履歴、根拠表示、監査ログでユーザー境界を維持する。
- IAM、ストレージ設計、アプリケーション層で多層防御を行う。
- API入力は型、長さ、形式、サイズを検証する。
- ユーザー向けエラーには内部パス、スタックトレース、トークン、機密情報を含めない。

### Security Baseline適用サマリー

| 項目 | 設計反映 |
|---|---|
| ログ・機微情報 | Audit Loggerでサニタイズし、トークンや全文プロンプトを記録しない |
| 入力検証 | Conversation APIで全入力を検証する |
| 最小権限 | Retrieval、History、Auditの権限をコンポーネント単位に分離する |
| 認可 | User Context Resolverと各バックエンドサービスで強制する |
| 多層防御 | アプリケーション層、IAM、ストレージ設計で `user_id` 境界を維持する |
| 監査 | 認証・認可失敗、検索、生成、履歴保存失敗を記録する |

---

## 9. 失敗処理

| 失敗種別 | システム応答 |
|---|---|
| 認証失敗 | 再認証を要求し、活動ログや履歴を返さない |
| 認可失敗 | 権限不足として拒否し、対象データを返さない |
| 入力不正 | 修正可能なエラーとして返す |
| 検索失敗 | 根拠なしの断定回答を避ける |
| 根拠不足 | 情報不足を明示し、断定を避ける |
| LLM失敗 | 回答生成失敗として再試行可能性を示す |
| 履歴保存失敗 | 回答表示可能な場合は表示し、履歴保存失敗を通知する |

---

## 10. PBT引き継ぎ

Property-Based Testingの詳細は後続の機能設計で扱う。ただし、アプリケーション設計から以下を不変条件候補として引き継ぐ。

- 任意の検索処理で、認証済み `user_id` による分離条件が必ず含まれる。
- 任意の履歴保存・取得処理で、`UserContext.user_id` が必ず条件に含まれる。
- クライアントから `user_id` が送られても、認可判断には使われない。
- `SearchResult` から `EvidenceItem` へ変換しても、参照元識別子、時刻、ソース種別が失われない。
- APIリクエストとレスポンスはシリアライズ後も必須フィールドを保持する。

---

## 11. 後続ステージへの引き継ぎ

作業単位計画では、以下を候補単位へ分解する。

- Windows Pythonデスクトップクライアント
- Cognito認証クライアント
- Conversation REST API
- User Context Resolver
- Retrieval Adapter
- Answer Generator
- Evidence Formatter
- Conversation History Store
- Audit Logger
- セキュリティ境界とPBT候補

Constructionフェーズは今回開始しない。

---

## 12. 関連成果物

- `components.md`
- `component-methods.md`
- `services.md`
- `component-dependency.md`
