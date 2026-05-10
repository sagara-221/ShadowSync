# User Stories: Daily Log System

## Story Organization

- **Breakdown Approach**: User Journey-Based
- **Persona Scope**: ShadowSync 利用者
- **Priority Notation**: MVP / Later
- **Acceptance Criteria Detail**: ユーザー結果に加え、対象日判定、`user_id` 分離、Notion 出力、PBT に落とし込める検証観点を含める

## DL-US-001: 対象日の日報を自動生成する

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、日付境界を越えた後に対象日の日報を自動生成してほしい。そうすることで、手動操作なしで前日の活動を振り返れる。

### Acceptance Criteria

- ユーザーごとのタイムゾーン設定に基づいて対象日の開始・終了が判定される。
- 対象日の開始・終了は DynamoDB 検索に利用できる UTC 範囲へ変換される。
- 対象日境界を越えた後、EventBridge 起点で日報生成処理が開始される。
- 初期版では単一ユーザーを対象に処理される。
- 処理対象の `user_id` が明示的に指定され、他ユーザーのログは日報に含まれない。
- 対象日範囲外のログは日報に含まれない。

### INVEST Check

- **Independent**: Notion 出力や要約生成とは分離して対象日判定を検証できる。
- **Negotiable**: 実行時刻や対象日の決め方は後続設計で調整可能。
- **Valuable**: 利用者が手動で日報生成を開始しなくて済む。
- **Estimable**: EventBridge、Lambda、日付範囲計算として見積もれる。
- **Small**: 対象日判定と自動起動に範囲を限定している。
- **Testable**: 日付範囲、タイムゾーン、`user_id` 分離を検証できる。

## DL-US-002: Notion で時系列タイムラインを確認する

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、Notion 日報で対象日の活動を時系列に確認したい。そうすることで、1日の行動を正確に振り返れる。

### Acceptance Criteria

- DynamoDB の正規化済みアクティビティから対象日のログが取得される。
- 取得条件には必ず `user_id` と対象日時範囲が含まれる。
- 日報には対象日の活動タイムラインが含まれる。
- タイムラインは活動時刻に基づいて読める順序で整理される。
- 対象日外のログや他ユーザーのログはタイムラインに含まれない。
- 初期版では S3 元データ参照や S3 Vector 検索を必須としない。

### INVEST Check

- **Independent**: タイムライン生成は要約や Notion 連携と分離して検証できる。
- **Negotiable**: タイムラインの表示粒度は後続設計で調整可能。
- **Valuable**: 利用者がその日の行動の流れを把握できる。
- **Estimable**: DynamoDB 読み取りとタイムライン整形として見積もれる。
- **Small**: 対象日のログ取得と時系列整理に範囲を限定している。
- **Testable**: ログの日時順、対象日範囲、`user_id` 分離を検証できる。

## DL-US-003: 主要トピックと要約を短時間で確認する

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、日報で主要トピックと1日の要約を確認したい。そうすることで、長いログをすべて読まなくても重要な活動を把握できる。

### Acceptance Criteria

- 対象日のログをルールベースで抽出した後、LLM でタイムライン分類、トピック抽出、要約文生成が行われる。
- 日報には主要トピックが含まれる。
- 日報には1日の要約が含まれる。
- LLM に渡す入力は対象ユーザー・対象日分に限定される。
- LLM 入力全文や個人情報はアプリケーションログへ出力されない。
- LLM 呼び出しが失敗した場合は日報生成が失敗扱いになり、エラーログが残る。

### INVEST Check

- **Independent**: 要約生成は Notion ページ作成と分離して検証できる。
- **Negotiable**: LLM モデルやプロンプトは後続設計で調整可能。
- **Valuable**: 利用者が短時間で1日の要点を把握できる。
- **Estimable**: LLM 呼び出し、分類、要約処理として見積もれる。
- **Small**: 主要トピックと要約生成に範囲を限定している。
- **Testable**: 対象データ限定、出力セクション存在、失敗時ログを検証できる。

## DL-US-004: 日報を安全に Notion へ出力する

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、自分の Notion Database に日報を自動作成してほしい。そうすることで、日々の振り返りを普段使っている Notion に蓄積できる。

### Acceptance Criteria

- ユーザーごとの Notion Database ID と Integration Token を利用して日報ページが作成される。
- Notion Integration Token は安全に保管され、アプリケーションログへ出力されない。
- Notion ページには対象日、`user_id`、生成日時、タイムライン、主要トピック、要約が含まれる。
- 出力先 Notion Database は処理対象ユーザーの設定に基づく。
- 他ユーザーの Notion 設定や活動データは利用されない。
- Notion API 呼び出しが失敗した場合は日報生成が失敗扱いになり、秘匿情報を含まないエラーログが残る。

### INVEST Check

