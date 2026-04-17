# Notification System Design

A comprehensive notification system design integrating with Payment, User, and Order services. This document covers architecture, data flow, components, and operational concerns.

---

## 1. High-Level Architecture

Shows how the notification service sits between business services (Payment, User, Order) and delivery channels.

```mermaid
graph TB
    subgraph "Client Layer"
        WEB[Web App]
        MOB[Mobile App]
    end

    subgraph "API Gateway"
        GW[API Gateway / Load Balancer]
    end

    subgraph "Business Services"
        US[User Service]
        OS[Order Service]
        PS[Payment Service]
    end

    subgraph "Message Broker"
        MQ[(Kafka / RabbitMQ<br/>Event Bus)]
    end

    subgraph "Notification System"
        NS[Notification Service]
        TPL[Template Engine]
        PREF[Preference Manager]
        SCHED[Scheduler]
        RL[Rate Limiter]
        DEDUP[Deduplication]
    end

    subgraph "Channel Workers"
        EW[Email Worker]
        SW[SMS Worker]
        PW[Push Worker]
        IW[In-App Worker]
        WW[WhatsApp Worker]
    end

    subgraph "External Providers"
        SES[AWS SES / SendGrid]
        TWL[Twilio / Vonage]
        FCM[FCM / APNs]
        WAP[WhatsApp Business API]
    end

    subgraph "Data Layer"
        DB[(PostgreSQL<br/>Notifications + Logs)]
        CACHE[(Redis<br/>Cache + Rate Limits)]
        BLOB[(S3<br/>Attachments)]
    end

    WEB --> GW
    MOB --> GW
    GW --> US
    GW --> OS
    GW --> PS

    US -->|UserRegistered<br/>PasswordReset| MQ
    OS -->|OrderPlaced<br/>OrderShipped<br/>OrderDelivered| MQ
    PS -->|PaymentSucceeded<br/>PaymentFailed<br/>RefundIssued| MQ

    MQ --> NS
    NS --> TPL
    NS --> PREF
    NS --> SCHED
    NS --> RL
    NS --> DEDUP

    NS --> EW
    NS --> SW
    NS --> PW
    NS --> IW
    NS --> WW

    EW --> SES
    SW --> TWL
    PW --> FCM
    WW --> WAP
    IW --> WEB
    IW --> MOB

    NS --> DB
    NS --> CACHE
    TPL --> BLOB

    style NS fill:#4A90E2,color:#fff
    style MQ fill:#F5A623,color:#fff
    style DB fill:#7ED321,color:#fff
```

---

## 2. Sequence Diagram — Order Placed Notification Flow

End-to-end flow when a user places an order, gets paid, and receives confirmation across multiple channels.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant OS as Order Service
    participant PS as Payment Service
    participant MQ as Event Bus (Kafka)
    participant NS as Notification Service
    participant PREF as Preference Mgr
    participant TPL as Template Engine
    participant US as User Service
    participant EW as Email Worker
    participant PW as Push Worker
    participant SES as Email Provider
    participant FCM as Push Provider
    participant DB as Notification DB

    User->>OS: Place Order
    OS->>PS: Initiate Payment
    PS-->>OS: Payment Confirmed
    OS->>MQ: Publish "OrderPlaced" event<br/>{orderId, userId, items, total}
    PS->>MQ: Publish "PaymentSucceeded" event

    MQ->>NS: Consume events
    NS->>NS: Deduplicate (event_id)

    NS->>US: Get user details (userId)
    US-->>NS: {email, phone, deviceTokens, locale}

    NS->>PREF: Get user preferences
    PREF-->>NS: {email: ON, push: ON, sms: OFF}

    NS->>TPL: Render templates<br/>(locale, channel)
    TPL-->>NS: Rendered content per channel

    NS->>DB: Persist notification records (status=PENDING)

    par Email Delivery
        NS->>EW: Dispatch email job
        EW->>SES: Send email
        SES-->>EW: 202 Accepted
        EW->>DB: Update status=SENT
    and Push Delivery
        NS->>PW: Dispatch push job
        PW->>FCM: Send push
        FCM-->>PW: success
        PW->>DB: Update status=SENT
    end

    SES-->>NS: Webhook: delivered/opened
    FCM-->>NS: Webhook: delivered
    NS->>DB: Update status=DELIVERED

    User-->>EW: Opens email (tracked)
    EW->>DB: Update status=OPENED
