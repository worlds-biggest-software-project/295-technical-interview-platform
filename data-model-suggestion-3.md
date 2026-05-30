# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Technical Interview Platform (#295)
> Approach: PostgreSQL with strategic JSONB columns for semi-structured data

## Summary

A pragmatic hybrid architecture that uses PostgreSQL as the single database engine, combining normalized relational tables for core entities (organizations, users, candidates, sessions) with JSONB columns for semi-structured, variable-shape data (evaluation rubrics, AI-generated reports, problem metadata, whiteboard state, proctoring details). This approach captures the best of both worlds: referential integrity and JOIN performance for structured relationships, and schema flexibility for the many areas of the interview platform where data shapes vary by context, evolve rapidly, or are inherently document-like.

This is the recommended approach for most teams because it avoids the operational complexity of a separate document database (MongoDB) or event store while providing the flexibility that a purely normalized schema lacks.

---

## Design Philosophy

The key insight is that a technical interview platform has two distinct data profiles:

1. **Structured, relational data**: Organizations, users, candidates, applications, job positions, interview schedules, ATS integrations. These have stable schemas, well-defined relationships, and benefit from foreign keys, indexes, and JOIN operations.

2. **Semi-structured, variable-shape data**: Evaluation rubrics (which differ per organization), AI evaluation reports (which evolve as models improve), whiteboard state (Excalidraw JSON), code editor state (CRDT snapshots), proctoring event details, problem metadata and constraints, and webhook payloads from diverse ATS providers. These benefit from JSONB's schema flexibility.

The hybrid model uses **relational columns for identity, relationships, and frequently-queried attributes**, and **JSONB columns for everything that varies, nests, or evolves**.

---

## Key Entities and Schema

### Core Relational Tables (Normalized)

```sql
-- === ORGANIZATIONS & USERS ===
-- Fully normalized; stable schema; heavy JOIN targets

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',    -- org-level config (timezone, branding, retention policies)
    sso_config      JSONB,                 -- flexible SSO provider config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(20) NOT NULL CHECK (role IN ('admin', 'interviewer', 'recruiter', 'observer')),
    preferences     JSONB DEFAULT '{}',    -- UI preferences, notification settings
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (organization_id, email)
);

CREATE TABLE candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    anonymized_id   VARCHAR(64) UNIQUE,
    profile         JSONB DEFAULT '{}',    -- resume data, skills, links (variable per candidate)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE job_positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           VARCHAR(255) NOT NULL,
    department      VARCHAR(100),
    seniority_level VARCHAR(50),
    status          VARCHAR(20) DEFAULT 'open',
    requirements    JSONB DEFAULT '{}',    -- skills, experience, qualifications (variable per role)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE applications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    job_position_id UUID NOT NULL REFERENCES job_positions(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'applied',
    source          VARCHAR(50),
    ats_external_id VARCHAR(255),
    ats_metadata    JSONB DEFAULT '{}',    -- provider-specific data from Greenhouse/Lever/Workday
    applied_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_applications_candidate ON applications(candidate_id);
CREATE INDEX idx_applications_status ON applications(status);
```

### Interview Sessions (Hybrid)