- **Independent**: Notion 出力は日付判定や要約生成後の出力処理として分離できる。
- **Negotiable**: Notion ページの具体的なプロパティは後続設計で調整可能。
- **Valuable**: 利用者が Notion 上で日報を確認・蓄積できる。
- **Estimable**: Notion API 連携とページ生成として見積もれる。
- **Small**: Notion への新規ページ作成に範囲を限定している。
- **Testable**: 出力先、ページ内容、token 非露出、`user_id` 分離を検証できる。

## DL-US-005: 既存日報を上書きせず再生成結果を残す

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、同じ対象日の生成処理が再度走った場合でも既存の日報を残してほしい。そうすることで、過去の出力を失わずに再生成結果を比較できる。

### Acceptance Criteria

- 同じ対象日の Notion 日報が既に存在しても、既存ページは上書きされない。
- 再生成時は新しい Notion ページが作成される。
- 新しいページにも対象日、`user_id`、生成日時が含まれる。
- 対象日と `user_id` の組み合わせが同じ場合でも、既存ページの内容は変更されない。
- 重複扱いのルールは後続設計でテスト可能な形に分離される。

### INVEST Check

- **Independent**: 再生成時の Notion 出力方針として独立して検証できる。
- **Negotiable**: 将来的に更新方式へ変更する余地がある。
- **Valuable**: 利用者が過去の日報を失わずに済む。
- **Estimable**: 既存ページ確認と新規ページ作成として見積もれる。
- **Small**: 既存日報の上書き禁止に範囲を限定している。
- **Testable**: 既存ページ非更新、新規ページ作成、対象日と `user_id` の保持を検証できる。

## DL-US-006: 失敗時に秘匿情報を出さずログを残す

- **Priority**: MVP
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、日報生成に失敗した場合でも Notion token や個人情報がログに出ないようにしてほしい。そうすることで、安全に問題調査できる。

### Acceptance Criteria

- Notion API、DynamoDB 読み取り、LLM 呼び出しの失敗時にはエラーログが残る。
- エラーログには処理ステップ、対象日、`user_id`、エラー種別が含まれる。
- エラーログには Notion Integration Token、認証情報、LLM 入力全文、個人情報が含まれない。
- 失敗時は日報生成が失敗扱いで終了する。
- 初期版では利用者通知や自動再実行までは必須としない。

### INVEST Check

- **Independent**: エラー処理とログ出力の方針として独立して検証できる。
- **Negotiable**: 通知や再実行は Later ストーリーとして追加可能。
- **Valuable**: 利用者が秘匿情報漏えいのリスクを抑えて利用できる。
- **Estimable**: 例外処理、構造化ログ、マスキングとして見積もれる。
- **Small**: 失敗扱いと安全なログ出力に範囲を限定している。
- **Testable**: エラーログ内容と秘匿情報非出力を検証できる。

## DL-US-007: 日報項目をカスタムする

- **Priority**: Later
- **Persona**: ShadowSync 利用者
- **User Story**: ShadowSync 利用者として、日報に含める項目をカスタムしたい。そうすることで、自分の振り返り方に合った日報を作れる。

### Acceptance Criteria

- 日報の出力セクションは設定可能な構造になっている。
- 初期版の必須項目はタイムライン、主要トピック、要約とする。
- 将来的にプロジェクト別、アプリ別、Webサイト別、集中時間、改善提案などを追加できる。
- カスタム設定がない場合は MVP の標準構成で日報が生成される。

### INVEST Check

- **Independent**: 日報出力セクション設定として独立して扱える。
- **Negotiable**: 追加項目の種類や設定方法は後続フェーズで調整可能。
- **Valuable**: 利用者が自分に合った振り返りを得られる。
- **Estimable**: 設定モデルと出力テンプレート拡張として見積もれる。
- **Small**: 出力項目のカスタムに範囲を限定している。
- **Testable**: デフォルト出力とカスタム出力の差分を検証できる。

## Coverage Mapping

| Requirement | Covered By |
|---|---|
| FR-1 日報生成対象 | DL-US-001, DL-US-004 |
| FR-2 対象日の判定 | DL-US-001, DL-US-002 |
| FR-3 データ取得 | DL-US-002 |
| FR-4 日報生成 | DL-US-002, DL-US-003, DL-US-007 |
| FR-5 Notion 連携 | DL-US-004, DL-US-005 |
| FR-6 自動実行 | DL-US-001 |
| FR-7 エラーハンドリング | DL-US-003, DL-US-004, DL-US-006 |
| Security Baseline | DL-US-002, DL-US-004, DL-US-006 |
| PBT 対象候補 | DL-US-001, DL-US-002, DL-US-005 |

## Notes for Later Phases

- Functional Design では、日付範囲判定、タイムゾーン変換、`user_id` 分離、Notion 出力モデル、既存ページ非上書きを Testable Properties として明示する。
- Infrastructure Design では、Secrets Manager、IAM 最小権限、CloudWatch Logs、EventBridge、Lambda の構成を Security Baseline に沿って検証する。
- Code Generation では、Python と Hypothesis を前提に PBT 対象の純粋関数を切り出す。
