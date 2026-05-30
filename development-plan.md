# Technical Interview Platform — Phased Development Plan

> Project: 295-technical-interview-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. All core research is present. The database design follows **Data Model Suggestion 3 (Hybrid Relational + JSONB)**, which the research explicitly recommends as the best balance of integrity and flexibility for this domain. The graph-based intelligence concepts from Suggestion 4 are deferred to a late, optional phase.

---

## Product Synopsis

An AI-native, open-source platform for live coding interviews, system-design whiteboarding, and automated evaluation in technical hiring. It is the only open-source entrant in a market of nine proprietary SaaS incumbents (CoderPad, Codility, iMocha, HackerEarth, etc.). Core value: a single browser-based workspace combining real-time collaborative code editing, sandboxed code execution, HD video, collaborative whiteboarding, session recording, and structured evaluation — augmented throughout by AI (post-interview reports, co-interviewer suggestions, bias detection, problem-variant generation).

**Primary personas:** engineering hiring managers, technical recruiters/coordinators, panel interviewers, and candidates. **Deployment model:** self-hostable SaaS (Docker Compose for MVP; Kubernetes-ready). **Key differentiators:** open source, first-class system-design tooling, and AI features absent from all incumbents (real-time co-interviewer, bias detection, adaptive difficulty, isomorphic problem variants).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) end-to-end | The product is API- and frontend-heavy with intense real-time requirements (WebSocket sync, WebRTC signalling, live editor). One language across server, client, and shared schema types minimises duplication. The Yjs CRDT ecosystem and Excalidraw are TypeScript-native. |
| Backend framework | NestJS | Modular DI structure suits a large multi-domain app (sessions, evaluation, ATS, billing). First-class WebSocket gateways, guards for RBAC, and OpenAPI generation via `@nestjs/swagger`. |
| Real-time collaborative editor | Yjs (CRDT, MIT) + `y-websocket` provider | `standards.md` recommends evaluating Yjs as the open-source collaborative-editing foundation; CRDTs converge without central op-ordering and are the modern choice (Figma, VS Code Live Share). Avoids hand-rolling OT. |
| Code editor UI | CodeMirror 6 with `y-codemirror.next` binding | Lightweight, accessible (WCAG-friendly keyboard nav), native Yjs binding, supports 10+ language modes. |
| Video conferencing | WebRTC (SFU via mediasoup) | `standards.md` (W3C WebRTC / RFC 8825) is the foundational standard. An SFU scales to 5+ panel interviewers far better than mesh P2P. mediasoup is open source and self-hostable. |
| Whiteboard | Excalidraw (embedded, MIT) with Yjs sync | Multiple incumbents (CodeInterview.io, TechInterview.live) use Excalidraw; exports to PNG/SVG and is Mermaid-adjacent for post-interview docs (per `standards.md`). |
| Code execution sandbox | Judge0 CE (self-hosted) running each submission in an isolated gVisor container | `standards.md` names Judge0 CE as the reference open-source execution backend (60+ languages). Known sandbox-escape risks mandate gVisor + isolated network + OWASP ASVS L2 review. |
| Primary database | PostgreSQL 16 with JSONB | Data Model Suggestion 3 (hybrid relational + JSONB). Normalised tables for core entities; JSONB for variable-shape data (rubrics, AI reports, whiteboard state, proctor details). Single engine, no MongoDB ops overhead. |
| ORM / migrations | Prisma | Recommended in Suggestion 1/3; type-safe, generates TS types shared with the API layer; `prisma migrate` for versioned schema. JSONB columns typed via Prisma `Json`. |
| Cache / pub-sub / queue | Redis 7 | Session presence, hot session state, WebSocket fan-out across server instances, and BullMQ job queue for async work (AI evaluation, ATS sync, recording transcode). |
| Async job queue | BullMQ (Redis-backed) | Long-running AI calls, recording processing, and ATS webhook delivery must run off the request path with retries and dead-letter handling. |
| Object storage | S3-compatible (MinIO self-hosted; AWS S3 in cloud) | Session recordings, whiteboard exports, code playback blobs. Lifecycle policies enforce GDPR retention. |
| LLM provider | Provider-agnostic gateway (Anthropic + OpenAI) behind an internal `LlmClient` interface | AI features are core differentiators; abstraction allows model swaps and self-host (Ollama) for privacy-sensitive customers. |
| Frontend | React 19 + Vite + TanStack Router/Query | SPA for the live workspace and dashboards; Vite for fast HMR; TanStack Query for server-state caching. |
| UI components | shadcn/ui + Tailwind CSS | Accessible primitives (Radix) help meet WCAG 2.2 AA; fast to build dashboards and forms. |
| Auth | OIDC/OAuth 2.1 (PKCE) via an `AuthModule` wrapping a provider (Keycloak self-host; Auth0 cloud) | `standards.md` mandates OIDC for enterprise SSO (Okta/Azure AD/Google). Keycloak is open source and self-hostable. |
| API auth (machine) | API keys (RFC 7617-style bearer) + OAuth 2.0 for ATS | Matches incumbent integration patterns (CoderPad, HackerEarth API keys; Greenhouse/Lever OAuth). |
| API contract | OpenAPI 3.1 (auto-generated) + JSON Schema Draft 2020-12 | `standards.md` requires OAS 3.1 for public/ATS APIs and JSON Schema for webhook/report payloads. |
| Containerisation | Docker + Docker Compose (MVP); Helm chart (later) | Self-hosted target; reproducible local dev including Judge0, Postgres, Redis, MinIO, mediasoup. |
| Testing | Vitest (unit), Supertest (API integration), Playwright (E2E), Testcontainers (real Postgres/Redis) | Standard TS stack; Testcontainers gives real-DB integration tests; Playwright covers the multi-user live workspace. |
| Code quality | ESLint + Prettier + `tsc --noEmit` (strict) | Standard TS toolchain; strict typing across shared schema. |
| Package manager / monorepo | pnpm workspaces + Turborepo | Shared `packages/shared` (types, Zod schemas) consumed by `apps/api` and `apps/web`; Turborepo caches builds/tests. |
| Observability | OpenTelemetry → Prometheus/Grafana; structured pino logs | SOC 2 availability/monitoring evidence; trace real-time and AI latency. |