```sql
CREATE TABLE interview_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    application_id  UUID NOT NULL REFERENCES applications(id),
    session_type    VARCHAR(30) NOT NULL,
    title           VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    
    -- Relational: frequently queried, indexed, used in JOINs
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    language        VARCHAR(30),
    is_anonymized   BOOLEAN DEFAULT FALSE,
    
    -- JSONB: variable session configuration that differs by session type
    session_config  JSONB DEFAULT '{}',
    /*
    session_config examples:
    
    Live coding:
    {
      "allowed_languages": ["python", "javascript", "go"],
      "time_limit_mins": 60,
      "ai_copilot_enabled": true,
      "code_execution_enabled": true,
      "starter_files": [
        {"name": "solution.py", "content": "# Write your solution here"}
      ]
    }
    
    System design:
    {
      "whiteboard_template": "blank",
      "diagram_tools": ["excalidraw"],
      "time_limit_mins": 45,
      "reference_materials_url": "https://..."
    }
    
    Take-home:
    {
      "deadline": "2026-06-01T23:59:59Z",
      "max_submissions": 3,
      "repo_template_url": "https://github.com/...",
      "review_checklist": ["correctness", "code_quality", "documentation"]
    }
    */
    
    -- JSONB: video/recording metadata
    media_config    JSONB DEFAULT '{}',
    /*
    {
      "video_room_id": "room_abc123",
      "video_provider": "daily.co",
      "recording_enabled": true,
      "recordings": [
        {"type": "video", "url": "s3://...", "duration_secs": 3600, "size_bytes": 524288000},
        {"type": "code_playback", "url": "s3://...", "duration_secs": 3600}
      ]
    }
    */
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_application ON interview_sessions(application_id);
CREATE INDEX idx_sessions_scheduled ON interview_sessions(scheduled_at);
CREATE INDEX idx_sessions_status ON interview_sessions(status);
CREATE INDEX idx_sessions_config ON interview_sessions USING GIN (session_config);
```

### Session Participants

```sql
CREATE TABLE session_participants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    user_id         UUID REFERENCES users(id),
    candidate_id    UUID REFERENCES candidates(id),
    role            VARCHAR(20) NOT NULL,
    joined_at       TIMESTAMPTZ,
    left_at         TIMESTAMPTZ,
    participation_data JSONB DEFAULT '{}',
    /*
    {
      "talk_time_secs": 1200,
      "questions_asked": 8,
      "hints_given": 2,
      "interruptions": 1,
      "screen_shares": 3
    }
    */
    CONSTRAINT participant_identity CHECK (
        (user_id IS NOT NULL AND candidate_id IS NULL) OR
        (user_id IS NULL AND candidate_id IS NOT NULL)
    )
);
```

### Problem Library (Hybrid)

```sql
CREATE TABLE problems (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    title           VARCHAR(255) NOT NULL,
    difficulty      VARCHAR(20) NOT NULL,
    problem_type    VARCHAR(30) NOT NULL,
    is_public       BOOLEAN DEFAULT FALSE,
    
    -- JSONB: the problem content is inherently document-shaped and varies by type
    content         JSONB NOT NULL,
    /*
    Algorithm problem:
    {
      "description": "Given an array of integers...",
      "constraints": ["1 <= n <= 10^5", "−10^9 <= nums[i] <= 10^9"],
      "examples": [
        {"input": "[2,7,11,15], target=9", "output": "[0,1]", "explanation": "..."}
      ],
      "starter_code": {"python": "def two_sum(nums, target):", "javascript": "function twoSum(nums, target) {"},
      "solution_code": {"python": "def two_sum(nums, target): ..."},
      "hints": ["Consider using a hash map", "Single pass is possible"],
      "tags": ["array", "hash-table", "two-pointers"],
      "skills_tested": ["data_structures", "time_complexity_optimization"],
      "estimated_time_mins": 30
    }
    
    System design problem:
    {
      "description": "Design a URL shortener service...",
      "requirements": ["functional", "non_functional"],
      "evaluation_areas": ["scalability", "api_design", "data_modeling", "caching"],
      "reference_diagram": "<excalidraw_json>",
      "discussion_prompts": ["How would you handle 10x traffic?", "What if a link goes viral?"],
      "tags": ["system-design", "distributed-systems", "caching"],
      "estimated_time_mins": 45
    }
    */
    
    -- JSONB: test cases stored as a flexible array
    test_cases      JSONB DEFAULT '[]',
    /*
    [
      {"input": "[2,7,11,15]\n9", "expected": "[0,1]", "hidden": false, "weight": 1.0},
      {"input": "[3,2,4]\n6", "expected": "[1,2]", "hidden": true, "weight": 1.0},
      {"input": "[3,3]\n6", "expected": "[0,1]", "hidden": true, "weight": 2.0}
    ]
    */
    
    -- JSONB: AI-generated variants
    variants        JSONB DEFAULT '[]',
    /*
    [
      {"variant_id": "v1", "difficulty": "easy", "description": "...", "generated_by": "gpt-4o", "generated_at": "..."},
      {"variant_id": "v2", "difficulty": "hard", "description": "...", "generated_by": "gpt-4o", "generated_at": "..."}
    ]
    */
    
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_problems_difficulty ON problems(difficulty);
CREATE INDEX idx_problems_type ON problems(problem_type);
CREATE INDEX idx_problems_content_tags ON problems USING GIN ((content -> 'tags'));
CREATE INDEX idx_problems_content_skills ON problems USING GIN ((content -> 'skills_tested'));

-- Full-text search across problem descriptions
CREATE INDEX idx_problems_fts ON problems USING GIN (
    to_tsvector('english', title || ' ' || COALESCE(content ->> 'description', ''))
);
```

