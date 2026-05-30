# Data Model Suggestion 4: Graph-Based Skill & Competency Mapping (Neo4j + PostgreSQL)

> Project: Technical Interview Platform (#295)
> Approach: Polyglot persistence — Neo4j graph database for skills, competencies, and candidate-problem matching; PostgreSQL for transactional data

## Summary

A polyglot persistence architecture that introduces a graph database (Neo4j) alongside PostgreSQL to model the domain's most complex relationship network: skills, competencies, problems, candidates, and evaluation outcomes. The graph captures how skills relate to each other, which problems test which skill combinations, how candidates demonstrate competencies across sessions, and how interviewer evaluations map onto a skills taxonomy. PostgreSQL continues to handle transactional operations (session management, user accounts, ATS integration), while Neo4j powers the intelligence layer: adaptive problem recommendation, skill gap analysis, natural-language problem search, bias detection across skill dimensions, and candidate-to-role matching.

This approach is uniquely suited to the platform's AI-native differentiators -- particularly natural-language problem selection, adaptive difficulty calibration, and post-interview impact tracking -- because these features are fundamentally graph traversal problems.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                 Application Layer               │
│  (API Server, WebSocket Server, AI Services)    │
└───────┬──────────────────────────┬──────────────┘
        │                          │
        │  Transactional ops       │  Graph queries
        │  (sessions, users,       │  (skills, matching,
        │   evaluations, ATS)      │   recommendations)
        │                          │
┌───────▼──────────┐      ┌───────▼──────────────┐
│   PostgreSQL     │      │      Neo4j           │
│                  │      │                      │
│  organizations   │      │  Skills Taxonomy     │
│  users           │◄────►│  Problem Graph       │
│  candidates      │ sync │  Candidate Profiles  │
│  applications    │      │  Evaluation Results  │
│  sessions        │      │  Role Requirements   │
│  code_submissions│      │  Interviewer Patterns│
│  audit_logs      │      │                      │
│  ats_integrations│      │                      │
└──────────────────┘      └──────────────────────┘
```

---

## Neo4j Graph Model

### Node Types

```cypher
// === SKILLS TAXONOMY ===

// Skill domains: broad categories of technical knowledge
(:SkillDomain {
  id: "sd-backend",
  name: "Backend Development",
  description: "Server-side programming, APIs, databases"
})

// Skills: specific competencies within a domain
(:Skill {
  id: "sk-python-async",
  name: "Python Async I/O",
  description: "asyncio, aiohttp, async/await patterns",
  difficulty_range: ["mid", "senior", "staff"],
  aliases: ["asyncio", "python coroutines", "async python"]
})

// Skill levels: proficiency tiers for a skill
(:SkillLevel {
  id: "sl-python-async-senior",
  level: "senior",
  description: "Can design async architectures, handle complex concurrency patterns",
  indicators: ["Implements custom event loops", "Designs async middleware", "Handles backpressure"]
})

// === PROBLEMS ===

(:Problem {
  id: "prob-001",
  title: "Async Rate Limiter",
  difficulty: "hard",
  type: "algorithm",
  time_limit_mins: 45,
  usage_count: 234,
  avg_completion_rate: 0.42,
  avg_time_to_solve_mins: 32
})

(:ProblemVariant {
  id: "pv-001-easy",
  parent_problem_id: "prob-001",
  difficulty: "easy",
  generated_by: "gpt-4o",
  title: "Simple Rate Counter"
})

// === PEOPLE ===

(:Candidate {
  id: "cand-xyz",
  anonymized_id: "anon-abc",
  pg_candidate_id: "uuid-from-postgres"  // link to PostgreSQL record
})

(:Interviewer {
  id: "int-abc",
  pg_user_id: "uuid-from-postgres",
  total_interviews: 156,
  avg_score_given: 3.2,
  consistency_rating: 0.87
})

// === ROLES ===

(:JobRole {
  id: "role-backend-senior",
  title: "Senior Backend Engineer",
  department: "Engineering",
  seniority: "senior"
})

// === TAGS ===

(:Tag {
  id: "tag-dp",
  name: "dynamic-programming",
  category: "algorithm_technique"
})
```

### Relationship Types

```cypher
// === SKILL RELATIONSHIPS ===

// Skill taxonomy hierarchy
(:SkillDomain)-[:CONTAINS]->(:Skill)
// e.g., (Backend Development)-[:CONTAINS]->(Python Async I/O)

// Skill prerequisites and adjacencies
(:Skill)-[:REQUIRES {strength: 0.8}]->(:Skill)
// e.g., (Python Async I/O)-[:REQUIRES]->(Python Fundamentals)

(:Skill)-[:RELATED_TO {similarity: 0.7}]->(:Skill)
// e.g., (Python Async I/O)-[:RELATED_TO]->(Node.js Event Loop)

(:Skill)-[:HAS_LEVEL]->(:SkillLevel)
// e.g., (Python Async I/O)-[:HAS_LEVEL]->(Senior Level)

// === PROBLEM-SKILL MAPPING ===

// Which skills does a problem test?
(:Problem)-[:TESTS {weight: 0.8, is_primary: true}]->(:Skill)
// e.g., (Async Rate Limiter)-[:TESTS {weight: 0.8}]->(Python Async I/O)

(:Problem)-[:TESTS {weight: 0.3, is_primary: false}]->(:Skill)
// e.g., (Async Rate Limiter)-[:TESTS {weight: 0.3}]->(Data Structures)

(:Problem)-[:TAGGED_WITH]->(:Tag)
// e.g., (Async Rate Limiter)-[:TAGGED_WITH]->(concurrency)

(:Problem)-[:HAS_VARIANT]->(:ProblemVariant)
// e.g., (Async Rate Limiter)-[:HAS_VARIANT]->(Simple Rate Counter)

// === CANDIDATE SKILL EVIDENCE ===

// Demonstrated competency from an interview session
(:Candidate)-[:DEMONSTRATED {
  session_id: "sess-123",
  score: 4,
  max_score: 5,
  evidence: "Correctly implemented sliding window with async locks",
  evaluated_at: datetime("2026-05-20T14:30:00Z")
}]->(:Skill)

// Problem attempt results
(:Candidate)-[:ATTEMPTED {
  session_id: "sess-123",
  solved: true,
  time_mins: 28,
  hints_used: 1,
  code_quality_score: 4.2,
  complexity_achieved: "O(n)",
  optimal_complexity: "O(n)"
}]->(:Problem)

// === ROLE REQUIREMENTS ===

// What skills does a role require?
(:JobRole)-[:REQUIRES_SKILL {
  min_level: "senior",
  importance: "critical",
  weight: 1.0
}]->(:Skill)

(:JobRole)-[:REQUIRES_SKILL {
  min_level: "mid",
  importance: "nice_to_have",
  weight: 0.3
}]->(:Skill)

// === INTERVIEWER PATTERNS ===

// Interviewer's evaluation patterns (for bias detection)
(:Interviewer)-[:EVALUATED {
  session_id: "sess-123",
  score_given: 4,
  decision: "yes",
  talk_time_ratio: 0.35,
  questions_asked: 8,
  evaluated_at: datetime("2026-05-20T15:00:00Z")
}]->(:Candidate)

(:Interviewer)-[:SPECIALIZES_IN {
  assessment_count: 45,
  avg_candidate_score: 3.8
}]->(:Skill)

// === RECOMMENDATION EDGES ===

// Problems that follow well from each other
(:Problem)-[:GOOD_FOLLOW_UP {
  reason: "tests_deeper_understanding",
  difficulty_delta: 1
}]->(:Problem)
```

---

## Example Graph Queries

### 1. Natural-Language Problem Selection

"Find mid-level Python problems that test async I/O and error handling"

```cypher
MATCH (p:Problem)-[:TESTS]->(s:Skill)
WHERE s.name CONTAINS 'Async' OR s.name CONTAINS 'Error Handling'
  AND p.difficulty IN ['medium', 'hard']
MATCH (p)-[:TESTS]->(s2:Skill)<-[:CONTAINS]-(sd:SkillDomain {name: 'Python'})
RETURN p.title, p.difficulty, p.avg_completion_rate,
       COLLECT(DISTINCT s.name) AS skills_tested
ORDER BY p.usage_count DESC
LIMIT 10
```

### 2. Adaptive Difficulty Calibration

"Given this candidate's performance so far, what difficulty should the next problem be?"

```cypher
MATCH (c:Candidate {id: $candidateId})-[a:ATTEMPTED]->(p:Problem)-[:TESTS]->(s:Skill)
WITH c, s, AVG(a.code_quality_score) AS avg_score, COUNT(a) AS attempts
WHERE attempts >= 1
WITH c, COLLECT({skill: s.name, score: avg_score}) AS skill_profile,
     AVG(avg_score) AS overall_avg
RETURN
  CASE
    WHEN overall_avg >= 4.0 THEN 'hard'
    WHEN overall_avg >= 2.5 THEN 'medium'
    ELSE 'easy'
  END AS recommended_difficulty,
  skill_profile
```

### 3. Candidate-Role Fit Score

"How well does this candidate match the Senior Backend Engineer role?"

```cypher
MATCH (role:JobRole {id: $roleId})-[req:REQUIRES_SKILL]->(s:Skill)
OPTIONAL MATCH (c:Candidate {id: $candidateId})-[demo:DEMONSTRATED]->(s)
WITH role, s, req,
     COALESCE(demo.score, 0) AS candidate_score,
     req.weight AS skill_weight,
     req.importance AS importance
WITH role,
     SUM(candidate_score * skill_weight) AS weighted_score,
     SUM(5 * skill_weight) AS max_possible_score,
     COLLECT({
       skill: s.name,
       required_level: req.min_level,
       candidate_score: candidate_score,
       importance: importance,
       gap: CASE WHEN candidate_score = 0 THEN 'missing'
                 WHEN candidate_score < 3 THEN 'below'
                 ELSE 'meets' END
     }) AS skill_breakdown
RETURN
  ROUND(100.0 * weighted_score / max_possible_score, 1) AS fit_percentage,
  skill_breakdown
```

### 4. Bias Detection Across Interviewers

"Compare interviewer scoring patterns for the same skills"

```cypher
MATCH (i:Interviewer)-[e:EVALUATED]->(c:Candidate)-[:DEMONSTRATED]->(s:Skill)
WHERE s.id = $skillId
WITH i, AVG(e.score_given) AS avg_score, STDEV(e.score_given) AS score_stddev,
     COUNT(e) AS eval_count, AVG(e.talk_time_ratio) AS avg_talk_time
WHERE eval_count >= 10
RETURN i.id, avg_score, score_stddev, eval_count, avg_talk_time
ORDER BY avg_score
```

### 5. Problem Recommendation After Current Problem

"What problem should follow this one based on the candidate's performance?"

```cypher
MATCH (current:Problem {id: $currentProblemId})-[:TESTS]->(s:Skill)
MATCH (c:Candidate {id: $candidateId})-[a:ATTEMPTED {session_id: $sessionId}]->(current)
WITH current, s, a,
     CASE WHEN a.solved AND a.time_mins < current.time_limit_mins * 0.6 THEN 'increase'
          WHEN NOT a.solved THEN 'decrease'
          ELSE 'maintain' END AS difficulty_direction

MATCH (next:Problem)-[:TESTS]->(s)
WHERE next.id <> current.id
  AND (difficulty_direction = 'increase' AND next.difficulty > current.difficulty
       OR difficulty_direction = 'decrease' AND next.difficulty < current.difficulty
       OR difficulty_direction = 'maintain' AND next.difficulty = current.difficulty)
OPTIONAL MATCH (current)-[fu:GOOD_FOLLOW_UP]->(next)
RETURN next.title, next.difficulty, next.avg_completion_rate,
       COLLECT(DISTINCT s.name) AS overlapping_skills,
       fu IS NOT NULL AS is_curated_follow_up
ORDER BY is_curated_follow_up DESC, next.usage_count DESC
LIMIT 5
```

### 6. Skills Gap Analysis for a Hiring Pipeline

"What skills are we failing to assess across our open roles?"

```cypher
MATCH (role:JobRole)-[req:REQUIRES_SKILL {importance: 'critical'}]->(s:Skill)
OPTIONAL MATCH (p:Problem)-[:TESTS]->(s)
WITH s, COUNT(DISTINCT role) AS roles_requiring, COUNT(DISTINCT p) AS problems_available
WHERE problems_available < 3
RETURN s.name, roles_requiring, problems_available,
       roles_requiring - problems_available AS coverage_gap
ORDER BY coverage_gap DESC
```

---

## PostgreSQL Schema (Transactional Layer)

The PostgreSQL schema remains similar to Suggestion 1 or 3, handling transactional operations. Key additions for graph synchronization:

```sql
-- Sync tracking between PostgreSQL and Neo4j
CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(50) NOT NULL,   -- 'candidate', 'problem', 'evaluation', 'session'
    entity_id       UUID NOT NULL,
    sync_direction  VARCHAR(10) NOT NULL,    -- 'pg_to_neo4j', 'neo4j_to_pg'
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'pending',
    error_message   TEXT,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Skills taxonomy cache in PostgreSQL for simple lookups
CREATE TABLE skills_cache (
    id              UUID PRIMARY KEY,
    domain          VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    difficulty_range VARCHAR(100),
    neo4j_node_id   BIGINT,
    last_synced     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Problem-skill mapping cache for fast filtering in the API layer
CREATE TABLE problem_skills_cache (
    problem_id      UUID NOT NULL REFERENCES problems(id),
    skill_id        UUID NOT NULL REFERENCES skills_cache(id),
    weight          NUMERIC(3,2),
    is_primary      BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (problem_id, skill_id)
);
```

---

## Data Synchronization Strategy

```
PostgreSQL (source of truth for transactional data)
    │
    │  CDC (Change Data Capture) or Application Events
    │
    ├──► Neo4j Graph Writer Service
    │      - Creates/updates nodes when candidates, problems, evaluations change
    │      - Maintains relationship edges
    │      - Runs on schedule or event-driven
    │
    └──► Graph Query Service
           - Serves recommendation queries from Neo4j
           - Caches results in Redis for hot paths
           - Falls back to PostgreSQL if Neo4j is unavailable
```

### Sync Rules

| PostgreSQL Event | Neo4j Action |
|-----------------|--------------|
| New candidate created | Create `(:Candidate)` node |
| New problem created | Create `(:Problem)` node, create `[:TESTS]` edges to skills |
| Evaluation submitted | Create `[:DEMONSTRATED]` edges from candidate to skills |
| Code submission scored | Update `[:ATTEMPTED]` edge properties |
| New job position created | Create `(:JobRole)` node, create `[:REQUIRES_SKILL]` edges |
| Session completed | Update `[:EVALUATED]` edges for bias analysis |

---

## Pros and Cons

### Pros

- **Natural skill relationship modeling**: Skills form a graph (prerequisites, adjacencies, hierarchies, similarities). A graph database represents these relationships natively, enabling traversal queries that would require recursive CTEs or multiple JOINs in a relational database.
- **Powerful recommendation queries**: "Find problems that test skills adjacent to what this candidate has demonstrated" is a simple 2-hop traversal in Neo4j. In PostgreSQL, this requires complex self-joins on junction tables with poor performance at scale.
- **Adaptive difficulty is a graph problem**: Determining the next best problem for a candidate based on their skill profile and performance trajectory is a weighted graph traversal, which Neo4j handles efficiently.
- **Bias detection across dimensions**: Comparing interviewer scoring patterns across skill domains, candidate demographics, and time periods involves multi-dimensional relationship analysis that graphs excel at.
- **Skills ontology alignment**: The HR Open Standards and emerging skills ontology frameworks (ESCO, O*NET, Lightcast) define skills as graphs. Using a graph database aligns the data model with industry standards for skill classification.
- **Candidate-role matching**: Computing a "fit score" between a candidate's demonstrated competencies and a role's requirements is a natural graph matching problem.
- **Flexible taxonomy evolution**: Adding new skills, re-categorizing existing ones, or merging duplicate skills requires changing edges in the graph -- not schema migrations.

### Cons

- **Operational complexity**: Running two databases (PostgreSQL + Neo4j) doubles the operational burden: backup strategies, monitoring, connection pooling, failover, and version upgrades for two systems.
- **Data synchronization risk**: Keeping PostgreSQL and Neo4j in sync requires a reliable CDC or event pipeline. Sync failures can cause stale recommendations or inconsistent skill profiles. Eventual consistency must be tolerated.
- **Team expertise requirements**: Neo4j and Cypher are less widely known than SQL. Hiring developers and debugging graph queries require specialized expertise.
- **Neo4j licensing considerations**: Neo4j Community Edition is open source (GPLv3), but advanced features (clustering, role-based access, full-text search) require Neo4j Enterprise (commercial licence). Alternative: consider Apache AGE (PostgreSQL graph extension) to avoid a separate database.
- **Not suited for transactional workloads**: Neo4j is not designed for high-throughput transactional writes (e.g., code submissions, proctoring events). These must stay in PostgreSQL.
- **Graph query performance pitfalls**: Poorly written Cypher queries (e.g., unbounded traversals, Cartesian products) can cause severe performance issues. Query optimization requires graph-specific expertise.
- **Overhead for small deployments**: For a platform with fewer than 10,000 problems and 50,000 candidates, the skill relationships may not be complex enough to justify a separate graph database. PostgreSQL with well-designed junction tables may suffice.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Graph database** | Neo4j Community Edition (self-hosted) or Neo4j AuraDB (managed); alternatively, Apache AGE (PostgreSQL extension) for simpler deployments |
| **Transactional database** | PostgreSQL 16+ |
| **Sync pipeline** | Debezium CDC (PostgreSQL WAL to Kafka) + custom Neo4j writer; or application-level dual-write with outbox pattern |
| **Graph query layer** | Neo4j JavaScript/Python driver with connection pooling |
| **Skills ontology source** | ESCO (European Skills/Competences), Lightcast Open Skills, or O*NET |
| **ORM** | Prisma (PostgreSQL) + neo4j-driver (graph queries) |
| **Cache** | Redis for hot graph query results (problem recommendations, candidate fit scores) |
| **Search** | Neo4j full-text search for skill/problem queries; PostgreSQL FTS for general search |

---

## Migration and Scaling Considerations

### Migration Path

1. **MVP**: PostgreSQL only with junction tables for problem-skill mapping. Implement basic problem search and recommendation in SQL. Design the skills taxonomy data model but store it in PostgreSQL tables.
2. **Growth (v1.1)**: Introduce Neo4j for the skills graph. Migrate the skills taxonomy, problem-skill mappings, and candidate-skill evidence to the graph. Keep PostgreSQL as the transactional source of truth with CDC sync to Neo4j.
3. **Scale**: Add GraphRAG capabilities -- use the skills knowledge graph with an LLM to power natural-language problem search and candidate evaluation summaries. Partition Neo4j by organization for multi-tenant isolation.
4. **Enterprise**: Neo4j Enterprise Edition for clustering and RBAC. Multi-region graph replication. Integration with external skills ontologies (ESCO, Lightcast) for standardized skill taxonomy alignment.

### Apache AGE Alternative

For teams that want graph querying without a separate database, Apache AGE adds graph capabilities directly to PostgreSQL:

```sql
-- Apache AGE: graph queries inside PostgreSQL
SELECT * FROM cypher('skills_graph', $$
  MATCH (p:Problem)-[:TESTS]->(s:Skill {name: 'Python Async I/O'})
  RETURN p.title, p.difficulty
$$) AS (title agtype, difficulty agtype);
```

This eliminates the sync problem entirely but with reduced graph performance compared to native Neo4j.

### Graph Size Estimates (Year 1)

| Node Type | Estimated Count |
|-----------|----------------|
| Skill | 500-2,000 |
| SkillDomain | 30-50 |
| SkillLevel | 1,500-6,000 |
| Problem | 5,000-20,000 |
| ProblemVariant | 10,000-50,000 |
| Candidate | 50,000-200,000 |
| Interviewer | 1,000-5,000 |
| JobRole | 500-2,000 |
| Tag | 200-500 |

| Relationship Type | Estimated Count |
|-------------------|----------------|
| TESTS | 20,000-80,000 |
| DEMONSTRATED | 100,000-500,000 |
| ATTEMPTED | 100,000-500,000 |
| REQUIRES_SKILL | 2,000-10,000 |
| RELATED_TO | 5,000-20,000 |
| REQUIRES | 2,000-8,000 |
| EVALUATED | 50,000-200,000 |

This graph size is well within Neo4j Community Edition's capabilities and would support sub-second query performance for all recommendation and matching queries.

---

## When to Choose This Approach

This approach is the best choice when:

- **Intelligent problem recommendation** is a core differentiator (not just a feature backlog item)
- **Skills-based hiring** alignment with industry ontologies (ESCO, O*NET) is required by enterprise buyers
- **Adaptive difficulty calibration** and **natural-language problem search** are planned for early releases
- **Post-hire impact tracking** (correlating interview scores with on-job performance) is in scope
- The team has or can acquire **graph database expertise**
- The platform will manage **10,000+ problems** and **complex skill taxonomies**

For teams that do not need these capabilities at launch, start with Suggestion 3 (hybrid JSONB) and migrate skill-related data to a graph database when the recommendation and matching features reach production priority.
