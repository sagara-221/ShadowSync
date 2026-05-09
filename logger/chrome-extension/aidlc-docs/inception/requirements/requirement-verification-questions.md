# 要求仕様の確認事項 (Requirement Verification Questions)

ShadowSync用のChrome拡張ロガーツールについて、要件を明確化するための質問です。各質問の `[Answer]:` の後にご回答をお願いいたします。

## 機能要件について
### Q1. ログ取得のトリガー
閲覧ログ（URL、タイトル、HTML等）を取得・送信するタイミングはどれが適切ですか？

A) ページ遷移時（URLが変更された時）のみ
B) ページ遷移時 ＋ 定期的な時間間隔（例: 1分ごと）
C) ページ遷移時 ＋ ユーザーの特定のアクション時（スクロールなど）
X) その他（以下に記述してください）

[Answer]: A

### Q2. 設定画面 (Options Page) の要件
`user_id` や `device_id`、Cognitoの設定値（Identity Pool ID、リージョンなど）は、拡張機能の設定画面（Options）からユーザーが手動で入力する想定でよいでしょうか？

A) はい、設定画面を作成し、ユーザーが手動で入力・保存する
B) いいえ、別のツール（SSツールなど）から自動で連携・注入される仕組みを想定している
X) その他（以下に記述してください）

[Answer]: A

### Q3. プライバシー・除外設定
特定のURLやドメイン（例: 銀行のサイト、シークレットモードでの閲覧など）のログ取得を除外する設定は必要ですか？

A) 必要（設定画面でユーザーが除外ドメインのリストを管理できるようにする）
B) 不要（すべてのアクセスをログとして送信する）
X) その他（以下に記述してください）

[Answer]: A

### Q4. HTMLスニペットの扱い
HTMLスニペットの送信は、データ量が大きくなる可能性がありパフォーマンスに影響する場合があります。取得有無をユーザーが設定で切り替えられるようにしますか？

A) はい、HTML取得のON/OFFを設定画面で切り替えられるようにする
B) いいえ、常に取得して送信する
C) いいえ、HTMLスニペットの取得自体を仕様から外す
X) その他（以下に記述してください）

[Answer]: A

---

## 拡張機能（Extensions）の適用について

### Q5. Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)
B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

### Q6. Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)
B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)
C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)
X) Other (please describe after [Answer]: tag below)

[Answer]: B

