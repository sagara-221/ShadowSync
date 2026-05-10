# 作業単位定義: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - 作業単位生成  
**バージョン**: 1.0  
**日付**: 2026-05-10  
**ステータス**: 承認済み

---

## 1. 分解方針

digital-twinは、デスクトップ、API、検索、生成、履歴、監査の機能別に作業単位へ分解する。複数人または複数エージェントが並行して進められるよう、責務、入出力、依存関係を明確にする。

本ドキュメントはINCEPTION成果物であり、実装、詳細設計、テスト作成、インフラ構築は行わない。

---

## 2. 作業単位一覧

| 作業単位ID | 作業単位 | 主な責務 | 主な配置候補 |
|---|---|---|---|
| DT-UOW-001 | Desktop App | PySide6 UI、Auth Client、API呼び出し、回答・根拠・履歴表示 | `desktop/` |
| DT-UOW-002 | Conversation API & Orchestrator | REST API、入力検証、User Context Resolver、会話処理統括 | `backend/` |
| DT-UOW-003 | Retrieval Adapter | data-accumulation/S3 Vectors連携、検索条件、`user_id` 分離 | `backend/` |
| DT-UOW-004 | Answer & Evidence | Bedrock Nova回答生成、根拠整形、根拠不足時応答 | `backend/` |
| DT-UOW-005 | Conversation History | 専用DynamoDB会話履歴、セッション取得、履歴文脈 | `backend/` |
| DT-UOW-006 | Audit & Security Validation | 監査ログ、ログサニタイズ、横断セキュリティ検証、PBT候補 | `backend/`, `tests/` |

---

## 3. Greenfieldコード構成戦略

後続でコードを生成する場合、以下の大枠を候補とする。

```text
digital-twin/
  desktop/
    app/
    auth/
    api_client/
    views/
  backend/
    api/
    orchestrator/
    retrieval/
    answer/
    history/
    audit/
  shared/
    models/
    schemas/
    errors/
  tests/
    contract/
    integration/
    property/
  iac/
  aidlc-docs/
```

### 配置方針

- `desktop/`: Windows対応Python + PySide6アプリを置く。
- `backend/`: Lambda中心のConversation API関連実装を置く。
- `shared/`: APIリクエスト、レスポンス、検索結果、根拠、履歴、監査イベントの共通モデルを置く。
- `tests/`: 後続フェーズで契約テスト、統合テスト、PBT候補を整理する。
- `iac/`: 後続のインフラ設計・構築フェーズでAWSリソース定義を置く。

---

## 4. DT-UOW-001 Desktop App

### 目的

ユーザーがWindowsデスクトップ上で質問し、回答、根拠、会話履歴、認証状態を確認できるアプリケーション体験を提供する。

### 含むコンポーネント

- Desktop Client
- Auth Client
- API Client

### 責務

- PySide6による質問入力、回答表示、根拠表示、履歴表示。
- Cognitoログイン、ログアウト、セッション更新。
- Conversation APIへのAuthorizationヘッダー付きREST呼び出し。
- 認証切れ、応答中、エラー状態の表示。
- 根拠の並び替え、折りたたみ、表示密度の調整。

### 含めないもの

- Cognitoトークンのサーバー側検証。
- `user_id` による最終認可判断。
- S3 Vectors、Bedrock、DynamoDBへの直接アクセス。

### 主要API・インターフェース候補

- `POST /conversation/messages`
- `GET /conversation/sessions`
- `GET /conversation/sessions/{session_id}`

### 想定ディレクトリ

```text
desktop/
  app/
  auth/
  api_client/
  views/
```

### 対象ストーリー

- 主: DT-US-001, DT-US-002, DT-US-006, DT-US-007
- 関連: DT-US-003, DT-US-008

### テスト観点

- 未認証時に質問送信できない。
- 認証切れ時に再認証導線を出す。
- APIレスポンスの回答、根拠、エラーを表示できる。
- クライアントから認可用 `user_id` を送らない。

---

## 5. DT-UOW-002 Conversation API & Orchestrator

### 目的

デスクトップアプリからの質問を受け付け、認証済みユーザー文脈のもとで検索、回答生成、履歴保存、監査を統括する。

### 含むコンポーネント

- Conversation API
- User Context Resolver
- Conversation Orchestrator