### Code Submissions

```sql
CREATE TABLE code_submissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    language        VARCHAR(30) NOT NULL,
    source_code     TEXT NOT NULL,
    submitted_by    UUID,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- JSONB: execution results vary by language and configuration
    execution_result JSONB,
    /*
    {
      "stdin": "5\n1 2 3 4 5",
      "stdout": "15",
      "stderr": "",
      "exit_code": 0,
      "execution_time_ms": 42,
      "memory_used_kb": 3200,
      "test_results": [
        {"test_case_id": 0, "passed": true, "actual": "[0,1]", "expected": "[0,1]", "time_ms": 5},
        {"test_case_id": 1, "passed": true, "actual": "[1,2]", "expected": "[1,2]", "time_ms": 3},
        {"test_case_id": 2, "passed": false, "actual": "timeout", "expected": "[0,1]", "time_ms": 5000}
      ],
      "passed_count": 2,
      "total_count": 3,
      "complexity_estimate": "O(n^2)"
    }
    */
);

CREATE INDEX idx_submissions_session ON code_submissions(session_id);
```

### Evaluation and Scoring (Hybrid)

```sql
CREATE TABLE evaluation_rubrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    session_type    VARCHAR(30),
    
    -- JSONB: rubric structure is inherently flexible and org-specific
    rubric_definition JSONB NOT NULL,
    /*
    {
      "criteria": [
        {
          "id": "correctness",
          "name": "Code Correctness",
          "description": "Does the solution produce correct output for all test cases?",
          "max_score": 5,
          "weight": 2.0,
          "scoring_guide": {
            "1": "No working solution",
            "2": "Partial solution, major bugs",
            "3": "Works for basic cases, misses edge cases",
            "4": "Correct for all visible test cases",
            "5": "Correct for all cases including edge cases"
          }
        },
        {
          "id": "code_quality",
          "name": "Code Quality & Style",
          "description": "Is the code readable, well-structured, and idiomatic?",
          "max_score": 5,
          "weight": 1.0,
          "scoring_guide": { ... }
        },
        {
          "id": "communication",
          "name": "Communication & Problem-Solving Approach",
          "description": "Did the candidate explain their thinking clearly?",
          "max_score": 5,
          "weight": 1.5,
          "scoring_guide": { ... }
        }
      ],
      "decision_options": ["strong_yes", "yes", "neutral", "no", "strong_no"],
      "version": 3
    }
    */
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    evaluator_id    UUID NOT NULL REFERENCES users(id),
    rubric_id       UUID NOT NULL REFERENCES evaluation_rubrics(id),
    
    -- Relational: indexed for aggregation queries
    overall_decision VARCHAR(20),
    submitted_at    TIMESTAMPTZ,
    
    -- JSONB: scores conform to the rubric definition but are stored flexibly
    scores          JSONB NOT NULL DEFAULT '{}',
    /*
    {
      "criteria_scores": {
        "correctness": {"score": 4, "notes": "Solved main cases, missed one edge case with empty input"},
        "code_quality": {"score": 5, "notes": "Clean, well-named variables, good use of list comprehensions"},
        "communication": {"score": 3, "notes": "Could have explained trade-offs more explicitly"}
      },
      "weighted_total": 4.1,
      "private_notes": "Strong candidate for mid-level. Consider for team X.",
      "strengths": ["data_structures", "python_idioms"],
      "areas_for_improvement": ["edge_case_handling", "verbal_communication"]
    }
    */
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_evaluations_session ON evaluations(session_id);
CREATE INDEX idx_evaluations_decision ON evaluations(overall_decision);

-- AI-generated evaluations stored with full flexibility
CREATE TABLE ai_evaluations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    evaluation_type VARCHAR(30) NOT NULL,
    model_version   VARCHAR(50),
    
    -- JSONB: AI reports vary significantly by type and model version
    report          JSONB NOT NULL,
    /*
    Code quality report:
    {
      "overall_score": 7.5,
      "dimensions": {
        "correctness": {"score": 8, "evidence": ["Line 15: correctly handles null case", "Line 22: missed integer overflow"]},
        "time_complexity": {"score": 7, "analysis": "O(n log n) solution; O(n) possible with hash map", "optimal": "O(n)"},
        "space_complexity": {"score": 8, "analysis": "O(n) auxiliary space", "optimal": "O(n)"},
        "code_quality": {"score": 7, "issues": ["Variable 'x' could be more descriptive", "Missing type hints"]},
        "edge_cases": {"score": 6, "covered": ["empty_array", "single_element"], "missed": ["all_duplicates", "negative_numbers"]}
      },
      "summary": "Strong algorithmic thinking with room for improvement on edge cases...",
      "transcript_citations": [
        {"timestamp": "00:12:34", "quote": "I think we need to sort first...", "analysis": "Correct instinct but suboptimal approach"}
      ]
    }
    
    Bias detection report:
    {
      "interviewer_id": "...",
      "metrics": {
        "talk_time_ratio": 0.35,
        "questions_asked": 8,
        "follow_up_depth": 2.3,
        "wait_time_avg_secs": 4.2,
        "interruptions": 1,
        "hint_frequency": 0.25
      },
      "comparison_to_baseline": {
        "talk_time_deviation": -0.05,
        "question_count_deviation": 0,
        "flags": []
      },
      "recommendation": "No bias indicators detected. Interview conducted consistently with baseline."
    }
    */
    
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_evaluations_session ON ai_evaluations(session_id);
CREATE INDEX idx_ai_evaluations_type ON ai_evaluations(evaluation_type);
```

