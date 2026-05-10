# 要件確認質問 (Requirement Verification Questions)

以下の質問に回答してください。各質問の `[Answer]:` タグの後に、選択肢の記号を記入してください。

---

## Question 1
スクリーンショットの撮影間隔（インターバル）はどの程度を想定していますか？

A) 30秒ごと
B) 1分ごと
C) 5分ごと
D) ユーザーが設定ファイルで任意に指定できるようにする
X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Question 2
「特定のトリガー」でのSS撮影について、intent.mdに記載がありますが、具体的にどのようなトリガーを想定していますか？

A) アクティブウィンドウが切り替わった時
B) 定期インターバルのみ（トリガーは不要）
C) OS APIロガー側のイベント（ウィンドウ切替等）をトリガーとして受信する
D) ユーザーの手動操作（ホットキー等）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 3
マルチモニター環境での撮影はどのように扱いますか？

A) プライマリモニターのみ撮影
B) すべてのモニターを個別に撮影（1モニター = 1画像）
C) すべてのモニターを結合した1枚の画像として撮影
D) ユーザーが設定で撮影対象のモニターを選択できるようにする
X) Other (please describe after [Answer]: tag below)

[Answer]: X　アクティブウィンドウの内容のみ撮影

## Question 4
画像の圧縮形式はどれを想定していますか？

A) PNG（ロスレス、高品質だがファイルサイズ大）
B) JPEG（非可逆圧縮、バランス型）
C) WebP（高圧縮率、モダンフォーマット）
D) 設定で切り替え可能にする
X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Question 5
送信完了後のローカル画像削除やオフライン時の一時保存について、容量上限の目安はどの程度ですか？

A) 100MB
B) 500MB
C) 1GB
D) ユーザーが設定で容量上限を指定できるようにする
X) Other (please describe after [Answer]: tag below)

[Answer]: D

## Question 6
開発言語・ランタイムの選定について、どれを使用しますか？

A) Python（開発速度優先、AWS SDK boto3 が豊富）
B) Rust（パフォーマンス・省リソース優先）
C) C# / .NET（Windowsネイティブとの親和性が高い）
D) Go（コンパイル済みバイナリ、軽量）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 7
アプリケーションの実行形態はどれを想定していますか？

A) Windowsサービスとしてバックグラウンド常駐
B) タスクトレイ常駐アプリケーション（GUI最小限）
C) コンソールアプリケーション（手動起動）
D) タスクスケジューラで定期起動
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 8
`user_id` と `device_id` はどのように管理しますか？

A) ローカルの設定ファイル（JSON/YAML等）にユーザーが手動で設定する
B) 初回起動時のセットアップウィザードで入力させる
C) AWS IoT Coreのデバイス証明書に紐づけて自動取得する
D) 環境変数で指定する
X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 9
S3へのアップロードに使用する認証方式はどれですか？（intent.mdでは「IAMまたはPresigned URL」と記載）

A) IoT Coreのデバイス証明書に紐づくIAMロール（Credential Provider経由）で直接S3にPutObject
B) バックエンド（Lambda等）からPresigned URLを取得してアップロード
C) まだ決まっていない（この子AI-DLCで設計する）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Question 10: セキュリティ拡張
このプロジェクトにセキュリティ拡張ルールを適用しますか？

A) Yes — すべてのセキュリティルールをブロッキング制約として適用する（本番品質のアプリケーション推奨）
B) No — すべてのセキュリティルールをスキップする（PoC、プロトタイプ、実験プロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 11: Property-Based Testing 拡張
このプロジェクトにProperty-Based Testing (PBT) ルールを適用しますか？

A) Yes — すべてのPBTルールをブロッキング制約として適用する（ビジネスロジック、データ変換、シリアライゼーション、ステートフルコンポーネントを含むプロジェクト推奨）
B) Partial — 純粋関数とシリアライゼーション往復テストのみにPBTルールを適用する（アルゴリズムの複雑性が限定的なプロジェクト向け）
C) No — すべてのPBTルールをスキップする（シンプルなCRUDアプリ、UIのみ、薄い統合レイヤー向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: B
