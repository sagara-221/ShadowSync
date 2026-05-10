# Component Methods: Daily Log System

## Domain Models

```python
UserId = str
DeviceId = str
SecretId = str
TimezoneName = str

@dataclass(frozen=True)
class DateRange:
    target_date: date
    timezone: TimezoneName
    start_utc: datetime
    end_utc: datetime

@dataclass(frozen=True)
class UserConfiguration:
    user_id: UserId
    timezone: TimezoneName
    notion_database_id: str
    notion_token_secret_id: SecretId
    bedrock_model_id: str | None

@dataclass(frozen=True)
class ActivityRecord:
    user_id: UserId
    timestamp: datetime
    activity_type: str
    title: str | None
    summary: str | None
    metadata: dict[str, Any]

@dataclass(frozen=True)
class TimelineItem:
    timestamp: datetime
    title: str
    description: str
    source_type: str

@dataclass(frozen=True)
class DailyReport:
    user_id: UserId
    target_date: date
    timezone: TimezoneName
    generated_at: datetime
    timeline: list[TimelineItem]
    topics: list[str]
    summary: str
```

## ScheduleHandler

```python
def lambda_handler(event: dict[str, Any], context: Any) -> dict[str, Any]
```

- **Purpose**: Lambda entry point for EventBridge.
- **Input**: EventBridge payload and Lambda context.
- **Output**: Status dictionary.
- **Notes**: Performs dependency wiring and delegates business flow to `DailyLogService`.

## DailyLogService

```python
class DailyLogService:
    def generate_daily_log(
        self,
        user_id: UserId,
        target_date: date | None,
        execution_id: str,
    ) -> DailyLogResult: ...
```

- **Purpose**: Execute daily report generation for one user and one target date.
- **Input**: User ID, optional target date, execution ID.
- **Output**: `DailyLogResult`.
- **Notes**: Owns orchestration, not detailed formatting or SDK calls.

```python
@dataclass(frozen=True)
class DailyLogResult:
    user_id: UserId
    target_date: date
    status: Literal["success", "failed"]
    published_page_id: str | None
    error_code: str | None
```

## UserConfigurationProvider

```python
class UserConfigurationProvider(Protocol):
    def get_configuration(self, user_id: UserId) -> UserConfiguration: ...
```

- **Purpose**: Load non-secret user configuration.
- **Adapter**: `DynamoDBUserConfigurationProvider`.
- **Error behavior**: Raises normalized configuration errors.

## SecretProvider

```python
class SecretProvider(Protocol):
    def get_secret_value(self, secret_id: SecretId) -> str: ...
```

- **Purpose**: Resolve sensitive values such as Notion integration token.
- **Adapter**: `SecretsManagerSecretProvider`.
- **Security**: Returned values must never be logged.

## DateRangeResolver

```python
class DateRangeResolver:
    def resolve(
        self,
        target_date: date | None,
        timezone: TimezoneName,
        now_utc: datetime,
    ) -> DateRange: ...
```

- **Purpose**: Determine target date and UTC half-open interval.
- **PBT relevance**: Start before end, no overlap between adjacent days, stable timezone conversion.

## ActivityRepository

```python
class ActivityRepository(Protocol):
    def list_activities(
        self,
        user_id: UserId,
        date_range: DateRange,
    ) -> list[ActivityRecord]: ...
```

- **Purpose**: Load normalized activities.
- **Adapter**: `DynamoDBActivityRepository`.
- **Security**: Query must include `user_id`.

## ActivityFilter

```python
class ActivityFilter:
    def filter_for_user_and_range(
        self,
        activities: Iterable[ActivityRecord],
        user_id: UserId,
        date_range: DateRange,
    ) -> list[ActivityRecord]: ...
```

- **Purpose**: Defensive filtering and ordering.
- **PBT relevance**: Output contains only requested `user_id` and timestamps in range.

## DailyLogGenerator

```python
class DailyLogGenerator:
    def generate(
        self,
        activities: list[ActivityRecord],
        user_config: UserConfiguration,
        date_range: DateRange,
    ) -> DailyReportContent: ...
```

- **Purpose**: Build timeline and use LLM for classification, topics, and summary.
- **Output**: Intermediate report content consumed by `ReportModelBuilder`.

```python
@dataclass(frozen=True)
class DailyReportContent:
    timeline: list[TimelineItem]
    topics: list[str]
    summary: str
```

## LLMClient

```python
class LLMClient(Protocol):
    def classify_timeline(
        self,
        activities: list[ActivityRecord],
        model_id: str | None,
    ) -> list[TimelineItem]: ...

    def extract_topics(
        self,
        activities: list[ActivityRecord],
        model_id: str | None,
    ) -> list[str]: ...

    def generate_summary(
        self,
        activities: list[ActivityRecord],
        topics: list[str],
        model_id: str | None,
    ) -> str: ...
```

- **Adapter**: `BedrockLLMClient`.
- **Security**: Must not log raw prompt or full activity payload.

## ReportModelBuilder

```python
class ReportModelBuilder:
    def build(
        self,
        user_config: UserConfiguration,
        date_range: DateRange,
        content: DailyReportContent,
        generated_at: datetime,
    ) -> DailyReport: ...
```

- **Purpose**: Build domain `DailyReport`.
- **PBT relevance**: Required fields exist; timeline remains ordered; model is SDK-independent.

## ReportPublisher

```python
class ReportPublisher(Protocol):
    def publish(
        self,
        report: DailyReport,
        destination: ReportDestination,
    ) -> PublishResult: ...
```

```python
@dataclass(frozen=True)
class ReportDestination:
    notion_database_id: str
    notion_token: str

@dataclass(frozen=True)
class PublishResult:
    page_id: str
    url: str | None
```

- **Adapter**: `NotionReportPublisher`.
- **Behavior**: Creates new page; does not overwrite existing page.

## StructuredLogger

```python
class StructuredLogger:
    def info(self, event_name: str, fields: dict[str, Any]) -> None: ...
    def warning(self, event_name: str, fields: dict[str, Any]) -> None: ...
    def error(self, event_name: str, fields: dict[str, Any]) -> None: ...
```

- **Purpose**: Centralized structured logging.
- **Security**: Redacts or rejects known sensitive fields.

## ErrorMapper

```python
class ErrorMapper:
    def normalize(self, error: Exception, component: str) -> DailyLogError: ...
```

```python
@dataclass(frozen=True)
class DailyLogError(Exception):
    code: str
    component: str
    safe_message: str
    retryable: bool
```

- **Purpose**: Convert adapter-specific exceptions to safe application errors.
- **Notes**: Detailed retry behavior is deferred to later phases.
