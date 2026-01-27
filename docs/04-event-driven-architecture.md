# Event-Driven Architecture

## Overview

The ATS platform uses Apache Kafka as its event backbone, enabling loose coupling between services, reliable message delivery, and comprehensive audit trails for compliance requirements.

## Why Kafka over Redis Pub/Sub?

| Requirement | Redis Pub/Sub | Kafka | Our Need |
|-------------|---------------|-------|----------|
| Message Persistence | ❌ Fire-and-forget | ✅ Durable log | Critical for ATS |
| Replay Capability | ❌ No | ✅ Offset-based | Compliance audits |
| Consumer Groups | ⚠️ Limited | ✅ Native | Multiple AI services |
| Ordering Guarantee | ❌ No | ✅ Per partition | Application workflows |
| Dead Letter Queue | ❌ Manual | ✅ Built-in | AI processing failures |

**Conclusion**: Kafka is essential for enterprise-grade reliability, especially for candidate applications and compliance requirements.

## Event Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         APACHE KAFKA CLUSTER                             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                         TOPICS                                   │    │
│  ├─────────────────────────────────────────────────────────────────┤    │
│  │                                                                  │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │    │
│  │  │ candidate.       │  │ candidate.       │  │ job.          │  │    │
│  │  │ applications     │  │ profiles         │  │ postings      │  │    │
│  │  │ ──────────────── │  │ ──────────────── │  │ ───────────── │  │    │
│  │  │ P:3 R:2 Ret:30d  │  │ P:3 R:2 Ret:30d  │  │ P:3 R:2 Ret:7d│  │    │
│  │  └──────────────────┘  └──────────────────┘  └───────────────┘  │    │
│  │                                                                  │    │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │    │
│  │  │ resume.parsing   │  │ ranking.         │  │ notifications │  │    │
│  │  │ .queue           │  │ queue            │  │ .outbound     │  │    │
│  │  │ ──────────────── │  │ ──────────────── │  │ ───────────── │  │    │
│  │  │ P:6 R:2 Ret:7d   │  │ P:6 R:2 Ret:3d   │  │ P:3 R:2 Ret:3d│  │    │
│  │  └──────────────────┘  └──────────────────┘  └───────────────┘  │    │
│  │                                                                  │    │
│  │  ┌──────────────────┐  ┌──────────────────┐                     │    │
│  │  │ interview.       │  │ audit.           │                     │    │
│  │  │ events           │  │ trail            │                     │    │
│  │  │ ──────────────── │  │ ──────────────── │                     │    │
│  │  │ P:3 R:2 Ret:30d  │  │ P:3 R:3 Ret:365d │                     │    │
│  │  └──────────────────┘  └──────────────────┘                     │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │   Schema Registry    │  │   Dead Letter Queue  │                     │
│  │   (Avro/JSON Schema) │  │   (Failed Messages)  │                     │
│  └──────────────────────┘  └──────────────────────┘                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

