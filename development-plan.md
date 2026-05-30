# Student Advising Platform — Phased Development Plan

> Project: 427-student-advising-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the canonical model — it is the best fit for a multi-tenant, AI-native, single-engine open-source platform serving institutions with widely varying catalog structures. The graph-traversal patterns from Suggestion 4 are incorporated as an **optional optimisation in Phase 11** (a derived read model), not as a hard dependency. The audit-trail strengths of Suggestion 2 are captured via an append-only `audit_log` table (Phase 1) rather than full event sourcing, keeping operational complexity appropriate for self-hosting institutional IT teams.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | The product is API + dashboard + student portal heavy with modest ML. A single language across backend and the two React frontends reduces team surface area for the open-source community. AI orchestration (Anthropic/OpenAI SDKs, MCP) is first-class in TS. |
| Runtime / package manager | Node.js 22 LTS + pnpm workspaces | pnpm workspaces give a clean monorepo (`api`, `web-advisor`, `web-student`, `packages/*`) with strict, fast, disk-efficient installs. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput, first-class JSON Schema validation on every route, and automatic **OpenAPI 3.1** generation (required by `standards.md`). Schema-first routes double as request validation and published contract. |
| Database | PostgreSQL 16 | Single engine serving relational + JSONB (Suggestion 3). RLS for FERPA tenant isolation, GIN indexes for JSONB queryability, declarative partitioning for large tables, `pgcrypto` for at-rest field encryption. |
| ORM / query builder | Drizzle ORM | Type-safe schema-as-code, first-class JSONB column typing, migration generation, and raw-SQL escape hatch for recursive degree-audit CTEs. Recommended by Suggestion 3. |
| Migrations | drizzle-kit | Version-controlled, generated from the Drizzle schema; CI-checked for drift. |
| Cache / queue | Redis 7 + BullMQ | SIS/LMS sync jobs, risk-scoring batches, AI briefing generation, and bulk outreach are async workloads. BullMQ gives retries, scheduling (repeatable jobs for nightly syncs), and dead-letter handling. Redis also caches degree-audit results. |
| Auth (machine) | OAuth 2.0 client-credentials (RFC 6749) | Server-to-server SIS/LMS ingestion. |
| Auth (human SSO) | SAML 2.0 SP + OIDC RP (`@node-saml/node-saml`, `openid-client`) | Higher-ed identity reality: InCommon/Shibboleth (SAML) and Azure AD/Okta (OIDC). Both required per `standards.md`. |
| Session / RBAC | Signed httpOnly cookies + role claims | Roles: `student`, `advisor`, `faculty`, `admin`, `system`. Enforced in middleware and mirrored to PostgreSQL RLS session variables. |
| Degree-audit engine | Custom rule evaluator over the JSONB requirement tree | No off-the-shelf open-source engine matches institutional catalog complexity; the evaluator is the product's core IP. Pure, deterministic, fully unit-testable. |
| ML / risk scoring | Python sidecar service (FastAPI + scikit-learn / XGBoost) | Risk modelling and SHAP explainability belong in Python. Exposed to the TS backend over an internal HTTP contract; deployed as a separate container. Institution-specific models per `research.md`. |
| LLM integration | Vercel AI SDK + Anthropic provider (configurable) | Pre-meeting briefings, AI rule authoring, transfer-equivalency suggestions, case-note drafts. Provider-pluggable so self-hosters can use any model. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Exposes degree-audit state, student risk profiles, and advising history as LLM tools (backlog feature, `standards.md` MCP note). |
| Frontends | React 19 + Vite + TanStack Router/Query + shadcn/ui + Tailwind | Two SPAs (advisor dashboard, student portal). shadcn/Tailwind accelerates a **WCAG 2.2 AA**-conformant UI. TanStack Query handles server cache + optimistic updates. |
| Validation (shared) | Zod + JSON Schema | Zod schemas in a shared `packages/contracts` package; converted to JSON Schema for Fastify routes and to OpenAPI. Single source of truth for types across API and frontends. |
| Accessibility testing | axe-core + Playwright | Automated WCAG 2.2 AA checks in CI against both portals (Section 508 / EAA). |
| Containerisation | Docker + docker-compose | Self-hosted deployment is a core differentiator. Compose stack: api, ml-service, web-advisor, web-student, postgres, redis, mailpit (dev). |
| Testing | Vitest (unit/integration TS), Playwright (E2E), pytest (ML service), Testcontainers (real Postgres/Redis) | Fast unit runner, real-dependency integration via Testcontainers, browser E2E. |
| Code quality | ESLint + Prettier + `tsc --noEmit` (TS); ruff + mypy (Python) | Enforced in CI. |
| CI | GitHub Actions | Lint, typecheck, unit, integration (Testcontainers), accessibility, OpenAPI-diff, Docker build. |
| Observability | pino (structured logs) + OpenTelemetry + Prometheus | FERPA/ASVS audit logging, plus operational metrics. |
| Licence | AGPL-3.0 | Matches the only open-source incumbent (FlightPath); strongest copyleft protection for an open advising commons. |

### Project Structure