### Project Structure

```
technical-interview-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # api, web, postgres, redis, minio, judge0, mediasoup, keycloak
├── Dockerfile.api
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── shared/                       # framework-agnostic, imported by api + web
│   │   ├── src/
│   │   │   ├── schemas/              # Zod schemas → JSON Schema + TS types
│   │   │   │   ├── session.ts
│   │   │   │   ├── evaluation.ts
│   │   │   │   ├── problem.ts
│   │   │   │   ├── ats.ts
│   │   │   │   └── ai-report.ts
│   │   │   ├── enums.ts              # SessionStatus, ParticipantRole, etc.
│   │   │   └── index.ts
│   └── config/                       # shared eslint/tsconfig/prettier presets
├── apps/
│   ├── api/                          # NestJS backend
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   ├── migrations/
│   │   │   └── seed.ts
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── common/               # guards, interceptors, RBAC, audit, errors
│   │   │   ├── auth/                 # OIDC, API keys, RBAC
│   │   │   ├── organizations/
│   │   │   ├── users/
│   │   │   ├── candidates/
│   │   │   ├── sessions/             # lifecycle, participants
│   │   │   ├── realtime/             # Yjs ws gateway, presence, signalling
│   │   │   ├── execution/            # Judge0 client, submissions
│   │   │   ├── whiteboard/           # Excalidraw sync + snapshots
│   │   │   ├── recording/            # SFU recording orchestration, transcode jobs
│   │   │   ├── problems/             # library, test cases, variants
│   │   │   ├── evaluation/           # rubrics, scores, decisions
│   │   │   ├── ai/                   # LlmClient, reports, co-interviewer, bias, variants
│   │   │   ├── proctoring/           # proctor events
│   │   │   ├── ats/                  # connectors, webhooks, sync
│   │   │   ├── analytics/            # hiring funnel + bias dashboards
│   │   │   ├── jobs/                 # BullMQ processors
│   │   │   └── storage/             # S3 client
│   │   └── test/                     # e2e + integration (Testcontainers)
│   └── web/                          # React SPA
│       ├── src/
│       │   ├── routes/
│       │   ├── features/
│       │   │   ├── workspace/        # editor + video + whiteboard + console
│       │   │   ├── dashboard/
│       │   │   ├── problems/
│       │   │   ├── evaluation/
│       │   │   └── reports/
│       │   ├── lib/                  # api client, yjs provider, webrtc client
│       │   └── components/ui/        # shadcn
│       └── tests/                    # Playwright
└── infra/
    ├── judge0/                       # Judge0 CE compose + gVisor config
    └── helm/                         # later
```

---

## Phase 1: Foundation, Schema & Auth

### Purpose
Establish the monorepo, shared schema layer, database, and authentication/RBAC. Nothing real-time or domain-specific works yet, but every later phase depends on a typed schema, migrations, a running API skeleton, and a working login with org/role enforcement. After this phase a user can sign in via OIDC and the API enforces role-based access.

### Tasks

#### 1.1 — Monorepo & shared schema package

**What**: Stand up the pnpm/Turborepo workspace with `packages/shared` exposing Zod schemas, enums, and generated TS types.

**Design**:
- `pnpm-workspace.yaml` lists `apps/*` and `packages/*`. `turbo.json` defines `build`, `lint`, `test`, `typecheck` pipelines with caching.
- Shared enums in `packages/shared/src/enums.ts`:
  ```ts
  export enum SessionType { LIVE_CODING='live_coding', SYSTEM_DESIGN='system_design', TAKE_HOME='take_home', PAIR_PROGRAMMING='pair_programming', BEHAVIORAL='behavioral' }
  export enum SessionStatus { SCHEDULED='scheduled', IN_PROGRESS='in_progress', COMPLETED='completed', CANCELLED='cancelled', NO_SHOW='no_show' }
  export enum ParticipantRole { INTERVIEWER='interviewer', CANDIDATE='candidate', OBSERVER='observer', AI_COPILOT='ai_copilot' }
  export enum UserRole { ADMIN='admin', INTERVIEWER='interviewer', RECRUITER='recruiter', OBSERVER='observer' }
  export enum Decision { STRONG_YES='strong_yes', YES='yes', NEUTRAL='neutral', NO='no', STRONG_NO='strong_no' }
  ```
- Zod schemas are the single source of truth. A build script runs `zod-to-json-schema` to emit JSON Schema Draft 2020-12 artefacts into `packages/shared/dist/json-schema/` for webhook/report contract validation (per `standards.md`).
- Strict `tsconfig` base in `packages/config` (`strict: true`, `noUncheckedIndexedAccess: true`).

**Testing**:
- `Unit: Zod session schema parses a valid session payload → typed object`.
- `Unit: invalid session_type → ZodError listing allowed values`.
- `Unit: generated JSON Schema for AiReport validates a sample report fixture (ajv) → no errors`.
- `Build: turbo build compiles all packages with zero tsc errors`.

#### 1.2 — Database schema & migrations (hybrid relational + JSONB)

**What**: Implement the Prisma schema per Data Model Suggestion 3 and generate the initial migration plus seed data.

**Design**:
- Core normalised tables (relational): `organizations`, `users`, `candidates`, `job_positions`, `applications`, `interview_sessions`, `session_participants`, `code_submissions`, `problems`, `problem_test_cases`, `tags`, `problem_tags`, `evaluation_rubrics`, `evaluation_scores`, `ai_evaluations`, `proctor_events`, `audit_logs`, `ats_integrations`, `ats_sync_logs`, `session_recordings`, `whiteboard_snapshots`, `api_keys`.
- JSONB columns (the hybrid element) for variable-shape data:
  - `evaluation_rubrics.criteria JSONB` — array of `{ id, name, description, maxScore, weight }` instead of a separate `rubric_criteria` table (rubrics are document-like and edited as a whole).
  - `evaluation_scores.criterion_scores JSONB` — `{ [criterionId]: { score, notes } }`.
  - `ai_evaluations.payload JSONB` — model output (summary, evidence citations, dimension scores) whose shape varies by `evaluation_type`.
  - `problems.metadata JSONB` — flexible skill/topic metadata.
  - `whiteboard_snapshots.scene JSONB` — Excalidraw scene.
  - `proctor_events.details JSONB`.
  - `ats_integrations.credentials_enc BYTEA` (encrypted) + `ats_integrations.config JSONB`.