P = Partitions, R = Replication Factor, Ret = Retention
```

## Topic Configuration

| Topic | Partitions | Retention | Purpose |
|-------|------------|-----------|---------|
| `candidate.applications` | 3 | 30 days | Application submissions, status changes |
| `candidate.profiles` | 3 | 30 days | Profile updates, resume uploads |
| `job.postings` | 3 | 7 days | Job CRUD operations |
| `resume.parsing.queue` | 6 | 7 days | AI processing queue (high volume) |
| `ranking.queue` | 6 | 3 days | Candidate-job matching queue |
| `notifications.outbound` | 3 | 3 days | Email/SMS/Push triggers |
| `interview.events` | 3 | 30 days | Interview scheduling, feedback |
| `audit.trail` | 3 | 365 days | Compliance logging (extended retention) |

## Event Schema

### Application Submitted Event

```json
{
  "eventId": "evt_uuid_001",
  "eventType": "APPLICATION_SUBMITTED",
  "eventTime": "2025-01-27T10:30:00Z",
  "version": "1.0",
  "tenantId": "tenant_uuid",
  "correlationId": "corr_uuid_001",
  "payload": {
    "applicationId": "app_uuid_001",
    "candidateId": "cand_uuid_001",
    "jobId": "job_uuid_001",
    "source": "LINKEDIN",
    "resumeUrl": "https://storage.../resume.pdf",
    "appliedAt": "2025-01-27T10:30:00Z"
  },
  "metadata": {
    "userId": "user_uuid",
    "userAgent": "Mozilla/5.0...",
    "ipAddress": "192.168.1.1"
  }
}
```

### Avro Schema (Schema Registry)

```json
{
  "type": "record",
  "name": "ApplicationEvent",
  "namespace": "com.ats.events.candidate",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "eventType", "type": {"type": "enum", "name": "EventType", 
      "symbols": ["APPLICATION_SUBMITTED", "APPLICATION_REVIEWED", "APPLICATION_REJECTED", "APPLICATION_HIRED"]}},
    {"name": "eventTime", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "version", "type": "string", "default": "1.0"},
    {"name": "tenantId", "type": "string"},
    {"name": "correlationId", "type": ["null", "string"], "default": null},
    {"name": "payload", "type": {
      "type": "record",
      "name": "ApplicationPayload",
      "fields": [
        {"name": "applicationId", "type": "string"},
        {"name": "candidateId", "type": "string"},
        {"name": "jobId", "type": "string"},
        {"name": "source", "type": "string"},
        {"name": "resumeUrl", "type": ["null", "string"], "default": null}
      ]
    }}
  ]
}
```

## Producer Implementation

### Spring Kafka Producer

```java
@Service
@Slf4j
public class ApplicationEventProducer {

    private final KafkaTemplate<String, ApplicationEvent> kafkaTemplate;
    
    private static final String TOPIC = "candidate.applications";

