# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Student Advising Platform (427) | Generated: 2026-05-25

---

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS), where every state change in the student advising domain is captured as an immutable event in an append-only event store. The current state of any entity (student, degree progress, alert, referral) is derived by replaying its event stream. Read-optimised projections (materialised views) are maintained separately for advisor dashboards, degree audit displays, and analytics queries.

This approach is motivated by the domain's strong need for a complete, tamper-proof audit trail of student record changes (FERPA compliance), the ability to reconstruct any student's state at any historical point in time, and the natural fit between academic events (enrolled, graded, alerted, referred) and an event-first data model.

---

## Architecture Overview

```
                         ┌──────────────────┐
                         │   Command API    │
                         │  (writes only)   │
                         └────────┬─────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │      Event Store        │
                    │  (append-only, ordered)  │
                    │  PostgreSQL / EventStoreDB│
                    └────────────┬────────────┘
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼             ▼
            ┌─────────┐  ┌───────────┐  ┌──────────┐
            │ Degree   │  │ Advisor   │  │Analytics │
            │ Audit    │  │ Dashboard │  │ & Risk   │
            │Projection│  │Projection │  │Projection│
            └─────────┘  └───────────┘  └──────────┘
                    │            │             │
                    ▼            ▼             ▼
            ┌─────────┐  ┌───────────┐  ┌──────────┐
            │ Read DB  │  │ Read DB   │  │ Analytics│
            │(Postgres)│  │(Postgres) │  │(ClickHouse│
            └─────────┘  └───────────┘  │ or DuckDB)│
                                        └──────────┘
                         ┌──────────────────┐
                         │    Query API     │
                         │  (reads only)    │
                         └──────────────────┘
```

---

## Key Entities: Event Store Schema

### Event Store Core

```sql
-- The append-only event store
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,              -- aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,        -- student, alert, referral, plan, etc.
    event_type      VARCHAR(100) NOT NULL,       -- e.g. StudentEnrolled, GradePosted
    event_version   INTEGER NOT NULL,            -- version within the stream (optimistic concurrency)
    payload         JSONB NOT NULL,              -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, ip
    institution_id  UUID NOT NULL,               -- tenant isolation
    occurred_at     TIMESTAMPTZ NOT NULL,        -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when stored
    UNIQUE (stream_id, event_version)
);

-- Partitioned by month for scalability
CREATE INDEX idx_events_stream ON events (stream_id, event_version);
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_institution ON events (institution_id, occurred_at);

-- Subscriptions track projection positions
CREATE TABLE projections (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_position   BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain Event Types

```typescript
// Student lifecycle events
type StudentRegistered = {
  studentId: string;
  institutionId: string;
  sisStudentId: string;
  firstName: string;
  lastName: string;
  email: string;
  admitTermId: string;
};

type ProgramDeclared = {
  studentId: string;
  programId: string;
  catalogYearId: string;
  declarationType: 'major' | 'minor' | 'certificate';
  isPrimary: boolean;
};

type ProgramWithdrawn = {
  studentId: string;
  programId: string;
  reason: string;
};

// Enrollment events
type CourseEnrolled = {
  studentId: string;
  sectionId: string;
  courseId: string;
  termId: string;
  credits: number;
};

type CourseDropped = {
  studentId: string;
  sectionId: string;
  reason: string;
  effectiveDate: string;
};

type GradePosted = {
  studentId: string;
  sectionId: string;
  courseId: string;
  grade: string;
  gradePoints: number;
  creditsEarned: number;
};

type TransferCreditEvaluated = {
  studentId: string;
  sourceInstitution: string;
  sourceCourseCode: string;
  equivalentCourseId: string | null;
  credits: number;
  grade: string;
  evaluatedBy: string;
};

// Early alert events
type EarlyAlertRaised = {
  alertId: string;
  studentId: string;
  raisedBy: string;
  alertType: 'academic' | 'attendance' | 'financial' | 'engagement';
  severity: 'low' | 'medium' | 'high' | 'critical';
  triggerSource: 'faculty_submitted' | 'system_generated' | 'ml_model';
  description: string;
};

type EarlyAlertAssigned = {
  alertId: string;
  assignedTo: string;
};

type EarlyAlertResolved = {
  alertId: string;
  resolvedBy: string;
  resolution: string;
  notes: string;
};

// Referral events
type ReferralCreated = {
  referralId: string;
  studentId: string;
  referredBy: string;
  alertId: string | null;
  serviceType: string;
  serviceProvider: string;
};