```
student-advising-platform/
├── pnpm-workspace.yaml
├── package.json
├── turbo.json                        # task orchestration across workspaces
├── docker-compose.yml
├── docker-compose.prod.yml
├── .github/workflows/ci.yml
├── apps/
│   ├── api/                          # Fastify backend
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── server.ts             # app bootstrap, plugin registration
│   │   │   ├── config.ts             # env parsing (Zod)
│   │   │   ├── db/
│   │   │   │   ├── schema/            # Drizzle table definitions (one file per domain)
│   │   │   │   ├── client.ts
│   │   │   │   ├── rls.ts             # session var helpers for RLS
│   │   │   │   └── migrations/        # drizzle-kit output
│   │   │   ├── plugins/              # auth, rls, swagger, rate-limit, audit
│   │   │   ├── modules/
│   │   │   │   ├── students/
│   │   │   │   ├── courses/
│   │   │   │   ├── programs/
│   │   │   │   ├── catalog/           # degree requirement sets
│   │   │   │   ├── audit-engine/      # degree audit evaluator (pure)
│   │   │   │   ├── planning/          # academic plans, prereq validation
│   │   │   │   ├── transfer/          # transfer credit articulation
│   │   │   │   ├── alerts/            # early alerts + rule triggers
│   │   │   │   ├── referrals/
│   │   │   │   ├── appointments/
│   │   │   │   ├── case-notes/
│   │   │   │   ├── caseload/          # advisor dashboards
│   │   │   │   ├── risk/              # client to ML sidecar
│   │   │   │   ├── ai/                # briefings, rule authoring, suggestions
│   │   │   │   ├── analytics/
│   │   │   │   └── auth/              # SAML, OIDC, sessions, RBAC
│   │   │   ├── integrations/
│   │   │   │   ├── sis/               # Banner, Colleague(Ethos), PeopleSoft(Edu-API), Workday(RAAS)
│   │   │   │   ├── lms/               # Canvas, Blackboard, Caliper
│   │   │   │   └── shared/            # connector interface, staging, mappers
│   │   │   ├── jobs/                  # BullMQ processors (sync, scoring, briefings, outreach)
│   │   │   └── mcp/                   # MCP server (backlog)
│   │   └── test/
│   ├── ml-service/                   # Python FastAPI risk scoring
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   ├── app/
│   │   │   ├── main.py
│   │   │   ├── features.py           # feature engineering from student data
│   │   │   ├── train.py              # institution-specific training CLI
│   │   │   ├── score.py
│   │   │   └── explain.py            # SHAP explainability
│   │   └── tests/
│   ├── web-advisor/                  # React advisor dashboard
│   └── web-student/                  # React student portal
├── packages/
│   ├── contracts/                    # Zod schemas + generated OpenAPI types (shared)
│   ├── audit-engine/                 # framework-free degree audit library (also imported by api)
│   ├── connector-sdk/                # connector interface + helpers, publishable
│   └── ui/                           # shared shadcn components, WCAG-checked
└── docs/
    ├── data-model.md
    ├── connectors.md
    └── openapi.json                  # published spec artifact
```

---

## Phase 1: Foundation — Monorepo, Database, Multi-Tenancy, Auth Scaffold

### Purpose
Establish the monorepo, the PostgreSQL schema for the stable relational core, multi-tenant isolation via RLS, the append-only audit log that underpins FERPA compliance, and the Fastify server with health checks and OpenAPI generation. After this phase the system boots, connects to Postgres/Redis, enforces tenant isolation, and serves a documented (empty) API surface.

### Tasks

#### 1.1 — Monorepo and tooling bootstrap

**What**: Create the pnpm/turbo workspace with `apps/api`, `packages/contracts`, lint/format/typecheck config, and a docker-compose dev stack.

**Design**:
- `pnpm-workspace.yaml` declares `apps/*` and `packages/*`.
- `turbo.json` tasks: `build`, `lint`, `typecheck`, `test`, `test:integration`.
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `mailpit`, `api` (dev target with hot reload).
- `apps/api/src/config.ts` parses env with Zod:
```ts
export const Config = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().default(8080),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  ENCRYPTION_KEY: z.string().length(64), // hex, 32 bytes for pgcrypto field encryption
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});
export type Config = z.infer<typeof Config>;
```

**Testing**:
- `Unit: config parse with valid env → typed Config object`
- `Unit: config parse missing DATABASE_URL → ZodError naming DATABASE_URL`
- `Integration: docker-compose up → postgres and redis reachable from api container (healthcheck passes)`

#### 1.2 — Core relational schema (Drizzle)

**What**: Define `institutions`, `users`, `students`, `advisors`, `departments`, `caseload_assignments`, `academic_terms`, `courses`, `course_sections`, `enrollments` as Drizzle tables, with `institution_id` on every tenant-scoped table.

**Design**: Follows Suggestion 3's normalised core. Key tables (Drizzle pseudo-DDL → generated SQL):
```ts
export const institutions = pgTable("institutions", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  ipedsId: varchar("ipeds_id", { length: 20 }).unique(),
  config: jsonb("config").$type<InstitutionConfig>().notNull().default({}),
  timezone: varchar("timezone", { length: 50 }).notNull().default("America/New_York"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const students = pgTable("students", {
  id: uuid("id").primaryKey().defaultRandom(),
  institutionId: uuid("institution_id").notNull().references(() => institutions.id),
  sisStudentId: varchar("sis_student_id", { length: 50 }).notNull(),
  firstName: varchar("first_name", { length: 100 }).notNull(),
  lastName: varchar("last_name", { length: 100 }).notNull(),
  preferredName: varchar("preferred_name", { length: 100 }),
  email: varchar("email", { length: 255 }),
  phone: varchar("phone", { length: 30 }),
  dateOfBirth: date("date_of_birth"),
  gender: varchar("gender", { length: 20 }),
  ethnicity: varchar("ethnicity", { length: 50 }),
  firstGeneration: boolean("first_generation").default(false),
  pellRecipient: boolean("pell_recipient").default(false),
  customAttributes: jsonb("custom_attributes").$type<Record<string, unknown>>().notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  uniqSis: unique().on(t.institutionId, t.sisStudentId),
  customGin: index("idx_students_custom").using("gin", t.customAttributes),
}));
```
`enrollments`, `courses`, `course_sections`, `academic_terms` per Suggestion 3 (courses carry `catalog_data` JSONB; enrollments carry `sis_sync` JSONB). `users` holds auth identity (email, role, `institution_id`, `external_idp_subject`).

**Testing**:
- `Integration (Testcontainers): run migrations on fresh Postgres → all tables + indexes exist`
- `Integration: insert student with duplicate (institution_id, sis_student_id) → unique violation`
- `Unit: Drizzle schema → generated SQL snapshot matches committed migration (no drift)`

#### 1.3 — Multi-tenant Row-Level Security