- Representative Prisma model:
  ```prisma
  model InterviewSession {
    id            String   @id @default(uuid()) @db.Uuid
    application   Application @relation(fields: [applicationId], references: [id])
    applicationId String   @db.Uuid
    sessionType   String   // SessionType
    title         String?
    scheduledAt   DateTime?
    startedAt     DateTime?
    endedAt       DateTime?
    durationLimit Int?
    status        String   @default("scheduled")
    videoRoomId   String?
    language      String?
    isAnonymized  Boolean  @default(false)
    createdById   String?  @db.Uuid
    createdAt     DateTime @default(now())
    updatedAt     DateTime @updatedAt
    participants  SessionParticipant[]
    submissions   CodeSubmission[]
    @@index([applicationId])
    @@index([scheduledAt])
    @@index([status])
  }
  ```
- Multi-tenancy: every tenant-scoped table carries `organizationId`; Phase 1 enforces it in the query layer, with Postgres Row-Level Security policies added in the security phase.
- Indexes per Suggestion 1 (applications by candidate/job/status; sessions by application/scheduled/status; audit_logs by org+time desc).
- `seed.ts` inserts one demo org, an admin user, two job positions, and ~20 sample problems with test cases.

**Testing**:
- `Integration (Testcontainers Postgres): prisma migrate deploy → all tables/indexes exist (introspect)`.
- `Integration: insert session referencing missing application → FK violation`.
- `Integration: participant with both user_id and candidate_id set → CHECK constraint rejects (participant_identity)`.
- `Integration: write + read rubric.criteria JSONB → round-trips structurally`.
- `Integration: seed script runs idempotently twice → no duplicate-key errors`.

#### 1.3 — API skeleton, config, health, OpenAPI

**What**: Bootable NestJS app with typed config, `/healthz`/`/readyz`, global error handling, and auto-generated OpenAPI 3.1.

**Design**:
- `ConfigModule` validates env via Zod (`DATABASE_URL`, `REDIS_URL`, `S3_*`, `JUDGE0_URL`, `OIDC_*`, `LLM_*`); boot fails fast on missing required vars.
- Global `HttpExceptionFilter` returns RFC 7807 problem+json: `{ type, title, status, detail, instance }`.
- `@nestjs/swagger` serves OAS 3.1 at `/api/docs` and writes `openapi.json` at build for contract tests.
- `/healthz` (liveness) and `/readyz` (checks Postgres + Redis).

**Testing**:
- `Unit: config validation with missing DATABASE_URL → boot throws with field name`.
- `Integration: GET /readyz with DB+Redis up → 200; with Redis down → 503`.
- `Integration: unhandled error → 500 problem+json with type/title/status`.
- `Contract: generated openapi.json is valid OAS 3.1 (swagger-parser validate)`.

#### 1.4 — Authentication, sessions, and RBAC

**What**: OIDC login (Authorization Code + PKCE), JWT session, API-key auth for machine clients, and a role-based access guard.

**Design**:
- `AuthModule` integrates an OIDC provider (Keycloak dev realm). Flow: `/auth/login` → IdP → `/auth/callback` exchanges code (PKCE) → issues short-lived access JWT (15 min) + rotating refresh token (httpOnly cookie).
- JWT claims: `sub`, `organizationId`, `role`, `email`. `JwtAuthGuard` populates `req.user`.
- `@Roles(UserRole.ADMIN)` decorator + `RolesGuard` enforce RBAC. Candidates authenticate via signed, single-use, expiring **join tokens** (not full accounts) — `POST /sessions/:id/join-token` returns a JWT scoped to one session with `role=candidate`.
- `api_keys` table stores hashed keys (argon2); `ApiKeyGuard` validates `Authorization: Bearer <key>` and scopes by org. Endpoints accept either JWT or API key.
- All auth events written to `audit_logs`.

**Testing**:
- `Integration (mocked IdP): callback with valid code → access JWT + refresh cookie set`.
- `Integration: request to @Roles(ADMIN) endpoint as interviewer → 403`.
- `Integration: candidate join-token grants access only to its own session; other session → 403`.
- `Unit: API key stored hashed (argon2), never plaintext; lookup matches by hash`.
- `Integration: expired join-token → 401`.

---

## Phase 2: Session Lifecycle & Domain Core

### Purpose
Build the transactional heart of the product: organizations/users/candidates/applications CRUD and the interview-session lifecycle state machine with participants. After this phase a recruiter can create a job, add a candidate/application, schedule a session, and a candidate can be issued a join link — all enforced and audited — though the live workspace itself comes next.

### Tasks

#### 2.1 — Org/user/candidate/application CRUD

**What**: REST resources for the foundational entities with org-scoping and audit logging.

**Design**:
- Endpoints (all org-scoped, RBAC-guarded):
  - `POST/GET/PATCH /organizations` (admin), `POST/GET/PATCH /users`, `POST/GET/PATCH /candidates`, `POST/GET/PATCH /job-positions`, `POST/GET/PATCH /applications`.
- Candidate `anonymizedId` generated on create (`anon_<base32(random)>`) to support bias-free review later.
- `AuditInterceptor` records `(actor, action, resource_type, resource_id, ip)` for every mutating request.
- Pagination: cursor-based (`?limit=&cursor=`) returning `{ data, nextCursor }`.

**Testing**:
- `Integration: create candidate → anonymizedId populated and unique`.
- `Integration: user in org A cannot GET candidate in org B → 404 (not 403, to avoid existence leak)`.
- `Integration: every create/update writes one audit_logs row with correct actor`.
- `Unit: cursor pagination returns stable ordering across pages`.

#### 2.2 — Interview-session lifecycle state machine

