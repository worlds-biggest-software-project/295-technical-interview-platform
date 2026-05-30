# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Technical Interview Platform (#295)
> Approach: Traditional normalized relational schema using PostgreSQL

## Summary

A fully normalized relational database design (3NF/BCNF) using PostgreSQL as the primary data store. All entities are represented as distinct tables with foreign key constraints, enforcing referential integrity across the interview lifecycle. This approach prioritizes data consistency, well-understood query patterns, and strong tooling support for the ATS integration, evaluation, and compliance requirements central to an enterprise-grade interview platform.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organization ──< Team ──< TeamMember >── User
User ──< InterviewSession
User ──< EvaluationScore

Candidate ──< Application >── JobPosition ──< Organization
Application ──< InterviewSession
InterviewSession ──< SessionParticipant >── User
InterviewSession ──< CodeSubmission
InterviewSession ──< WhiteboardSnapshot
InterviewSession ──< SessionRecording
InterviewSession ──< EvaluationScore
InterviewSession ──< ProctorEvent

ProblemLibrary ──< Problem ──< ProblemVariant
Problem ──< ProblemTag >── Tag
InterviewSession ──< SessionProblem >── Problem

EvaluationRubric ──< RubricCriterion
EvaluationScore ──< CriterionScore >── RubricCriterion

ATSIntegration ──< ATSSyncLog
```

### Core Schema

```sql
-- === ORGANIZATIONS & USERS ===

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    sso_provider    VARCHAR(50),        -- 'okta', 'azure_ad', 'google'
    sso_config_url  TEXT,
    soc2_compliant  BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL CHECK (role IN ('admin', 'interviewer', 'recruiter', 'observer')),
    avatar_url      TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

-- === CANDIDATES & APPLICATIONS ===