### Proctoring Events

```sql
CREATE TABLE proctor_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    event_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL DEFAULT 'info',
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- JSONB: event details vary significantly by event type
    details         JSONB DEFAULT '{}',
    /*
    Tab switch:    {"tab_title": "LeetCode - Two Sum", "duration_secs": 12, "return_tab": "Interview Platform"}
    Face detection: {"alert": "multiple_faces", "confidence": 0.92, "face_count": 2}
    Copy paste:    {"content_length": 245, "source": "external", "content_hash": "abc123..."}
    Device change: {"old_device": "MacBook Pro", "new_device": "iPhone 14", "ip_changed": true}
    */
);

CREATE INDEX idx_proctor_session ON proctor_events(session_id);
CREATE INDEX idx_proctor_severity ON proctor_events(severity) WHERE severity IN ('warning', 'critical');
```

### ATS Integration

```sql
CREATE TABLE ats_integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        VARCHAR(30) NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    
    -- JSONB: each ATS provider has different config requirements
    config          JSONB NOT NULL,
    /*
    Greenhouse:
    {
      "api_key_encrypted": "enc:...",
      "harvest_api_url": "https://harvest.greenhouse.io/v1",
      "webhook_secret": "...",
      "job_board_token": "...",
      "sync_interval_mins": 15
    }
    
    Lever:
    {
      "oauth_client_id": "...",
      "oauth_client_secret_encrypted": "enc:...",
      "oauth_refresh_token_encrypted": "enc:...",
      "api_url": "https://api.lever.co/v1",
      "webhook_signing_token": "...",
      "sync_interval_mins": 10
    }
    
    Workday:
    {
      "tenant_url": "https://wd5-impl-services.workday.com",
      "isu_username": "...",
      "isu_password_encrypted": "enc:...",
      "api_version": "v40.2",
      "use_soap": false,
      "sync_interval_mins": 30
    }
    */
    
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE ats_webhook_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES ats_integrations(id),
    direction       VARCHAR(10) NOT NULL,
    event_type      VARCHAR(100),
    
    -- JSONB: webhook payloads are provider-specific and should be stored verbatim
    payload         JSONB NOT NULL,
    
    status          VARCHAR(20) NOT NULL DEFAULT 'received',
    error_message   TEXT,
    processed_at    TIMESTAMPTZ,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_log_integration ON ats_webhook_log(integration_id);
CREATE INDEX idx_webhook_log_status ON ats_webhook_log(status) WHERE status != 'processed';
```

