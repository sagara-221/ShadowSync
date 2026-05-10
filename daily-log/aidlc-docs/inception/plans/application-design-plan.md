# Application Design Plan

## Purpose

`daily-log` の要件とユーザーストーリーを、アプリケーションレベルのコンポーネント、サービス、依存関係、主要インターフェースに変換する。

## Context

- 対象システム: ShadowSync Daily Log System
- 実行環境: AWS Lambda / EventBridge
- 実装言語: Python
- 入力: `data-accumulation` の DynamoDB 正規化済みアクティビティ
- 出力: Notion 日報ページ
- 外部依存: DynamoDB, Secrets Manager, Bedrock, Notion API, CloudWatch Logs
- 有効な拡張ルール: Security Baseline, Property-Based Testing

## Application Design Checklist

- [x] Read `aidlc-docs/inception/requirements/requirements.md`
- [x] Read `aidlc-docs/inception/user-stories/stories.md`
- [x] Identify major application components
- [x] Define high-level responsibilities and interfaces for each component
- [x] Define component method signatures and input/output types
- [x] Define service orchestration patterns
- [x] Define dependency relationships and communication patterns
- [x] Generate `aidlc-docs/inception/application-design/components.md`
- [x] Generate `aidlc-docs/inception/application-design/component-methods.md`
- [x] Generate `aidlc-docs/inception/application-design/services.md`
- [x] Generate `aidlc-docs/inception/application-design/component-dependency.md`
- [x] Generate consolidated `aidlc-docs/inception/application-design/application-design.md`
- [x] Validate design completeness and consistency

## Preliminary Component Candidates

以下は現時点の候補であり、回答内容に基づいて確定する。

- **Schedule Handler**: EventBridge から起動され、対象日・対象ユーザーの処理を開始する。
- **User Configuration Provider**: ユーザーのタイムゾーン、Notion 設定、Bedrock 利用設定を取得する。
- **Date Range Resolver**: ユーザーのタイムゾーンから対象日の UTC 範囲を計算する。
- **Activity Repository**: DynamoDB から対象ユーザー・対象日範囲の正規化済みアクティビティを取得する。
- **Daily Log Generator**: タイムライン、主要トピック、要約の生成を統括する。
- **LLM Summarization Client**: Bedrock 呼び出しを担当する。
- **Notion Publisher**: Notion API を通じて日報ページを作成する。
- **Failure Logger / Audit Logger**: 構造化ログと失敗情報を出力する。

## Questions

以下の `[Answer]:` に選択肢の文字を記入してください。どの選択肢にも合わない場合は `X` を選び、具体的な希望を書いてください。

---

## Q1. コンポーネント分割の粒度
Application Design では、どの粒度でコンポーネントを分割しますか？

A) 粗め: Handler、Data Access、Generator、Publisher 程度にまとめる  
B) 中程度: 日付判定、設定取得、データ取得、LLM、Notion、ログを個別コンポーネントに分ける  
C) 細かめ: 各処理ステップや外部APIごとに小さく分け、テスト対象を明確にする  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q2. サービス層の中心
日報生成全体を統括するサービスは、どの形にしますか？

A) `DailyLogService` が全体のユースケースを直接オーケストレーションする  
B) `DailyLogWorkflowService` が処理順序を管理し、各ドメインサービスへ委譲する  
C) Lambda handler が軽いオーケストレーションを持ち、個別コンポーネントを直接呼ぶ  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q3. LLM 呼び出しの抽象化
Bedrock 等の LLM 呼び出しは、どのように扱いますか？

A) Bedrock 専用クライアントとして設計する  
B) `LLMClient` インターフェースを設け、初期実装を Bedrock にする  
C) `DailyLogGenerator` の内部実装として扱い、独立コンポーネントにはしない  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q4. Notion 出力の抽象化
Notion 連携は、どのように扱いますか？

A) Notion 専用 `NotionPublisher` として設計する  
B) `ReportPublisher` インターフェースを設け、初期実装を Notion にする  
C) 初期版では Lambda handler 内から直接 Notion API を呼ぶ  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q5. ユーザー設定の取得元
ユーザーのタイムゾーン、Notion Database ID、Notion token の参照方法はどの前提で設計しますか？

A) DynamoDB にユーザー設定を保存し、Notion token は Secrets Manager に保存する  
B) すべて Secrets Manager に保存する  
C) 初期版では環境変数に設定し、後で DynamoDB / Secrets Manager に移行する  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q6. 日報出力モデル
Notion に渡す前の日報データは、どのようなモデルとして設計しますか？

A) `DailyReport` ドメインモデルを作り、Notion 変換は Publisher 側で行う  
B) 最初から Notion block / page property に近い構造で組み立てる  
C) 辞書型の汎用データとして持ち、必要に応じて変換する  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q7. エラー処理の境界
外部 API 失敗時のエラー処理は、どのコンポーネントが主に担当しますか？

A) 各外部クライアントが例外を正規化し、オーケストレーション層が失敗扱いを決める  
B) すべてオーケストレーション層で try/catch し、外部クライアントは薄く保つ  
C) Lambda handler で最終的な例外処理とログ出力をまとめる  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q8. PBT を意識した設計境界
PBT 対象の純粋ロジックは、どのように切り出しますか？

A) Date Range Resolver、Activity Filter、Report Model Builder を純粋関数寄りにする  
B) Date Range Resolver のみ純粋関数として切り出し、他は通常ユニットテストにする  
C) PBT 対象は Functional Design で改めて決めるため、Application Design では切り出さない  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q9. 依存方向
依存関係はどの方向に寄せますか？

A) ドメインロジックは AWS / Notion SDK に依存せず、外部依存は adapter 側へ寄せる  
B) 実装速度を優先し、必要なコンポーネントから AWS / Notion SDK を直接使う  
C) Lambda handler を中心に SDK 呼び出しを集約し、ドメインロジックは薄くする  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Approval Gate

全回答が埋まり、曖昧さが解消された後、Application Design の成果物生成に進む。