```

---

## 3. Component / Class Diagram

Internal structure of the Notification Service showing core domain entities.

```mermaid
classDiagram
    class NotificationService {
        +processEvent(Event event)
        +sendNotification(Notification n)
        -resolveRecipient(userId)
        -applyPreferences(userId, channels)
    }

    class Notification {
        +UUID id
        +String userId
        +String eventType
        +NotificationStatus status
        +Channel channel
        +String templateKey
        +Map payload
        +Instant createdAt
        +Instant sentAt
        +int retryCount
    }

    class NotificationStatus {
        <<enumeration>>
        PENDING
        QUEUED
        SENT
        DELIVERED
        OPENED
        FAILED
        SUPPRESSED
    }

    class Channel {
        <<enumeration>>
        EMAIL
        SMS
        PUSH
        IN_APP
        WHATSAPP
    }

    class UserPreference {
        +String userId
        +Map~Channel,Boolean~ channelOptIn
        +Map~EventType,Boolean~ eventOptIn
        +String locale
        +String timezone
        +QuietHours quietHours
    }

    class Template {
        +String key
        +Channel channel
        +String locale
        +String subject
        +String body
        +int version
        +render(Map vars) String
    }

    class ChannelWorker {
        <<interface>>
        +send(Notification n)
        +supports(Channel c) boolean
    }

    class EmailWorker {
        -EmailProvider provider
        +send(Notification n)
    }

    class SmsWorker {
        -SmsProvider provider
        +send(Notification n)
    }

    class PushWorker {
        -PushProvider provider
        +send(Notification n)
    }

    class RateLimiter {
        +allow(userId, channel) boolean
        -RedisClient redis
    }

    class RetryHandler {
        +shouldRetry(Notification n) boolean
        +scheduleRetry(Notification n)
    }

    NotificationService --> Notification
    NotificationService --> UserPreference
    NotificationService --> Template
    NotificationService --> ChannelWorker
    NotificationService --> RateLimiter
    NotificationService --> RetryHandler
    Notification --> NotificationStatus
    Notification --> Channel
    ChannelWorker <|.. EmailWorker
    ChannelWorker <|.. SmsWorker
    ChannelWorker <|.. PushWorker
    UserPreference --> Channel
```

---

## 4. State Diagram — Notification Lifecycle

The lifecycle a single notification goes through, including failure and retry paths.

```mermaid
stateDiagram-v2
    [*] --> PENDING: Event received
    PENDING --> SUPPRESSED: User opted out<br/>or quiet hours
    PENDING --> QUEUED: Passed checks
    QUEUED --> SENDING: Worker picked up
    SENDING --> SENT: Provider accepted
    SENDING --> FAILED: Provider error
    FAILED --> RETRYING: Retry < max
    FAILED --> DEAD_LETTER: Retry exhausted
    RETRYING --> SENDING: Backoff elapsed
    SENT --> DELIVERED: Delivery webhook
    SENT --> BOUNCED: Bounce webhook
    DELIVERED --> OPENED: Open tracking
    OPENED --> CLICKED: Link clicked
    SUPPRESSED --> [*]
    DEAD_LETTER --> [*]
    BOUNCED --> [*]
    CLICKED --> [*]
    DELIVERED --> [*]