### 責務

- REST APIエンドポイントの定義。
- API入力の型、長さ、形式、サイズ検証。
- Cognitoトークン検証と `UserContext` 確定。
- クライアント指定 `user_id` の無視。
- Retrieval、Answer & Evidence、Conversation History、Auditの呼び出し順序制御。
- 認証失敗、認可失敗、検索失敗、LLM失敗、履歴保存失敗の分類。

### 含めないもの

- PySide6 UI。
- S3 Vectors固有実装。
- Bedrockプロンプト詳細。
- DynamoDB会話履歴スキーマ詳細。

### 主要API・インターフェース候補

- `POST /conversation/messages`
- `GET /conversation/sessions`
- `GET /conversation/sessions/{session_id}`
- `GET /health`

### 想定ディレクトリ

```text
backend/
  api/
  orchestrator/
shared/
  models/
  errors/
```

### 対象ストーリー

- 主: DT-US-002, DT-US-003, DT-US-008
- 関連: DT-US-001, DT-US-004, DT-US-005, DT-US-007, DT-US-009

### テスト観点

- 無効トークンを拒否する。
- クライアント指定 `user_id` を認可判断に使わない。
- 入力不正を安全なエラーとして返す。
- 部分失敗時に安定したレスポンス形式を維持する。

---

## 6. DT-UOW-003 Retrieval Adapter

### 目的

data-accumulationが管理するS3 Vectors / Bedrock Knowledge Base系の検索基盤を隠蔽し、digital-twin内部へ標準化された検索結果を返す。

### 含むコンポーネント

- Retrieval Adapter
- SearchQueryモデル
- SearchResultモデル

### 責務

- `UserContext.user_id` に基づく検索分離条件の付与。
- data-accumulation検索基盤への問い合わせ。
- S3 Vectors固有レスポンスから `SearchResult` への変換。
- 検索失敗、タイムアウト、根拠不足候補の分類。
- data-accumulation側仕様変更の影響を局所化する。

### 含めないもの

- 回答生成。
- 根拠表示モデルへの最終整形。
- デスクトップ表示。
- 活動ログの収集、保存、ベクトル生成。

### 主要API・インターフェース候補

- `search(user_context, search_query) -> SearchResult[]`
- `apply_user_filter(user_context, search_query) -> SearchQuery`
- `map_results(external_results) -> SearchResult[]`

### 想定ディレクトリ

```text
backend/
  retrieval/
shared/
  models/
```

### 対象ストーリー

- 主: DT-US-004
- 関連: DT-US-005, DT-US-006, DT-US-008, DT-US-009

### テスト観点

- 任意の検索条件に認証済み `user_id` が付与される。
- 他ユーザーの検索条件を生成しない。
- 検索結果から時刻、ソース種別、参照元識別子、スコアが失われない。
- data-accumulation側の詳細が上位コンポーネントへ漏れない。

---

## 7. DT-UOW-004 Answer & Evidence

### 目的

検索結果と会話文脈からBedrock Nova系モデルで回答を生成し、ユーザーが確認できる標準根拠モデルを作る。

### 含むコンポーネント

- Answer Generator
- Evidence Formatter
- EvidenceItemモデル

### 責務

- 質問、直近会話履歴、検索結果をもとに回答生成プロンプトを構築する。
- Bedrock Nova系モデルを呼び出す。
- 活動ログ由来の根拠と会話履歴由来の文脈を区別する。
- `SearchResult` を `EvidenceItem` へ変換する。
- 根拠不足時は断定を避ける応答方針を適用する。

### 含めないもの

- RAG検索の実行。
- 会話履歴の永続保存。
- APIエンドポイント管理。

### 主要API・インターフェース候補

- `generate(question, history, search_results) -> GeneratedAnswer`
- `build_prompt(question, history, search_results) -> Prompt`
- `format_evidence(search_results) -> EvidenceItem[]`
- `classify_evidence_sufficiency(search_results) -> EvidenceStatus`

### 想定ディレクトリ

```text
backend/
  answer/
shared/
  models/
```

### 対象ストーリー

- 主: DT-US-005, DT-US-006
- 関連: DT-US-003, DT-US-004, DT-US-008, DT-US-009

### テスト観点

