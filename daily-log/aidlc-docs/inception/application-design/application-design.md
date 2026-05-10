# Application Design: Daily Log System

## Summary

The Daily Log System is designed as a Python AWS Lambda application with a thin Lambda handler, a central `DailyLogService`, domain components for pure business logic, and adapters for AWS / Notion integrations.

The design intentionally separates domain logic from AWS SDK and Notion SDK usage. This supports Security Baseline requirements, keeps sensitive dependencies isolated, and enables PBT for date, filtering, and report model logic.

## Key Decisions

- Use medium-grained components.
- Use `DailyLogService` as the central application service.
- Use `LLMClient` as an interface with `BedrockLLMClient` as the initial adapter.
- Use `ReportPublisher` as an interface with `NotionReportPublisher` as the initial adapter.
- Store user configuration in DynamoDB and Notion token in Secrets Manager.
- Use `DailyReport` as a domain model before Notion conversion.
- Normalize external API errors in adapters.
- Keep PBT-relevant logic pure-function oriented.
- Keep domain logic independent from AWS and Notion SDKs.

## Component Set

See `components.md` for complete component definitions.

Primary components:

- `ScheduleHandler`
- `DailyLogService`
- `UserConfigurationProvider`
- `SecretProvider`
- `DateRangeResolver`
- `ActivityRepository`
- `ActivityFilter`
- `DailyLogGenerator`
- `LLMClient`
- `ReportModelBuilder`
- `ReportPublisher`
- `StructuredLogger`
- `ErrorMapper`

Primary adapters:

- `DynamoDBUserConfigurationProvider`
- `SecretsManagerSecretProvider`
- `DynamoDBActivityRepository`
- `BedrockLLMClient`
- `NotionReportPublisher`

## Service Design

See `services.md` for service orchestration.

`DailyLogService` executes a single use case:

1. Load user configuration.
2. Resolve target date range.
3. Load activities.
4. Defensively filter activities.
5. Generate timeline, topics, and summary.
6. Build `DailyReport`.
7. Retrieve Notion token.
8. Publish report to Notion.
9. Log success or failure safely.

## Method Design

See `component-methods.md` for method signatures and domain models.

Important domain models:

- `DateRange`
- `UserConfiguration`
- `ActivityRecord`
- `TimelineItem`
- `DailyReportContent`
- `DailyReport`
- `DailyLogResult`

## Dependency Design

See `component-dependency.md` for dependency matrix and data flow.

Dependency direction:

```text
Handler -> Application Service -> Domain Components / Ports <- Adapters
```

## Security Baseline Compliance

### Applicable at Application Design

- **SECURITY-03 Application-Level Logging**: Design includes `StructuredLogger` and explicit sensitive-field exclusions.
- **SECURITY-06 Least-Privilege Access Policies**: Design isolates adapter responsibilities, preparing precise IAM permissions in Infrastructure Design.
- **SECURITY-12 Credential Management**: Notion token is stored in Secrets Manager and never in DynamoDB.
- **SECURITY-13 Data Integrity**: Critical generated report metadata includes `user_id`, target date, and generated timestamp.

### Not Applicable or Deferred

- Network intermediary logging and HTTP security headers are not applicable at this stage because the system has no public HTTP endpoint.
- IAM policy details are deferred to Infrastructure Design.
- Dependency pinning and vulnerability scanning are deferred to Code Generation / Build and Test.

### Blocking Findings

None.

## PBT Compliance

### Applicable at Application Design

- PBT-relevant logic is isolated into pure-function oriented components:
  - `DateRangeResolver`
  - `ActivityFilter`
  - `ReportModelBuilder`

### Deferred

- Concrete properties, generators, and Hypothesis test design are deferred to Functional Design and Code Generation.

### Blocking Findings

None.

## Coverage

| Requirement / Story Area | Design Coverage |
|---|---|
| Target date auto generation | `ScheduleHandler`, `DailyLogService`, `DateRangeResolver` |
| User timezone handling | `UserConfigurationProvider`, `DateRangeResolver` |
| DynamoDB activity loading | `ActivityRepository`, `DynamoDBActivityRepository` |
| User isolation | `ActivityRepository`, `ActivityFilter`, `DailyReport` metadata |
| LLM classification and summary | `DailyLogGenerator`, `LLMClient`, `BedrockLLMClient` |
| Notion publication | `ReportPublisher`, `NotionReportPublisher` |
| Existing page non-overwrite | `NotionReportPublisher` creates new page |
| Secret non-logging | `SecretProvider`, `StructuredLogger`, `NotionReportPublisher` |
| Failure handling | `ErrorMapper`, adapter error normalization, `DailyLogService` |

## Next Phase Notes

Functional Design should define:

- Exact target date selection rule.
- Activity filtering rules.
- Timeline ordering rules.
- LLM prompt input contract.
- `DailyReport` field-level validation.
- PBT properties for date range, activity filtering, and report model building.

NFR Design should define:

- Structured logging schema.
- Secret redaction rules.
- PBT framework configuration.
- Dependency pinning and scanning approach.

Infrastructure Design should define:

- Lambda runtime and environment variables.
- EventBridge schedule.
- DynamoDB tables and access patterns.
- Secrets Manager secret naming.
- IAM least-privilege policies.
- CloudWatch log group retention and alarms.
