# Inception解決事項 QAリスト

**作成日**: 2026-05-10  
**目的**: Construction詳細ではなく、Inception成果物として今同期すべき事項だけを確認する。  
**回答方法**: 各質問の `[Answer]:` に、選択肢の文字を記入してください。どの選択肢にも合わない場合は `X` を選び、同じ行または直後に具体的な希望を書いてください。  
**運用方針**: 回答後、親AI-DLC仕様、子AI-DLCのInception成果物、Inception確認レポートを回答内容に基づいて修正する。Presigned URL有効期限、RAG同期方式、Notionプロパティなどの詳細設計は `construction-design-qa-backlog.md` で扱う。

---

## Q1. ロガー送信スキーマの正本

logger実装とdata-accumulation実装のズレを避けるため、Inception成果物上はどの文書をスキーマ契約の正本にしますか？

A) 親AI-DLCの `aidlc-docs/inception/application-design/data-accumulation-interface.md` を正本にする  
B) `data-accumulation/aidlc-docs/inception/requirements/requirements.md` を正本にし、親文書は概要に留める  
C) 親AI-DLCに別途JSON Schema相当の契約文書を追加し、それを正本にする  
X) Other

[Answer]: A

**修正対象**:
- `aidlc-docs/inception/application-design/data-accumulation-interface.md`
- `data-accumulation/aidlc-docs/inception/requirements/requirements.md`
- `aidlc-docs/inception-confirmation-report.md`

---

## Q2. data-accumulationの旧API Gateway記述

data-accumulationの一部Inception成果物に残る旧API Gateway前提を、最終方針へ一括同期してよいですか？

最終方針:
- Chrome拡張: Lambda Function URL + Cognito IDプール由来IAM認証
- ss-tool: IoT Core Request/Response + X.509

A) はい。一括同期する  
B) いいえ。旧記述は履歴としてそのまま残すが、Superseded注記だけ追加する  
C) 主要文書だけ同期し、詳細な計画文書はConstructionで更新する  
X) Other

[Answer]: A

**修正対象**:
- `data-accumulation/aidlc-docs/inception/plans/*.md`
- `data-accumulation/aidlc-docs/inception/application-design/*.md`
- `data-accumulation/aidlc-docs/inception-verification-report.md`

---

## Q3. digital-twinのステータス同期

digital-twinの `aidlc-state.md` と監査ログでは承認済みですが、一部Inception成果物に「レビュー待ち」が残っていました。これらを承認済みへ同期してよいですか？

A) はい。承認済みへ同期する  
B) いいえ。レビュー待ちのまま残す  
C) ステータスは完了にせず、注記で「監査ログ上は承認済み」とだけ書く  
X) Other

[Answer]: A

**修正対象**:
- `digital-twin/aidlc-docs/inception/requirements/requirements.md`
- `digital-twin/aidlc-docs/inception/plans/execution-plan.md`
- `digital-twin/aidlc-docs/inception/application-design/unit-of-work*.md`

---

## Q4. 古い検証レポートの扱い

`data-accumulation/aidlc-docs/inception-verification-report.md` は初回確認時点の条件付き不合格を含みますが、後続の最終評価で解決済みです。この旧レポートをどう扱いますか？

A) 冒頭にSuperseded注記を追加し、履歴として残す  
B) 内容を全面的に更新し、最終評価と同じ結論に書き換える  
C) 旧レポートを削除する  
X) Other

[Answer]: A

**修正対象**:
- `data-accumulation/aidlc-docs/inception-verification-report.md`
- `data-accumulation/aidlc-docs/inception-final-evaluation.md`

---

## Q5. QAの分離方針

既存QAにはConstruction詳細質問が多く含まれていました。Inception QAとConstruction QAを分離してよいですか？

A) はい。Inception QAはこの5問に絞り、詳細質問はConstruction設計QAバックログへ移す  
B) いいえ。1つのQAファイルに全て残す  
C) Inception QAは削除し、Construction準備文書だけに統合する  
X) Other

[Answer]: A

**修正対象**:
- `aidlc-docs/inception-resolution-qa.md`
- `aidlc-docs/construction-design-qa-backlog.md`
- `aidlc-docs/construction-readiness-issues.md`