**What**: Apply RLS policies on all tenant-scoped tables keyed off a session variable `app.institution_id`, and a request plugin that sets it per request.

**Design**:
- Migration enables RLS and adds policy per table:
```sql
ALTER TABLE students ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON students
  USING (institution_id = current_setting('app.institution_id')::uuid);
```
- `db/rls.ts` exposes `withTenant(institutionId, fn)` that runs `SET LOCAL app.institution_id = $1` inside a transaction before delegating to `fn(tx)`.
- Fastify `rls` plugin reads `institution_id` from the authenticated session and wraps the handler's DB access.

**Testing**:
- `Integration: query students under tenant A → returns only tenant A rows`
- `Integration: query students with no app.institution_id set → 0 rows (fail-closed)`
- `Integration: attempt cross-tenant update by id → 0 rows affected`

#### 1.4 — Append-only audit log

**What**: An `audit_log` table and helper recording every write (who, what entity, before/after, when, IP), satisfying FERPA accountability and OWASP ASVS logging.

**Design**:
```sql
CREATE TABLE audit_log (
  id BIGSERIAL PRIMARY KEY,
  institution_id UUID NOT NULL,
  actor_user_id UUID,
  actor_role VARCHAR(20),
  action VARCHAR(50) NOT NULL,        -- create, update, delete, view_pii, export
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID,
  diff JSONB,                          -- {before, after} for mutations
  ip INET,
  occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);
```
- Monthly partitions created by a maintenance job. No UPDATE/DELETE grants (append-only).
- `audit()` helper called from a Fastify `onResponse` hook for mutating routes and explicitly for PII reads/exports.

**Testing**:
- `Unit: audit() with create action → row with diff.after populated, diff.before null`
- `Integration: update student name → audit_log row with diff {before, after} and actor_user_id`
- `Integration: revoke UPDATE on audit_log; attempt UPDATE → permission denied`

#### 1.5 — Fastify server, health, OpenAPI scaffold

**What**: Boot Fastify with pino logging, `@fastify/swagger` (OpenAPI 3.1) + Scalar UI, rate limiting, and `/healthz` / `/readyz`.

**Design**:
- `GET /healthz` → `{ status: "ok" }` (liveness).
- `GET /readyz` → checks Postgres + Redis → 200 or 503.
- Swagger served at `/docs`; spec emitted to `docs/openapi.json` via a build script.
- Every route registers a Zod-derived JSON Schema for body/query/response.

**Testing**:
- `Integration: GET /healthz → 200 {status:"ok"}`
- `Integration: stop redis → GET /readyz → 503`
- `Integration: build openapi → docs/openapi.json validates against OpenAPI 3.1 meta-schema`

---

## Phase 2: Identity, SSO, and RBAC

### Purpose
Real institutional authentication. After this phase, users sign in via SAML 2.0 or OIDC, sessions carry verified role + institution claims, and RBAC + RLS are wired together. This unblocks every authenticated feature.

### Tasks

#### 2.1 — Local + session foundation

**What**: Session management with signed httpOnly cookies and a `users` ↔ `students`/`advisors` linkage.

**Design**:
- `@fastify/secure-session`; session payload: `{ userId, institutionId, role, studentId?, advisorId? }`.
- `requireRole(...roles)` preHandler factory; sets `app.institution_id` + `app.user_role` for RLS.
- Roles enum: `student | advisor | faculty | admin | system`.

**Testing**:
- `Unit: requireRole('advisor') with student session → 403`
- `Integration: login sets cookie; subsequent request resolves session → 200`
- `Integration: tampered cookie signature → 401`

#### 2.2 — OIDC Relying Party

**What**: OIDC authorization-code login against institutional IdPs (Azure AD, Okta).

**Design**:
- `openid-client`; per-institution IdP config stored in `institutions.config.oidc` (issuer, clientId, clientSecret-ref, scopes).
- Routes: `GET /auth/oidc/:institutionId/login` → redirect; `GET /auth/oidc/callback` → validate ID token, map claims (`sub`, `email`, role claim) to a `users` row (JIT provisioning), establish session.
- Claim→role mapping configurable per institution (`config.oidc.roleClaim`, `config.oidc.roleMap`).

**Testing**:
- `Integration (mocked IdP): valid code → ID token validated → session created, user JIT-provisioned`
- `Integration (mocked IdP): expired/invalid ID token → 401, no session`
- `Unit: claim mapping with configured roleMap → correct internal role`

#### 2.3 — SAML 2.0 Service Provider

**What**: SAML SP login for Shibboleth/InCommon institutions.

**Design**:
- `@node-saml/node-saml`; SP metadata exposed at `/auth/saml/:institutionId/metadata`.
- `POST /auth/saml/:institutionId/acs` consumes the assertion, validates signature against the IdP cert in `institutions.config.saml`, maps attributes (`eduPersonPrincipalName`, `eduPersonAffiliation`) to user + role.

**Testing**:
- `Integration: signed assertion (fixture) → session created`
- `Integration: assertion with wrong signature → 401`
- `Integration: GET metadata → valid SAML SP XML`

---

## Phase 3: Academic Catalog & Degree Requirement Authoring

### Purpose
Model courses, programs, terms, prerequisites, and the JSONB degree-requirement tree — the data the audit engine consumes. After this phase, an admin can author and publish versioned, catalog-year-scoped requirement sets validated against a formal schema. This is the foundation of the core value proposition.

### Tasks

#### 3.1 — Courses, programs, terms, prerequisites CRUD

**What**: CRUD endpoints + schema for `courses`, `programs`, `departments`, `academic_terms`, `course_sections`, `course_prerequisites`.

**Design**:
- `course_prerequisites` per Suggestion 1 (groups for OR logic, `min_grade`, `is_corequisite`).
- Endpoints (JSON:API-style pagination/filtering): `GET/POST /courses`, `GET/PATCH/DELETE /courses/:id`, analogous for programs/terms/sections.
- Response/request schemas live in `packages/contracts`.

