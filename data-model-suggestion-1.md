# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Student Advising Platform (427) | Generated: 2026-05-25

---

## Summary

A fully normalized relational data model implemented in PostgreSQL, following third normal form (3NF) conventions for all core entities. This approach prioritises data integrity, ACID compliance, and alignment with established higher education data standards (1EdTech Edu-API, PESC transcript schema). Every entity has a well-defined table, foreign key relationships enforce referential integrity, and complex degree requirement logic is modelled through a recursive rule hierarchy.

This is the most conservative and well-understood approach, mirroring the architectural patterns used by incumbent systems like Ellucian Degree Works and CollegeSource uAchieve, which have modelled degree audit rules relationally for over 40 years.

---

## Key Entities and Relationships

### Core Student and Institution Domain

```sql
-- Multi-tenant institution support
CREATE TABLE institutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    ipeds_id        VARCHAR(20) UNIQUE,        -- IPEDS institution code
    ope_id          VARCHAR(20),               -- Office of Postsecondary Education ID
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Students as known to the institution
CREATE TABLE students (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    sis_student_id  VARCHAR(50) NOT NULL,      -- ID from the source SIS
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    preferred_name  VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    date_of_birth   DATE,
    gender          VARCHAR(20),
    ethnicity       VARCHAR(50),               -- for equity-disaggregated reporting
    first_generation BOOLEAN DEFAULT FALSE,
    pell_recipient  BOOLEAN DEFAULT FALSE,
    residency_status VARCHAR(30),
    admit_term_id   UUID REFERENCES academic_terms(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, sis_student_id)
);

-- Advisors and staff
CREATE TABLE advisors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    department_id   UUID REFERENCES departments(id),
    title           VARCHAR(100),
    max_caseload    INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Caseload assignments
CREATE TABLE caseload_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    student_id      UUID NOT NULL REFERENCES students(id),
    assignment_type VARCHAR(30) NOT NULL,       -- primary, secondary, specialty
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    UNIQUE (advisor_id, student_id, assignment_type)
);
```

### Academic Catalog and Degree Requirements

```sql
-- Catalog years define requirement snapshots
CREATE TABLE catalog_years (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    year_start      INTEGER NOT NULL,          -- e.g. 2025
    year_end        INTEGER NOT NULL,          -- e.g. 2026
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (institution_id, year_start, year_end)
);

-- Academic programs (majors, minors, certificates)
CREATE TABLE programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    program_type    VARCHAR(30) NOT NULL,       -- major, minor, certificate, concentration
    degree_level    VARCHAR(30) NOT NULL,       -- associate, bachelor, master, doctoral
    department_id   UUID REFERENCES departments(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (institution_id, code)
);

-- Student program declarations
CREATE TABLE student_programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    program_id      UUID NOT NULL REFERENCES programs(id),
    catalog_year_id UUID NOT NULL REFERENCES catalog_years(id),
    declaration_date DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- active, completed, withdrawn
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE
);

-- Courses in the institution catalog
CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    subject_code    VARCHAR(10) NOT NULL,      -- e.g. MATH
    course_number   VARCHAR(10) NOT NULL,      -- e.g. 201
    title           VARCHAR(255) NOT NULL,
    credits_min     NUMERIC(4,1) NOT NULL,
    credits_max     NUMERIC(4,1) NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (institution_id, subject_code, course_number)
);

-- Prerequisite relationships between courses
CREATE TABLE course_prerequisites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(id),
    prerequisite_course_id UUID REFERENCES courses(id),
    prerequisite_group INTEGER NOT NULL DEFAULT 1, -- groups for OR logic
    min_grade       VARCHAR(5),                    -- minimum grade required
    is_corequisite  BOOLEAN NOT NULL DEFAULT FALSE
);

-- Recursive degree requirement tree
CREATE TABLE degree_requirements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES programs(id),
    catalog_year_id UUID NOT NULL REFERENCES catalog_years(id),
    parent_id       UUID REFERENCES degree_requirements(id), -- recursive hierarchy
    name            VARCHAR(255) NOT NULL,
    requirement_type VARCHAR(30) NOT NULL,      -- group, course, elective, gpa_check, credit_total
    logic_operator  VARCHAR(5) DEFAULT 'AND',   -- AND, OR for child requirements
    min_courses     INTEGER,                    -- minimum courses to satisfy
    min_credits     NUMERIC(5,1),               -- minimum credits to satisfy
    min_gpa         NUMERIC(3,2),               -- minimum GPA if gpa_check
    sort_order      INTEGER NOT NULL DEFAULT 0
);

-- Courses that can satisfy a requirement
CREATE TABLE requirement_courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requirement_id  UUID NOT NULL REFERENCES degree_requirements(id),
    course_id       UUID NOT NULL REFERENCES courses(id),
    is_recommended  BOOLEAN NOT NULL DEFAULT FALSE
);
```

