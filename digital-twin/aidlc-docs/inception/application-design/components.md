# コンポーネント設計: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計  
**バージョン**: 1.0  
**日付**: 2026-05-10

---

## 1. 設計方針

digital-twinは、Windows対応のPythonデスクトップアプリからConversation APIを呼び出し、data-accumulationが管理する活動ログ検索基盤とAmazon Bedrock Nova系モデルを利用して、根拠付き回答を返す。

本設計では、実装コードではなく、コンポーネント責務、境界、入出力モデル、セキュリティ制約を定義する。

---

## 2. コンポーネント一覧

| ID | コンポーネント | 配置 | 主な責務 |
|---|---|---|---|
| DT-CMP-001 | Desktop Client | ローカルWindows | 質問入力、回答表示、根拠表示、履歴表示、認証UI |
| DT-CMP-002 | Auth Client | ローカルWindows | Cognitoログイン、トークン保持、セッション更新 |
| DT-CMP-003 | Conversation API | AWS API Gateway / Lambda | RESTリクエスト受付、入力検証、応答返却 |
| DT-CMP-004 | User Context Resolver | AWS Lambda内論理コンポーネント | 認証トークン由来の `user_id` 確定、認可境界の起点 |
| DT-CMP-005 | Conversation Orchestrator | AWS Lambda内論理コンポーネント | 検索、回答生成、根拠整形、履歴保存、監査の統括 |
| DT-CMP-006 | Retrieval Adapter | AWS Lambda内論理コンポーネント | data-accumulationのS3 Vectors連携を隠蔽 |
| DT-CMP-007 | Answer Generator | AWS Lambda内論理コンポーネント | Bedrock Nova系モデルによる回答生成 |
| DT-CMP-008 | Evidence Formatter | AWS Lambda内論理コンポーネント | 検索結果を標準根拠モデルへ変換 |
| DT-CMP-009 | Conversation History Store | DynamoDB連携コンポーネント | digital-twin専用会話履歴の保存・取得 |
| DT-CMP-010 | Audit Logger | AWSログ基盤連携コンポーネント | 監査イベント、失敗イベント、処理結果の記録 |

---

## 3. 主要責務

### DT-CMP-001 Desktop Client

- PySide6を第一候補とするPythonデスクトップアプリとして設計する。
- ユーザーの質問入力、回答表示、根拠の折りたたみ・並び替え、会話履歴閲覧を担当する。
- Conversation APIとの通信はHTTPS RESTを基本とする。
- 将来のストリーミング応答に備え、回答表示領域は段階的な表示更新に対応できる構造にする。
- `user_id` を認可目的でAPIへ送信しない。ユーザー境界はサーバー側で確定する。

### DT-CMP-002 Auth Client

- Amazon Cognito User Poolと連携してユーザー認証を行う。
- アクセストークンの取得、保持、更新、ログアウトを扱う。
- API呼び出し時にAuthorizationヘッダーへトークンを付与する。
- トークンや認証情報をログ出力しない。

### DT-CMP-003 Conversation API

- デスクトップアプリからのRESTリクエストを受け付ける。
- 入力の型、長さ、必須項目、形式を検証する。
- 認証済みリクエストのみを後続処理へ渡す。
- 初期は同期REST応答とし、将来のストリーミング化を妨げないレスポンスモデルを採用する。

### DT-CMP-004 User Context Resolver

- Cognitoトークンから信頼できる `user_id` を解決する。
- クライアント指定の `user_id` は無視する。
- 後続コンポーネントへ `UserContext` を渡し、検索、履歴、監査で同一のユーザー境界を強制する。

### DT-CMP-005 Conversation Orchestrator

- 会話処理の中心として、検索、回答生成、根拠整形、履歴保存、監査を順序立てて実行する。
- S3 VectorsやBedrockの詳細には直接依存せず、Retrieval AdapterとAnswer Generatorを通じて利用する。
- 根拠不足、検索失敗、LLM失敗、履歴保存失敗を区別して扱う。