**Testing**:
- `Integration: create course → 201, appears in GET list filtered by subject_code`
- `Integration: create prerequisite referencing nonexistent course → 422`
- `Unit: pagination cursor encode/decode round-trips`

#### 3.2 — Degree requirement set schema & versioning

**What**: `degree_requirement_sets` table (JSONB `requirements` tree) with draft→published→archived lifecycle and immutable published versions.

**Design**: Table per Suggestion 3. The requirement tree has a formal Zod schema in `packages/contracts`:
```ts
type ReqNode =
  | { id: string; name: string; type: "group"; logic: "AND" | "OR";
      minCredits?: number; minGpa?: number; children: ReqNode[] }
  | { id: string; name: string; type: "course_list"; logic: "ALL" | "CHOOSE";
      minCourses?: number; minCredits?: number;
      courses: { courseId: string; minGrade?: string }[] }
  | { id: string; name: string; type: "attribute_filter"; attribute: string;
      minCourses?: number; minCredits?: number }
  | { id: string; name: string; type: "credit_bucket"; minCredits: number;
      exclusions?: string[] };
type RequirementSet = {
  name: string; totalCreditsRequired: number; minimumGpa?: number;
  residencyCredits?: number; schemaVersion: "1.0"; children: ReqNode[];
};
```
- `POST /programs/:id/requirement-sets` (draft), `POST .../:setId/publish` (validates referenced `courseId`s exist — application-level integrity per Suggestion 3 cons; freezes the version), `POST .../:setId/archive`.
- Publishing a new version increments `version`; old published rows are never mutated.

**Testing**:
- `Unit: validate well-formed tree → ok; tree with unknown node type → ZodError`
- `Integration: publish set referencing missing courseId → 422 listing dangling refs`
- `Integration: edit a published set → 409 (immutable); must create new draft version`

#### 3.3 — AI-assisted rule authoring (catalog PDF → draft tree)

**What**: Upload a catalog PDF/section; LLM produces a draft `RequirementSet` for human review (backlog feature, surfaced early to validate the AI pipeline).

**Design**:
- `POST /programs/:id/requirement-sets/ai-draft` (multipart): extract text (pdf-parse), chunk, prompt the LLM to emit JSON conforming to the `RequirementSet` schema, validate with Zod, store as a **draft** flagged `aiGenerated: true`.
- System prompt template (stored in `modules/ai/prompts/rule-authoring.ts`):
  > "You convert university catalog text into a structured degree-requirement tree. Output ONLY JSON matching this schema: {schema}. Map each requirement heading to a group node; map explicit course lists to course_list nodes with logic ALL when 'all of' and CHOOSE when 'one of'/'choose N'. Never invent course IDs; emit subject+number and leave courseId null for human linkage. {catalog_text}"
- Course linkage is a separate human step (match subject+number → courseId).

**Testing**:
- `Integration (mocked LLM): fixture catalog text → valid draft RequirementSet stored, aiGenerated=true`
- `Integration (mocked LLM): LLM returns invalid JSON → 422, nothing persisted, error logged`
- `Unit: course-number linkage maps 'CS 201' → existing course id; unmatched → flagged for review`

---

## Phase 4: Degree Audit Engine (Core Value Proposition)

### Purpose
The heart of the product: a deterministic engine that evaluates a student's completed/in-progress courses + transfer credits against a published requirement tree and produces a structured, explainable audit. After this phase the platform delivers its primary value. Built as a framework-free library in `packages/audit-engine` so it is exhaustively unit-testable and reusable by the API, planning, and AI modules.

### Tasks

#### 4.1 — Audit evaluator core

**What**: Pure function `evaluateAudit(input) → AuditResult` over the requirement tree.

**Design**:
```ts
interface CompletedCourse { courseId: string; subject: string; number: string;
  credits: number; grade: string; gradePoints: number; inProgress: boolean;
  attributes: string[]; isTransfer: boolean; }
interface AuditInput {
  requirementSet: RequirementSet;
  completed: CompletedCourse[];
  overrides: Substitution[];          // advisor exceptions (Phase 8 feeds these)
  cumulativeGpa: number;
}
type NodeStatus = "satisfied" | "in_progress" | "incomplete";
interface AuditNodeResult {
  id: string; name: string; type: string; status: NodeStatus;
  creditsApplied: number; creditsRequired?: number;
  coursesApplied: { courseId: string; status: "complete" | "in_progress" }[];
  unmet?: { neededCredits?: number; neededCourses?: number };
  children?: AuditNodeResult[];
}
interface AuditResult {
  overallStatus: NodeStatus; totalCreditsEarned: number;
  totalCreditsRequired: number; gpaSatisfied: boolean;
  tree: AuditNodeResult; usedCourseIds: string[]; // for double-count prevention
}
```
- **Allocation rules**: post-order traversal; a course may satisfy at most one `course_list`/`credit_bucket` leaf unless the node is explicitly `shareable` (greedy best-fit: apply to the most-constrained unmet leaf first). `attribute_filter` matches on `course.attributes`. `min_grade` enforced via `gradePoints`. `group` AND requires all children satisfied; OR requires one. GPA checks compare `cumulativeGpa`/sub-GPA to `minGpa`. In-progress courses yield `in_progress` not `satisfied`.

**Testing**:
- `Unit: single course_list ALL, all completed with adequate grades → satisfied`
- `Unit: course_list CHOOSE minCourses=1, none taken → incomplete, unmet.neededCourses=1`
- `Unit: in-progress course on a leaf → leaf in_progress, overall in_progress`
- `Unit: grade below min_grade → course not applied, leaf incomplete`
- `Unit: double-count prevention — one course matches two leaves → applied to one only`
- `Unit: GPA 1.9 vs minGpa 2.0 → gpaSatisfied false`
- `Fixture: full B.S. CS tree + transcript fixture → snapshot of AuditResult`

#### 4.2 — Audit API + caching

**What**: `GET /students/:id/audit?programId=&catalogYear=` assembling inputs from DB and returning `AuditResult`, with Redis caching keyed on inputs.