```

---

## 5. ER Diagram — Data Model

Database schema for notifications, preferences, templates, and delivery logs.

```mermaid
erDiagram
    USERS ||--o{ NOTIFICATIONS : receives
    USERS ||--|| USER_PREFERENCES : has
    USERS ||--o{ DEVICE_TOKENS : owns
    NOTIFICATIONS ||--o{ DELIVERY_ATTEMPTS : has
    NOTIFICATIONS }o--|| TEMPLATES : uses
    TEMPLATES ||--o{ TEMPLATE_VERSIONS : has
    EVENTS ||--o{ NOTIFICATIONS : triggers

    USERS {
        uuid user_id PK
        string email
        string phone
        string locale
        string timezone
        timestamp created_at
    }

    USER_PREFERENCES {
        uuid user_id PK
        jsonb channel_preferences
        jsonb event_preferences
        time quiet_hours_start
        time quiet_hours_end
        timestamp updated_at
    }

    DEVICE_TOKENS {
        uuid token_id PK
        uuid user_id FK
        string token
        string platform
        boolean active
        timestamp last_seen
    }

    EVENTS {
        uuid event_id PK
        string event_type
        string source_service
        jsonb payload
        timestamp occurred_at
    }

    NOTIFICATIONS {
        uuid notification_id PK
        uuid user_id FK
        uuid event_id FK
        string template_key FK
        string channel
        string status
        jsonb rendered_content
        int retry_count
        timestamp created_at
        timestamp sent_at
    }

    DELIVERY_ATTEMPTS {
        uuid attempt_id PK
        uuid notification_id FK
        int attempt_number
        string provider
        string provider_message_id
        string status
        text error_message
        timestamp attempted_at
    }

    TEMPLATES {
        string template_key PK
        string channel
        string locale
        int current_version
    }

    TEMPLATE_VERSIONS {
        string template_key PK
        int version_num PK
        string subject
        text body
        timestamp created_at
    }
```

---

## 6. Retry & Failure Handling Flow

How failed notifications are retried with exponential backoff and eventually moved to a dead letter queue.

```mermaid
flowchart TD
    START([Notification Send Attempt]) --> SEND[Call Provider API]
    SEND --> CHECK{Response?}

    CHECK -->|2xx Success| SUCCESS[Mark SENT<br/>Log delivery_attempt]
    CHECK -->|4xx Client Error| ANALYZE{Error Type?}
    CHECK -->|5xx Server Error| RETRY_DECISION
    CHECK -->|Timeout| RETRY_DECISION

    ANALYZE -->|Invalid recipient| SUPPRESS[Mark SUPPRESSED<br/>Add to suppression list]
    ANALYZE -->|Rate limited 429| BACKOFF[Wait + retry]
    ANALYZE -->|Auth/Config| ALERT[Page on-call<br/>Mark FAILED]

    RETRY_DECISION{Retry count<br/>&lt; max?}
    RETRY_DECISION -->|Yes| SCHEDULE[Schedule with<br/>exponential backoff<br/>2^n seconds + jitter]
    RETRY_DECISION -->|No| DLQ[Move to Dead Letter Queue<br/>Mark DEAD_LETTER]

    BACKOFF --> SCHEDULE
    SCHEDULE --> WAIT[Wait in delay queue]
    WAIT --> SEND

    DLQ --> NOTIFY_OPS[Alert Ops Team]
    DLQ --> MANUAL[Manual review/replay]

    SUCCESS --> END([Done])
    SUPPRESS --> END
    ALERT --> END
    NOTIFY_OPS --> END

    style SUCCESS fill:#7ED321,color:#fff
    style DLQ fill:#D0021B,color:#fff
    style SUPPRESS fill:#F5A623,color:#fff
    style ALERT fill:#D0021B,color:#fff
```

---

## Key Design Decisions

**Event-driven via message broker.** Business services publish domain events to Kafka/RabbitMQ. The notification service consumes them, decoupling producers from notification logic. Adding a new notification type doesn't touch the Order or Payment service.

**Channel workers are pluggable.** Each channel (email, SMS, push) has its own worker pool. This isolates failures — if Twilio is down, email and push keep flowing. It also lets you scale workers independently based on volume.

**Idempotency via event_id.** Every event carries a unique ID. The notification service deduplicates on this before processing, so retries from the broker don't cause duplicate sends.

**User preferences as a first-class concern.** The Preference Manager checks opt-in status, quiet hours, and locale before any send. This keeps compliance (GDPR, CAN-SPAM) and UX concerns out of business services.

**Templates are versioned.** Marketing/product can update templates without code deploys. Versioning allows A/B testing and rollback.

**Retry with exponential backoff + jitter.** Transient failures retry with `2^n + random(0, 1000ms)` to avoid thundering herd. Permanent failures (invalid email) skip retry and go to a suppression list.

**Webhook ingestion for delivery status.** Providers like SES, Twilio, and FCM POST back delivery/bounce/open events. These update notification status for analytics and bounce handling.

**Rate limiting per user per channel.** Redis-backed token bucket prevents notification spam (e.g., max 10 push notifications per user per hour).
