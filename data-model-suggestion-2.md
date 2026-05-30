# Data Model Suggestion 2: Event-Sourced / CQRS Architecture

> Project: Technical Interview Platform (#295)
> Approach: Event Sourcing with Command Query Responsibility Segregation (CQRS)

## Summary

An event-sourced architecture where every state change across the interview lifecycle is captured as an immutable event in an append-only event store. The system separates write operations (commands) from read operations (queries) using CQRS, with materialized read models (projections) optimized for specific query patterns. This approach is a natural fit for a technical interview platform because the core domain -- live collaborative coding sessions, real-time evaluation, and audit-critical hiring decisions -- is inherently event-driven. Every keystroke, code execution, evaluator action, and proctoring alert is an event that must be recorded, replayed, and analysed.

---

## Architecture Overview

```
                    ┌──────────────┐
                    │  Commands    │
                    │  (Write Side)│
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Aggregates  │
                    │  (Domain     │
                    │   Logic)     │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Event Store │  ◄── Append-only, immutable
                    │  (Source of  │
                    │   Truth)     │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──┐  ┌──────▼──┐  ┌──────▼──────┐
       │ Session  │  │ Scoring │  │ Analytics   │
       │ Read     │  │ Read    │  │ Read Model  │
       │ Model    │  │ Model   │  │ (ClickHouse)│
       └──────────┘  └─────────┘  └─────────────┘
              │            │            │
       ┌──────▼────────────▼────────────▼──┐
       │         Query Side (Reads)        │
       └───────────────────────────────────┘
```

---

## Key Aggregates and Event Streams

### Aggregate 1: InterviewSession

The central aggregate managing the lifecycle of a single interview session.

```
Stream: interview-session-{sessionId}

Events:
├── SessionScheduled        { sessionId, applicationId, sessionType, scheduledAt, participants[] }
├── SessionStarted          { sessionId, startedAt, videoRoomId }
├── ParticipantJoined       { sessionId, participantId, role, joinedAt }
├── ParticipantLeft         { sessionId, participantId, leftAt }
├── CodeChanged             { sessionId, fileId, language, delta, authorId, timestamp }
├── CodeExecutionRequested  { sessionId, submissionId, language, sourceCode, stdin }
├── CodeExecutionCompleted  { sessionId, submissionId, stdout, stderr, exitCode, executionTimeMs, memoryUsedKb }
├── WhiteboardUpdated       { sessionId, snapshotId, deltaOps[], authorId, timestamp }
├── ProblemAssigned         { sessionId, problemId, assignedBy }
├── ProblemHintRevealed     { sessionId, problemId, hintLevel }
├── SessionPaused           { sessionId, pausedAt, reason }
├── SessionResumed          { sessionId, resumedAt }
├── SessionCompleted        { sessionId, endedAt, durationMins }
└── SessionCancelled        { sessionId, cancelledAt, reason }
```

### Aggregate 2: Evaluation

Manages scoring and feedback for a session, separate from the session itself.

```
Stream: evaluation-{sessionId}-{evaluatorId}

Events:
├── EvaluationStarted       { sessionId, evaluatorId, rubricId }
├── CriterionScored         { sessionId, evaluatorId, criterionId, score, notes }
├── OverallDecisionRecorded  { sessionId, evaluatorId, decision, privateNotes }
├── EvaluationSubmitted     { sessionId, evaluatorId, submittedAt }
├── EvaluationRevised       { sessionId, evaluatorId, revisedAt, changes[] }
└── AIEvaluationGenerated   { sessionId, evaluationType, modelVersion, score, summary, evidence }
```

### Aggregate 3: Candidate

Tracks candidate lifecycle across applications.

```
Stream: candidate-{candidateId}

Events:
├── CandidateRegistered     { candidateId, email, displayName }
├── CandidateProfileUpdated { candidateId, fields[] }
├── ApplicationSubmitted    { applicationId, candidateId, jobPositionId, source }
├── ApplicationStatusChanged { applicationId, oldStatus, newStatus, changedBy }
├── CandidateAnonymized     { candidateId, anonymizedId }
└── CandidateDataDeleted    { candidateId, deletedAt, reason }  -- GDPR erasure
```

### Aggregate 4: ProblemLibrary

Manages the assessment problem catalog.

```
Stream: problem-{problemId}

Events:
├── ProblemCreated          { problemId, title, description, difficulty, problemType, starterCode }
├── ProblemUpdated          { problemId, fieldsChanged[] }
├── TestCaseAdded           { problemId, testCaseId, input, expectedOutput, isHidden }
├── ProblemTagged           { problemId, tagId, tagName }
├── ProblemVariantGenerated { problemId, variantId, difficulty, generatedBy }
├── ProblemArchived         { problemId, archivedAt }
└── ProblemUsageRecorded    { problemId, sessionId, candidatePerformance }
```

### Aggregate 5: Proctoring

Dedicated stream for security-sensitive monitoring events.

```
Stream: proctor-{sessionId}

Events:
├── ProctorSessionStarted   { sessionId, proctorConfig }
├── TabSwitchDetected       { sessionId, timestamp, tabTitle, duration }
├── FaceDetectionAlert      { sessionId, timestamp, alertType, confidence }
├── VoiceDetectionAlert     { sessionId, timestamp, alertType }
├── CopyPasteDetected       { sessionId, timestamp, contentLength, source }
├── DeviceChangeDetected    { sessionId, timestamp, deviceInfo }
├── ScreenShareStopped      { sessionId, timestamp }
└── ProctorViolationFlagged { sessionId, timestamp, severity, details }
```

---

## Event Store Schema

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    global_position   BIGSERIAL PRIMARY KEY,
    stream_id         VARCHAR(255) NOT NULL,
    stream_position   INTEGER NOT NULL,
    event_type        VARCHAR(100) NOT NULL,
    event_data        JSONB NOT NULL,
    metadata          JSONB,              -- correlation_id, causation_id, actor_id, ip_address
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, stream_position)
);

