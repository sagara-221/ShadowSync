# 作業単位依存関係: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - 作業単位生成  
**バージョン**: 1.0  
**日付**: 2026-05-10  
**ステータス**: 承認済み

---

## 1. 依存方針

作業単位は並行開発を想定して分割する。ただし、ユーザー文脈、API契約、検索結果モデル、根拠モデル、会話履歴モデルは共有契約として先に合意する必要がある。

---

## 2. 依存一覧

| 依存元 | 依存先 | 依存種別 | 内容 |
|---|---|---|---|
| DT-UOW-001 Desktop App | DT-UOW-002 Conversation API & Orchestrator | Runtime / Contract | REST API契約、認証ヘッダー、レスポンスモデル |
| DT-UOW-002 Conversation API & Orchestrator | DT-UOW-003 Retrieval Adapter | Runtime | RAG検索実行、SearchResult取得 |
| DT-UOW-002 Conversation API & Orchestrator | DT-UOW-004 Answer & Evidence | Runtime | 回答生成、根拠整形 |
| DT-UOW-002 Conversation API & Orchestrator | DT-UOW-005 Conversation History | Runtime | 直近履歴取得、会話履歴保存 |
| DT-UOW-002 Conversation API & Orchestrator | DT-UOW-006 Audit & Security Validation | Runtime / Cross-cutting | 監査イベント記録、セキュリティ制約 |
| DT-UOW-003 Retrieval Adapter | data-accumulation | External | S3 Vectors / Bedrock Knowledge Base系検索基盤 |
| DT-UOW-004 Answer & Evidence | Amazon Bedrock | External | Nova系モデル呼び出し |
| DT-UOW-005 Conversation History | DynamoDB | External | digital-twin専用会話履歴保存 |
| DT-UOW-006 Audit & Security Validation | 全作業単位 | Cross-cutting | `user_id` 分離、ログサニタイズ、PBT候補 |

---

## 3. 依存マトリクス

| 作業単位 | Desktop | API | Retrieval | Answer | History | Audit/Security |
|---|---:|---:|---:|---:|---:|---:|
| DT-UOW-001 Desktop App | - | uses | - | - | via API | must follow |
| DT-UOW-002 Conversation API & Orchestrator | serves | - | uses | uses | uses | uses |
| DT-UOW-003 Retrieval Adapter | - | called by | - | - | - | must follow |
| DT-UOW-004 Answer & Evidence | - | called by | consumes results | - | consumes context | must follow |
| DT-UOW-005 Conversation History | via API | called by | - | provides context | - | must follow |
| DT-UOW-006 Audit & Security Validation | constrains | constrains | constrains | constrains | constrains | - |

---

## 4. 推奨開発順序

並行開発を想定しつつ、契約定義は先行させる。

1. 共有モデル契約を確定する。
   - `UserContext`
   - `ConversationMessageRequest`
   - `ConversationResponse`
   - `SearchQuery`
   - `SearchResult`
   - `EvidenceItem`
   - `ConversationRecord`
   - `AuditEvent`
2. DT-UOW-002 Conversation API & OrchestratorのAPI契約を確定する。
3. DT-UOW-001 Desktop AppはAPIモック前提で進める。
4. DT-UOW-003 Retrieval Adapter、DT-UOW-004 Answer & Evidence、DT-UOW-005 Conversation Historyを並行して進める。
5. DT-UOW-006 Audit & Security Validationは横断条件として各単位へ適用し、監査イベント仕様を整える。

---

## 5. 並行開発の境界

| 並行可能な作業 | 前提 |
|---|---|
| Desktop AppとConversation API | API契約とレスポンスモデルを固定する |
| Retrieval AdapterとAnswer & Evidence | `SearchResult` モデルを固定する |
| Conversation HistoryとAnswer & Evidence | 会話文脈モデルを固定する |
| Audit & Security Validationと各機能単位 | `UserContext` と `AuditEvent` モデルを固定する |

---

## 6. 外部依存

| 外部依存 | 関連作業単位 | 注意点 |
|---|---|---|
| Amazon Cognito User Pool | DT-UOW-001, DT-UOW-002 | Desktop側ログインとサーバー側トークン検証を分ける |
| data-accumulation検索基盤 | DT-UOW-003 | S3 Vectors仕様をRetrieval Adapter内に閉じる |
| Amazon Bedrock Nova系モデル | DT-UOW-004 | モデル設定、プロンプト、タイムアウトをAnswer側へ閉じる |
| DynamoDB | DT-UOW-005 | 活動ログとは別のdigital-twin専用会話履歴テーブル |
| AWSログ基盤 | DT-UOW-006 | 機微情報を含めない監査イベントだけを記録する |

---

## 7. 禁止依存

| 禁止依存 | 理由 |
|---|---|
| Desktop App -> S3 Vectors | 検索基盤の詳細とユーザー分離責務をクライアントに漏らさないため |
| Desktop App -> DynamoDB | 会話履歴の認可境界をバックエンドに集約するため |
| Desktop App -> Bedrock | モデル設定やプロンプトをローカルに持たせないため |
| Answer & Evidence -> data-accumulation直接 | 検索仕様はRetrieval Adapterで隠蔽するため |
| 任意作業単位 -> クライアント指定 `user_id` | 認可境界の改ざんを防ぐため |

---

## 8. リスクと対策

| リスク | 影響 | 対策 |
|---|---|---|
| S3 Vectors / Knowledge Base仕様変更 | 検索連携の変更 | Retrieval Adapter内に閉じ込める |
| API契約の揺れ | Desktop Appとの結合不良 | 共有モデルと契約テストを先に定義する |
| `user_id` 分離漏れ | 他ユーザーデータ混入 | UserContext必須化、PBT候補、監査イベントで検証する |
| 履歴保存失敗と回答生成失敗の混同 | ユーザー表示が曖昧になる | Orchestratorで失敗分類を分ける |
| ログへの機微情報混入 | セキュリティ事故 | Audit Loggerでサニタイズを必須化する |