CREATE TABLE candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    anonymized_id   VARCHAR(64) UNIQUE,   -- for bias-free review
    phone           VARCHAR(50),
    resume_url      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE job_positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           VARCHAR(255) NOT NULL,
    department      VARCHAR(100),
    seniority_level VARCHAR(50),          -- 'junior', 'mid', 'senior', 'staff', 'principal'
    description     TEXT,
    status          VARCHAR(20) DEFAULT 'open' CHECK (status IN ('draft', 'open', 'closed', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    job_position_id UUID NOT NULL REFERENCES job_positions(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'applied'
                    CHECK (status IN ('applied', 'screening', 'interviewing', 'offer', 'hired', 'rejected', 'withdrawn')),
    source          VARCHAR(50),           -- 'greenhouse', 'lever', 'direct', etc.
    ats_external_id VARCHAR(255),          -- ID in external ATS
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === INTERVIEW SESSIONS ===

CREATE TABLE interview_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    session_type    VARCHAR(30) NOT NULL CHECK (session_type IN ('live_coding', 'system_design', 'take_home', 'pair_programming', 'behavioral')),
    title           VARCHAR(255),
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    duration_limit  INTEGER,               -- max duration in minutes
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled', 'in_progress', 'completed', 'cancelled', 'no_show')),
    video_room_id   VARCHAR(255),          -- WebRTC room identifier
    recording_url   TEXT,
    language        VARCHAR(30),           -- primary programming language
    is_anonymized   BOOLEAN DEFAULT FALSE,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE session_participants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    user_id         UUID REFERENCES users(id),
    candidate_id    UUID REFERENCES candidates(id),
    role            VARCHAR(20) NOT NULL CHECK (role IN ('interviewer', 'candidate', 'observer', 'ai_copilot')),
    joined_at       TIMESTAMPTZ,
    left_at         TIMESTAMPTZ,
    CONSTRAINT participant_identity CHECK (
        (user_id IS NOT NULL AND candidate_id IS NULL) OR
        (user_id IS NULL AND candidate_id IS NOT NULL)
    )
);

-- === CODE SUBMISSIONS & EXECUTION ===

CREATE TABLE code_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    language        VARCHAR(30) NOT NULL,
    source_code     TEXT NOT NULL,
    stdin           TEXT,
    stdout          TEXT,
    stderr          TEXT,
    exit_code       INTEGER,
    execution_time_ms   INTEGER,
    memory_used_kb      INTEGER,
    submitted_by    UUID,                  -- candidate or interviewer UUID
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === PROBLEM LIBRARY ===

CREATE TABLE problems (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id), -- NULL = global library
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    difficulty      VARCHAR(20) NOT NULL CHECK (difficulty IN ('easy', 'medium', 'hard', 'expert')),
    problem_type    VARCHAR(30) NOT NULL CHECK (problem_type IN ('algorithm', 'data_structure', 'system_design', 'debugging', 'frontend', 'database', 'devops')),
    starter_code    TEXT,
    solution_code   TEXT,
    time_limit_mins INTEGER DEFAULT 45,
    created_by      UUID REFERENCES users(id),
    is_public       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE problem_test_cases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    problem_id      UUID NOT NULL REFERENCES problems(id) ON DELETE CASCADE,
    input           TEXT NOT NULL,
    expected_output TEXT NOT NULL,
    is_hidden       BOOLEAN DEFAULT FALSE,
    weight          NUMERIC(3,2) DEFAULT 1.00,
    sort_order      INTEGER DEFAULT 0
);

CREATE TABLE tags (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name    VARCHAR(100) UNIQUE NOT NULL,
    category VARCHAR(50)  -- 'language', 'skill', 'topic', 'seniority'
);

CREATE TABLE problem_tags (
    problem_id UUID NOT NULL REFERENCES problems(id) ON DELETE CASCADE,
    tag_id     UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (problem_id, tag_id)
);

-- === EVALUATION & SCORING ===

CREATE TABLE evaluation_rubrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    session_type    VARCHAR(30),           -- applies to which session types
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE rubric_criteria (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rubric_id       UUID NOT NULL REFERENCES evaluation_rubrics(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,    -- e.g. 'Code Correctness', 'Communication', 'Problem Decomposition'
    description     TEXT,
    max_score       INTEGER NOT NULL DEFAULT 5,
    weight          NUMERIC(3,2) DEFAULT 1.00,
    sort_order      INTEGER DEFAULT 0
);

CREATE TABLE evaluation_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    evaluator_id    UUID NOT NULL REFERENCES users(id),
    rubric_id       UUID NOT NULL REFERENCES evaluation_rubrics(id),
    overall_score   NUMERIC(5,2),
    overall_decision VARCHAR(20) CHECK (overall_decision IN ('strong_yes', 'yes', 'neutral', 'no', 'strong_no')),
    private_notes   TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE criterion_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    evaluation_id   UUID NOT NULL REFERENCES evaluation_scores(id) ON DELETE CASCADE,
    criterion_id    UUID NOT NULL REFERENCES rubric_criteria(id),
    score           INTEGER NOT NULL,
    notes           TEXT
);

-- === AI EVALUATION ===

CREATE TABLE ai_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    evaluation_type VARCHAR(30) NOT NULL,   -- 'code_quality', 'complexity_analysis', 'soft_skills', 'bias_detection'
    model_version   VARCHAR(50),
    score           NUMERIC(5,2),
    summary         TEXT NOT NULL,
    evidence        TEXT,                    -- transcript citations
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === PROCTORING & AUDIT ===

CREATE TABLE proctor_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    event_type      VARCHAR(30) NOT NULL CHECK (event_type IN ('tab_switch', 'face_detection', 'voice_detection', 'device_change', 'copy_paste', 'screen_share_stop')),
    severity        VARCHAR(10) DEFAULT 'info' CHECK (severity IN ('info', 'warning', 'critical')),
    details         TEXT,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    actor_id        UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    details         TEXT,
    ip_address      INET,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === ATS INTEGRATION ===

CREATE TABLE ats_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        VARCHAR(30) NOT NULL CHECK (provider IN ('greenhouse', 'lever', 'workday', 'icims', 'workable')),
    api_key_enc     BYTEA,                 -- encrypted API key
    oauth_config    TEXT,                   -- encrypted OAuth credentials
    webhook_secret  VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ats_sync_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES ats_integrations(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    entity_type     VARCHAR(30) NOT NULL,   -- 'candidate', 'application', 'interview_result'
    entity_id       UUID,
    external_id     VARCHAR(255),
    status          VARCHAR(20) NOT NULL CHECK (status IN ('pending', 'success', 'failed', 'retrying')),
    error_message   TEXT,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- === SESSION RECORDINGS & WHITEBOARD ===

CREATE TABLE session_recordings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    recording_type  VARCHAR(20) NOT NULL CHECK (recording_type IN ('video', 'audio', 'code_playback', 'whiteboard')),
    storage_url     TEXT NOT NULL,
    duration_secs   INTEGER,
    file_size_bytes BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE whiteboard_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    snapshot_data   TEXT NOT NULL,           -- Excalidraw JSON or SVG export
    label           VARCHAR(255),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Key Indexes

```sql
CREATE INDEX idx_applications_candidate ON applications(candidate_id);
CREATE INDEX idx_applications_job ON applications(job_position_id);
CREATE INDEX idx_applications_status ON applications(status);
CREATE INDEX idx_sessions_application ON interview_sessions(application_id);
CREATE INDEX idx_sessions_scheduled ON interview_sessions(scheduled_at);
CREATE INDEX idx_sessions_status ON interview_sessions(status);
CREATE INDEX idx_code_submissions_session ON code_submissions(session_id);
CREATE INDEX idx_evaluation_scores_session ON evaluation_scores(session_id);
CREATE INDEX idx_problems_difficulty ON problems(difficulty);
CREATE INDEX idx_problems_type ON problems(problem_type);
CREATE INDEX idx_audit_logs_org_time ON audit_logs(organization_id, timestamp DESC);
CREATE INDEX idx_proctor_events_session ON proctor_events(session_id);
```

---

## Pros and Cons

### Pros

- **Strong data integrity**: Foreign key constraints and CHECK constraints enforce business rules at the database level. Interview scores always reference valid sessions and evaluators.
- **Mature tooling**: PostgreSQL has decades of ecosystem support including ORMs (Prisma, TypeORM, Drizzle, SQLAlchemy), migration tools (Flyway, Alembic, Prisma Migrate), monitoring (pganalyze), and backup solutions.
- **Clear query patterns**: JOIN-based queries for reports (e.g., "show all interview scores for a candidate across all sessions") are straightforward and well-optimized by the PostgreSQL query planner.
- **SOC 2 / GDPR compliance**: Row-level security, audit logging, and data deletion (CASCADE) are natively supported, simplifying compliance requirements.
- **ATS integration clarity**: Separate ATS integration and sync log tables provide a clean boundary for webhook processing and external system reconciliation.
- **Transparent cost modeling**: No specialized database licensing; PostgreSQL is open source. Managed services (AWS RDS, Supabase, Neon) offer predictable scaling.
- **Well-understood by engineering teams**: Most developers are proficient with relational schemas, reducing onboarding time.

### Cons

- **Schema rigidity**: Adding new evaluation dimensions, session types, or problem metadata requires schema migrations. Frequent schema changes during early product development can slow iteration.
- **Poor fit for semi-structured data**: Evaluation rubrics, AI-generated reports, and whiteboard snapshots have variable structures that are awkward to normalize. Storing Excalidraw JSON or variable-length AI evidence in TEXT columns loses queryability.
- **Real-time collaboration overhead**: The collaborative code editor generates high-frequency events (keystrokes, cursor moves) that do not map well to row-per-event in a relational table. A separate real-time layer (Redis, CRDT) is required regardless.
- **Scaling write-heavy workloads**: During live sessions with proctoring, code execution, and video, the platform generates many concurrent writes. PostgreSQL handles this well up to moderate scale but may require read replicas and partitioning for high-volume enterprise deployments.
- **Complex reporting queries**: Multi-dimensional analytics (e.g., "bias detection across interviewer panels grouped by demographic and question difficulty") require complex JOINs across many tables, potentially benefiting from a dedicated analytics store.
- **Rigid problem taxonomy**: The fixed tag/category model may not capture the nuanced skill relationships needed for intelligent problem recommendation (e.g., "this problem tests both async I/O and error handling patterns").

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Primary database** | PostgreSQL 16+ (managed via AWS RDS, Supabase, or Neon) |
| **ORM / Query builder** | Prisma (TypeScript) or SQLAlchemy (Python) |
| **Migrations** | Prisma Migrate or Alembic |
| **Real-time layer** | Redis Pub/Sub or Yjs CRDT (separate from relational store) |
| **Search** | PostgreSQL full-text search for problems; Elasticsearch for advanced problem search |
| **Code execution** | Judge0 CE (self-hosted, isolated) |
| **Object storage** | S3-compatible (recordings, whiteboard exports) |
| **Cache** | Redis for session state and hot data |

---

## Migration and Scaling Considerations

### Migration Path

1. **MVP**: Single PostgreSQL instance with all tables. Use connection pooling (PgBouncer) from day one.
2. **Growth (100-1,000 daily sessions)**: Add read replicas for reporting and analytics queries. Partition `audit_logs` and `proctor_events` by month using PostgreSQL declarative partitioning.
3. **Scale (1,000+ daily sessions)**: Extract real-time collaboration data to a dedicated event store or Redis Streams. Consider moving analytics to a columnar store (ClickHouse, TimescaleDB) for bias detection and trend analysis.
4. **Enterprise multi-tenancy**: Implement row-level security (RLS) policies keyed on `organization_id` to isolate tenant data within a shared schema.

### Data Retention

- Session recordings: Configurable retention per organization (30-365 days) with automated S3 lifecycle policies.
- Audit logs: 7-year retention for SOC 2 compliance, partitioned by year.
- Code submissions: Retained for the lifetime of the application record; purged on candidate data deletion request (GDPR right to erasure).

### Estimated Table Sizes (Year 1, 10,000 interviews/month)

| Table | Estimated Rows/Month | Growth Rate |
|-------|---------------------|-------------|
| interview_sessions | 10,000 | Linear |
| code_submissions | 50,000 | Linear |
| proctor_events | 200,000 | Linear |
| audit_logs | 500,000 | Linear |
| evaluation_scores | 20,000 | Linear |
| problems | 5,000 (cumulative) | Logarithmic |

This model is the safest starting point for an MVP, providing strong guarantees and broad ecosystem support, with clear upgrade paths to hybrid or event-sourced architectures as the platform matures.