**Design**:
- Loader gathers enrollments (+ in-progress), transfer credits (Phase 7), the published requirement set for the student's catalog year, and cumulative GPA.
- Cache key: `audit:{studentId}:{programId}:{requirementSetVersion}:{enrollmentDigest}`; invalidated on enrollment/grade/override change (event hooks).

**Testing**:
- `Integration: seed student+transcript+requirement set → audit returns expected statuses`
- `Integration: second call hits cache (no recompute) → identical result, faster path asserted via spy`
- `Integration: post a grade → cache invalidated → audit reflects new grade`

#### 4.3 — What-if analysis

**What**: `POST /students/:id/audit/what-if` projecting against a different program/catalog year/added-dropped courses without persisting.

**Design**:
- Body: `{ programId?, catalogYear?, addCompleted?: CompletedCourse[], removeCourseIds?: string[] }`. Clone inputs, apply hypotheticals, run `evaluateAudit`, return result tagged `whatIf:true`. No writes.

**Testing**:
- `Integration: what-if switch to different major → returns audit for that program, no DB mutation`
- `Integration: what-if drop a completed course → previously satisfied leaf becomes incomplete`
- `Unit: what-if never calls any write/repository method (mock asserts zero writes)`

---

## Phase 5: SIS & LMS Integration Layer

### Purpose
Populate the platform with real institutional data and address the market's biggest gap — data-sync lag. After this phase, the platform ingests enrollments, grades, holds (SIS) and engagement signals (LMS) through a pluggable connector framework with a staging→validate→merge pipeline, scheduled via BullMQ.

### Tasks

#### 5.1 — Connector SDK & staging pipeline

**What**: A connector interface (`packages/connector-sdk`), `integration_connections` + `sync_log` tables (Suggestion 3), and a staging-then-merge ingestion pipeline.

**Design**:
```ts
interface SisConnector {
  name: string;
  testConnection(cfg: ConnectionConfig): Promise<boolean>;
  fetchEnrollments(cfg, since?: Date): AsyncIterable<RawEnrollment>;
  fetchGrades(cfg, since?: Date): AsyncIterable<RawGrade>;
  fetchHolds(cfg, since?: Date): AsyncIterable<RawHold>;
}
```
- Records land in `staging_*` tables, are validated (Zod) and mapped via per-institution field mappings in `connection.config` (e.g., Banner `SPRIDEN_PIDM → sis_student_id`), then upserted into canonical tables inside a tenant transaction. `sync_log` records counts/errors.
- Encrypted credentials at rest via pgcrypto (`ENCRYPTION_KEY`).

**Testing**:
- `Unit: field mapper applies config mapping → canonical record`
- `Integration: staged batch with 1 invalid row → 1 errored in sync_log, valid rows merged`
- `Integration: re-run same batch → idempotent (no duplicate enrollments)`

#### 5.2 — SIS connectors: Edu-API, Banner, Colleague (Ethos), Workday (RAAS)

**What**: Four connector implementations behind the SDK.

**Design**:
- **Edu-API** (per `standards.md` first-mover): OAuth2 client-credentials, maps Edu-API `Person`/`Enrollment`/`AcademicSession` to canonical entities. Primary, standards-aligned path.
- **Banner**: Ellucian REST / Ethos with Ethos API key.
- **Colleague**: via Ethos.
- **Workday**: RAAS report endpoints + OAuth2.
- Each ships a connector manifest + sample `config` and a sandbox/mock mode.

**Testing**:
- `Integration (mocked Edu-API): paged Person+Enrollment feed → students + enrollments upserted`
- `Integration (mocked Banner/Ethos): grade feed → grades posted, audit cache invalidated`
- `Unit: Workday RAAS row → canonical enrollment mapping`

#### 5.3 — LMS connectors + Caliper engagement signals

**What**: Canvas and Blackboard connectors plus a Caliper event consumer for engagement signals feeding risk/alerts.

**Design**:
- Canvas REST (`canvasapi`-equivalent calls, OAuth2): pull per-course grades, submission/attendance proxies.
- Caliper endpoint `POST /caliper/events` accepting v1.2 envelopes → normalised `engagement_events` (login frequency, submissions, late submissions).
- Signals stored for the alert engine (Phase 6) and risk features (Phase 9).

**Testing**:
- `Integration (mocked Canvas): grade pull → lms grade signals stored`
- `Integration: post Caliper SessionEvent batch → engagement_events rows`
- `Unit: Caliper envelope → normalised engagement signal`

#### 5.4 — Scheduled sync jobs

**What**: BullMQ repeatable jobs per active connection on a configurable cadence (default 15 min incremental).

**Design**:
- Repeatable job per `integration_connections.config.sync_frequency`; incremental using `last_sync_at`; full nightly reconciliation. Failures retried with backoff, surfaced in `sync_log` + admin UI.

**Testing**:
- `Integration: enqueue repeatable job → processor runs connector, updates last_sync_at`
- `Integration: connector throws → job retried, sync_log status=error`

---

## Phase 6: Early Alerts & Rule Triggers

### Purpose
Proactive risk surfacing. After this phase, faculty submit alerts via simple forms, configurable rule triggers auto-generate alerts from SIS/LMS signals, and alerts route to advisor task queues.

### Tasks

#### 6.1 — Early alert model & lifecycle

**What**: `early_alerts` table (Suggestion 3 hybrid: normalised core + `trigger_details`/`resolution` JSONB) with lifecycle open→assigned→in_progress→resolved.

**Design**: Table per Suggestion 3. Endpoints: `POST /alerts` (faculty/system), `POST /alerts/:id/assign`, `POST /alerts/:id/resolve`, `GET /alerts?assignedTo=&status=&severity=`.

**Testing**:
- `Integration: faculty creates alert → routed to student's primary advisor queue`
- `Integration: resolve alert → status=resolved, resolution JSONB populated, audit logged`
- `Unit: invalid severity → 422`

#### 6.2 — Faculty alert submission form (API + portal route)

**What**: A simplified, non-technical faculty submission endpoint and minimal UI.

