# サービス設計: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計  
**バージョン**: 1.0  
**日付**: 2026-05-10

---

## 1. サービス境界

| サービス | 主なコンポーネント | 境界 |
|---|---|---|
| Desktop App Service | Desktop Client, Auth Client | Windowsローカルアプリ。UI、認証操作、API呼び出しを担当する |
| Conversation Service | Conversation API, User Context Resolver, Conversation Orchestrator | AWS Lambda中心のバックエンドAPI。会話処理の入口 |
| Retrieval Service | Retrieval Adapter | data-accumulationのRAG検索基盤を隠蔽する |
| Answer Generation Service | Answer Generator | Bedrock Nova系モデル呼び出しを隠蔽する |
| Evidence Service | Evidence Formatter | 検索結果を標準根拠モデルに整形する |
| Conversation History Service | Conversation History Store | digital-twin専用DynamoDBテーブルを扱う |
| Audit Service | Audit Logger | 監査イベントを記録する |

---

## 2. Desktop App Service

### 役割

- ユーザーが質問、回答確認、根拠確認、履歴確認を行う画面を提供する。
- Cognito認証を開始し、取得したトークンをAPI呼び出しに使用する。
- REST APIのレスポンスを画面表示へ変換する。

### 設計判断

- Windows対応を前提に、Python + PySide6を第一候補とする。
- Desktop App ServiceはAWSサービスを直接操作しない。Conversation APIを通じて処理する。
- 根拠表示の並び替え、折りたたみ、表示密度の調整はクライアント側で扱う。

---

## 3. Conversation Service

### 役割

- 認証済みRESTリクエストを受け付ける。
- 入力検証、ユーザー文脈解決、検索、回答生成、根拠整形、履歴保存、監査を統括する。
- 成功時も失敗時も、クライアントに安定したレスポンスモデルを返す。

### 主要API候補

| API | メソッド | 目的 |
|---|---|---|
| `/conversation/messages` | POST | 質問を送信し、回答と根拠を取得する |
| `/conversation/sessions` | GET | 会話セッション一覧を取得する |
| `/conversation/sessions/{session_id}` | GET | セッション内の会話履歴を取得する |
| `/health` | GET | APIの疎通確認を行う |

### 設計判断

- 初期はREST APIを基本にする。
- レスポンスモデルは、将来ストリーミング応答に移行しても再利用しやすいよう、メッセージ単位と根拠単位を分離する。
- `user_id` はAPI入力から受け取らず、User Context Resolverで確定する。

---

## 4. Retrieval Service

### 役割

- data-accumulationが構築するS3 Vectorsベースの検索基盤に問い合わせる。
- `user_id` 分離条件を必ず付与する。
- 検索結果を `SearchResult` に標準化する。

### 設計判断

- S3 Vectors固有のパラメータ、メタデータフィルタ、レスポンス形式はRetrieval Adapter内に閉じ込める。
- Conversation Orchestrator、Answer Generator、Desktop ClientはS3 Vectorsの詳細を知らない。
- data-accumulation側の仕様変更に備え、境界は `SearchQuery` と `SearchResult` に限定する。

---

## 5. Answer Generation Service

### 役割

- Bedrock Nova系モデルを使って日本語回答を生成する。
- 質問、直近会話文脈、検索結果をプロンプトへ組み込む。
- 根拠不足時は断定を避ける回答を生成する。

### 設計判断

- モデル固有設定はAnswer Generatorに閉じ込める。
- 会話履歴由来の文脈と活動ログ由来の根拠をプロンプト内で区別する。
- 生成結果には、使用モデル、処理結果、根拠不足フラグを付与できるようにする。

---

## 6. Evidence Service

### 役割

- `SearchResult` を `EvidenceItem` へ変換する。
- 根拠の時刻、ソース種別、関連抜粋、参照元識別子、スコアを標準化する。
- クライアントが表示調整できるよう、表示モデルを安定化させる。

### 設計判断

- バックエンドは標準モデルまで整形する。
- 表示順や折りたたみはDesktop Clientに任せる。
- 根拠がない場合も、根拠不足を表す状態を明示する。

---

## 7. Conversation History Service

### 役割

- 会話履歴をdigital-twin専用DynamoDBテーブルに保存する。
- 活動ログとは分離して保存する。
- 後続質問の文脈として直近履歴を提供する。

### データ境界

- パーティション設計はユーザー別、セッション別の取得を前提にする。
- 履歴保存・取得では必ず `UserContext.user_id` を条件に含める。
- 履歴保存失敗は回答生成失敗と区別して扱う。

---

## 8. Audit Service

### 役割

- 認証・認可失敗、検索実行、回答生成、履歴保存失敗を監査できる形で記録する。
- リクエストID、時刻、認証済みユーザー識別子、イベント種別、処理結果、エラー分類を保持する。
- 機微情報のログ混入を防ぐ。

### 初期イベント候補

| イベント | 発生タイミング | 機微情報制御 |
|---|---|---|
| `AUTH_FAILURE` | トークン不正または期限切れ | トークン本体は記録しない |
| `AUTHZ_DENIED` | 権限違反 | 参照対象の詳細本文は記録しない |
| `RETRIEVAL_EXECUTED` | RAG検索実行 | 検索結果全文は記録しない |
| `ANSWER_GENERATED` | 回答生成完了 | 全文プロンプトは記録しない |
| `HISTORY_SAVE_FAILED` | 会話履歴保存失敗 | 質問・回答全文は原則記録しない |
| `REQUEST_FAILED` | 主要処理失敗 | スタックトレースはユーザー応答に含めない |

---

## 9. 標準処理フロー

1. Desktop ClientがAuth Clientからアクセストークンを取得する。
2. Desktop Clientが質問をConversation APIへPOSTする。
3. Conversation APIが入力を検証する。
4. User Context Resolverがトークンから `UserContext` を作る。
5. Conversation Orchestratorが直近会話履歴を取得する。
6. Retrieval Adapterが `user_id` 分離条件付きでRAG検索を実行する。
7. Answer Generatorが検索結果と履歴文脈から回答を生成する。
8. Evidence Formatterが根拠モデルを作る。
9. Conversation History Storeが会話履歴を保存する。
10. Audit Loggerが検索、生成、失敗イベントを記録する。
11. Conversation APIが回答、根拠、状態をDesktop Clientへ返す。

---

## 10. 失敗時フロー

| 失敗種別 | 応答方針 | 監査 |
|---|---|---|
| 認証失敗 | 再認証を要求し、データを返さない | `AUTH_FAILURE` |
| 認可失敗 | 権限不足として拒否し、対象データを返さない | `AUTHZ_DENIED` |
| 入力不正 | 修正可能なエラーとして返す | 必要に応じて `REQUEST_FAILED` |
| 検索失敗 | 根拠なしの断定回答を避ける | `REQUEST_FAILED` |
| 根拠不足 | 不足情報を明示し、断定を避ける | `ANSWER_GENERATED` |
| LLM失敗 | 回答生成失敗として再試行可能性を示す | `REQUEST_FAILED` |
| 履歴保存失敗 | 回答表示が可能なら表示し、履歴保存失敗を通知する | `HISTORY_SAVE_FAILED` |
