# Unit of Work Plan

## Project Context
- **Project Type**: Greenfield AWS Serverless Backend
- **Architecture**: Multi-service serverless data accumulation system
- **Deployment Model**: CloudFormation-based Infrastructure as Code
- **Primary Components**: IoT Core, Kinesis, Lambda, DynamoDB, S3, Bedrock, API Gateway

---

## Decomposition Questions

### 1. Story Grouping Strategy
このプロジェクトはインフラストラクチャ中心のサーバーレスシステムです。ユニット分割の基準について確認させてください：

**Q1.1**: ユニット分割の基準として、どのアプローチが適切でしょうか？
- A) **機能別** - データ取り込み、データ処理、ストレージ、API、モニタリングなど機能ごとに分割
- B) **データフロー別** - 構造化データフロー、スクリーンショットフロー、エラーハンドリングフローなど
- C) **AWSサービス別** - IoT Core、Kinesis、Lambda、DynamoDB、S3など各サービスを独立したユニット
- D) **その他** - 別のアプローチがあればご指定ください

[Answer]: A

**Q1.2**: Lambda関数は個別のユニットとして扱うべきでしょうか、それとも関連する機能でグループ化すべきでしょうか？
- A) **個別ユニット** - 各Lambda関数を独立したユニット（Router、Structured、Screenshot Meta、Bedrock、Presigned URL）
- B) **機能グループ** - 関連するLambda関数をグループ化（例：データ処理系、API系）
- C) **単一ユニット** - すべてのLambda関数を1つのユニット

[Answer]: A

### 2. Infrastructure as Code Organization
CloudFormationスタック構成について確認させてください：

**Q2.1**: CloudFormationスタックの分割方針はどうしますか？
- A) **単一スタック** - すべてのリソースを1つのスタックで管理（シンプル、依存関係管理が容易）
- B) **レイヤー別スタック** - 基盤層（ネットワーク、IAM）、データ層（DynamoDB、S3）、処理層（Lambda、Kinesis）など
- C) **機能別スタック** - 取り込み、処理、ストレージ、APIなど機能ごとにスタック分割
- D) **サービス別スタック** - 各AWSサービスを独立したスタック

[Answer]: C

**Q2.2**: スタック間の依存関係管理はどうしますか？
- A) **CloudFormation Export/Import** - スタック間でOutputs/Importsを使用
- B) **SSM Parameter Store** - パラメータストアで値を共有
- C) **単一スタックのため不要** - Q2.1でAを選択した場合

[Answer]: A

### 3. Code Organization (Greenfield)
コードの配置構造について確認させてください：

**Q3.1**: Lambda関数のコード配置はどうしますか？
- A) **関数別ディレクトリ** - `lambda/router/`, `lambda/structured/`, `lambda/bedrock/`など
- B) **機能別ディレクトリ** - `lambda/ingestion/`, `lambda/processing/`, `lambda/api/`など
- C) **フラット構造** - `lambda/`直下にすべての関数

[Answer]: B

**Q3.2**: 共通ライブラリやユーティリティの配置はどうしますか？
- A) **Lambda Layer** - 共通コードをLambda Layerとして配置（`layers/common/`）
- B) **各関数にコピー** - 共通コードを各Lambda関数ディレクトリにコピー
- C) **共有ディレクトリ** - `shared/`や`common/`ディレクトリを作成し、デプロイ時にバンドル

[Answer]: A

**Q3.3**: CloudFormationテンプレートの配置はどうしますか？
- A) **iac/ディレクトリ** - `iac/ingestion.yaml`, `iac/processing.yaml`など
- B) **cloudformation/ディレクトリ** - `cloudformation/`配下に配置
- C) **各機能ディレクトリ内** - Lambda関数と同じディレクトリに配置
- D) **ルート直下** - プロジェクトルートに配置

[Answer]: A

### 4. Dependencies and Integration
ユニット間の依存関係について確認させてください：

**Q4.1**: データ取り込み（IoT Core + Kinesis）とデータ処理（Lambda）の関係をどう扱いますか？
- A) **別ユニット** - 取り込み基盤と処理ロジックを分離
- B) **統合ユニット** - 取り込みから処理までを1つのユニット
- C) **データフロー別** - 構造化データとスクリーンショットで分離

[Answer]: B

**Q4.2**: 認証基盤（Cognito、X.509）はどう扱いますか？
- A) **独立ユニット** - 認証・認可を専用ユニット
- B) **取り込みユニットに含める** - IoT Coreと一緒に管理
- C) **各ユニットで個別管理** - 必要なユニットごとに認証設定

[Answer]: A

### 5. Testing Strategy
テスト戦略について確認させてください：

**Q5.1**: ユニットテストの範囲はどうしますか？
- A) **Lambda関数のみ** - ビジネスロジックを持つLambda関数のみテスト
- B) **すべてのコンポーネント** - Lambda、IoT Rules、DynamoDB操作など
- C) **統合テスト中心** - ユニットテストは最小限、E2Eテスト重視

[Answer]: B