**Design**: `POST /alerts/faculty` with `{ studentId|sisStudentId, courseContext, concernType, note }`; resolves advisor assignment via `caseload_assignments`.

**Testing**:
- `Integration: submit by sisStudentId → resolves to canonical student, alert created`
- `Integration: unknown student → 404 with actionable message`

#### 6.3 — Configurable rule-trigger engine

**What**: Per-institution at-risk rules evaluating SIS/LMS signals to auto-create alerts (GPA threshold, missing assignments, attendance, financial hold, no-contact-in-N-days).

**Design**:
- Rules stored in `institutions.config.alertRules` as JSON: `{ id, name, when: {metric, op, value}, severity, alertType, cooldownDays }`.
- A nightly BullMQ job evaluates each active student against rules; dedup via cooldown; emits `system_generated` alerts with `trigger_details` capturing the firing condition.

**Testing**:
- `Unit: rule gpa < 2.0 against student gpa 1.85 → fires; against 2.5 → no fire`
- `Integration: nightly job over seeded cohort → expected alerts created, none within cooldown`
- `Unit: cooldown suppresses duplicate within window`

---

## Phase 7: Transfer Credit Articulation

### Purpose
Automate and assist transfer evaluation — a fragmented area across incumbents. After this phase, institutions store equivalency rules, evaluate incoming transfer credits (auto + advisor exception workflow), and the audit engine consumes accepted equivalencies. Includes AI-suggested mappings with human confirmation.

### Tasks

#### 7.1 — Equivalency rules & transfer credit records

**What**: `transfer_equivalency_rules` (JSONB mappings, Suggestion 3) and `transfer_credits` with evaluation workflow.

**Design**: Tables per Suggestion 3. `POST /transfer-credits` (ingest from SIS or manual), auto-match against equivalency rules → set `equivalent_course_id` + status; unmatched → `pending` for advisor. `POST /transfer-credits/:id/evaluate` (advisor decision).

**Testing**:
- `Integration: incoming course matching a rule → auto-evaluated, equivalent_course_id set`
- `Integration: no rule match → pending, appears in advisor exception queue`
- `Integration: evaluated transfer credit → included in audit (counts toward requirement)`

#### 7.2 — AI-suggested equivalency mapping

**What**: For unmatched transfer courses, LLM proposes an equivalency + confidence into `transfer_credits.ai_suggestion`; advisor confirms.

**Design**: `POST /transfer-credits/:id/ai-suggest` → prompt with source course title/description/credits + candidate home courses → JSON `{ suggestedCourseId, confidence, reasoning }`; stored, never auto-applied. Confirmation reuses 7.1's evaluate endpoint.

**Testing**:
- `Integration (mocked LLM): unmatched course → ai_suggestion stored, status still pending`
- `Integration: advisor confirms suggestion → equivalency persisted, audit updated`
- `Unit: low-confidence suggestion flagged for mandatory review`

#### 7.3 — Prospective transfer preview (public)

**What**: Unauthenticated `POST /transfer-preview` returning likely equivalencies for a prospective student's course list (Transferology-style).

**Design**: Rate-limited public endpoint; uses equivalency rules only (no PII); returns matched/unmatched summary. No persistence.

**Testing**:
- `Integration: public preview with known source courses → equivalencies returned`
- `Integration: preview is rate-limited → 429 after threshold`

---

## Phase 8: Advising Workflow — Caseload, Appointments, Case Notes, Referrals

### Purpose
The advisor's daily workspace. After this phase, advisors see ranked caseloads, book appointments, log structured case notes, manage substitutions/exceptions, and run referral loops with outcome tracking.

### Tasks

#### 8.1 — Caseload dashboard

**What**: `GET /caseload` returning the advisor's assigned students enriched with risk level, open alerts, next appointment, last contact, and graduation proximity.

**Design**: Computed read (joins enrollments, alerts, appointments, latest risk score, audit summary). Filter/sort by risk, alert count, milestone proximity. Optionally backed by a materialised `read_advisor_dashboard` projection (Suggestion 2) refreshed on relevant writes for large caseloads.

**Testing**:
- `Integration: advisor with 3 students → 3 enriched rows; sorted by risk desc`
- `Integration: RLS — advisor A cannot see advisor B's caseload`

#### 8.2 — Appointments & scheduling

**What**: `appointments` table, advisor availability, student-facing booking, calendar (.ics) export.

**Design**: Endpoints `GET /advisors/:id/availability`, `POST /appointments` (conflict-checked), `POST /appointments/:id/cancel`. `appointments.ai_briefing` JSONB reserved for Phase 10.

**Testing**:
- `Integration: book overlapping slot → 409`
- `Integration: student books open slot → appointment created, advisor notified`
- `Unit: .ics generation valid`

#### 8.3 — Case notes & exceptions/substitutions

**What**: `case_notes` (Suggestion 3 metadata JSONB with tags/action items) and a substitutions table feeding the audit engine's `overrides`.

**Design**: `POST /students/:id/case-notes`; `is_confidential` gates visibility by role. `POST /students/:id/substitutions` (advisor override: course X satisfies requirement node Y) → invalidates audit cache.

**Testing**:
- `Integration: confidential note hidden from non-authorised role`
- `Integration: add substitution → audit re-evaluates with override applied`
- `Unit: action-item metadata schema validated`

#### 8.4 — Referrals with outcome loop

**What**: `referrals` table with service routing and outcome tracking (Suggestion 3 `outcome_data` JSONB).

**Design**: `POST /referrals` (optionally linked to an alert), `POST /referrals/:id/outcome`. Mental-health referrals store minimal data (privacy). Outcome closure surfaces in analytics.

**Testing**:
- `Integration: create referral from alert → linked, status pending`
- `Integration: record outcome → status closed, outcome_data set`
- `Integration: mental_health referral stores no sensitive detail beyond flags`

---

## Phase 9: Predictive Risk Scoring (ML Sidecar)

