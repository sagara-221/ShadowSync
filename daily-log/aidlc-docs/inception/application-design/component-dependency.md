# Component Dependency: Daily Log System

## Dependency Direction

The design uses a ports-and-adapters direction:

```text
Lambda Handler
    -> DailyLogService
        -> Domain Components
        -> Ports
            <- Adapters
```

Domain logic does not depend on AWS SDK or Notion SDK. External dependencies are isolated behind adapters.

## Dependency Matrix

| Component | Depends On | Dependency Type |
|---|---|---|
| `ScheduleHandler` | `DailyLogService` | Direct call |
| `DailyLogService` | `UserConfigurationProvider` | Port |
| `DailyLogService` | `SecretProvider` | Port |
| `DailyLogService` | `DateRangeResolver` | Domain component |
| `DailyLogService` | `ActivityRepository` | Port |
| `DailyLogService` | `ActivityFilter` | Domain component |
| `DailyLogService` | `DailyLogGenerator` | Domain service |
| `DailyLogService` | `ReportModelBuilder` | Domain component |
| `DailyLogService` | `ReportPublisher` | Port |
| `DailyLogService` | `StructuredLogger` | Cross-cutting |
| `DailyLogGenerator` | `LLMClient` | Port |
| `DynamoDBUserConfigurationProvider` | AWS SDK DynamoDB | Adapter dependency |
| `SecretsManagerSecretProvider` | AWS SDK Secrets Manager | Adapter dependency |
| `DynamoDBActivityRepository` | AWS SDK DynamoDB | Adapter dependency |
| `BedrockLLMClient` | AWS SDK Bedrock | Adapter dependency |
| `NotionReportPublisher` | Notion API client | Adapter dependency |

## Runtime Flow

```text
EventBridge
    -> ScheduleHandler
        -> DailyLogService
            -> UserConfigurationProvider
                -> DynamoDBUserConfigurationProvider
            -> DateRangeResolver
            -> ActivityRepository
                -> DynamoDBActivityRepository
            -> ActivityFilter
            -> DailyLogGenerator
                -> LLMClient
                    -> BedrockLLMClient
            -> ReportModelBuilder
            -> SecretProvider
                -> SecretsManagerSecretProvider
            -> ReportPublisher
                -> NotionReportPublisher
            -> StructuredLogger
```

## Data Flow

1. EventBridge triggers Lambda.
2. `ScheduleHandler` extracts execution input.
3. `DailyLogService` loads `UserConfiguration`.
4. `DateRangeResolver` produces `DateRange`.
5. `ActivityRepository` returns `ActivityRecord` list.
6. `ActivityFilter` validates and orders records.
7. `DailyLogGenerator` produces `DailyReportContent`.
8. `ReportModelBuilder` produces `DailyReport`.
9. `SecretProvider` resolves Notion token.
10. `ReportPublisher` publishes new Notion page.
11. `StructuredLogger` records success or failure.

## Communication Patterns

- In-process function/class calls within Lambda.
- AWS SDK calls only in AWS adapters.
- HTTP Notion API calls only in `NotionReportPublisher`.
- Bedrock API calls only in `BedrockLLMClient`.
- Structured logs emitted to CloudWatch Logs through runtime logging.

## Security Boundaries

- Notion token crosses only from `SecretProvider` to `ReportPublisher`.
- Raw activity payloads should not be logged.
- LLM prompt content should not be logged.
- `user_id` is required in activity query and defensive filtering.
- Domain components are testable without cloud credentials.

## PBT Boundaries

| Component | Testable Property Candidates |
|---|---|
| `DateRangeResolver` | UTC range is half-open, start is before end, adjacent local dates do not overlap. |
| `ActivityFilter` | Output contains only requested `user_id`; output timestamps are inside range. |
| `ReportModelBuilder` | Report has required fields; timeline remains ordered; user/date metadata is preserved. |

Detailed properties will be finalized in Functional Design.