### Whiteboard State

```sql
CREATE TABLE whiteboard_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES interview_sessions(id),
    label           VARCHAR(255),
    captured_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- JSONB: Excalidraw state is inherently a JSON document
    excalidraw_state JSONB NOT NULL,
    /*
    {
      "type": "excalidraw",
      "version": 2,
      "elements": [
        {"type": "rectangle", "x": 100, "y": 200, "width": 150, "height": 80, ...},
        {"type": "text", "text": "API Gateway", ...},
        {"type": "arrow", "startBinding": {...}, "endBinding": {...}, ...}
      ],
      "appState": {"viewBackgroundColor": "#ffffff", "gridSize": 20}
    }
    */
);

CREATE INDEX idx_whiteboard_session ON whiteboard_snapshots(session_id);
```

### Audit Log

```sql
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    
    -- JSONB: audit context varies by action
    context         JSONB DEFAULT '{}',
    /*
    { "ip": "192.168.1.1", "user_agent": "...", "old_value": "screening", "new_value": "interviewing" }
    */
    
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE audit_logs_2026_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions created by automation

CREATE INDEX idx_audit_org_time ON audit_logs(organization_id, timestamp DESC);
```

---

## JSONB Indexing Strategy

```sql
-- GIN indexes for JSONB containment queries (@> operator)
CREATE INDEX idx_org_settings ON organizations USING GIN (settings);
CREATE INDEX idx_session_config ON interview_sessions USING GIN (session_config);
CREATE INDEX idx_problem_content ON problems USING GIN (content);

-- Partial indexes for specific JSONB paths
CREATE INDEX idx_problems_tags ON problems USING GIN ((content -> 'tags'));
CREATE INDEX idx_eval_strengths ON evaluations USING GIN ((scores -> 'strengths'));

-- Expression indexes for frequently accessed JSONB fields
CREATE INDEX idx_sessions_time_limit ON interview_sessions ((session_config ->> 'time_limit_mins'));
```

---

## Pros and Cons

### Pros

- **Single database, dual paradigm**: PostgreSQL handles both relational and document queries, eliminating the operational overhead of running a separate MongoDB or document store. One backup strategy, one connection pool, one set of monitoring tools.
- **Schema flexibility where it matters**: Evaluation rubrics, AI reports, ATS webhook payloads, and whiteboard state all have variable shapes that are awkward to normalize. JSONB handles these naturally while keeping stable attributes in typed columns.
- **Strong querying of semi-structured data**: PostgreSQL's JSONB operators (`->`, `->>`, `@>`, `?`, `#>`) and GIN indexes enable efficient queries into document data. "Find all problems tagged with 'dynamic-programming' at 'hard' difficulty" works with a single indexed query.
- **Incremental migration**: Start with a normalized schema and progressively move semi-structured fields to JSONB as requirements clarify. No architectural rewrites needed; the database engine stays the same.
- **Full-text search built in**: `to_tsvector` and GIN indexes on JSONB content provide natural-language problem search ("find Python async I/O questions for mid-level candidates") without requiring Elasticsearch for the MVP.
- **Referential integrity preserved**: Foreign keys still enforce relationships between core entities. JSONB columns add flexibility within entities, not between them.
- **GDPR compliance**: Row-level deletion and RLS policies work identically for relational columns and JSONB fields. No special handling needed for document data.
- **Familiar to most teams**: Any team comfortable with PostgreSQL can adopt this pattern immediately. JSONB is not a separate technology to learn -- it is a column type with query operators.

### Cons