### Purpose
Replace rigid thresholds with institution-specific ML risk models that emit explainable scores. After this phase, students carry weekly risk scores with SHAP factor explanations, feeding the caseload dashboard and alert routing.

### Tasks

#### 9.1 — Feature pipeline & ML service contract

**What**: Python FastAPI `ml-service` with a feature builder and a scoring contract consumed by the TS API.

**Design**:
- Feature inputs (de-identified per `standards.md` FERPA note): term GPA trend, credit-completion ratio, LMS login/submission frequency, advising-contact recency, hold count, etc.
- Contract: `POST /score` `{ studentId, features } → { score, riskLevel, explanation: {factors:[{name,value,impact,direction}], confidence} }` (matches Suggestion 3 `risk_scores.explanation`).

**Testing**:
- `Unit (pytest): feature builder produces expected vector from sample student JSON`
- `Integration: TS api calls ml-service /score (mock) → risk_scores row written`

#### 9.2 — Institution-specific training CLI

**What**: `train.py` builds/validates a model from an institution's historical outcomes; persists versioned artifacts.

**Design**: XGBoost classifier; train/validate split by cohort; outputs model + metrics + feature importances; `model_version` recorded. Fairness check: per-subgroup performance disparity report (equity per `research.md`).

**Testing**:
- `Unit (pytest): training on synthetic dataset → model file + metrics produced`
- `Unit: fairness report computes per-subgroup AUC deltas`

#### 9.3 — Weekly scoring job & explainability surfacing

**What**: BullMQ weekly job scores all active students; `risk_scores` (Suggestion 3) updated; explanations surfaced in API.

**Design**: Batched scoring; `GET /students/:id/risk` returns latest score + factors. High-risk transitions can auto-create `ml_model` alerts (reuses Phase 6).

**Testing**:
- `Integration: weekly job over cohort → latest risk_scores per student`
- `Integration: risk crossing high threshold → ml_model alert created`
- `Integration: GET /students/:id/risk → factors with impact/direction`

---

## Phase 10: AI Pre-Meeting Briefings & Outreach

### Purpose
Eliminate manual data assembly — the headline AI-native advantage. After this phase, advisors get LLM-generated briefings assembling cross-system context, AI-drafted case notes, and personalised bulk outreach.

### Tasks

#### 10.1 — Briefing generation

**What**: `POST /appointments/:id/briefing` (and pre-generation job) producing a structured briefing into `appointments.ai_briefing` (Suggestion 3 shape).

**Design**: Assemble context (audit summary, recent grades, alerts, risk + factors, last notes, plan status) → LLM with a structured-output prompt → `{ summary, degreeProgress, recentAlerts, recentGrades, suggestedTopics, riskContext }`. Context assembly only includes data the requesting advisor may view (RLS-respecting).

**Testing**:
- `Integration (mocked LLM): briefing assembles correct context, stored as JSONB`
- `Integration: advisor lacking access to a student → 403, no briefing`
- `Unit: context assembler excludes confidential notes from other advisors`

#### 10.2 — AI case-note drafts

**What**: `POST /students/:id/case-notes/ai-draft` turning an advisor's short summary into a structured draft note (never auto-saved).

**Design**: LLM drafts body + suggested tags/action-items → returned for edit; advisor saves via 8.3. `metadata.ai_generated=true`.

**Testing**:
- `Integration (mocked LLM): summary → draft with tags + action items, not persisted`
- `Integration: advisor edits + saves → persisted note flagged ai_generated`

#### 10.3 — Bulk outreach campaigns

**What**: Email/SMS campaigns to filtered cohorts with optional LLM-personalised templating.

**Design**: `POST /campaigns` `{ filter, channel, template }`; BullMQ fan-out; per-recipient personalisation token expansion (+ optional LLM rewrite). Email via SMTP (mailpit in dev), SMS via pluggable provider. Consent/opt-out enforced.

**Testing**:
- `Integration: campaign over filtered cohort → N queued messages, opt-outs excluded`
- `Integration (mocked LLM): personalised template → per-recipient body`
- `Unit: filter compiles to correct query`

---

## Phase 11: Academic Planning & Optional Knowledge-Graph Acceleration

### Purpose
Multi-term planning with prerequisite/availability awareness and AI pathway generation. Optionally introduce the Neo4j/Memgraph knowledge graph (Suggestion 4) as a derived read model to accelerate audit, what-if, credential discovery, and pathway optimisation. The graph is an optimisation, not a system of record.

### Tasks

#### 11.1 — Academic plans & prerequisite validation

**What**: `academic_plans` + `plan_items` (Suggestion 1) with term-by-term planning and prereq/availability validation.

**Design**: `POST /students/:id/plans`, `POST /plans/:id/items`. `GET /plans/:id/validate` flags prerequisite violations (course planned before its prereq is completed/planned-earlier) and availability gaps (course not `typically_offered` that term). Validation reuses prerequisite data + audit engine.

**Testing**:
- `Unit: plan with course before its prereq → violation reported`
- `Unit: course planned in a term it is never offered → availability warning`
- `Integration: validate plan → structured list of conflicts`

#### 11.2 — AI graduation pathway generation

**What**: `POST /students/:id/plans/generate` producing an LLM/heuristic optimised remaining-courses pathway across terms.

**Design**: Compute remaining requirements from audit; topologically order by prerequisites; pack into terms respecting load + availability; LLM refines ordering with student preferences. Output a draft plan.

**Testing**:
- `Integration: generate for partially-complete student → valid plan covering all unmet requirements, no prereq violations`
- `Unit: topological ordering respects prerequisite DAG`

#### 11.3 — Optional knowledge graph & credential discovery

**What**: A derived graph (Memgraph default for open-source/licensing per Suggestion 4) synced via CDC, powering credential discovery and fast traversals; graceful fallback to recursive SQL CTEs if disabled.

**Design**: Sync service maps Postgres changes to Course/Program/Requirement/Student nodes + REQUIRES/SATISFIED_BY/COMPLETED/DECLARED edges (Suggestion 4). **No PII in the graph.** `GET /students/:id/credential-discovery` runs the ≥85%-complete query; if graph disabled, an equivalent CTE path. Feature-flagged via `institutions.config.graphEnabled`.