### Enrollment and Academic Records

```sql
-- Academic terms / semesters
CREATE TABLE academic_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    name            VARCHAR(50) NOT NULL,       -- e.g. "Fall 2025"
    term_type       VARCHAR(20) NOT NULL,       -- fall, spring, summer, winter
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    census_date     DATE,
    is_current      BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (institution_id, name)
);

-- Course sections offered in a term
CREATE TABLE course_sections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    course_id       UUID NOT NULL REFERENCES courses(id),
    term_id         UUID NOT NULL REFERENCES academic_terms(id),
    section_number  VARCHAR(10) NOT NULL,
    instructor_name VARCHAR(255),
    capacity        INTEGER,
    enrolled_count  INTEGER DEFAULT 0,
    meeting_pattern VARCHAR(100),               -- e.g. MWF 10:00-10:50
    delivery_mode   VARCHAR(20),                -- in_person, online, hybrid
    UNIQUE (course_id, term_id, section_number)
);

-- Student enrollments (completed and in-progress)
CREATE TABLE enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    section_id      UUID NOT NULL REFERENCES course_sections(id),
    status          VARCHAR(20) NOT NULL,       -- enrolled, completed, withdrawn, failed
    grade           VARCHAR(5),
    grade_points    NUMERIC(4,2),
    credits_attempted NUMERIC(4,1),
    credits_earned  NUMERIC(4,1),
    is_repeat       BOOLEAN NOT NULL DEFAULT FALSE,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (student_id, section_id)
);

-- Transfer credits from external institutions
CREATE TABLE transfer_credits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    source_institution VARCHAR(255) NOT NULL,
    source_course_code VARCHAR(30),
    source_course_title VARCHAR(255),
    credits         NUMERIC(4,1) NOT NULL,
    grade           VARCHAR(5),
    equivalent_course_id UUID REFERENCES courses(id),
    evaluation_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    evaluated_by    UUID REFERENCES advisors(id),
    evaluated_at    TIMESTAMPTZ
);
```

### Advising Workflow

```sql
-- Advising appointments
CREATE TABLE appointments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    appointment_type VARCHAR(30) NOT NULL,      -- drop_in, scheduled, virtual
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    location        VARCHAR(255),
    meeting_url     VARCHAR(500),
    intake_form_id  UUID,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Case notes from advising sessions
CREATE TABLE case_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    appointment_id  UUID REFERENCES appointments(id),
    note_type       VARCHAR(30) NOT NULL,       -- advising, early_alert, referral, general
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    is_confidential BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Early alerts
CREATE TABLE early_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    raised_by       UUID NOT NULL REFERENCES users(id),  -- faculty or system
    assigned_to     UUID REFERENCES advisors(id),
    alert_type      VARCHAR(30) NOT NULL,       -- academic, attendance, financial, engagement
    severity        VARCHAR(10) NOT NULL,       -- low, medium, high, critical
    trigger_source  VARCHAR(30) NOT NULL,       -- faculty_submitted, system_generated, ml_model
    description     TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    resolved_at     TIMESTAMPTZ,
    resolution_notes TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Referrals to support services
CREATE TABLE referrals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    referred_by     UUID NOT NULL REFERENCES advisors(id),
    early_alert_id  UUID REFERENCES early_alerts(id),
    service_type    VARCHAR(50) NOT NULL,       -- tutoring, mental_health, disability, financial_aid, career
    service_provider VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    outcome         VARCHAR(50),                -- attended, no_show, declined, completed
    outcome_notes   TEXT,
    referred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    outcome_at      TIMESTAMPTZ
);

-- Risk scores (from ML models)
CREATE TABLE risk_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    model_id        VARCHAR(50) NOT NULL,
    model_version   VARCHAR(20) NOT NULL,
    score           NUMERIC(5,4) NOT NULL,      -- 0.0000 to 1.0000
    risk_level      VARCHAR(10) NOT NULL,       -- low, medium, high, critical
    explanation     JSONB,                      -- explainability factors
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_risk_scores_student_latest 
    ON risk_scores (student_id, scored_at DESC);
```

### Academic Planning

```sql
-- Student academic plans
CREATE TABLE academic_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    program_id      UUID NOT NULL REFERENCES programs(id),
    name            VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, active, archived
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Planned courses per term in a plan
CREATE TABLE plan_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES academic_plans(id),
    term_id         UUID NOT NULL REFERENCES academic_terms(id),
    course_id       UUID NOT NULL REFERENCES courses(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',  -- planned, registered, completed, dropped
    sort_order      INTEGER NOT NULL DEFAULT 0,
    notes           TEXT
);
```

