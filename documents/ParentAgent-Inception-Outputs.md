# Parent AI-DLC: Inceptionフェーズ 最終アウトプット一覧

本プロジェクト（個人活動ログ自動集計・活用システム: ShadowSync）の全体設計を担う「親AI-DLC」のInceptionフェーズにおける最終成果物（アウトプット）の一覧です。
これらのドキュメントは、次に開始する各「子AI-DLC」への入力（Input）として機能します。

## 1. プロジェクト要件と実行計画 (Requirements & Plans)
*   **[requirements.md](../aidlc-docs/inception/requirements/requirements.md)**
    *   システム全体の要件と、「親AI-DLCは設計と委譲に専念する」という基本方針を定義したドキュメント。
*   **[unit-of-work-plan.md](../aidlc-docs/inception/plans/unit-of-work-plan.md)**
    *   親AI-DLCがどのようにシステムを分割し、要件ドキュメントを生成するかの合意プロセスと計画を記録したドキュメント。
*   **[execution-plan.md](../aidlc-docs/inception/plans/execution-plan.md)**
    *   親AI-DLCのタスク実行計画（構築フェーズをスキップし設計のみに留める旨）を定義したドキュメント。

## 2. アーキテクチャとインターフェース設計 (Application Design)
*   **[unit-of-work.md](../aidlc-docs/inception/application-design/unit-of-work.md)**
    *   システムをどの単位（ユニット）に分割し、それぞれのディレクトリがどのような役割・責任範囲を持つかを定義した設計書。
*   **[unit-of-work-dependency.md](../aidlc-docs/inception/application-design/unit-of-work-dependency.md)**
    *   各ユニット間のデータフロー（依存関係や通信プロトコル）のマトリクス。
*   **[unit-of-work-story-map.md](../aidlc-docs/inception/application-design/unit-of-work-story-map.md)**
    *   高レベルの要件（ユーザーストーリー）が、具体的にどのユニット（子AI-DLC）で実現されるかをマッピングしたドキュメント。
*   **[data-accumulation-interface.md](../aidlc-docs/inception/application-design/data-accumulation-interface.md)**
    *   各ロガーからAWS（データ蓄積システム）への通信方法（MQTT仕様、Cognito WebSockets、S3 Presigned URL、JSONペイロードのスキーマ等）を確定させたインターフェース定義書。

## 3. 各子AI-DLCへの開発指示書 (Units Generation Intents)
親AI-DLCから各サブシステムへ引き継がれる個別の要件指示書です。各子・孫ディレクトリ内に配置されています。

*   **ロガーシステム (Logger)**
    *   [OS APIロガー](../logger/osapi/documents/intent.md)
    *   [Chrome拡張ロガー](../logger/chrome-extension/documents/intent.md)
    *   [スクリーンショットロガー](../logger/ss-tool/documents/intent.md)
*   **データ蓄積システム (Data Accumulation)**
    *   [データ蓄積システム](../data-accumulation/documents/intent.md)
*   **日誌作成システム (Daily Log)**
    *   [日誌作成システム](../daily-log/documents/intent.md)
*   **デジタルツインシステム (Digital Twin)**
    *   [デジタルツインシステム](../digital-twin/documents/intent.md)