**Testing**:
- `Integration: enrollment in Postgres → COMPLETED edge appears after sync`
- `Integration: credential-discovery returns programs ≥85% complete (graph and CTE paths agree on fixture)`
- `Integration: graph disabled → CTE fallback returns identical result`

---

## Phase 12: Student Portal, Analytics, Public API & MCP, Hardening

### Purpose
Complete the student-facing experience, institutional reporting, the published API surface, the MCP server, and production hardening (accessibility, security, observability).

### Tasks

#### 12.1 — Student self-service portal

**What**: `web-student` SPA: degree progress, plan view, graduation eligibility, appointment booking, intake forms — WCAG 2.2 AA.

**Design**: Consumes audit, plan, appointment endpoints with student-scoped sessions. Accessible authentication, target size, focus appearance (WCAG 2.2 new criteria). axe-core checks in CI.

**Testing**:
- `E2E (Playwright): student logs in → sees audit, books appointment`
- `Accessibility: axe scan of all portal pages → zero serious/critical violations`

#### 12.2 — Institutional analytics & equity reporting

**What**: Retention, time-to-graduation, advising-contact frequency, alert response, referral closure, and demographic-disaggregated equity metrics.

**Design**: `read_analytics_retention`-style aggregates (Suggestion 2 projection) refreshed nightly; `GET /analytics/*` endpoints; cohort filters; equity breakdowns. Propensity-score-matched program-effectiveness report (Civitas-style) as an analytics job.

**Testing**:
- `Integration: retention endpoint over seeded cohorts → correct counts`
- `Integration: equity disaggregation by subgroup → per-group metrics`
- `Unit: propensity matching pairs treated/control correctly on synthetic data`

#### 12.3 — Published OpenAPI 3.1 spec, data-model JSON Schema & MCP server

**What**: Finalise and publish `docs/openapi.json`, publish the data model as JSON Schema (data-portability differentiator per `standards.md`), and ship the MCP server.

**Design**: CI fails on undocumented routes or breaking OpenAPI diffs. MCP server (`apps/api/src/mcp`) exposes tools: `getDegreeAudit`, `getStudentRiskProfile`, `getAdvisingHistory`, `runWhatIf` — RLS/role-scoped via a service token. JSON Schema for core entities published under `docs/`.

**Testing**:
- `Integration: every route present in openapi.json (route-vs-spec coverage assertion)`
- `Integration: MCP getDegreeAudit tool → returns audit matching REST endpoint`
- `Integration: MCP call without authorised scope → denied`

#### 12.4 — Security & compliance hardening

**What**: OWASP ASVS L2 pass, NIST 800-171-aligned controls, FERPA audit completeness, field encryption, rate limiting, secrets management.

**Design**: Verify RLS on all tenant tables; PII-read audit coverage; pgcrypto encryption for credentials + sensitive fields; security headers; dependency scanning; pen-test checklist. GLBA safeguards for financial-aid-derived data.

**Testing**:
- `Integration: cross-tenant access attempts across all endpoints → all denied`
- `Integration: PII export → audit_log action=export recorded`
- `Security: ASVS L2 checklist automated where feasible; dependency scan clean in CI`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, RLS, audit log, server)      ─── required by everything
    │
Phase 2: Identity / SSO / RBAC                          ─── requires 1
    │
Phase 3: Catalog & Requirement Authoring                ─── requires 1,2
    │
Phase 4: Degree Audit Engine  ★ core value              ─── requires 3
    │
    ├── Phase 5: SIS/LMS Integration                    ─── requires 1,3 (feeds 4); parallel with 6,7
    ├── Phase 6: Early Alerts & Triggers                ─── requires 5 signals (manual alerts need only 2,3)
    ├── Phase 7: Transfer Credit Articulation           ─── requires 3,4; parallel with 5,6
    │
Phase 8: Advising Workflow (caseload/appts/notes)       ─── requires 4,6 (caseload enrich)
    │
Phase 9: Predictive Risk Scoring (ML)                   ─── requires 5 (signals); parallel with 8 after 5
    │
Phase 10: AI Briefings & Outreach                       ─── requires 4,6,8,9 (assembles all context)
    │
Phase 11: Planning + optional Knowledge Graph           ─── requires 4 (graph optional, parallel-able)
    │
Phase 12: Student Portal, Analytics, API/MCP, Hardening ─── requires the relevant feature phases
```

Parallelism:
- After Phase 4, **Phases 5, 7** can be built concurrently.
- After Phase 5, **Phases 6, 9** can be built concurrently.
- **Phase 11's knowledge graph (11.3)** can be developed independently any time after Phase 4, since it is a derived read model with a CTE fallback.
- Frontends (`web-advisor`, `web-student`) can be scaffolded early and built incrementally as each backend module lands.

---

## Definition of Done (per phase)

1. All tasks implemented and merged behind passing CI.
2. All unit and integration tests pass (integration uses Testcontainers for real Postgres/Redis).
3. ESLint + Prettier clean; `tsc --noEmit` passes (and ruff + mypy for `ml-service` where touched).
4. Docker images build; `docker-compose up` brings the stack to a ready state.
5. Feature works end-to-end (demonstrable via API call, job run, or E2E test).
6. New config options documented in `docs/` and reflected in `config.ts`/`institutions.config` schema.
7. New/changed API endpoints appear in `docs/openapi.json`; OpenAPI diff reviewed (no unintended breaking changes).
8. Drizzle migrations generated, reviewed, and drift-checked in CI.
9. RLS policies present on any new tenant-scoped table; mutating routes write `audit_log` entries.
10. Accessibility: any new portal UI passes axe-core (zero serious/critical) for WCAG 2.2 AA.
11. Any LLM-backed feature has a mocked-provider integration test and never auto-applies AI output without human confirmation where a human-in-the-loop is specified.
```
