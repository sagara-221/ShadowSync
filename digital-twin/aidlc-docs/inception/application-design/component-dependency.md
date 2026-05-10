# コンポーネント依存関係: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計  
**バージョン**: 1.0  
**日付**: 2026-05-10

---

## 1. 依存方向の原則

- UIはConversation APIに依存するが、AWS内部サービスやRAG検索基盤の詳細には依存しない。
- Conversation Orchestratorは処理の統括を行うが、S3 VectorsやBedrockのSDK詳細には直接依存しない。
- Retrieval Adapter、Answer Generator、Conversation History Storeが外部サービス詳細を局所化する。
- `UserContext` はユーザー固有データアクセスの必須入力とする。

---

## 2. 依存一覧

| 依存元 | 依存先 | 依存内容 |
|---|---|---|
| Desktop Client | Auth Client | ログイン状態、アクセストークン取得 |
| Desktop Client | Conversation API | 質問送信、回答取得、履歴取得 |
| Conversation API | User Context Resolver | 認証トークン検証、`UserContext` 解決 |
| Conversation API | Conversation Orchestrator | 会話処理委譲 |
| Conversation Orchestrator | Conversation History Store | 直近履歴取得、履歴保存 |
| Conversation Orchestrator | Retrieval Adapter | RAG検索 |
| Conversation Orchestrator | Answer Generator | 回答生成 |
| Conversation Orchestrator | Evidence Formatter | 根拠表示モデル生成 |
| Conversation Orchestrator | Audit Logger | 監査イベント記録 |
| Retrieval Adapter | data-accumulation検索基盤 | S3 Vectors検索 |
| Answer Generator | Amazon Bedrock | Nova系モデル呼び出し |
| Conversation History Store | DynamoDB | 会話履歴の永続保存 |
| Auth Client | Amazon Cognito User Pool | 認証、トークン取得 |
| Audit Logger | AWSログ基盤 | 監査ログ出力 |

---

## 3. 外部依存

| 外部サービス | 用途 | 隠蔽するコンポーネント |
|---|---|---|
| Amazon Cognito User Pool | ユーザー認証 | Auth Client, User Context Resolver |
| API Gateway | REST API公開 | Conversation API |
| AWS Lambda | Conversation Service実行 | Conversation API配下 |
| data-accumulation S3 Vectors連携 | 活動ログRAG検索 | Retrieval Adapter |
| Amazon Bedrock Nova系モデル | 回答生成 | Answer Generator |
| DynamoDB | 会話履歴保存 | Conversation History Store |
| AWSログ基盤 | 監査ログ | Audit Logger |

---

## 4. 依存マトリクス

| コンポーネント | Client | Auth | API | Context | Orchestrator | Retrieval | Answer | Evidence | History | Audit |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Desktop Client | - | uses | uses | - | - | - | - | - | - | - |
| Auth Client | - | - | - | - | - | - | - | - | - | - |
| Conversation API | - | - | - | uses | uses | - | - | - | - | uses |
| User Context Resolver | - | - | - | - | - | - | - | - | - | uses |
| Conversation Orchestrator | - | - | - | uses | - | uses | uses | uses | uses | uses |
| Retrieval Adapter | - | - | - | uses | - | - | - | - | - | uses |
| Answer Generator | - | - | - | - | - | - | - | - | - | uses |
| Evidence Formatter | - | - | - | - | - | - | - | - | - | - |
| Conversation History Store | - | - | - | uses | - | - | - | - | - | uses |
| Audit Logger | - | - | - | - | - | - | - | - | - | - |

---

## 5. 禁止する依存

| 禁止依存 | 理由 |
|---|---|
| Desktop Client -> S3 Vectors | ユーザー分離と検索仕様をクライアントに漏らさないため |
| Desktop Client -> DynamoDB | 会話履歴の認可境界をバックエンドに集約するため |
| Desktop Client -> Bedrock | モデル設定、プロンプト、機微情報をクライアントに持たせないため |
| Conversation Orchestrator -> S3 Vectors SDK詳細 | 検索基盤の変更影響をRetrieval Adapterに閉じ込めるため |
| Conversation Orchestrator -> Bedrock SDK詳細 | モデル差し替え影響をAnswer Generatorに閉じ込めるため |
| 任意コンポーネント -> クライアント指定 `user_id` | 認可境界の改ざんを防ぐため |

---

## 6. セキュリティ境界

| 境界 | 強制内容 |
|---|---|
| Desktop Client / Conversation API | HTTPS、Authorizationヘッダー、入力検証 |
| Conversation API / User Context Resolver | トークン検証、`UserContext` 確定 |
| User Context / Retrieval Adapter | `user_id` 分離条件の必須化 |
| User Context / Conversation History Store | ユーザー別・セッション別取得条件の必須化 |
| Orchestrator / Audit Logger | 機微情報を除去した監査イベントのみ記録 |

---

## 7. 変更容易性

- UIフレームワークをPySide6から別Python GUIへ変更する場合、影響はDesktop Clientに閉じる。
- S3 Vectors連携方式が変わる場合、影響はRetrieval Adapterに閉じる。
- LLMモデルをNova系から別モデルへ切り替える場合、影響はAnswer Generatorに閉じる。
- 会話履歴保存先をDynamoDB以外に変更する場合、影響はConversation History Storeに閉じる。
