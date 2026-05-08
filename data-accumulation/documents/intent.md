# 開発インテント (データ蓄積システム)

**目的**: ロガーシステムから送信された様々な形式のデータを受信し、Amazon Bedrockを活用して構造化・共通形式に変換し、蓄積（RAG構築）するバックエンドシステムの開発。

**要件**:
- AWS IoT Core, Glue, DynamoDB, S3, Lambda などの利用を想定。
- **エンドポイントと認証基盤**: 各ロガーが通信できるよう、IoT Coreのエンドポイント整備に加え、「X.509証明書認証（ローカルアプリ用）」と「Cognito IDプール認証（Chrome拡張用）」の両方を構築・許可すること。
- **大容量データ受け入れ**: SSツール向けに、S3への直接アップロード機構（Presigned URLの発行など）を構築すること。
- インターフェースの詳細は、親AI-DLCで定義した `../aidlc-docs/inception/application-design/data-accumulation-interface.md` を必ず参照・準拠すること。
