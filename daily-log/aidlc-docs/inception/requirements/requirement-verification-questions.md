# 要求仕様の確認事項 (Requirement Verification Questions)

ShadowSync の `daily-log` 日誌作成システムについて、要件を明確化するための質問です。
各質問の `[Answer]:` の後に、選択肢の文字を記入してください。どの選択肢にも合わない場合は `X` を選び、同じ行または次の行に具体的な希望を書いてください。

---

## Q1. 日報生成の対象ユーザー
日次バッチでは、どのユーザーの日報を生成しますか？

A) 登録済みの全ユーザーを毎日一括処理する  
B) 指定された `user_id` のみを処理する手動実行も可能にし、定期実行では全ユーザーを処理する  
C) まずは単一ユーザーのみを対象にする  
X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Q2. 対象日の判定基準
1日分のログ範囲は、どのタイムゾーンと日付境界で判定しますか？

A) ユーザーごとのタイムゾーン設定に従う  
B) システム固定で Asia/Tokyo の 00:00:00 から 23:59:59 まで  
C) UTC の 00:00:00 から 23:59:59 まで  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q3. 日報生成の実行タイミング
EventBridge 等による定期実行は、いつ行う想定ですか？

A) 毎日 23:55 に当日分を生成する  
B) 翌日早朝に前日分を生成する  
C) 定期実行に加えて、任意の日付を再生成できる手動実行を用意する  
X) Other (please describe after [Answer]: tag below)

[Answer]: X:2.対象日の判定基準で問われた日付を超えた後に自動的に作成される

## Q4. 入力データソース
`data-accumulation` から読み取る主なデータソースはどれですか？

A) DynamoDB の正規化済みアクティビティのみ  
B) DynamoDB の正規化済みアクティビティ + S3 の元データ参照  
C) S3 Vector 等の検索用データも併用し、日報の要約精度を高める  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q5. 日報に含める内容
Notion に出力する日報には、どの粒度の情報を含めますか？

A) 時系列タイムライン、1日の要約、主要トピックのみ  
B) 上記に加えて、プロジェクト別・アプリ別・Webサイト別の集計を含める  
C) 上記に加えて、集中時間、割り込み、改善提案などの分析コメントを含める  
X) Other (please describe after [Answer]: tag below)

[Answer]: X：とりあえずはAで良いがカスタムできるようにしてほしい

## Q6. LLM 利用方針
日報生成で Bedrock 等の LLM をどの程度利用しますか？

A) ルールベースで集計し、LLM は要約文生成のみ使用する  
B) タイムライン分類、トピック抽出、要約文生成に LLM を使用する  
C) コスト削減を優先し、初期版では LLM を使用しない  
X) Other (please describe after [Answer]: tag below)

[Answer]:X:2.対象日の判定を利用して絞り込みを行い、日報作成を行う

## Q7. Notion 連携方式
Notion への出力はどのような構成にしますか？

A) ユーザーごとに Notion Database ID と Integration Token を設定する  
B) システム共通の Notion Database に `user_id` で分離して出力する  
C) 初期版では単一ユーザー・単一 Notion Database のみ対応する  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q8. Notion ページの再生成・重複防止
同じ対象日の Notion 日報が既に存在する場合、どう扱いますか？

A) 既存ページを更新する  
B) 新しいページを作成し、古いページは残す  
C) 既存ページがある場合は処理をスキップする  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q9. Lambda の 15分制限を超える場合の構成
ユーザー数やログ量が増えて Lambda 単体で完了しない場合、どの構成を優先しますか？

A) Step Functions でユーザー単位・日付単位に分割して処理する  
B) EventBridge でユーザーごとのジョブを分散起動する  
C) まずは Lambda 単体で実装し、必要になったら分割する  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q10. 失敗時の扱いと再実行
Notion API、データ取得、LLM 呼び出しなどが失敗した場合の扱いはどうしますか？

A) 自動リトライし、最終的に DLQ または失敗記録に残して手動再実行する  
B) 失敗したユーザーだけを記録し、次回実行時に再処理する  
C) エラーログのみ残し、日報生成は失敗扱いで終了する  
X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Q11. IaC ツール
インフラストラクチャ定義にはどのツールを使いますか？

A) CloudFormation  
B) AWS SAM  
C) AWS CDK  
D) Terraform  
X) Other (please describe after [Answer]: tag below)

[Answer]: X:後回しでよい

## Q12. 開発言語・実行環境
Lambda 実装の主な言語・ランタイムはどれを想定しますか？

A) Python  
B) TypeScript / Node.js  
C) Go  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q13. テスト方針
日誌作成システムでは、どの範囲のテストを必須にしますか？

A) ユニットテストのみ  
B) ユニットテスト + Notion API をモックした統合テスト  
C) ユニットテスト + AWS 開発環境での統合テスト + Notion 連携確認  
X) Other (please describe after [Answer]: tag below)

[Answer]: X:後回しでよい

## Q14. Security Extensions
このプロジェクトで Security Baseline 拡張ルールを適用しますか？

A) Yes - 本番品質の制約として SECURITY ルールを強制する  
B) No - PoC、プロトタイプ、実験用途として SECURITY ルールをスキップする  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q15. Property-Based Testing Extension
このプロジェクトで Property-Based Testing (PBT) 拡張ルールを適用しますか？

A) Yes - ビジネスロジック、データ変換、シリアライズ、状態管理に PBT ルールを強制する  
B) Partial - 純粋関数とシリアライズの往復テストに限定して PBT ルールを適用する  
C) No - 単純な連携処理として PBT ルールをスキップする  
X) Other (please describe after [Answer]: tag below)

[Answer]: A
