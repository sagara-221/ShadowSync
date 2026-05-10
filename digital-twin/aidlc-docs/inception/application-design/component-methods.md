# コンポーネントメソッド設計: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計  
**バージョン**: 1.0  
**日付**: 2026-05-10

---

## 1. 目的

各コンポーネントが公開する主要操作を定義する。ここでは実装コード、詳細アルゴリズム、AWS SDKの具体呼び出しは定義しない。

---

## 2. Desktop Client

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `start_application` | なし | アプリ状態 | 初期画面、認証状態、履歴導線を準備する |
| `submit_question` | `session_id`, `message` | `ConversationResponse` | 質問をConversation APIへ送信する |
| `render_answer` | `ConversationResponse` | なし | 回答本文と状態を表示する |
| `render_evidence` | `EvidenceItem[]` | なし | 根拠を表示し、折りたたみや並び替えを行う |
| `show_history` | `session_id` | なし | 自分の会話履歴を表示する |
| `show_error` | `error_code`, `message` | なし | ユーザー向けエラーを表示する |

---

## 3. Auth Client

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `login` | 認証入力 | トークンセット | Cognitoログインを実行する |
| `logout` | なし | なし | ローカルセッションを終了する |
| `refresh_session` | リフレッシュトークン | トークンセット | セッションを更新する |
| `get_access_token` | なし | アクセストークン | API呼び出し用トークンを取得する |
| `is_authenticated` | なし | 真偽値 | 認証状態を判定する |

---

## 4. Conversation API

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `post_message` | `ConversationMessageRequest`, Authorizationヘッダー | `ConversationResponse` | 質問を受け付けて回答を返す |
| `get_session_history` | `session_id`, Authorizationヘッダー | `ConversationRecord[]` | 指定セッションの会話履歴を返す |
| `list_sessions` | Authorizationヘッダー | セッション一覧 | ユーザーの会話セッション一覧を返す |
| `health_check` | なし | ヘルス状態 | API疎通確認を行う |

---

## 5. User Context Resolver

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `resolve` | Authorizationヘッダー | `UserContext` | Cognitoトークンを検証し、信頼できるユーザー文脈を作る |
| `validate_scope` | `UserContext`, 操作種別 | 認可結果 | 操作に必要な権限を確認する |
| `reject_client_user_id` | リクエスト本文 | 検証結果 | クライアント指定 `user_id` を認可判断から排除する |

---

## 6. Conversation Orchestrator

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `handle_message` | `UserContext`, `ConversationMessageRequest` | `ConversationResponse` | 会話処理全体を統括する |
| `load_context` | `UserContext`, `session_id` | 直近会話履歴 | 回答生成に使う履歴文脈を取得する |
| `build_retrieval_query` | `UserContext`, 質問 | `SearchQuery` | 検索条件を組み立てる |
| `handle_partial_failure` | エラー種別, 中間結果 | `ConversationResponse` | 検索失敗、LLM失敗、履歴保存失敗を区別して応答を作る |

---

## 7. Retrieval Adapter

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `search` | `UserContext`, `SearchQuery` | `SearchResult[]` | ユーザー分離条件付きでRAG検索を実行する |
| `apply_user_filter` | `UserContext`, 検索条件 | 検索条件 | `user_id` 分離条件を付与する |
| `map_results` | S3 Vectors検索結果 | `SearchResult[]` | 外部検索結果を標準モデルへ変換する |

---

## 8. Answer Generator

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `generate` | 質問, 履歴, `SearchResult[]` | 生成回答 | Bedrock Nova系モデルで回答を生成する |
| `build_prompt` | 質問, 履歴, `SearchResult[]` | プロンプト | 活動ログ根拠と会話文脈を区別して構築する |
| `classify_evidence_sufficiency` | `SearchResult[]` | 判定結果 | 根拠不足かどうかを判定する |

---

## 9. Evidence Formatter

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `format` | `SearchResult[]` | `EvidenceItem[]` | 表示用根拠モデルへ変換する |
| `extract_excerpt` | `SearchResult` | 関連抜粋 | 回答根拠として見せる抜粋を作る |
| `normalize_source` | `SearchResult` | ソース情報 | ソース種別と参照元識別子を標準化する |

---

## 10. Conversation History Store

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `save_record` | `UserContext`, `ConversationRecord` | 保存結果 | 会話履歴をDynamoDBへ保存する |
| `get_session` | `UserContext`, `session_id` | `ConversationRecord[]` | ユーザー別・セッション別に履歴を取得する |
| `list_sessions` | `UserContext` | セッション一覧 | ユーザーのセッション一覧を取得する |
| `append_status` | `ConversationRecord`, 処理結果 | `ConversationRecord` | 失敗分類や生成モデル情報を付与する |

---

## 11. Audit Logger

| メソッド | 入力 | 出力 | 責務 |
|---|---|---|---|
| `record_auth_failure` | `request_id`, エラー分類 | なし | 認証・認可失敗を記録する |
| `record_retrieval` | `UserContext`, 検索結果概要 | なし | 検索実行イベントを記録する |
| `record_generation` | `UserContext`, モデル情報, 結果 | なし | 回答生成イベントを記録する |
| `record_history_failure` | `UserContext`, エラー分類 | なし | 履歴保存失敗を記録する |
| `sanitize` | イベント候補 | `AuditEvent` | トークン、全文プロンプト、機微情報を除去する |

---

## 12. 横断的不変条件

- 認可判断に使用する `user_id` は `UserContext` 由来のみとする。
- Retrieval AdapterとConversation History Storeは、`UserContext` なしではユーザー固有データへアクセスしない。
- Audit Loggerは機微情報の除去を通したイベントだけを記録する。
- Desktop Clientは表示責務を持つが、ユーザー分離の最終責任を持たない。
