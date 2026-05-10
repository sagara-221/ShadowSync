# Story Generation Plan

## Purpose

`daily-log` の要件を、ユーザー視点のストーリー、ペルソナ、受け入れ基準に変換する。

## Context

- 対象システム: ShadowSync Daily Log System
- 要件定義: `aidlc-docs/inception/requirements/requirements.md`
- 初期版の対象: 単一ユーザー
- 将来拡張: ユーザーごとの Notion 設定と `user_id` 分離を前提に複数ユーザー対応可能にする
- 主な成果物: Notion 日報ページ

## Story Development Checklist

- [x] Read `aidlc-docs/inception/requirements/requirements.md`
- [x] Identify personas from requirements and user workflow
- [x] Select story breakdown approach from approved answers
- [x] Generate `aidlc-docs/inception/user-stories/personas.md`
- [x] Generate `aidlc-docs/inception/user-stories/stories.md`
- [x] Ensure each story follows INVEST criteria
- [x] Add acceptance criteria for each story
- [x] Map personas to relevant user stories
- [x] Verify stories cover requirements FR-1 through FR-7 and NFR concerns relevant to user outcomes
- [x] Verify Security Baseline and PBT implications are represented where user-visible or acceptance-test relevant

## Story Breakdown Options

### Option A: User Journey-Based

ユーザーが「1日を終える」「自動生成を待つ」「Notion で振り返る」「必要に応じて再生成や確認をする」という流れに沿ってストーリーを整理する。

- **Pros**: 実際の利用体験に沿う。日報の価値が見えやすい。
- **Cons**: 内部処理やエラー処理の粒度が曖昧になりやすい。

### Option B: Feature-Based

対象日判定、データ取得、LLM 生成、Notion 出力、エラー処理、セキュリティのように機能単位で整理する。

- **Pros**: 実装・設計・テストに接続しやすい。
- **Cons**: ユーザー価値の流れが分断されやすい。

### Option C: Persona-Based

利用者、運用者、将来の複数ユーザー利用者などのペルソナごとに整理する。

- **Pros**: 利害関係者ごとの期待値が明確になる。
- **Cons**: 初期版が単一ユーザーのため、過剰に見える可能性がある。

### Option D: Hybrid

主要ストーリーは User Journey-Based で整理し、エラー処理・セキュリティ・テスト観点は Feature-Based の補助ストーリーとして整理する。

- **Pros**: ユーザー価値と実装可能性のバランスがよい。
- **Cons**: ストーリー分類のルールを明確にする必要がある。

## Questions

以下の `[Answer]:` に選択肢の文字を記入してください。どの選択肢にも合わない場合は `X` を選び、具体的な希望を書いてください。

---

## Q1. ストーリー分解方針
`daily-log` のユーザーストーリーは、どの方針で整理しますか？

A) User Journey-Based: 利用者が日報を受け取り、振り返る流れを中心にする  
B) Feature-Based: 対象日判定、データ取得、LLM 生成、Notion 出力など機能単位にする  
C) Persona-Based: 利用者、運用者、将来の複数ユーザーなどペルソナ単位にする  
D) Hybrid: 主要体験は User Journey-Based、内部品質やエラー処理は Feature-Based にする  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q2. ペルソナ範囲
初期の `personas.md` には、どのペルソナまで含めますか？

A) 日報を読む単一の ShadowSync 利用者のみ  
B) ShadowSync 利用者 + システム運用者  
C) ShadowSync 利用者 + システム運用者 + 将来の複数ユーザー利用者  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q3. 受け入れ基準の粒度
各ストーリーの受け入れ基準は、どの粒度にしますか？

A) ユーザーが確認できる結果を中心に簡潔にする  
B) ユーザー結果に加えて、対象日判定、`user_id` 分離、Notion 出力など検証観点も含める  
C) Given/When/Then 形式で詳細に書く  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q4. ストーリーの優先度表記
ストーリーに優先度を付けますか？

A) Must / Should / Could で優先度を付ける  
B) MVP / Later で初期版と将来版を分ける  
C) 優先度は付けず、全ストーリーを同列で扱う  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Q5. Notion 日報のユーザー価値
Notion 日報のストーリーでは、どの価値を最も強調しますか？

A) 1日の行動を時系列で正確に振り返れること  
B) 主要トピックと要約により短時間で振り返れること  
C) 将来的に日報項目をカスタムできること  
D) A と B を同程度に重視し、C は将来拡張として扱う  
X) Other (please describe after [Answer]: tag below)

[Answer]: D

## Q6. エラー・失敗時ストーリー
Notion API や LLM 呼び出し失敗のストーリーは、どの程度扱いますか？

A) 最小限にし、ログを残して失敗扱いにする要件だけを書く  
B) 利用者または運用者が失敗に気づけることまで含める  
C) 再実行や失敗ジョブ管理まで将来ストーリーとして含める  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q7. セキュリティ関連ストーリー
Security Baseline が有効ですが、セキュリティ関連はストーリーとしてどう扱いますか？

A) ユーザー価値に直結する `user_id` 分離と token 非露出だけをストーリーに含める  
B) `user_id` 分離、token 管理、IAM 最小権限、ログ秘匿を受け入れ基準にも含める  
C) セキュリティは後続の設計成果物で扱い、ユーザーストーリーには含めない  
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Q8. PBT 関連ストーリー
PBT が有効ですが、ストーリー内ではテスト可能性をどこまで明示しますか？

A) ストーリーには明示せず、後続の Functional Design で扱う  
B) 日付判定、ユーザー分離、重複扱いなどは受け入れ基準に検証観点として入れる  
C) 各ストーリーに PBT 対象かどうかを明記する  
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Approval Gate

この計画への回答がすべて埋まり、曖昧さが解消された後、ユーザーストーリー生成に進む。生成前にこの計画への明示的な承認を必要とする。