---

## Entity Relationship Summary

```
Institution 1--* Program
Institution 1--* AcademicTerm
Institution 1--* Course
Institution 1--* CatalogYear

Student *--1 Institution
Student 1--* StudentProgram --* Program
Student 1--* Enrollment --* CourseSection --* Course
Student 1--* TransferCredit
Student 1--* EarlyAlert
Student 1--* Referral
Student 1--* RiskScore
Student 1--* Appointment --* Advisor
Student 1--* CaseNote
Student 1--* AcademicPlan 1--* PlanItem

Program + CatalogYear 1--* DegreeRequirement (recursive tree)
DegreeRequirement *--* Course (via RequirementCourse)

Course 1--* CoursePrerequisite
Course 1--* CourseSection --* AcademicTerm

Advisor 1--* CaseloadAssignment --* Student
Advisor 1--* CaseNote
Advisor 1--* Appointment
```

---

## Pros

- **Data integrity**: Foreign keys and constraints enforce referential integrity across all entities. Grade changes, enrollment updates, and requirement modifications are ACID-compliant. This is critical for FERPA-protected student records.
- **Mature tooling**: PostgreSQL is the most widely deployed open-source relational database. Ecosystem includes pgAdmin, pg_dump, logical replication, and extensive monitoring tooling. Institutional IT teams are familiar with relational administration.
- **Standards alignment**: The normalized model maps cleanly to 1EdTech Edu-API entities (Person, CourseTemplate, CourseOffering, AcademicSession, Enrollment), PESC transcript schemas, and Ed-Fi UDM conventions. This simplifies connector development.
- **Query flexibility**: Complex degree audit queries (recursive requirement traversal, GPA calculations, credit aggregation) are well-served by PostgreSQL's CTEs, window functions, and aggregate operations.
- **Regulatory auditability**: Normalised tables with timestamp columns support audit trail queries. Row-level security in PostgreSQL can enforce FERPA access controls at the database layer.
- **Proven at scale**: Systems like Degree Works and uAchieve have used normalised relational models for 40+ years at thousands of institutions serving millions of students.

---

## Cons

- **Schema rigidity**: Adding institution-specific fields (e.g., custom student attributes, non-standard requirement types) requires schema migrations. Different institutions may need different columns, creating deployment complexity for a multi-tenant SaaS.
- **Complex requirement modelling**: The recursive degree requirement tree is difficult to query efficiently. Deeply nested AND/OR logic with exceptions, substitutions, and GPA sub-requirements produces complex SQL that is hard to optimise and debug.
- **Migration overhead**: Schema changes require coordinated migrations across all tenants. In a SaaS deployment, rolling schema updates must be backward-compatible.
- **Impedance mismatch with AI**: ML model features and risk score explanations are inherently semi-structured. Forcing them into normalised tables produces either many sparse columns or awkward key-value patterns.
- **Join-heavy queries**: Assembling a complete student advising picture (enrollments + degree progress + alerts + risk scores + case notes) requires joining 10+ tables. Performance depends on careful indexing and query optimisation.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| ORM / Query Builder | Drizzle ORM or Prisma (TypeScript), SQLAlchemy (Python) |
| Migrations | Flyway or dbmate for version-controlled schema migrations |
| Connection pooling | PgBouncer for multi-tenant connection management |
| Audit logging | pgAudit extension for FERPA-compliant access logging |
| Row-level security | PostgreSQL RLS policies for tenant isolation |
| Full-text search | PostgreSQL tsvector or pg_trgm for student/course lookup |
| Backup | pg_basebackup + WAL archiving for point-in-time recovery |

---

## Migration and Scaling Considerations

- **Multi-tenancy**: Use a shared-schema approach with `institution_id` on all tables and RLS policies, or a schema-per-tenant approach for stronger isolation. Shared-schema is simpler to manage; schema-per-tenant provides better data isolation for FERPA-sensitive institutions.
- **SIS data synchronisation**: Design an ETL/CDC pipeline with a staging schema for incoming SIS data. Validate and transform before merging into the canonical model. Track sync timestamps per entity to support incremental updates.
- **Partitioning**: Partition `enrollments`, `risk_scores`, and `early_alerts` by term or date range for performance at scale. PostgreSQL declarative partitioning handles this natively.
- **Read replicas**: Deploy read replicas for analytics and reporting queries to avoid impacting transactional workloads (advisor dashboards, degree audits).
- **Data archival**: Implement a retention policy that archives graduated/withdrawn student records to cold storage after a configurable period, retaining summary records for institutional reporting.
- **Version control for requirements**: Catalog year scoping means requirement changes create new rows rather than modifying existing ones. Historical audits always reflect the requirements in effect at the time of the audit.