**What**: Create/schedule sessions and transition them through a validated state machine.

**Design**:
- States: `scheduled → in_progress → completed`; plus `scheduled → cancelled` and `scheduled → no_show`. Illegal transitions (e.g. `completed → in_progress`) rejected with 409.
  ```
  scheduled ──start──▶ in_progress ──end──▶ completed
      │                                   
      ├──cancel──▶ cancelled              
      └──no_show──▶ no_show               
  ```
- `POST /sessions` (recruiter/interviewer): body `{ applicationId, sessionType, scheduledAt, durationLimit, language, problemIds[] }`. Generates `videoRoomId` (uuid).
- `POST /sessions/:id/start`, `/end`, `/cancel`. `start` stamps `startedAt`; `end` stamps `endedAt` and enqueues the post-interview AI report job (Phase 7) and recording finalisation (Phase 6).
- `SessionParticipant` rows created when interviewers are assigned and when a candidate join-token is redeemed.

**Testing**:
- `Unit: transition completed→in_progress → InvalidTransitionError (409)`.
- `Integration: start sets startedAt and status=in_progress; end sets endedAt`.
- `Integration: end on a completed session → 409, no duplicate AI-report job enqueued`.
- `Integration: assign 6 interviewers to a panel-capped session → rejects beyond 5 (panel limit per features.md)`.

#### 2.3 — Session-scoped authorization

**What**: Ensure only assigned participants (and org admins/recruiters) can access a session's data.

**Design**:
- `SessionAccessGuard` resolves the session, checks the requester is a participant, the creator, or an org admin/recruiter; observers get read-only.
- Candidate join-tokens grant access to exactly one session and only to candidate-visible resources (editor, problem statement, whiteboard) — never evaluation scores or private notes.

**Testing**:
- `Integration: interviewer not on the panel requests session → 403`.
- `Integration: candidate token requests /sessions/:id/evaluations → 403`.
- `Integration: observer attempts POST evaluation → 403; GET allowed`.

---

## Phase 3: Real-Time Collaborative Workspace

### Purpose
Deliver the core differentiating experience: a real-time collaborative code editor with presence and cursor sync, scaled across server instances. This is the "heart" of the product and ships early per the phase-design principles. After this phase two browsers in the same session edit the same document live with visible cursors.

### Tasks

#### 3.1 — Yjs WebSocket gateway with auth & persistence

**What**: A NestJS WebSocket gateway that hosts Yjs documents per session, authorises connections, and persists document state.

**Design**:
- Transport: WebSocket (RFC 6455). One Yjs `Doc` per `sessionId`, addressed as room `session:<id>:editor`.
- Connection handshake requires a valid JWT or candidate join-token whose `sessionId` matches the requested room; otherwise the socket is closed with code 4401.
- Multi-instance fan-out: Yjs updates relayed across API replicas via Redis pub/sub (`y-redis`-style adapter), so participants on different instances converge.
- Persistence: debounced (every 2 s / on last-participant-leave) snapshot of the encoded Yjs state to `interview_sessions`-linked storage (Postgres `bytea` or S3 for large docs). On first join, the server seeds the doc from the latest snapshot or from the problem's `starter_code`.
- Document model: a Yjs `Y.Text` per open file keyed in a `Y.Map` (`files`), enabling the multi-file environment competitors offer.

**Testing**:
- `Integration: two ws clients edit concurrently → both converge to identical text (CRDT)`.
- `Integration: connect with token for a different session → socket closed 4401`.
- `Integration: clients on two API instances (Redis relay) → updates propagate`.
- `Integration: disconnect all, reconnect → document restored from snapshot`.

#### 3.2 — Presence, cursors & language selection

**What**: Awareness layer for participant presence, live cursors/selections, and per-session language switching.

**Design**:
- Uses Yjs `awareness` for ephemeral state: `{ userId, displayName, color, cursor: {anchor, head}, role }`.
- Anonymisation: when `session.isAnonymized`, candidate awareness shows `anonymizedId` and a neutral colour to reviewers.
- Language switch broadcasts a `Y.Map` field `language`; editor swaps CodeMirror language mode reactively.

**Testing**:
- `Integration: participant joins → awareness list includes them; leaves → removed within timeout`.
- `Integration: anonymized session → candidate displayName never sent to interviewer clients`.
- `Unit: cursor awareness encodes/decodes anchor+head correctly`.

#### 3.3 — Web workspace shell (editor)

**What**: React workspace route binding CodeMirror 6 to the Yjs doc with a participant rail and language picker.

**Design**:
- `features/workspace/EditorPanel.tsx` uses `y-codemirror.next`; connects via `lib/yjs-provider.ts` (WebsocketProvider with auth token in the URL/protocol header).
- Accessibility (WCAG 2.2 AA): full keyboard operability, visible focus rings, contrast-checked theme, ARIA labels on controls.
- Layout: resizable panes (editor | side panel) using a split-pane component; side panel hosts problem statement (Phase 5) and console (Phase 4).

**Testing**:
- `E2E (Playwright, 2 browser contexts): both join session → typing in A appears in B with cursor`.
- `E2E: keyboard-only navigation reaches all workspace controls (tab order)`.
- `E2E: switch language in A → B's editor mode updates`.

---

## Phase 4: Code Execution

### Purpose
Add sandboxed code execution so candidates get immediate output — a table-stakes feature and prerequisite for automated evaluation. After this phase, participants run the current editor code against stdin and see stdout/stderr/exit-code/timing in a console.

### Tasks

#### 4.1 — Judge0 client & isolated runner

**What**: A backend `ExecutionService` that submits code to a self-hosted, isolated Judge0 CE instance and returns results.

**Design**:
- `infra/judge0/` runs Judge0 CE; each submission executes inside a gVisor-sandboxed container on an isolated Docker network with no egress (mitigating the documented Judge0 sandbox-escape risk in `standards.md`; reviewed against OWASP ASVS L2).
- `submit(language, sourceCode, stdin, limits)` → creates a Judge0 submission with `cpu_time_limit`, `memory_limit`, `wall_time_limit` (defaults: 5 s CPU, 256 MB, 10 s wall) and polls (or uses Judge0 callback) until done.
- Language IDs mapped in a config table (Judge0 attributes). Result persisted to `code_submissions` (`stdout`, `stderr`, `exit_code`, `execution_time_ms`, `memory_used_kb`).
- Rate-limited per session (e.g. 1 concurrent run, 30 runs/min) via Redis token bucket to prevent abuse.