### DT-CMP-006 Retrieval Adapter

- data-accumulationが提供するS3 Vectorsベースの検索基盤を隠蔽する。
- すべての検索で `UserContext.user_id` に基づく分離条件を付与する。
- 他コンポーネントにはS3 Vectors固有のレスポンスではなく、標準化された `SearchResult` を返す。

### DT-CMP-007 Answer Generator

- ユーザー質問、直近会話履歴、検索結果を入力として、Bedrock Nova系モデルへ回答生成を依頼する。
- 活動ログ由来の根拠と会話履歴由来の文脈を区別してプロンプトへ反映する。
- 根拠が不足する場合は断定を避ける回答方針を適用する。

### DT-CMP-008 Evidence Formatter

- `SearchResult` をクライアント表示向けの標準根拠モデルへ変換する。
- 標準モデルには、時刻、ソース種別、関連抜粋、参照元識別子、スコアを含める。
- 表示順、折りたたみ、グルーピングなどのUI上の見せ方はDesktop Clientが調整する。

### DT-CMP-009 Conversation History Store

- digital-twin専用DynamoDBテーブルに会話履歴を保存する。
- 活動ログストアとは分離する。
- ユーザー別、セッション別に保存・取得できる設計とする。
- 履歴取得時も必ず `UserContext.user_id` を条件に含める。

### DT-CMP-010 Audit Logger

- 認証・認可失敗、検索実行、回答生成、履歴保存失敗、主要エラーを監査イベントとして記録する。
- 監査ログには、時刻、リクエストID、認証済みユーザー識別子、イベント種別、処理結果、エラー分類を含める。
- 認証トークン、全文プロンプト、検索結果全文、機微情報は不用意に記録しない。

---

## 4. 標準データモデル

| モデル | 主な項目 | 用途 |
|---|---|---|
| `UserContext` | `user_id`, `tenant_scope`, `auth_provider`, `request_id` | サーバー側で確定したユーザー文脈 |
| `ConversationMessageRequest` | `session_id`, `message`, `client_timestamp`, `options` | デスクトップアプリからの質問 |
| `ConversationResponse` | `answer`, `evidence`, `session_id`, `message_id`, `status`, `errors` | APIからの回答 |
| `SearchQuery` | `query_text`, `user_id`, `time_range`, `limit` | Retrieval Adapter内部の検索条件 |
| `SearchResult` | `content`, `timestamp`, `source_type`, `source_id`, `score`, `metadata` | RAG検索結果の標準モデル |
| `EvidenceItem` | `source_id`, `timestamp`, `source_type`, `excerpt`, `score`, `relation` | クライアント表示用根拠 |
| `ConversationRecord` | `user_id`, `session_id`, `message_id`, `question`, `answer`, `evidence`, `model`, `created_at`, `status` | 会話履歴保存モデル |
| `AuditEvent` | `request_id`, `user_id`, `event_type`, `result`, `error_class`, `created_at` | 監査ログモデル |

---

## 5. セキュリティ設計制約

- 認可に使う `user_id` は必ずCognitoトークン由来とする。
- RAG検索、会話履歴、根拠表示、監査ログのすべてでユーザー境界を維持する。
- IAM、ストレージ設計、アプリケーション層の複数層で分離を強制する。
- API入力は型、長さ、形式、サイズを検証する。
- ログには認証トークン、全文プロンプト、機微情報を不用意に出力しない。

---

## 6. PBT引き継ぎ候補

- 任意のリクエストに対して、検索条件と履歴取得条件には認証済み `user_id` が含まれる。
- クライアント指定の `user_id` が存在しても、サーバー側の `UserContext.user_id` が優先される。
- `SearchResult` から `EvidenceItem` への変換で参照元識別子と時刻が失われない。
- `ConversationMessageRequest` と `ConversationResponse` はシリアライズ後も必須項目を保持する。
