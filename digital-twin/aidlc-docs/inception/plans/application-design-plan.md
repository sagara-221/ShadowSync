# アプリケーション設計計画

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - アプリケーション設計計画  
**状態**: 完了

---

## 1. 目的

digital-twinの主要コンポーネント、サービス境界、インターフェース、依存関係を設計する。詳細な業務ロジックや実装コードはCONSTRUCTIONフェーズの機能設計・コード生成で扱うため、本ステージでは高レベル設計に限定する。

---

## 2. 設計対象

要件とユーザーストーリーに基づき、以下を設計対象とする。

- デスクトップクライアント
- 認証・ユーザーコンテキスト
- Conversation API
- 会話オーケストレーション
- RAG検索連携
- 回答生成
- 根拠表示
- 会話履歴
- 監査・セキュリティイベント
- data-accumulation連携

---

## 3. 実行チェックリスト

- [x] 承認済み要件を確認する。
- [x] 承認済みユーザーストーリーとペルソナを確認する。
- [x] コンポーネント候補と責務境界を確定する。
- [x] サービス層のオーケストレーション方針を確定する。
- [x] コンポーネント間の通信方式と依存方向を整理する。
- [x] Security Baselineを設計制約として反映する。
- [x] PBTにつながる設計上の不変条件候補を補足として残す。
- [x] `aidlc-docs/inception/application-design/components.md` を生成する。
- [x] `aidlc-docs/inception/application-design/component-methods.md` を生成する。
- [x] `aidlc-docs/inception/application-design/services.md` を生成する。
- [x] `aidlc-docs/inception/application-design/component-dependency.md` を生成する。
- [x] `aidlc-docs/inception/application-design/application-design.md` を生成する。
- [x] 設計の完全性と一貫性を検証する。

---

## 4. 想定コンポーネント案

初期案として、以下のコンポーネント境界を想定する。

| コンポーネント | 主な責務 |
|---|---|
| Desktop Client | 質問入力、回答表示、根拠表示、履歴表示、認証UI |
| Auth Client | Cognitoログイン、トークン取得、セッション更新 |
| API Gateway / Conversation API | 認証済みリクエスト受付、入力検証、応答返却 |
| User Context Resolver | トークン由来の `user_id` 確定、クライアント指定 `user_id` の無視 |
| Conversation Orchestrator | 検索、回答生成、履歴保存、監査を統括 |
| Retrieval Adapter | data-accumulationのS3 Vectors検索連携、`user_id` 分離条件付与 |
| Answer Generator | Bedrock Novaによる回答生成、根拠不足時の応答制御 |
| Evidence Formatter | 参照元、時刻、ソース種別、関連抜粋の整形 |
| Conversation History Store | 会話履歴の永続保存、ユーザー別・セッション別取得 |
| Audit Logger | 監査ログ、認可失敗、検索・回答生成イベント記録 |

---

## 5. 計画質問

各質問の `[Answer]:` に選択肢の文字を記入してください。該当する選択肢がない場合は `X) その他` を選び、同じ行または直後に内容を書いてください。

### 質問1: デスクトップアプリ技術方針
デスクトップクライアントはどの前提で設計しますか？

A) Tauri + React/TypeScriptを前提にする  
B) Electron + React/TypeScriptを前提にする  
C) ネイティブmacOSアプリを前提にする  
D) 技術を固定せず、UI/API境界だけを設計する  
X) その他

[Answer]: X) Windows対応のPythonデスクトップアプリを前提にする。UIはPySide6またはCustomTkinter等を候補とし、バックエンドAPIとの境界はHTTPベースで設計する。


### 質問2: バックエンド実行形態
Conversation APIと関連サービスはどの実行形態を前提にしますか？

A) AWS Lambda中心のサーバーレスAPI  
B) ECS/FargateなどのコンテナAPI  
C) デスクトップアプリ内で直接AWSサービスを呼ぶ薄いバックエンド  
D) 現時点では固定せず、アプリケーション設計では論理サービス境界だけ定義する  
X) その他

[Answer]: A

### 質問3: Conversation APIの通信方式
デスクトップアプリとConversation APIの通信方式はどれを優先しますか？

A) REST API  
B) WebSocket API  
C) REST APIを基本にし、将来のストリーミング応答に備える  
D) 最初からストリーミング応答を前提にする  
X) その他

[Answer]: C

### 質問4: 会話履歴の保存先境界
会話履歴ストアはどのように設計しますか？

A) digital-twin専用のDynamoDBテーブルとして設計する  
B) data-accumulation側の既存ストレージに寄せる  
C) 初期はローカル保存、後でクラウド永続化する  
D) 保存先は未定とし、抽象インターフェースだけ定義する  
X) その他

[Answer]: A

### 質問5: RAG検索連携の境界
data-accumulationのS3 Vectors連携はどの境界で隠蔽しますか？

A) Retrieval Adapterに完全に閉じ込め、他コンポーネントは検索結果モデルだけ扱う  
B) Conversation OrchestratorがS3 Vectorsの詳細も知る  
C) Desktop Clientが検索条件の一部を構築する  
D) 現時点では固定せず、後続設計で決める  
X) その他

[Answer]: A

### 質問6: 根拠表示の責務分担
根拠の整形と表示の責務はどのように分けますか？

A) バックエンドで根拠表示モデルまで整形し、クライアントは表示に専念する  
B) バックエンドは生の検索結果を返し、クライアントで表示用に整形する  
C) バックエンドで標準モデルを返し、クライアントで表示順や折りたたみを調整する  
D) 後続のUI設計で決める  
X) その他

[Answer]: C

### 質問7: セキュリティ責務の配置
認証・認可・`user_id` 分離の責務はどこに置きますか？

A) User Context Resolverと各バックエンドサービスで一貫して強制する  
B) Conversation Orchestratorに集約する  
C) API Gateway認証に主に任せ、アプリケーション層は補助にする  
D) 後続のSecurity Baseline設計で決める  
X) その他

[Answer]: A

### 質問8: 監査ログの設計粒度
アプリケーション設計で監査ログをどの粒度まで扱いますか？

A) 認証・認可失敗、検索実行、回答生成、履歴保存失敗まで論理イベントを定義する  
B) 認証・認可失敗だけ定義する  
C) Audit Loggerコンポーネントの存在だけ定義し、イベント詳細は後続設計に回す  
D) 監査ログはインフラ設計で扱う  
X) その他

[Answer]: A

---

## 6. 承認ゲート

すべての `[Answer]:` が埋まった後、回答の曖昧さや矛盾を確認する。問題がなければ、この計画をもとに以下のアプリケーション設計成果物を生成する。

- `aidlc-docs/inception/application-design/components.md`
- `aidlc-docs/inception/application-design/component-methods.md`
- `aidlc-docs/inception/application-design/services.md`
- `aidlc-docs/inception/application-design/component-dependency.md`
- `aidlc-docs/inception/application-design/application-design.md`

---

## 7. 追加確認

### 追加質問1: PythonデスクトップUIフレームワーク
質問1の回答では「PySide6またはCustomTkinter等を候補」とあり、設計前提として複数案が残っています。アプリケーション設計ではどれを第一候補として固定しますか？

A) PySide6を第一候補にする  
B) CustomTkinterを第一候補にする  
C) Pythonデスクトップアプリ前提だけ固定し、UIフレームワークは後続設計で決める  
X) その他

[Answer]: A
