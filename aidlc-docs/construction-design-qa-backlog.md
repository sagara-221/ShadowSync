# Construction設計QAバックログ

**作成日**: 2026-05-10  
**目的**: Inceptionで決め切る必要はないが、ConstructionのFunctional Design / Infrastructure Designで回答が必要な質問を保管する。  
**運用方針**: 各子AI-DLCの該当ステージ開始時に、このバックログから必要な質問だけを取り出して回答・設計へ反映する。

---

## 0. Inception完了後のConstruction移行タスク

以下はInceptionフェーズの残作業ではなく、Constructionで具体化するためのタスクとして扱う。

| ID | タスク | 扱うステージ |
|---|---|---|
| CT-CQ-01 | 親AI-DLCの共通インターフェース文書を参照し、各子AI-DLCのFunctional Design冒頭で送受信スキーマ契約を確定する | Functional Design |
| CT-CQ-02 | 旧検討履歴や確認メモはConstruction実装判断の参考情報として扱い、提出用Inception成果物の正本には含めない | Construction開始時 |
| CT-CQ-03 | 子AI-DLCのステータスはInception完了として扱い、以降の詳細検討事項はConstructionタスクとして進捗管理する | Construction開始時 |
| CT-CQ-04 | Presigned URL、RAG同期、Notion出力、認証フローなどの詳細QAは、本バックログから各子AI-DLCの設計質問へ展開する | Functional / Infrastructure Design |

---

## 1. data-accumulation向け

| ID | 質問 | 扱うステージ |
|---|---|---|
| DA-CQ-01 | Chrome拡張の `html_snippet` を送信するか、送信する場合の最大サイズはいくつか | Functional Design |
| DA-CQ-02 | `html_snippet` がDynamoDB / IoT Core制限を超える場合、S3退避・切り詰め・保存しない、どれを採用するか | Functional Design |
| DA-CQ-03 | Presigned URLの有効期限を何分または何時間にするか | Functional / Infrastructure Design |
| DA-CQ-04 | S3 object keyをクライアント生成、Lambda生成、またはLambda強制生成のどれにするか | Functional Design |
| DA-CQ-05 | IoT CoreからDynamoDBまでの取り込み経路を、Lambda直接、Kinesis、SQS、段階移行のどれにするか | Infrastructure Design |
| DA-CQ-06 | DynamoDBからS3 Vectors / Bedrock Knowledge Baseへの同期方式を、Streams、EventBridge、日次バッチ、後回しのどれにするか | Functional / Infrastructure Design |
| DA-CQ-07 | RAGのマルチテナント分離を、ユーザー別リソース、namespace、メタデータフィルタ、単一ユーザーPoCのどれにするか | Functional / Infrastructure Design |
| DA-CQ-08 | X.509証明書と `user_id/device_id` の対応を、IoT Thing属性、DynamoDB対応表、ポリシー制約、手動設定のどれで管理するか | Infrastructure Design |

---

## 2. logger/ss-tool向け

| ID | 質問 | 扱うステージ |
|---|---|---|
| SS-CQ-01 | IoT Core Request/Responseのレスポンス待機を同期処理、非同期処理、URLプールのどれにするか | Functional Design |
| SS-CQ-02 | S3アップロード成功後にMQTTメタデータ送信が失敗した場合、再送、マージ、孤児削除、許容のどれにするか | Functional Design |
| SS-CQ-03 | ローカルキューの重複送信とDynamoDB冪等性をどう担保するか | Functional Design |

---

## 3. logger/osapi / logger/chrome-extension向け

| ID | 質問 | 扱うステージ |
|---|---|---|
| LG-CQ-01 | OS APIの `event_type` を現行3種で固定するか、簡略化するか | Code Generation前 |
| LG-CQ-02 | Chrome拡張のIndexedDB再送時の順序と重複対策をどうするか | Code Generation |
| LG-CQ-03 | Chrome拡張のOptions Pageでどの設定値を必須検証するか | Code Generation |

---

## 4. daily-log向け

| ID | 質問 | 扱うステージ |
|---|---|---|
| DL-CQ-01 | 日報生成タイミングをユーザータイムゾーン翌日00:05、01:00、06:00、固定時刻起動のどれにするか | Functional / Infrastructure Design |
| DL-CQ-02 | 日報生成失敗時または手動再生成時の再実行方式をどうするか | Functional Design |
| DL-CQ-03 | Notion日報ページの識別プロパティをどうするか | Functional Design |
| DL-CQ-04 | LLMプロンプト入力契約とログに残さない情報の境界をどうするか | Functional / NFR Design |

---

## 5. digital-twin向け

| ID | 質問 | 扱うステージ |
|---|---|---|
| DT-CQ-01 | data-accumulationのRAG I/F確定後に進めるか、Retrieval Adapterを仮I/Fで先行するか | Functional Design |
| DT-CQ-02 | 回答根拠として最低限表示するメタデータをどこまで含めるか | Functional Design |
| DT-CQ-03 | デスクトップアプリのCognito認証をHosted UI + PKCE、アプリ内フォーム、固定ユーザーPoCのどれにするか | Functional / Infrastructure Design |
| DT-CQ-04 | 10秒以内応答目標に対する計測点、タイムアウト、フォールバックをどう設計するか | NFR Design |