type ReferralOutcomeRecorded = {
  referralId: string;
  outcome: 'attended' | 'no_show' | 'declined' | 'completed';
  notes: string;
};

// Advising events
type AppointmentScheduled = {
  appointmentId: string;
  studentId: string;
  advisorId: string;
  scheduledStart: string;
  scheduledEnd: string;
  appointmentType: string;
};

type CaseNoteCreated = {
  noteId: string;
  studentId: string;
  advisorId: string;
  appointmentId: string | null;
  noteType: string;
  subject: string;
  body: string;
  isConfidential: boolean;
};

// Risk scoring events
type RiskScoreCalculated = {
  studentId: string;
  modelId: string;
  modelVersion: string;
  score: number;
  riskLevel: string;
  explanationFactors: Record<string, number>;
};

// Academic plan events
type AcademicPlanCreated = {
  planId: string;
  studentId: string;
  programId: string;
  createdBy: string;
};

type PlanCourseAdded = {
  planId: string;
  termId: string;
  courseId: string;
};

type PlanCourseRemoved = {
  planId: string;
  termId: string;
  courseId: string;
  reason: string;
};
```

### Read Model Projections

```sql
-- Projection: Current student state (materialised from events)
CREATE TABLE read_student_profiles (
    student_id      UUID PRIMARY KEY,
    institution_id  UUID NOT NULL,
    sis_student_id  VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    current_gpa     NUMERIC(4,3),
    total_credits   NUMERIC(5,1),
    current_term_credits NUMERIC(4,1),
    active_programs JSONB,          -- [{programId, name, type, catalogYear}]
    enrollment_status VARCHAR(20),  -- enrolled, graduated, withdrawn
    risk_level      VARCHAR(10),
    risk_score      NUMERIC(5,4),
    open_alerts     INTEGER DEFAULT 0,
    pending_referrals INTEGER DEFAULT 0,
    last_advising_contact TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Projection: Degree audit state per student-program
CREATE TABLE read_degree_audit (
    student_id      UUID NOT NULL,
    program_id      UUID NOT NULL,
    catalog_year_id UUID NOT NULL,
    requirements_total INTEGER NOT NULL,
    requirements_met INTEGER NOT NULL,
    credits_required NUMERIC(5,1),
    credits_earned  NUMERIC(5,1),
    gpa_required    NUMERIC(3,2),
    gpa_current     NUMERIC(4,3),
    audit_details   JSONB NOT NULL,  -- full requirement tree with satisfaction status
    projected_graduation_term VARCHAR(50),
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (student_id, program_id)
);

-- Projection: Advisor caseload dashboard
CREATE TABLE read_advisor_dashboard (
    advisor_id      UUID NOT NULL,
    student_id      UUID NOT NULL,
    student_name    VARCHAR(255) NOT NULL,
    risk_level      VARCHAR(10),
    open_alerts     INTEGER DEFAULT 0,
    pending_tasks   INTEGER DEFAULT 0,
    next_appointment TIMESTAMPTZ,
    last_contact    TIMESTAMPTZ,
    graduation_proximity VARCHAR(20), -- on_track, at_risk, overdue
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (advisor_id, student_id)
);

-- Projection: Analytics (denormalised for reporting)
CREATE TABLE read_analytics_retention (
    institution_id  UUID NOT NULL,
    term_id         UUID NOT NULL,
    cohort_type     VARCHAR(30) NOT NULL,       -- admit_term, demographic, program
    cohort_value    VARCHAR(100) NOT NULL,
    enrolled_count  INTEGER NOT NULL,
    retained_count  INTEGER NOT NULL,
    graduated_count INTEGER NOT NULL,
    alerts_raised   INTEGER NOT NULL,
    referrals_made  INTEGER NOT NULL,
    avg_gpa         NUMERIC(4,3),
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (institution_id, term_id, cohort_type, cohort_value)
);
```

---

## How It Works: Degree Audit Example

1. **Command**: Student enrolls in CHEM 201.
2. **Event stored**: `CourseEnrolled { studentId, sectionId, courseId, termId, credits }`.
3. **Degree audit projection** receives the event, replays the student's enrollment and grade events, re-evaluates each degree requirement node against the updated course list, and writes the updated `read_degree_audit` row with the full requirement tree.
4. **What-if analysis**: Create a temporary in-memory projection, inject hypothetical events (e.g., `ProgramDeclared` for a different major), replay, and return the projected audit without persisting anything.

---

## Pros

- **Complete audit trail by design**: Every change to a student's record is captured as an immutable event. FERPA compliance is inherent: who changed what, when, and why is permanently recorded. No data is ever lost or overwritten.
- **Point-in-time reconstruction**: An advisor can see exactly what a student's degree progress looked like on any historical date by replaying events up to that timestamp. This is invaluable for dispute resolution, academic appeals, and regulatory audits.
- **Natural domain model**: Academic events (enrolled, graded, alerted, referred, advised) map directly to domain events. The event stream is a natural narrative of the student's journey through the institution.
- **Independent read model optimisation**: Each projection (degree audit, advisor dashboard, analytics) can be structured and indexed for its specific query patterns without compromising the write model. Analytics can use ClickHouse or DuckDB for columnar aggregation while advising dashboards use PostgreSQL.
- **Robust what-if analysis**: Hypothetical degree audits are just temporary event stream replays with injected events, requiring no complex query logic or temporary database state.
- **Decoupled systems**: New features (e.g., an AI briefing generator) simply subscribe to the event stream and build their own projections without modifying existing code.

---

## Cons

- **Complexity**: Event sourcing is significantly more complex than CRUD. Developers must understand aggregates, event handlers, projections, eventual consistency, and idempotency. Hiring and onboarding are harder.
- **Eventual consistency**: Read models lag behind writes (typically milliseconds to seconds, but potentially longer under load). An advisor saving a case note may not see it immediately in the dashboard. Users must be informed about consistency windows.
- **Schema evolution**: Changing event schemas over time requires careful versioning. Old events must remain deserializable. Upcasting (transforming old event formats to new) adds maintenance burden. In a domain with 40+ year catalog histories, this is non-trivial.
- **Projection rebuilds**: If a projection has a bug or a new projection is needed, all events must be replayed. For a large institution with millions of enrollment events, full replays can take hours.
- **Overkill for simple entities**: Not every entity benefits from event sourcing. Courses, programs, and catalog requirements are relatively static reference data that are more naturally modelled as CRUD tables. Applying event sourcing to everything adds unnecessary complexity.
- **Debugging difficulty**: Tracing why a degree audit shows an unexpected result requires replaying and inspecting the event stream, which is harder than querying a single normalised table.
- **Storage growth**: The event store grows continuously. A large institution with 50,000 students and 15 events per student per term generates ~750,000 events per term. Over a decade, this accumulates to tens of millions of events requiring storage and replay management.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL with append-only events table (simpler) or EventStoreDB (purpose-built) |
| Event bus | Apache Kafka or NATS JetStream for event distribution to projections |
| Read databases | PostgreSQL for transactional read models, ClickHouse or DuckDB for analytics |
| Projection framework | Custom event handlers or Marten (.NET), Eventide (Ruby), or Axon (Java) |
| Serialisation | JSON with schema registry (Confluent Schema Registry or custom) for event versioning |
| API layer | Separate command API (write) and query API (read) services |
| Snapshotting | Periodic aggregate snapshots (every N events) to avoid full replay on every read |

---

## Migration and Scaling Considerations

- **Hybrid approach**: Apply event sourcing selectively to high-value aggregates (student academic record, early alerts, referrals) while keeping reference data (courses, programs, requirements) as standard CRUD tables. This significantly reduces complexity.
- **Snapshotting strategy**: For aggregates with long event histories (e.g., a student with 8 years of enrollment events), store periodic snapshots and replay only from the latest snapshot. Snapshot every 50-100 events per aggregate.
- **Event store partitioning**: Partition the events table by `institution_id` and month for multi-tenant performance. Use logical partitioning in Kafka topics for event distribution.
- **Projection catch-up**: Design projections to be idempotent and re-buildable from scratch. Maintain a "projection checkpoint" table tracking the last processed event per projection. Use parallel projection rebuilds with partitioned event ranges.
- **SIS integration**: Inbound SIS data feeds are naturally modelled as event producers. An SIS sync connector emits `CourseEnrolled`, `GradePosted`, and `HoldPlaced` events as it processes incoming data, keeping the event store as the single source of truth.
- **Data retention**: Events are immutable and cannot be deleted under normal operation. For FERPA right-to-deletion requests, implement "tombstone" events that logically delete PII while preserving the event structure for audit purposes. Alternatively, use a separate PII store referenced by event metadata, allowing PII deletion without modifying events.
- **Testing**: Event-sourced systems require different testing strategies. Test aggregates by replaying event sequences and asserting resulting state. Test projections by feeding known event streams and verifying read model output.
