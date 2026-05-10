# Services: Daily Log System

## Service Layer Overview

The application service layer is centered on `DailyLogService`.

`DailyLogService` represents one use case: generating and publishing a daily report for a user and target date. Lambda handler remains thin. External APIs are accessed only through ports and adapters.

## Primary Service

### DailyLogService

#### Responsibilities

- Coordinate daily report generation.
- Enforce the processing sequence.
- Use ports for configuration, secrets, activity loading, LLM, and publishing.
- Decide whether the run succeeded or failed.
- Emit safe structured logs through `StructuredLogger`.

#### Orchestration Flow

1. Receive `user_id`, optional `target_date`, and `execution_id`.
2. Load user configuration with `UserConfigurationProvider`.
3. Resolve target date range with `DateRangeResolver`.
4. Load normalized activities with `ActivityRepository`.
5. Defensively filter activities with `ActivityFilter`.
6. Generate timeline, topics, and summary with `DailyLogGenerator`.
7. Build `DailyReport` with `ReportModelBuilder`.
8. Resolve Notion token with `SecretProvider`.
9. Publish report with `ReportPublisher`.
10. Log success or normalized failure.

#### Failure Handling

- Adapter components normalize SDK/API errors into safe application errors.
- `DailyLogService` treats any unrecovered external API failure as daily-log generation failure.
- Failure logs include safe metadata: `user_id`, target date, execution ID, component, and error code.
- Failure logs exclude Notion token, raw prompt text, full activity payload, and authentication material.

## Domain Services and Helpers

### DateRangeResolver

- Pure-function oriented.
- Converts local target date and timezone to UTC half-open range.
- Used by `DailyLogService`.

### ActivityFilter

- Pure-function oriented.
- Applies defensive `user_id` and range filtering.
- Helps protect against accidental repository misconfiguration.

### DailyLogGenerator

- Coordinates report content generation.
- Delegates model calls to `LLMClient`.
- Produces `DailyReportContent`, not Notion-specific structures.

### ReportModelBuilder

- Pure-function oriented.
- Builds `DailyReport` from user config, date range, and generated content.

## Ports

### UserConfigurationProvider

- Abstracts user configuration loading.
- Initial adapter: `DynamoDBUserConfigurationProvider`.

### SecretProvider

- Abstracts secret retrieval.
- Initial adapter: `SecretsManagerSecretProvider`.

### ActivityRepository

- Abstracts normalized activity loading.
- Initial adapter: `DynamoDBActivityRepository`.

### LLMClient

- Abstracts LLM operations.
- Initial adapter: `BedrockLLMClient`.

### ReportPublisher

- Abstracts report publishing.
- Initial adapter: `NotionReportPublisher`.

## Adapter Services

### DynamoDBUserConfigurationProvider

- Reads user timezone and Notion database settings from DynamoDB.
- Returns token secret ID, not token value.

### SecretsManagerSecretProvider

- Reads Notion token from Secrets Manager.
- Does not log secret values.

### DynamoDBActivityRepository

- Queries activities by `user_id` and UTC date range.
- Maps DynamoDB items to `ActivityRecord`.

### BedrockLLMClient

- Calls Amazon Bedrock for timeline classification, topic extraction, and summary.
- Normalizes Bedrock errors.
- Redacts prompt content from logs.

### NotionReportPublisher

- Converts `DailyReport` to Notion page properties and blocks.
- Creates a new Notion page for every publication.
- Normalizes Notion API errors.

## Service Boundary Rules

- Lambda handler wires dependencies and delegates to `DailyLogService`.
- Domain services do not import AWS SDK or Notion SDK.
- Adapters may depend on AWS SDK or Notion SDK.
- Application service depends on ports, not concrete adapters.
- Detailed business rules are deferred to Functional Design.
