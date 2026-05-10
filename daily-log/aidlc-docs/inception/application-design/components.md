# Components: Daily Log System

## Design Decisions

- **Component granularity**: Medium
- **Service center**: `DailyLogService`
- **LLM abstraction**: `LLMClient` interface with Bedrock implementation
- **Publisher abstraction**: `ReportPublisher` interface with Notion implementation
- **User configuration**: DynamoDB for user configuration, Secrets Manager for Notion token
- **Report model**: `DailyReport` domain model, converted by publisher
- **Error boundary**: External clients normalize exceptions; orchestration layer decides failure handling
- **PBT boundary**: Date Range Resolver, Activity Filter, Report Model Builder are pure-function oriented
- **Dependency direction**: Domain logic does not depend on AWS / Notion SDK; external dependencies stay in adapters

## Component Overview

| Component | Type | Purpose |
|---|---|---|
| `ScheduleHandler` | Entry point | Receives EventBridge invocation and calls `DailyLogService`. |
| `DailyLogService` | Application service | Orchestrates one daily-log generation use case. |
| `UserConfigurationProvider` | Port | Loads user timezone, Notion database ID, and secret references. |
| `DynamoDBUserConfigurationProvider` | Adapter | Reads user configuration from DynamoDB. |
| `SecretProvider` | Port | Resolves sensitive values such as Notion integration token. |
| `SecretsManagerSecretProvider` | Adapter | Reads secrets from AWS Secrets Manager. |
| `DateRangeResolver` | Domain component | Converts user timezone and target date into UTC date range. |
| `ActivityRepository` | Port | Loads normalized activity records for user and date range. |
| `DynamoDBActivityRepository` | Adapter | Reads normalized activities from DynamoDB. |
| `ActivityFilter` | Domain component | Ensures activity records match `user_id` and target range. |
| `ReportModelBuilder` | Domain component | Builds `DailyReport` from activities and LLM-generated content. |
| `DailyLogGenerator` | Domain service | Coordinates timeline shaping, topic extraction, and summary generation. |
| `LLMClient` | Port | Classifies timeline, extracts topics, and generates summary. |
| `BedrockLLMClient` | Adapter | Calls Amazon Bedrock through AWS SDK. |
| `ReportPublisher` | Port | Publishes `DailyReport` to an external destination. |
| `NotionReportPublisher` | Adapter | Converts `DailyReport` to Notion page properties and blocks. |
| `StructuredLogger` | Cross-cutting | Emits structured logs without sensitive values. |
| `ErrorMapper` | Cross-cutting | Converts adapter-specific exceptions into application errors. |

## Component Responsibilities

### ScheduleHandler

- Parse EventBridge payload.
- Determine requested `user_id` and target date if provided.
- Call `DailyLogService.generate_daily_log`.
- Return success or failure response for Lambda runtime.
- Avoid business logic and direct SDK calls except dependency wiring.

### DailyLogService

- Coordinate the full daily-log generation use case.
- Load user configuration.
- Resolve target date range.
- Fetch normalized activities.
- Apply activity filtering as a defensive layer.
- Generate `DailyReport`.
- Publish the report through `ReportPublisher`.
- Decide final success or failure handling based on normalized errors.

### UserConfigurationProvider

- Provide non-secret user settings.
- Return timezone, Notion database ID, Notion token secret ID, and LLM settings.
- Keep secret values out of DynamoDB and logs.

### SecretProvider

- Resolve secret values by secret ID.
- Return secret values only to components that need them.
- Prevent logging of secret values.

### DateRangeResolver

- Convert user timezone and target date into a half-open UTC interval.
- Use `[start_utc, end_utc)` semantics.
- Stay independent from AWS SDK and Notion SDK.
- Serve as a primary PBT target.

### ActivityRepository

- Query normalized activities by `user_id` and UTC date range.
- Return domain-level activity records, not raw DynamoDB items.
- Normalize data-access errors into application errors.

### ActivityFilter

- Validate that all activities match requested `user_id`.
- Validate that all activities fall within target UTC range.
- Sort or prepare activity records for report generation.
- Serve as a PBT target for user isolation and range filtering.

### DailyLogGenerator

- Build a timeline from normalized activities.
- Use `LLMClient` for timeline classification, topic extraction, and summary generation.
- Produce structured content for `ReportModelBuilder`.
- Avoid Notion-specific formatting.

### ReportModelBuilder

- Build a `DailyReport` domain model.
- Ensure report contains `user_id`, target date, generated timestamp, timeline, topics, and summary.
- Keep report output independent from Notion page/block structures.
- Serve as a PBT target for output structure invariants.

### LLMClient / BedrockLLMClient

- `LLMClient` defines the model-facing operations required by `DailyLogGenerator`.
- `BedrockLLMClient` implements those operations using Amazon Bedrock.
- Bedrock-specific errors are normalized before leaving the adapter.
- LLM input text must not be emitted to application logs.

### ReportPublisher / NotionReportPublisher

- `ReportPublisher` defines publication of a `DailyReport`.
- `NotionReportPublisher` converts `DailyReport` to Notion API payloads.
- Existing reports are not overwritten; publication creates a new page.
- Notion token must not be logged.

### StructuredLogger

- Emit timestamp, level, execution ID, `user_id`, target date, stage, and result.
- Exclude Notion token, raw LLM input, authentication material, and unnecessary personal data.
- Support auditability without leaking sensitive content.

### ErrorMapper

- Normalize external API and SDK errors.
- Preserve safe error metadata.
- Remove secrets and sensitive payloads from error messages.

## Security Notes

- Secret values are only read via `SecretProvider`.
- Domain components never receive AWS SDK clients or Notion SDK clients.
- Logs must use safe metadata only.
- IAM least privilege will be detailed in Infrastructure Design.

## PBT Notes

Primary PBT candidates:

- `DateRangeResolver`
- `ActivityFilter`
- `ReportModelBuilder`

Functional Design must document concrete properties for these components.