- 根拠がある場合に回答と根拠が対応する。
- 根拠不足時に断定を避ける。
- EvidenceItemに時刻、ソース種別、関連抜粋、参照元識別子が含まれる。
- プロンプトやログに機微情報を不用意に残さない。

---

## 8. DT-UOW-005 Conversation History

### 目的

digital-twin専用の会話履歴を活動ログとは分離して保存し、後続質問の文脈として利用できるようにする。

### 含むコンポーネント

- Conversation History Store
- ConversationRecordモデル

### 責務

- 会話履歴をdigital-twin専用DynamoDBテーブルへ保存する。
- ユーザー別、セッション別に履歴を取得する。
- 回答生成に使う直近会話文脈を提供する。
- 履歴保存失敗を回答生成失敗と区別する。
- 履歴取得時に必ず `UserContext.user_id` を条件に含める。

### 含めないもの

- 活動ログ本体の保存。
- RAG検索。
- デスクトップ上の履歴表示UI。

### 主要API・インターフェース候補

- `save_record(user_context, conversation_record) -> SaveResult`
- `get_session(user_context, session_id) -> ConversationRecord[]`
- `list_sessions(user_context) -> SessionSummary[]`
- `load_recent_context(user_context, session_id) -> ConversationContext`

### 想定ディレクトリ

```text
backend/
  history/
shared/
  models/
```

### 対象ストーリー

- 主: DT-US-007
- 関連: DT-US-003, DT-US-005, DT-US-008, DT-US-009

### テスト観点

- 履歴保存・取得に `UserContext.user_id` が必ず使われる。
- 他ユーザーのセッション履歴を取得しない。
- 保存した会話履歴を取得しても必須項目が保持される。
- 履歴保存失敗時に回答表示と失敗通知を分離できる。

---

## 9. DT-UOW-006 Audit & Security Validation

### 目的

監査ログ、ログサニタイズ、`user_id` 分離検証、PBT候補を横断的に扱い、各作業単位の受け入れ条件にもセキュリティ制約を反映する。

### 含むコンポーネント

- Audit Logger
- AuditEventモデル
- セキュリティ検証観点
- PBT候補整理

### 責務

- 認証・認可失敗、検索実行、回答生成、履歴保存失敗の監査イベントを定義する。
- 監査ログから認証トークン、全文プロンプト、検索結果全文、機微情報を除外する。
- 各作業単位へ `user_id` 分離条件を受け入れ条件として割り当てる。
- PBT候補を後続の機能設計へ引き継ぐ。
- Security Baselineの適用観点を整理する。

### 含めないもの

- 各機能の業務ロジック実装。
- インフラ監視アラートの詳細設計。

### 主要API・インターフェース候補

- `record_auth_failure(request_id, error_class)`
- `record_retrieval(user_context, retrieval_summary)`
- `record_generation(user_context, model_info, result)`
- `record_history_failure(user_context, error_class)`
- `sanitize(event_candidate) -> AuditEvent`

### 想定ディレクトリ

```text
backend/
  audit/
tests/
  property/
  contract/
```

### 対象ストーリー

- 主: DT-US-009
- 関連: DT-US-002, DT-US-004, DT-US-007, DT-US-008

### テスト観点

- 監査ログにトークンや全文プロンプトを含めない。
- 認可失敗時に対象データを返さない。
- 任意の検索・履歴取得で `user_id` 分離が維持される。
- シリアライズ後も監査イベントの必須項目が保持される。

---

## 10. 横断ルール

- すべてのユーザー固有データアクセスは `UserContext.user_id` に基づく。
- Desktop AppはAuthorizationヘッダーを送るが、認可判断に使う `user_id` を送らない。
- Retrieval AdapterとConversation Historyは、`UserContext` なしでユーザー固有データへアクセスしない。
- Answer & Evidenceは他ユーザーの根拠を表示しない前提で、入力済み検索結果のメタデータを保持する。
- Audit & Security Validationは独立作業単位として存在しつつ、各作業単位の受け入れ条件にも分散して適用する。

---

## 11. 後続フェーズへの引き継ぎ

CONSTRUCTIONへ進む場合は、各作業単位ごとに機能設計、非機能要件、非機能設計、インフラ設計、コード生成を進める。今回の依頼範囲ではCONSTRUCTIONへは進まない。