- **No schema enforcement on JSONB**: Unlike typed columns, JSONB columns accept any valid JSON. Application-level validation (JSON Schema, Zod, etc.) is required to ensure data quality. A malformed evaluation rubric won't be caught by the database.
- **JSONB update performance**: Updating a single field within a large JSONB document requires rewriting the entire JSONB value. Frequently-updated JSONB columns can cause table bloat and require more aggressive vacuuming. The `jsonb_set()` function helps but does not eliminate the overhead.
- **GIN index maintenance cost**: GIN indexes on JSONB columns are powerful for reads but expensive to maintain during writes. High-write tables (proctor_events, code_submissions) may see write amplification from GIN indexes.
- **Query complexity**: JSONB path queries (`content -> 'dimensions' -> 'correctness' ->> 'score'`) are harder to read and debug than simple column references. Deep nesting increases query complexity.
- **No cross-document JOINs**: You cannot efficiently JOIN on values inside JSONB columns. If a query pattern emerges that requires joining on a JSONB field, that field should be promoted to a first-class column.
- **Reporting limitations**: Business intelligence tools and non-technical analysts often struggle with JSONB queries. An analytics/reporting layer may need to flatten JSONB into views or materialized views.
- **Risk of "JSONB everything"**: Without discipline, teams may dump too much data into JSONB columns, losing the benefits of typed schemas. Clear guidelines are needed for what belongs in JSONB vs. typed columns.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ (managed via Supabase, Neon, or AWS RDS) |
| **ORM** | Prisma (with `Json` type support) or Drizzle ORM (native JSONB support) |
| **Schema validation** | Zod (TypeScript) or Pydantic (Python) for JSONB content validation at the application layer |
| **Migrations** | Prisma Migrate or Drizzle Kit; JSONB columns migrated via application-level transformation |
| **Real-time** | Yjs CRDT for collaborative editor; Supabase Realtime or custom WebSocket for session state |
| **Search** | PostgreSQL full-text search for MVP; Elasticsearch for advanced problem search at scale |
| **Code execution** | Judge0 CE (self-hosted) |
| **Object storage** | S3-compatible for recordings and large assets |
| **Cache** | Redis for session state and hot queries |

---

## Migration and Scaling Considerations

### Migration Path

1. **MVP**: Single PostgreSQL instance. All tables in one database. JSONB validation via Zod schemas in the application layer. Connection pooling with PgBouncer.
2. **Growth**: Add read replicas for reporting queries. Create materialized views to flatten JSONB for analytics dashboards. Partition `audit_logs` and `proctor_events` by month.
3. **Scale**: Extract high-frequency real-time data (CRDT ops, cursor positions) to Redis Streams. Promote frequently-queried JSONB fields to typed columns as query patterns stabilize. Consider ClickHouse for analytical workloads.
4. **Enterprise**: Row-level security policies on `organization_id`. Column-level encryption for PII fields. JSONB content encryption for sensitive evaluation notes.

### JSONB Column Sizing Guidelines

| Column | Expected Size | Update Frequency | GIN Index? |
|--------|--------------|-------------------|------------|
| `organizations.settings` | 1-5 KB | Rare | Yes |
| `problems.content` | 2-20 KB | Rare | Yes |
| `problems.test_cases` | 1-10 KB | Rare | No |
| `evaluations.scores` | 1-5 KB | Once per eval | No |
| `ai_evaluations.report` | 5-50 KB | Once per session | No |
| `interview_sessions.session_config` | 1-5 KB | Rare | Yes |
| `proctor_events.details` | 0.1-1 KB | Write-once | No |
| `ats_webhook_log.payload` | 1-50 KB | Write-once | No |
| `whiteboard_snapshots.excalidraw_state` | 10-500 KB | Per snapshot | No |

### When to Promote JSONB to Columns

Move a field from JSONB to a typed column when:
- It appears in WHERE clauses or JOINs regularly
- It is needed for ORDER BY or aggregation (SUM, AVG)
- It has a stable, non-evolving schema
- It is referenced by foreign keys or unique constraints

This approach is the recommended starting point for most teams. It provides the relational safety net for core business logic while offering document-store flexibility for the many areas of a technical interview platform where data shapes are inherently variable.