**Q5.2**: テストコードの配置はどうしますか？
- A) **各ユニット内** - `lambda/router/tests/`のように各ユニット内にtestsディレクトリ
- B) **専用testsディレクトリ** - プロジェクトルートに`tests/`ディレクトリを作成
- C) **Lambda関数と同じファイル** - `router.py`と`test_router.py`を同じディレクトリに配置

[Answer]: A

### 6. Deployment and Operations
デプロイ戦略について確認させてください：

**Q6.1**: デプロイの単位はどうしますか？
- A) **ユニット単位** - 各ユニットを独立してデプロイ可能
- B) **全体一括** - すべてのユニットを同時にデプロイ
- C) **段階的** - 基盤→データ層→処理層の順にデプロイ

[Answer]: A

**Q6.2**: 環境管理（dev、staging、prod）はどうしますか？
- A) **パラメータファイル** - 環境ごとにパラメータファイルを用意（`params/dev.json`など）
- B) **環境変数** - デプロイ時に環境変数で切り替え
- C) **別スタック名** - スタック名に環境名を含める（`shadowsync-dev-ingestion`など）

[Answer]: B

---

## Follow-up Questions (Ambiguity Resolution)

回答を分析したところ、以下の点で矛盾が見られます。明確化のため追加質問にお答えください：

### FQ1: Lambda関数のユニット化とコード配置の整合性

**矛盾点**: 
- Q1.2で「Lambda関数を個別ユニット」と回答
- Q3.1で「機能別ディレクトリ（`lambda/ingestion/`, `lambda/processing/`）」と回答

**質問**: Lambda関数の扱いを明確にしてください：

**Option A - Lambda関数を個別ユニット化**:
```
ユニット構成:
- Unit 1: Lambda Router
- Unit 2: Lambda Structured
- Unit 3: Lambda Screenshot Meta
- Unit 4: Lambda Bedrock
- Unit 5: Lambda Presigned URL

コード配置:
lambda/
  ├── router/
  ├── structured/
  ├── screenshot-meta/
  ├── bedrock/
  └── presigned-url/
```

**Option B - Lambda関数を機能グループ化**:
```
ユニット構成:
- Unit 1: Data Ingestion (Router + Structured + Screenshot Meta)
- Unit 2: AI Processing (Bedrock)
- Unit 3: API (Presigned URL)

コード配置:
lambda/
  ├── ingestion/
  │   ├── router/
  │   ├── structured/
  │   └── screenshot-meta/
  ├── processing/
  │   └── bedrock/
  └── api/
      └── presigned-url/
```

[Answer]: A

### FQ2: データ取り込みと処理の分離方針

**矛盾点**:
- Q1.1で「機能別ユニット分割」（取り込み、処理を分離）と回答
- Q4.1で「取り込みと処理を統合ユニット」と回答

**質問**: データ取り込み（IoT Core + Kinesis）とデータ処理（Lambda）の関係を明確にしてください：

**Option A - 機能別に分離**:
```
ユニット構成:
- Unit: Data Ingestion (IoT Core + Kinesis + IoT Rules)
- Unit: Data Processing (Lambda Router + Structured + Screenshot Meta)
- Unit: AI Processing (Lambda Bedrock)
- Unit: API (Lambda Presigned URL)
- Unit: Storage (DynamoDB + S3)
- Unit: Authentication (Cognito + X.509)
```

**Option B - 取り込みと処理を統合**:
```
ユニット構成:
- Unit: Ingestion & Processing (IoT Core + Kinesis + Lambda Router + Structured + Screenshot Meta)
- Unit: AI Processing (Lambda Bedrock)
- Unit: API (Lambda Presigned URL)
- Unit: Storage (DynamoDB + S3)
- Unit: Authentication (Cognito + X.509)
```

[Answer]: B

---

## Unit of Work Decomposition Plan

### Phase 1: Planning and Documentation
- [x] Analyze all user answers and resolve any ambiguities
- [x] Generate `aidlc-docs/inception/application-design/unit-of-work.md` with:
  - Unit definitions based on approved decomposition approach
  - Responsibilities for each unit
  - Code organization strategy (directory structure)
  - Deployment model and stack organization
- [x] Generate `aidlc-docs/inception/application-design/unit-of-work-dependency.md` with:
  - Dependency matrix showing relationships between units
  - Integration points and data flow between units
  - CloudFormation stack dependencies (if multi-stack)
- [x] Generate `aidlc-docs/inception/application-design/unit-of-work-story-map.md` with:
  - Mapping of functional requirements to units
  - Coverage verification (all FRs assigned to units)

### Phase 2: Validation
- [x] Validate unit boundaries are clear and non-overlapping
- [x] Verify all functional requirements are covered by units
- [x] Ensure dependencies are manageable and not circular
- [x] Confirm code organization aligns with deployment strategy

### Phase 3: Approval
- [x] Present unit decomposition to user for review
- [x] Address any feedback or concerns
- [x] Obtain explicit approval to proceed to Units Generation

---

## Next Steps
1. User fills in all [Answer]: tags above
2. AI analyzes answers for ambiguities and asks follow-up questions if needed
3. User approves the plan
4. AI executes the decomposition plan to generate unit artifacts

---

**Document Status**: Awaiting User Input
**Required Action**: Please fill in all [Answer]: tags above