    public CompletableFuture<SendResult<String, ApplicationEvent>> publishApplicationSubmitted(
            Application application) {
        
        ApplicationEvent event = ApplicationEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .eventType(EventType.APPLICATION_SUBMITTED)
            .eventTime(Instant.now())
            .tenantId(TenantContext.getTenantId())
            .correlationId(MDC.get("correlationId"))
            .payload(ApplicationPayload.from(application))
            .build();

        // Partition by candidateId for ordering guarantee per candidate
        return kafkaTemplate.send(TOPIC, application.getCandidateId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish event: {}", event.getEventId(), ex);
                } else {
                    log.info("Published event {} to partition {} offset {}", 
                        event.getEventId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### Producer Configuration

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      acks: all  # Wait for all replicas
      retries: 3
      properties:
        enable.idempotence: true  # Exactly-once semantics
        max.in.flight.requests.per.connection: 5
        schema.registry.url: ${SCHEMA_REGISTRY_URL}
```

## Consumer Implementation

### Resume Parsing Consumer

```java
@Service
@Slf4j
public class ResumeParsingConsumer {

    private final ResumeParserService parserService;
    private final CandidateProfileProducer profileProducer;

    @KafkaListener(
        topics = "resume.parsing.queue",
        groupId = "resume-parser-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void processResume(
            @Payload ResumeParsingRequest request,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            Acknowledgment ack) {
        
        try {
            // Set tenant context for database operations
            TenantContext.setTenantId(request.getTenantId());
            
            log.info("Processing resume for candidate {} from partition {}", 
                request.getCandidateId(), partition);
            
            // Parse resume using AI service
            ParsedResume result = parserService.parse(request.getResumeUrl());
            
            // Publish parsed profile to candidate.profiles topic
            profileProducer.publishProfileUpdated(request.getCandidateId(), result);
            
            // Manual acknowledgment after successful processing
            ack.acknowledge();
            
        } catch (TransientException e) {
            // Retriable error - don't ack, will be reprocessed
            log.warn("Transient error processing resume, will retry", e);
            throw e;
            
        } catch (Exception e) {
            // Non-retriable - send to DLQ
            log.error("Failed to parse resume, sending to DLQ", e);
            sendToDeadLetterQueue(request, e);
            ack.acknowledge(); // Ack to prevent infinite retry
            
        } finally {
            TenantContext.clear();
        }
    }
}
```

### Consumer Configuration

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      enable-auto-commit: false  # Manual ack for reliability
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        schema.registry.url: ${SCHEMA_REGISTRY_URL}
        specific.avro.reader: true
    listener:
      ack-mode: manual
      concurrency: 3
```

## Event Flows

### Application Submission Flow

```
┌───────────┐     ┌───────────────┐     ┌─────────────────────────────────┐
│ Candidate │────▶│ Candidate     │────▶│ candidate.applications          │
│ Applies   │     │ Service       │     │ (Kafka Topic)                   │
└───────────┘     └───────────────┘     └─────────────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        ▼                             ▼                             ▼
              ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
              │ Notification    │          │ Analytics       │          │ Resume Parser   │
              │ Service         │          │ Service         │          │ Service         │
              │ ─────────────── │          │ ─────────────── │          │ ─────────────── │
              │ Email recruiter │          │ Update metrics  │          │ Queue for AI    │
              │ about new app   │          │ and dashboards  │          │ parsing         │
              └─────────────────┘          └─────────────────┘          └─────────────────┘
                                                                                 │
                                                                                 ▼
                                                                        ┌─────────────────┐
                                                                        │ resume.parsing  │
                                                                        │ .queue          │
                                                                        └─────────────────┘
                                                                                 │
                                                                                 ▼
                                                                        ┌─────────────────┐
                                                                        │ AI Resume       │
                                                                        │ Parser          │
                                                                        │ ─────────────── │
                                                                        │ Extract skills, │
                                                                        │ experience      │
                                                                        └─────────────────┘
                                                                                 │
                                                                                 ▼
                                                                        ┌─────────────────┐
                                                                        │ ranking.queue   │
                                                                        └─────────────────┘
                                                                                 │
                                                                                 ▼
                                                                        ┌─────────────────┐
                                                                        │ Ranking         │
                                                                        │ Service         │
                                                                        │ ─────────────── │
                                                                        │ Match candidate │
                                                                        │ to job          │
                                                                        └─────────────────┘
```

## Dead Letter Queue Strategy

```java
@Configuration
public class KafkaErrorHandling {

    @Bean
    public DeadLetterPublishingRecoverer deadLetterRecoverer(
            KafkaTemplate<String, Object> template) {
        
        return new DeadLetterPublishingRecoverer(template,
            (record, ex) -> {
                // Route to topic-specific DLQ
                String dlqTopic = record.topic() + ".dlq";
                return new TopicPartition(dlqTopic, record.partition());
            });
    }

    @Bean
    public DefaultErrorHandler errorHandler(
            DeadLetterPublishingRecoverer recoverer) {
        
        // Retry 3 times with exponential backoff, then DLQ
        return new DefaultErrorHandler(recoverer, 
            new ExponentialBackOff(1000L, 2.0)); // 1s, 2s, 4s
    }
}
```

## Monitoring & Alerting

### Key Metrics to Monitor

| Metric | Alert Threshold | Action |
|--------|-----------------|--------|
| Consumer Lag | > 10,000 messages | Scale consumers |
| DLQ Message Count | > 100/hour | Investigate failures |
| Producer Error Rate | > 1% | Check Kafka health |
| Processing Latency P99 | > 5 seconds | Optimize consumers |

### Grafana Dashboard Queries

```promql
# Consumer lag by group
kafka_consumer_group_lag{group="resume-parser-service"}

# Messages processed per second
rate(kafka_consumer_records_consumed_total[5m])

# DLQ message rate
rate(kafka_producer_record_send_total{topic=~".*\\.dlq"}[5m])
```

## Best Practices

1. **Idempotent Consumers**: Use `eventId` to deduplicate
2. **Partition Key Strategy**: Use `candidateId` for candidate events, `jobId` for job events
3. **Schema Evolution**: Use Avro with backward compatibility
4. **Monitoring**: Alert on consumer lag and DLQ growth
5. **Retention**: Balance storage costs with replay requirements

## Next Steps

- Set up Kafka Connect for CDC from MongoDB
- Implement KSQL for real-time analytics streams
- Configure cross-region replication for DR