**Testing**:
- `Integration (real Judge0 in compose, marked slow): print "hello" in Python → stdout="hello", exit 0`.
- `Integration: infinite loop → wall-time-limit exceeded status, no hang`.
- `Integration: code attempting network egress → blocked (no connection)`.
- `Unit: unsupported language → 400 with supported-language list`.
- `Integration: exceed rate limit → 429`.

#### 4.2 — Execution API + live console

**What**: Endpoint and UI to run editor code and stream results to all participants.

**Design**:
- `POST /sessions/:id/execute` body `{ language, stdin? }` — server reads current source from the Yjs doc (authoritative), submits, and broadcasts the result over the session's realtime channel (`session:<id>:console`) so all participants see the same output.
- Console UI shows status (queued/running/done), stdout/stderr tabs, exit code, time, and memory; a stdin field feeds runs.

**Testing**:
- `Integration: execute reads current Yjs source (not stale client copy)`.
- `Integration: result broadcast → both participants' consoles update`.
- `E2E: candidate runs code with stdin → output shown to interviewer too`.

---

## Phase 5: Problem Library, Whiteboard & Evaluation

### Purpose
Complete the manual interview workflow: a searchable problem library with test cases, a collaborative system-design whiteboard (the market's weakest area, per research), and structured evaluation rubrics/scoring. After this phase a complete interview can be run and scored end-to-end without AI. These three tasks have no inter-dependencies and can be built in parallel.

### Tasks

#### 5.1 — Problem library & test cases

**What**: CRUD and search for problems, variants placeholder, tags, and weighted test cases (hidden/visible).

**Design**:
- `problems` with `metadata JSONB`; `problem_test_cases` (`input`, `expected_output`, `is_hidden`, `weight`, `sort_order`); `tags`/`problem_tags`.
- Search: `GET /problems?q=&difficulty=&type=&tags=` using PostgreSQL full-text search over title/description (Suggestion 1's recommendation); returns paginated results.
- Attaching a problem to a session copies its `starter_code` into the session's Yjs doc on session start.
- Visibility: global library (`organizationId NULL`, `is_public`) vs org-private problems.

**Testing**:
- `Integration: FTS query "binary tree" → ranks matching problems`.
- `Integration: hidden test cases not returned to candidate-scoped requests`.
- `Integration: attach problem to session → starter_code seeds editor on start`.
- `Unit: test-case weights normalise for scoring (sum used as denominator)`.

#### 5.2 — Collaborative system-design whiteboard

**What**: Embedded Excalidraw synced via Yjs, with snapshot capture and SVG/PNG export.

**Design**:
- Excalidraw scene stored in a Yjs `Y.Map` per session (`session:<id>:whiteboard`), reusing the Phase 3 gateway and Redis relay.
- `POST /sessions/:id/whiteboard/snapshots` persists the current scene to `whiteboard_snapshots.scene JSONB` with a label; export endpoint renders SVG/PNG (Excalidraw export utils) for post-interview docs (Mermaid/draw.io-adjacent per `standards.md`).
- Snapshots auto-captured on session end.

**Testing**:
- `Integration: two clients add shapes concurrently → scenes converge`.
- `Integration: create snapshot → row persisted; export returns valid SVG`.
- `Integration: session end → auto snapshot created`.

#### 5.3 — Evaluation rubrics, scores & decisions

**What**: Rubric templates (JSONB criteria), per-evaluator scoring with private notes, and a hiring decision.

**Design**:
- `evaluation_rubrics.criteria JSONB`: `[{ id, name, description, maxScore, weight }]`. Default rubric seeded per session type (e.g. system-design rubric: Problem Decomposition, Trade-off Analysis, Scalability, Communication).
- `POST /sessions/:id/evaluations` body `{ rubricId, criterionScores: {[id]:{score,notes}}, overallDecision, privateNotes }`. `overall_score` computed as weighted mean. One evaluation per evaluator per session (unique constraint).
- Private notes never exposed to candidates or other-org users; visible to panel + recruiters.
- Decision enum: `strong_yes…strong_no`.

**Testing**:
- `Integration: submit scores → overall_score = correct weighted mean`.
- `Integration: candidate-scoped request for evaluations → 403`.
- `Integration: second evaluation by same evaluator → 409 (unique)`.
- `Unit: criterion score above maxScore → validation error`.

---

## Phase 6: Video & Session Recording

### Purpose
Add HD video conferencing (WebRTC/SFU) with screen sharing and panel support, plus session recording (video + code playback + whiteboard) for asynchronous review. After this phase a full live interview — video, code, whiteboard, execution — is recorded and replayable. Depends on Phases 2–3; can be developed in parallel with Phase 5.

### Tasks

#### 6.1 — WebRTC signalling & SFU media

**What**: mediasoup SFU with a signalling channel for multi-party audio/video and screen share.

**Design**:
- Signalling over the existing WebSocket gateway (room `session:<id>:media`): exchange of router RTP capabilities, transport creation, produce/consume per W3C WebRTC / RFC 8825/8834.
- SFU supports up to 5 interviewers + 1 candidate + observers (per `features.md` panel cap); observers consume-only.
- Screen-share is a separate producer track; speaker detection via audio-level observer broadcast to clients.
- TURN server (coturn) configured for NAT traversal.

**Testing**:
- `Integration (mocked mediasoup router): produce/consume handshake completes`.
- `E2E (Playwright fake media devices): two participants establish audio/video; third joins as observer (recv-only)`.
- `E2E: screen-share track appears for all participants`.

#### 6.2 — Recording, transcode & playback

**What**: Server-side recording of media + a synchronised code/whiteboard timeline, processed off-request and replayable.

**Design**:
- mediasoup `PlainTransport` pipes tracks to an ffmpeg/GStreamer recorder producing an MP4 to S3; URL stored in `session_recordings` (`recording_type='video'`).
- Code playback: the realtime gateway logs timestamped Yjs updates during the session to S3 (`recording_type='code_playback'`), replayed client-side by applying updates against the wall clock; whiteboard snapshots provide the diagram timeline.
- On `session end`, a BullMQ `recording.finalize` job transcodes, uploads, computes `duration_secs`/`file_size_bytes`, and marks the recording ready.
- Retention: per-org configurable (30–365 days) via S3 lifecycle policy.

**Testing**:
- `Integration: finalize job → MP4 in S3, session_recordings row with duration`.
- `Integration: code-playback log replays to the final document state`.
- `Integration (mocked S3): finalize failure → job retried, marked failed after max attempts (DLQ)`.
- `E2E: open completed session → video + code playback scrub in sync`.

---

## Phase 7: AI Evaluation & Co-Interviewer

### Purpose
Deliver the headline AI-native differentiators: automated post-interview reports with evidence citations, automated correctness/complexity scoring, and a real-time AI co-interviewer. After this phase every completed interview yields an AI report, and interviewers receive live follow-up suggestions. Depends on Phases 4, 5, 6 (needs code, execution results, transcript/recording, rubrics).

### Tasks

#### 7.1 — LLM gateway

**What**: A provider-agnostic `LlmClient` with retries, timeouts, token accounting, and prompt-caching.

**Design**:
- Interface: `complete({ system, messages, schema?, model }): Promise<{ text|json, usage }>`. `schema` enables structured JSON output validated against a shared Zod/JSON schema.
- Adapters for Anthropic and OpenAI; self-host (Ollama) optional. Calls run inside BullMQ jobs (off-request) with exponential backoff.
- Prompt caching for stable system prompts/rubrics to cut cost/latency. All AI inputs/outputs logged (model version) to `ai_evaluations` for auditability and EEOC defensibility.

**Testing**:
- `Unit (mocked provider): structured call returns JSON validated against schema; malformed → retried then error`.
- `Unit: provider timeout → backoff retry; persistent failure → job fails with reason`.
- `Unit: usage tokens recorded per call`.

#### 7.2 — Automated correctness & complexity scoring

**What**: Run candidate code against hidden test cases and produce a correctness score plus an LLM complexity estimate.

**Design**:
- On session end (or on demand), for the attached problem: execute the final source against all `problem_test_cases` via Phase 4, computing weighted pass rate → `correctness_score`.
- LLM estimates time/space complexity (Big-O) and flags edge-case gaps, emitting structured JSON `{ timeComplexity, spaceComplexity, edgeCasesCovered[], edgeCasesMissed[], rationale }` → stored in `ai_evaluations` (`evaluation_type='complexity_analysis'`).

**Testing**:
- `Integration: solution passing 8/10 weighted cases → correctness_score reflects weights`.
- `Integration (mocked LLM): complexity JSON validates and persists`.
- `Integration: missing test cases → correctness scoring skipped gracefully with note`.

#### 7.3 — Post-interview AI report

**What**: Generate a structured, evidence-cited report scoring technical and soft-skill dimensions.

**Design**:
- Inputs: final code, execution/correctness results, transcript (from recording/awareness log), problem statement, rubric.
- Prompt template (system): *"You are an impartial technical-hiring evaluator. Using only the provided evidence, score each dimension 1–5 and cite specific moments (timestamps or code line ranges). Do not infer demographic attributes. Output JSON matching the AiReport schema."* Dimensions: code quality, problem-solving approach, communication clarity, edge-case coverage.
- Output validated against `AiReport` schema; persisted to `ai_evaluations` (`evaluation_type='soft_skills'`/`'code_quality'`). Surfaced in the reports UI alongside human evaluations (never auto-deciding — EEOC: AI advises, humans decide).

**Testing**:
- `Integration (mocked LLM): report validates against AiReport JSON Schema and persists`.
- `Unit: prompt assembly includes rubric + redacts candidate name when session anonymized`.
- `Integration: session end enqueues exactly one report job (idempotent on retry)`.
- `E2E: completed session shows AI report with citations in reports UI`.

#### 7.4 — Real-time AI co-interviewer

**What**: A live assistant that suggests follow-up questions based on the candidate's evolving code.

**Design**:
- Participant of type `ai_copilot`. On significant editor pauses or run events, a debounced job sends the current code + problem to the LLM requesting 1–3 targeted follow-up questions (`{ question, rationale, probes }[]`).
- Suggestions stream only to interviewer/observer clients via `session:<id>:copilot` — never to candidates. Interviewers can dismiss/insert into notes.
- Strict rate limiting + cost cap per session.

**Testing**:
- `Integration (mocked LLM): pause event → suggestions broadcast to interviewer channel only`.
- `Integration: candidate client never receives copilot messages`.
- `Unit: debounce collapses rapid edits into one suggestion request`.

---

## Phase 8: ATS Integration

### Purpose
Connect the platform to enterprise hiring workflows — the adoption gate for enterprise buyers. Implement inbound webhooks (trigger interviews from ATS stages) and outbound result push, starting with Greenhouse and Lever. Depends on Phases 2 and 5 (needs sessions and evaluations to sync). Can run in parallel with Phase 7.

### Tasks

#### 8.1 — Connector abstraction & credential vault

**What**: A pluggable `AtsConnector` interface with encrypted credential storage and OAuth/API-key support.

**Design**:
- Interface: `fetchCandidate`, `pushAssessmentStatus`, `pushInterviewResult`, `verifyWebhook(sig, body)`. Implementations: `GreenhouseConnector` (Harvest API key + Assessment API + webhooks), `LeverConnector` (OAuth 2.0, HTTPS-only webhooks with signing token per `standards.md`).
- Credentials encrypted at rest (`credentials_enc BYTEA`, AES-GCM with a KMS/env key); `ats_integrations.config JSONB` for non-secret settings.
- Outbound results map to HR Open Standards (HR-JSON) shapes where applicable to reduce integration friction.

**Testing**:
- `Unit: credentials encrypted at rest; decrypt round-trips; key rotation re-encrypts`.
- `Unit (mocked HTTP): Greenhouse pushAssessmentStatus posts correct payload to Assessment API`.
- `Unit: connector registry resolves by provider; unknown provider → error`.

#### 8.2 — Inbound webhooks → session creation

**What**: Receive ATS webhook events (candidate reaches interview stage) and create candidate/application/session records.

**Design**:
- `POST /ats/:provider/webhook` verifies the signature (provider-specific) before processing; invalid → 401 and nothing enqueued.
- Valid events enqueue an `ats.inbound` BullMQ job that upserts candidate+application (idempotent on `ats_external_id`) and optionally auto-creates a scheduled session; logs to `ats_sync_logs`.

**Testing**:
- `Integration: valid signature → 200, job enqueued, candidate upserted`.
- `Integration: invalid signature → 401, nothing enqueued`.
- `Integration: duplicate webhook (same external_id) → single candidate (idempotent)`.

#### 8.3 — Outbound result sync

**What**: Push interview completion and results back to the ATS, with retries and reconciliation logging.

**Design**:
- On session completion + evaluation submission, enqueue `ats.outbound` to call `pushInterviewResult`/`pushAssessmentStatus`. Each attempt logged to `ats_sync_logs` (`status` pending→success/failed/retrying); failures retried with backoff and surfaced in an admin sync view.

**Testing**:
- `Integration (mocked ATS): completed+evaluated session → outbound job posts result, sync log success`.
- `Integration: ATS returns 500 → retried, status=retrying then failed after max attempts`.
- `Integration: outbound payload validates against HR-JSON schema`.

---

## Phase 9: Async Take-Home, Proctoring & Anonymisation Review

### Purpose
Round out the assessment surface with asynchronous take-home interviews, AI proctoring signals, and a candidate-anonymised review mode — all "should-have" features that broaden the product beyond live sessions. Depends on Phases 4–7.

### Tasks

#### 9.1 — Asynchronous take-home mode

**What**: Candidate completes a timed problem on their own schedule; interviewers review the recorded attempt later.

**Design**:
- `sessionType='take_home'` with no live interviewers present. Candidate opens a join link, the timer (`durationLimit`) starts on first edit, code-playback logging is on, and on submit/timeout the session auto-transitions to `completed`, enqueueing correctness + AI report jobs (Phase 7).
- Reviewers later open the playback + AI report and score via Phase 5 rubrics.

**Testing**:
- `Integration: timer expiry → session completed, jobs enqueued`.
- `Integration: submit before expiry → completed; further edits rejected`.
- `E2E: candidate completes take-home → reviewer sees playback + report`.

#### 9.2 — AI proctoring signals

**What**: Capture integrity signals (tab-switch, paste, face/voice presence, device change) as proctor events.

**Design**:
- Client emits browser-detectable events (visibility change, paste, devicechange) to `POST /sessions/:id/proctor-events` → `proctor_events` with `event_type`, `severity`, `details JSONB`.
- Optional in-browser face/voice presence detection (consent-gated, GDPR) emits `face_detection`/`voice_detection` warnings; results are advisory signals on the report, never auto-rejections (EEOC).
- A proctoring timeline appears in the report UI.

**Testing**:
- `Integration: tab-switch event → row with severity=warning`.
- `Integration: events scoped to session; cross-session access → 403`.
- `Unit: proctoring consent flag required before face/voice events accepted`.

#### 9.3 — Candidate anonymisation review mode

**What**: A review mode that hides candidate identity to reduce bias during scoring.

**Design**:
- When `session.isAnonymized` or org-level "anonymous review" is on, all review surfaces (reports, playback, evaluation) display `anonymizedId` and strip name/email/photo; de-anonymisation requires recruiter/admin role and is audit-logged.

**Testing**:
- `Integration: anonymized review → responses contain anonymizedId, never name/email`.
- `Integration: de-anonymise action by admin → audit_logs entry; by interviewer → 403`.

---

## Phase 10: Compliance, Security Hardening & Observability

### Purpose
Make the platform enterprise-ready: SOC 2 / ISO 27001 control posture, GDPR data-subject workflows, WCAG 2.2 AA verification, RLS multi-tenant isolation, and full observability. Depends on all prior phases (hardens what exists).

### Tasks

#### 10.1 — Row-level security & tenant isolation

**What**: Enforce `organizationId` isolation at the database via Postgres RLS.

**Design**:
- RLS policies on all tenant tables keyed on a per-request `app.current_org` GUC set from the JWT; the API sets it per transaction. Defence-in-depth atop application scoping.

**Testing**:
- `Integration: query as org A with RLS on → org B rows invisible even on raw SQL`.
- `Integration: missing org GUC → zero rows (fail closed)`.

#### 10.2 — GDPR data-subject workflows & retention

**What**: Right-to-erasure and export for candidates, plus enforced retention.

**Design**:
- `POST /candidates/:id/erase` cascades deletion/anonymisation of candidate PII, code submissions, recordings (S3 delete), and proctor data, while retaining audit_logs (7-yr SOC 2) in anonymised form. `GET /candidates/:id/export` returns a portable JSON bundle. S3 lifecycle + a scheduled job enforce per-org retention windows.

**Testing**:
- `Integration: erase → PII removed, recordings deleted, audit_logs retained but anonymised`.
- `Integration: export bundle validates against JSON Schema and includes all candidate data`.
- `Integration: retention job deletes recordings past window`.

#### 10.3 — Security hardening (OWASP ASVS) & audit completeness

**What**: Apply OWASP ASVS L2 controls and verify the audit trail.

**Design**:
- Rate limiting on auth + execution; input validation everywhere (Zod); CSP/security headers; secrets via env/KMS; dependency and container scanning in CI; the Judge0 sandbox isolation reviewed against ASVS L2 (per `standards.md`). Every privileged action present in `audit_logs`.

**Testing**:
- `Integration: brute-force login → throttled (429) after threshold`.
- `Integration: each privileged endpoint writes an audit row (table-driven test)`.
- `CI: SAST + dependency scan pass with no high-severity findings`.

#### 10.4 — Accessibility & observability

**What**: Verify WCAG 2.2 AA and ship tracing/metrics/dashboards.

**Design**:
- Automated axe-core checks in Playwright across key screens; manual keyboard/screen-reader pass on workspace. OpenTelemetry traces (HTTP, ws, AI, Judge0) → Prometheus/Grafana; pino structured logs with request/session correlation IDs; SLO dashboards for real-time latency and AI job duration (SOC 2 availability evidence).

**Testing**:
- `E2E: axe-core scan of dashboard, workspace, reports → no WCAG 2.2 AA violations`.
- `Integration: a request produces a trace spanning API→DB→Judge0 with correlation id`.
- `Integration: AI job emits duration + token metrics`.

---

## Phase 11 (Optional): Advanced AI Intelligence

### Purpose
Add the remaining backlog AI differentiators that benefit from richer modelling: bias detection, adaptive difficulty, isomorphic problem variants, natural-language problem search, and post-hire impact analytics. This phase optionally introduces the Neo4j skills graph from Data Model Suggestion 4 to power graph-traversal features. Depends on Phases 5 and 7.

### Tasks

#### 11.1 — Bias detection layer

**What**: Analyse interviewer behaviour for structured-interview violations (no incumbent does this).

**Design**:
- Per session/panel, compute metrics from the transcript/awareness log: interviewer-vs-candidate talk-time ratio, interruption count, question count/consistency across candidates for the same role. An LLM/stats layer flags outliers (e.g. talk-time skew, inconsistent difficulty) as advisory `ai_evaluations` (`evaluation_type='bias_detection'`). Aggregated, never used to auto-decide (EEOC).

**Testing**:
- `Integration: transcript with 80% interviewer talk-time → talk-time-skew flag`.
- `Integration: consistent panels → no false-positive flags`.

#### 11.2 — Isomorphic problem variants

**What**: Generate difficulty-graded variants of a seed problem to prevent answer leakage.

**Design**:
- `POST /problems/:id/variants` → LLM produces N isomorphic variants (same skills, different surface/values) at requested difficulties, each with regenerated test cases; stored as new `problems` linked via `metadata.seedProblemId`. Human review/approve before activation.

**Testing**:
- `Integration (mocked LLM): generate 3 variants → 3 problems with seed link + test cases`.
- `Integration: variants require approval before becoming selectable`.

#### 11.3 — Adaptive difficulty & NL problem search

**What**: Real-time difficulty adjustment and natural-language problem discovery.

**Design**:
- Adaptive: after early signals (correctness/time on first sub-task) the co-interviewer suggests escalating/de-escalating to a linked variant; interviewer confirms.
- NL search: `GET /problems/search/nl?q="mid-level Python async I/O"` embeds the query and ranks problems by embedding similarity (pgvector) — optionally backed by the Neo4j skills graph (Suggestion 4) for skill-relationship traversal.

**Testing**:
- `Integration: NL query maps to relevant tagged problems (top-k contains expected)`.
- `Integration: adaptive suggestion fires after first sub-task signal`.

#### 11.4 — Post-hire impact analytics

**What**: Correlate interview signals with on-the-job outcomes to surface predictive questions/behaviours.

**Design**:
- Ingest outcome labels (hired/performance) via ATS or manual entry; an analytics job correlates problem/difficulty/interviewer-behaviour with outcomes, exposed in an analytics dashboard. Heavy aggregations run against a columnar store (TimescaleDB/ClickHouse) per Suggestion 1's scaling path.

**Testing**:
- `Integration: seeded outcomes → correlation report ranks predictive problems`.
- `Integration: analytics queries run against the columnar store, not the OLTP primary`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema & Auth        ─── required by everything
    │
Phase 2: Session Lifecycle & Domain Core  ─── requires P1
    │
Phase 3: Real-Time Collaborative Workspace ── requires P1, P2  (core value)
    │
Phase 4: Code Execution                    ─── requires P3
    │
    ├── Phase 5: Library / Whiteboard / Evaluation ── requires P2,P3 (5.1/5.2/5.3 parallel)
    └── Phase 6: Video & Recording                 ── requires P2,P3 (parallel with P5)
             │
    ┌────────┴───────────────────────────────────┐
Phase 7: AI Evaluation & Co-Interviewer   ─── requires P4,P5,P6
    │
    ├── Phase 8: ATS Integration          ─── requires P2,P5 (parallel with P7)
    │
Phase 9: Take-Home / Proctoring / Anon Review ── requires P4–P7
    │
Phase 10: Compliance, Security & Observability ─ requires all prior (hardening)
    │
Phase 11 (optional): Advanced AI Intelligence ── requires P5,P7
```

**Parallelism opportunities:**
- Phase 5 (tasks 5.1, 5.2, 5.3) and Phase 6 can be developed concurrently once Phase 3 lands.
- Phase 8 (ATS) can proceed in parallel with Phase 7 (AI) after Phase 5.
- Within Phase 1, schema (1.2) and API skeleton (1.3) can progress alongside the shared package (1.1) once enums exist.

---

## Definition of Done (per phase)

Every phase is complete only when:

1. All tasks implemented and merged.
2. All unit and mocked-integration tests pass; real-dependency integration tests (Testcontainers/Judge0/mediasoup) pass in CI where applicable.
3. `eslint` and `prettier --check` pass.
4. `tsc --noEmit` passes in strict mode across all touched packages.
5. `docker compose up` builds and the affected services start healthy (`/readyz` green).
6. The phase's feature works end-to-end (Playwright E2E where user-facing).
7. New config/env vars added to `.env.example` and documented.
8. New/changed API endpoints appear in the generated `openapi.json` (OAS 3.1) and validate.
9. New webhook/report payloads have JSON Schema (Draft 2020-12) artefacts that validate against fixtures.
10. Prisma migration created, reviewed, and `migrate deploy` succeeds on a clean database.
11. Privileged actions write `audit_logs` entries (from Phase 2 onward).
12. No new high-severity findings from dependency/container scans (from Phase 10 onward, enforced in CI).
```
