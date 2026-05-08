# ユニット分割計画 (Unit of Work Plan)

## 生成タスクの計画

システム全体を独立して開発・デプロイ可能な単位（ユニット＝子AI-DLC）に分割するためのタスク計画です。

- [x] `aidlc-docs/inception/application-design/unit-of-work.md` の生成 (各ユニットの定義と責任範囲のドキュメント化)
- [x] `aidlc-docs/inception/application-design/unit-of-work-dependency.md` の生成 (ユニット間の依存関係マトリクスの作成)
- [x] `aidlc-docs/inception/application-design/unit-of-work-story-map.md` の生成 (高レベル要件と各ユニットのマッピング)
- [x] `unit-of-work.md` にコード構成戦略のドキュメント化 (ディレクトリ構造と親/子/孫AI-DLCの境界定義)
- [x] ユニットの境界と依存関係の妥当性検証
- [x] すべての要件が適切なユニットに割り当てられていることの確認
- [x] 各ユニット（子AI-DLC）ディレクトリへの「親からの要件指示書 (インテント)」の生成と配置

---

## ユニット境界に関する確認事項

以下の質問に対して、`[Answer]:` の後に回答を記入してください。

### Story Grouping & Team Alignment (開発体制について)
**Q1.** 各ユニット（子AI-DLC）の開発は、1人で順番に行う予定ですか？それとも並行して進める予定ですか？
[Answer]: 複数の人間が並列して進める予定です。

### Dependencies & Technical Considerations (依存関係と技術的考慮点)
**Q2.** データ蓄積システム（AWSバックエンド）とロガーシステム（ローカルPC）間のデータ送信において、通信プロトコルや認証方式の前提（例: IoT Core経由のMQTT、API Gatewayなど）はすでに決まっていますか？未定であれば「データ蓄積システム側に一任する」で構いません。
[Answer]: IoT Core経由のMQTTを想定しています。

### Business Domain & Boundaries (ドメインと境界)
**Q3.** 「ロガーシステム」の孫として「osapi」「chrome拡張」「ssツール」が存在しますが、これらを束ねる「親ロガー（データのバッファリングやAWS送信を統括するローカルモジュール）」を設けますか？それとも各ツールがそれぞれ独立してAWSへデータを送信する構成としますか？
[Answer]: 各ツールが独立してAWSへデータを送信する構成とします。