CREATE INDEX idx_events_stream ON events(stream_id, stream_position);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);

-- Snapshot store for aggregate rehydration optimization
CREATE TABLE snapshots (
    stream_id         VARCHAR(255) PRIMARY KEY,
    stream_position   INTEGER NOT NULL,
    snapshot_data     JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Alternative: EventStoreDB

For production deployments, consider EventStoreDB (open-source, purpose-built event store) instead of a PostgreSQL-based event store:

```
EventStoreDB streams:
  interview-session-{id}    → all session lifecycle events
  evaluation-{id}-{userId}  → all evaluation events per evaluator
  candidate-{id}            → candidate lifecycle
  problem-{id}              → problem library events
  proctor-{id}              → proctoring events
  $ce-InterviewSession      → category projection (all session events)
  $et-CodeChanged           → event-type projection (all code changes)
```

---

## Read Models (Projections)

### Projection 1: Session Dashboard

Materialized in PostgreSQL for the interviewer dashboard.

```sql
CREATE TABLE rm_session_dashboard (
    session_id        UUID PRIMARY KEY,
    application_id    UUID,
    candidate_name    VARCHAR(255),
    session_type      VARCHAR(30),
    status            VARCHAR(20),
    scheduled_at      TIMESTAMPTZ,
    started_at        TIMESTAMPTZ,
    ended_at          TIMESTAMPTZ,
    participant_count INTEGER,
    problems_assigned INTEGER,
    submissions_count INTEGER,
    proctor_alerts    INTEGER,
    last_updated      TIMESTAMPTZ
);
```

### Projection 2: Candidate Scorecard

Aggregates all evaluations for a candidate across sessions.

```sql
CREATE TABLE rm_candidate_scorecard (
    application_id    UUID,
    session_id        UUID,
    evaluator_id      UUID,
    evaluator_name    VARCHAR(255),
    criterion_name    VARCHAR(255),
    score             INTEGER,
    max_score         INTEGER,
    decision          VARCHAR(20),
    ai_score          NUMERIC(5,2),
    ai_summary        TEXT,
    evaluated_at      TIMESTAMPTZ,
    PRIMARY KEY (application_id, session_id, evaluator_id, criterion_name)
);
```

### Projection 3: Analytics (Bias Detection)

Denormalized for analytical queries, potentially materialized into ClickHouse.

```sql
CREATE TABLE rm_interview_analytics (
    session_id            UUID,
    interviewer_id        UUID,
    candidate_demographic VARCHAR(50),  -- anonymized demographic group
    talk_time_ratio       NUMERIC(5,4), -- interviewer talk time / total
    questions_asked       INTEGER,
    hints_given           INTEGER,
    difficulty_level      VARCHAR(20),
    score_given           NUMERIC(5,2),
    decision              VARCHAR(20),
    session_duration_mins INTEGER,
    recorded_at           TIMESTAMPTZ
);
```

### Projection 4: Code Playback Timeline

Optimized for session replay.

```sql
CREATE TABLE rm_code_timeline (
    session_id        UUID,
    sequence_num      INTEGER,
    event_type        VARCHAR(50),     -- 'code_changed', 'execution_requested', 'execution_completed'
    timestamp         TIMESTAMPTZ,
    delta_data        JSONB,           -- code diff or execution result
    author_id         UUID,
    PRIMARY KEY (session_id, sequence_num)
);
```

---

## Pros and Cons

### Pros

- **Complete audit trail by design**: Every action during an interview is an immutable event. SOC 2 compliance, EEOC auditing, and bias detection are built into the architecture rather than bolted on. You can answer "exactly what happened during this interview and in what order" by replaying the event stream.
- **Natural fit for session replay**: Code playback, whiteboard replay, and video synchronization are trivially implemented by replaying the event stream up to a given timestamp. This is the core use case for session review.
- **Temporal queries**: "What was the code state at minute 15?" or "When did the candidate first attempt a recursive approach?" are answered by replaying events to a point in time -- impossible with a mutable state model.
- **Decoupled read/write scaling**: Write throughput during live sessions (high-frequency code changes, proctoring events) is handled by the append-only event store. Read models for dashboards, analytics, and ATS sync are independently scalable.
- **Enables AI analysis pipelines**: AI evaluation, bias detection, and adaptive difficulty all consume event streams. Event sourcing provides a natural integration point for ML pipelines that process session events asynchronously.
- **Safe schema evolution**: New event types can be added without modifying existing data. New projections can be built by replaying historical events, enabling feature development without data migration.
- **GDPR compliance via crypto-shredding**: Instead of deleting events (which violates immutability), candidate PII can be encrypted with a per-candidate key. Erasure is achieved by destroying the key, rendering events unreadable while preserving the event stream structure.

### Cons

- **Significant architectural complexity**: Event sourcing and CQRS require understanding of aggregates, projections, eventual consistency, idempotency, and event versioning. The engineering team must be experienced with these patterns or invest significant learning time.
- **Eventual consistency challenges**: Read models are updated asynchronously after events are written. Interviewers may not see evaluation scores immediately after submission. The UI must be designed to tolerate and communicate eventual consistency.
- **Projection maintenance burden**: Each read model is a separate projection that must be built, maintained, and rebuilt when requirements change. With 4-6 projections, this is manageable; at 20+ it becomes a significant operational burden.
- **Event versioning complexity**: As the domain evolves, event schemas change. Upcasting old events to new formats, maintaining backward compatibility, and handling event schema migrations requires disciplined versioning practices.
- **Higher infrastructure cost**: Running an event store, projection processors, and multiple read model databases costs more than a single PostgreSQL instance. Managed EventStoreDB or Kafka adds operational complexity.
- **Overkill for simple CRUD entities**: Organizations, users, job positions, and ATS integrations are simple CRUD entities that don't benefit from event sourcing. A pure event-sourced system forces unnecessary complexity on these domains.
- **Debugging difficulty**: Tracing a bug requires replaying events and inspecting projections rather than querying a single table. Tooling for event store debugging is less mature than relational database tooling.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event store** | EventStoreDB (open-source, purpose-built) or PostgreSQL with append-only events table |
| **Command processing** | Node.js with custom aggregate framework, or Axon Framework (Java/Kotlin) |
| **Projection engine** | Custom event handlers writing to PostgreSQL read models |
| **Read model database** | PostgreSQL for operational queries; ClickHouse for analytics |
| **Message bus** | EventStoreDB subscriptions (built-in) or Apache Kafka for cross-service event distribution |
| **Real-time sync** | Yjs CRDT for collaborative editor (events derived from CRDT operations) |
| **Code execution** | Judge0 CE; execution requests and results are events in the session stream |
| **Object storage** | S3-compatible for recordings and large binary assets |
| **Cache** | Redis for session state projections and hot read models |

---

## Migration and Scaling Considerations

### Migration Path

1. **MVP**: PostgreSQL-based event store with 2-3 read model projections. Event sourcing for InterviewSession and Evaluation aggregates only; simple CRUD for organizations, users, and problems.
2. **Growth**: Migrate event store to EventStoreDB for better subscription support and built-in projections. Add Kafka for cross-service event distribution.
3. **Scale**: Partition event streams by organization. Deploy ClickHouse for analytics projections. Add event archival to S3/Glacier for streams older than 1 year.
4. **Enterprise**: Multi-region event store replication for global interview scheduling. Per-tenant encryption keys for crypto-shredding GDPR compliance.

### Event Volume Estimates (10,000 interviews/month)

| Event Type | Events/Interview | Monthly Volume |
|------------|-----------------|----------------|
| CodeChanged | ~500-2,000 | 5-20 million |
| CodeExecutionCompleted | ~10-30 | 100-300K |
| ProctorEvent | ~20-100 | 200K-1M |
| WhiteboardUpdated | ~50-200 | 500K-2M |
| EvaluationEvents | ~10-20 | 100-200K |
| SessionLifecycle | ~5-10 | 50-100K |
| **Total** | **~600-2,400** | **~6-24 million** |

### Snapshot Strategy

- Snapshot InterviewSession aggregates every 500 events to avoid slow rehydration during live sessions.
- Snapshot Candidate aggregates every 100 events.
- Snapshots stored alongside events for fast aggregate loading.

### Event Retention

- Live event store: 90 days of recent events for fast replay.
- Archive: All events archived to S3 in Parquet format for long-term compliance (7+ years for SOC 2).
- Projections rebuilt from archive if needed (rare, but possible for new analytics requirements).

This architecture is the strongest choice if the platform's differentiators (session replay, AI evaluation, bias detection, audit compliance) are non-negotiable from launch. The upfront complexity investment pays dividends as these features mature, but it requires an engineering team comfortable with event-driven systems.
