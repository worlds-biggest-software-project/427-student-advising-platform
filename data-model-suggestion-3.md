# Data Model Suggestion 3: Hybrid Relational + JSONB / Document Approach

> Project: Student Advising Platform (427) | Generated: 2026-05-25

---

## Summary

A pragmatic hybrid architecture that combines PostgreSQL's relational strengths for well-defined, stable entities (students, courses, enrollments, terms) with JSONB document columns for semi-structured, institution-variable, or rapidly evolving data (degree requirement rules, risk model outputs, AI-generated briefings, custom student attributes, catalog exceptions). This approach recognises that a student advising platform serves hundreds of institutions, each with unique catalog structures, custom fields, and workflow variations that cannot be predicted at schema design time.

The core principle is: **normalise what is universal, document what is variable**. Stable entities that are consistent across all institutions get traditional relational columns with foreign keys. Institution-specific configurations, flexible rule definitions, and AI-generated content use JSONB columns with GIN indexes for queryability.

---

## Design Principles

1. **Stable core, flexible edges**: Student, enrollment, term, and course entities are fully normalised. Degree requirements, risk explanations, custom attributes, and integration payloads use JSONB.
2. **Single database engine**: PostgreSQL serves as both the relational and document store, eliminating the operational complexity of running a separate MongoDB or similar system.
3. **Schema validation at the application layer**: JSONB columns are validated by application-level JSON Schema definitions, not database constraints. This allows per-institution schema variations without DDL changes.
4. **Indexed JSONB for queryability**: GIN indexes on JSONB columns enable efficient containment queries (`@>`) and path-based lookups without sacrificing query performance.

---

## Key Entities and Relationships

### Core Relational Tables (Normalised)

```sql
-- Institutions
CREATE TABLE institutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    ipeds_id        VARCHAR(20) UNIQUE,
    config          JSONB NOT NULL DEFAULT '{}',  -- institution-level settings
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Students: normalised core + JSONB for institution-specific attributes
CREATE TABLE students (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    sis_student_id  VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    preferred_name  VARCHAR(100),
    email           VARCHAR(255),
    phone           VARCHAR(30),
    date_of_birth   DATE,
    -- Demographics for equity reporting (normalised columns for standard fields)
    gender          VARCHAR(20),
    ethnicity       VARCHAR(50),
    first_generation BOOLEAN DEFAULT FALSE,
    pell_recipient  BOOLEAN DEFAULT FALSE,
    -- Institution-specific custom fields (JSONB for flexibility)
    custom_attributes JSONB NOT NULL DEFAULT '{}',
    -- Examples of what custom_attributes might contain:
    -- { "veteran_status": true, "housing": "on_campus",
    --   "learning_community": "STEM Scholars",
    --   "cohort_tags": ["first-year", "transfer", "online"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, sis_student_id)
);

CREATE INDEX idx_students_custom ON students USING GIN (custom_attributes);

-- Academic terms (fully normalised — universal across institutions)
CREATE TABLE academic_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    name            VARCHAR(50) NOT NULL,
    term_type       VARCHAR(20) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    census_date     DATE,
    is_current      BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (institution_id, name)
);

-- Courses (normalised core + JSONB for variable catalog metadata)
CREATE TABLE courses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    subject_code    VARCHAR(10) NOT NULL,
    course_number   VARCHAR(10) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    credits_min     NUMERIC(4,1) NOT NULL,
    credits_max     NUMERIC(4,1) NOT NULL,
    -- Variable catalog metadata (descriptions, learning outcomes, attributes)
    catalog_data    JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- { "description": "...", "learning_outcomes": [...],
    --   "attributes": ["writing_intensive", "diversity_requirement"],
    --   "typically_offered": ["fall", "spring"],
    --   "delivery_modes": ["in_person", "online"] }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (institution_id, subject_code, course_number)
);

CREATE INDEX idx_courses_catalog ON courses USING GIN (catalog_data);

-- Enrollments (fully normalised — critical transactional data)
CREATE TABLE enrollments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    section_id      UUID NOT NULL REFERENCES course_sections(id),
    status          VARCHAR(20) NOT NULL,
    grade           VARCHAR(5),
    grade_points    NUMERIC(4,2),
    credits_attempted NUMERIC(4,1),
    credits_earned  NUMERIC(4,1),
    is_repeat       BOOLEAN NOT NULL DEFAULT FALSE,
    -- SIS sync metadata
    sis_sync        JSONB NOT NULL DEFAULT '{}',
    -- { "sis_enrollment_id": "...", "last_synced": "...", "sync_source": "banner" }
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (student_id, section_id)
);
```

### Degree Requirements (JSONB-Heavy — The Key Differentiator)

The degree requirement model is where hybrid shines most. Institutional catalog requirements are extraordinarily complex and vary wildly between institutions. A JSONB-based rule tree avoids the need for a deep recursive relational schema while maintaining queryability.

```sql
-- Programs (normalised shell)
CREATE TABLE programs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    code            VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    program_type    VARCHAR(30) NOT NULL,
    degree_level    VARCHAR(30) NOT NULL,
    department_id   UUID REFERENCES departments(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (institution_id, code)
);

-- Degree requirements stored as a JSONB rule tree per program + catalog year
CREATE TABLE degree_requirement_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_id      UUID NOT NULL REFERENCES programs(id),
    catalog_year    VARCHAR(9) NOT NULL,        -- e.g. "2025-2026"
    version         INTEGER NOT NULL DEFAULT 1,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, published, archived
    -- The full requirement tree as a JSONB document
    requirements    JSONB NOT NULL,
    -- Validation schema reference
    schema_version  VARCHAR(10) NOT NULL DEFAULT '1.0',
    created_by      UUID REFERENCES users(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (program_id, catalog_year, version)
);

-- Example of the requirements JSONB structure:
-- {
--   "name": "B.S. Computer Science",
--   "total_credits_required": 120,
--   "minimum_gpa": 2.0,
--   "residency_credits": 30,
--   "children": [
--     {
--       "id": "gen-ed",
--       "name": "General Education",
--       "type": "group",
--       "logic": "AND",
--       "min_credits": 42,
--       "children": [
--         {
--           "id": "gen-ed-writing",
--           "name": "Writing Requirement",
--           "type": "course_list",
--           "logic": "ALL",
--           "courses": [
--             {"course_id": "uuid-eng101", "subject": "ENG", "number": "101", "min_grade": "C"},
--             {"course_id": "uuid-eng102", "subject": "ENG", "number": "102", "min_grade": "C"}
--           ]
--         },
--         {
--           "id": "gen-ed-math",
--           "name": "Mathematics",
--           "type": "course_list",
--           "logic": "CHOOSE",
--           "min_courses": 1,
--           "min_credits": 3,
--           "courses": [
--             {"course_id": "uuid-math151", "subject": "MATH", "number": "151"},
--             {"course_id": "uuid-math161", "subject": "MATH", "number": "161"},
--             {"course_id": "uuid-stat200", "subject": "STAT", "number": "200"}
--           ]
--         },
--         {
--           "id": "gen-ed-diversity",
--           "name": "Diversity Requirement",
--           "type": "attribute_filter",
--           "attribute": "diversity_requirement",
--           "min_courses": 1,
--           "min_credits": 3
--         }
--       ]
--     },
--     {
--       "id": "core",
--       "name": "Computer Science Core",
--       "type": "group",
--       "logic": "AND",
--       "min_gpa": 2.5,
--       "children": [ ... ]
--     },
--     {
--       "id": "electives",
--       "name": "Free Electives",
--       "type": "credit_bucket",
--       "min_credits": 15,
--       "exclusions": ["courses already used in other requirements"]
--     }
--   ]
-- }
```

### Early Alerts and Risk Scoring (Hybrid)

```sql
-- Early alerts: normalised core with JSONB for trigger details
CREATE TABLE early_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    raised_by       UUID NOT NULL REFERENCES users(id),
    assigned_to     UUID REFERENCES advisors(id),
    alert_type      VARCHAR(30) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    trigger_source  VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    -- Flexible trigger details (varies by alert type and trigger source)
    trigger_details JSONB NOT NULL DEFAULT '{}',
    -- Examples:
    -- For system_generated: {"rule": "gpa_below_2.0", "current_gpa": 1.85, "term": "Fall 2025"}
    -- For ml_model: {"model": "retention_v3", "score": 0.72, "top_factors": [...]}
    -- For faculty_submitted: {"course": "CHEM 201", "concern": "missed 3 labs", "faculty_notes": "..."}
    resolution      JSONB,
    -- { "resolved_by": "uuid", "action": "referral_to_tutoring", "notes": "...", "resolved_at": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_trigger ON early_alerts USING GIN (trigger_details);

-- Risk scores with JSONB explainability
CREATE TABLE risk_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    model_id        VARCHAR(50) NOT NULL,
    model_version   VARCHAR(20) NOT NULL,
    score           NUMERIC(5,4) NOT NULL,
    risk_level      VARCHAR(10) NOT NULL,
    -- Rich explainability output (variable by model version)
    explanation     JSONB NOT NULL,
    -- { "factors": [
    --     {"name": "term_gpa", "value": 1.85, "impact": -0.15, "direction": "negative"},
    --     {"name": "lms_login_frequency", "value": 2, "impact": -0.12, "direction": "negative"},
    --     {"name": "advising_contact_recency", "value": 45, "unit": "days", "impact": -0.08},
    --     {"name": "credits_completed_ratio", "value": 0.85, "impact": 0.05, "direction": "positive"}
    --   ],
    --   "confidence": 0.82,
    --   "comparison_cohort": "same_major_same_term" }
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_risk_student_latest ON risk_scores (student_id, scored_at DESC);
```

### Advising Workflow (Mostly Normalised)

```sql
-- Appointments
CREATE TABLE appointments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    appointment_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    location        VARCHAR(255),
    meeting_url     VARCHAR(500),
    -- AI-generated pre-meeting briefing (JSONB for rich structured content)
    ai_briefing     JSONB,
    -- { "generated_at": "...", "model": "claude-4",
    --   "summary": "...",
    --   "degree_progress": { "credits_remaining": 24, "on_track": true },
    --   "recent_alerts": [...],
    --   "recent_grades": [...],
    --   "suggested_topics": ["discuss summer course plan", "review transfer credits"],
    --   "risk_context": { "score": 0.35, "level": "low", "trend": "improving" } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Case notes with structured and unstructured content
CREATE TABLE case_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    appointment_id  UUID REFERENCES appointments(id),
    note_type       VARCHAR(30) NOT NULL,
    subject         VARCHAR(255),
    body            TEXT NOT NULL,
    -- Structured tags and action items (JSONB for flexibility)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- { "tags": ["registration", "major_change"],
    --   "action_items": [
    --     {"task": "Submit major change form", "due": "2025-11-15", "status": "pending"},
    --     {"task": "Schedule follow-up", "due": "2025-12-01", "status": "pending"}
    --   ],
    --   "ai_generated": false,
    --   "topics_discussed": ["course_selection", "graduation_timeline"] }
    is_confidential BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_case_notes_metadata ON case_notes USING GIN (metadata);

-- Referrals with outcome tracking
CREATE TABLE referrals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    referred_by     UUID NOT NULL REFERENCES advisors(id),
    early_alert_id  UUID REFERENCES early_alerts(id),
    service_type    VARCHAR(50) NOT NULL,
    service_provider VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Flexible outcome tracking (varies by service type)
    outcome_data    JSONB,
    -- For tutoring: { "sessions_attended": 5, "subject": "Chemistry", "tutor_feedback": "..." }
    -- For mental_health: { "intake_completed": true, "ongoing": true }  (minimal for privacy)
    -- For financial_aid: { "emergency_fund_awarded": true, "amount": 500 }
    referred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    outcome_at      TIMESTAMPTZ
);
```

### Transfer Credit Articulation (Hybrid)

```sql
-- Institution-level transfer equivalency rules (JSONB for complex mappings)
CREATE TABLE transfer_equivalency_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    source_institution_name VARCHAR(255) NOT NULL,
    source_institution_id VARCHAR(20),          -- IPEDS or FICE code
    -- Equivalency mappings as a JSONB array
    mappings        JSONB NOT NULL,
    -- [
    --   {
    --     "source_course": {"code": "BIO 101", "title": "Intro Biology", "credits": 4},
    --     "equivalent_course_id": "uuid-bio110",
    --     "equivalent_subject": "BIO", "equivalent_number": "110",
    --     "credit_adjustment": null,
    --     "effective_from": "2020-08-01", "effective_to": null,
    --     "notes": "Lab component included"
    --   },
    --   ...
    -- ]
    last_reviewed   DATE,
    reviewed_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transfer_rules_source ON transfer_equivalency_rules 
    USING GIN (mappings jsonb_path_ops);

-- Individual student transfer credits
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
    -- AI-suggested equivalency (if applicable)
    ai_suggestion   JSONB,
    -- { "suggested_course_id": "uuid", "confidence": 0.92,
    --   "reasoning": "Course description match 94%, credit hours match, prerequisite alignment" }
    evaluated_by    UUID REFERENCES advisors(id),
    evaluated_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### SIS Integration Layer

```sql
-- Track integration state per institution per system
CREATE TABLE integration_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    system_type     VARCHAR(30) NOT NULL,       -- sis, lms, financial_aid
    system_name     VARCHAR(50) NOT NULL,       -- banner, colleague, canvas, workday
    -- Connection configuration (encrypted at rest)
    config          JSONB NOT NULL,
    -- { "api_base_url": "...", "auth_type": "oauth2",
    --   "sync_frequency": "15min", "entities": ["enrollment", "grades", "holds"] }
    status          VARCHAR(20) NOT NULL DEFAULT 'inactive',
    last_sync_at    TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    sync_stats      JSONB,
    -- { "records_processed": 1250, "errors": 3, "duration_seconds": 45 }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Sync event log
CREATE TABLE sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES integration_connections(id),
    sync_type       VARCHAR(20) NOT NULL,       -- full, incremental
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL,
    records_processed INTEGER DEFAULT 0,
    records_created INTEGER DEFAULT 0,
    records_updated INTEGER DEFAULT 0,
    records_errored INTEGER DEFAULT 0,
    error_details   JSONB                       -- array of error records
);
```

---

## Entity Relationship Summary

```
Institution 1--* Student (relational + custom_attributes JSONB)
Institution 1--* Course (relational + catalog_data JSONB)
Institution 1--* Program
Institution 1--* AcademicTerm
Institution 1--* IntegrationConnection

Program + CatalogYear --> DegreeRequirementSet (requirements as JSONB tree)

Student 1--* Enrollment (relational + sis_sync JSONB)
Student 1--* TransferCredit (relational + ai_suggestion JSONB)
Student 1--* EarlyAlert (relational + trigger_details JSONB + resolution JSONB)
Student 1--* RiskScore (relational + explanation JSONB)
Student 1--* Appointment (relational + ai_briefing JSONB)
Student 1--* CaseNote (relational + metadata JSONB)
Student 1--* Referral (relational + outcome_data JSONB)

Advisor 1--* CaseloadAssignment --* Student
```

---

## Pros

- **Best of both worlds**: Critical transactional data (enrollments, grades, student identity) gets full relational integrity with foreign keys and constraints. Variable data (requirement rules, AI outputs, custom attributes, integration payloads) gets schema flexibility without migrations.
- **Single database engine**: PostgreSQL handles both relational and document workloads. No need to operate, synchronise, or pay for a separate document database. Reduces operational complexity significantly.
- **Institution customisation without schema changes**: Each institution can define custom student attributes, unique requirement structures, and specialised alert triggers through JSONB without requiring DDL changes. This is critical for a multi-tenant SaaS serving diverse institutions.
- **Degree requirement flexibility**: The JSONB requirement tree can represent any degree structure (nested AND/OR groups, attribute-based filters, credit buckets, GPA sub-requirements, course lists) without a recursive relational schema. New requirement types can be added without migrations.
- **AI-friendly**: Risk score explanations, AI-generated briefings, and suggested transfer equivalencies are naturally semi-structured. JSONB columns store these directly without awkward relational mappings.
- **GIN-indexed queryability**: JSONB columns with GIN indexes support efficient queries like "find all students with custom_attributes containing veteran_status = true" or "find all requirement sets containing a course with subject MATH".
- **Incremental adoption**: Teams can start with a fully normalised model and selectively introduce JSONB columns as flexibility needs emerge. No big-bang architectural change required.

---

## Cons

- **No referential integrity for JSONB contents**: Course IDs referenced inside the JSONB requirement tree are not enforced by foreign keys. A deleted course could leave dangling references in requirement documents. Application-level validation must compensate.
- **Schema drift risk**: Without database-level constraints, JSONB documents can drift from expected shapes over time. Rigorous application-level JSON Schema validation and version tracking are essential.
- **Query complexity**: Queries that join relational columns with JSONB path expressions can be harder to write, read, and optimise. Developers need familiarity with PostgreSQL's JSONB operators (`->`, `->>`, `@>`, `jsonb_path_query`).
- **Reporting challenges**: Analytics queries that need to aggregate over JSONB fields (e.g., "count students by custom attribute X across all institutions") are slower and more complex than equivalent queries on normalised columns.
- **Indexing overhead**: GIN indexes on large JSONB columns consume significant storage and slow down writes. Careful index design is needed to balance read performance with write throughput.
- **Testing complexity**: The same codebase must handle both relational and document-style data. Test suites must validate JSONB schema conformance, GIN index usage, and the interaction between relational joins and JSONB filters.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with JSONB, GIN indexes, and RLS |
| Schema validation | JSON Schema (ajv library in Node.js, jsonschema in Python) at application layer |
| ORM | Drizzle ORM or Prisma with custom JSONB column support; or Kysely for type-safe raw SQL |
| Degree audit engine | Custom rule evaluator operating on the JSONB requirement tree with student enrollment data |
| Search | PostgreSQL full-text search (tsvector) + JSONB GIN indexes; Elasticsearch/Meilisearch for advanced search |
| Caching | Redis for frequently accessed student profiles and audit results |
| Migrations | Flyway or dbmate for relational schema; application-level JSONB schema versioning |

---

## Migration and Scaling Considerations

- **JSONB schema versioning**: Include a `schema_version` field in every JSONB-heavy table. When the application expects a new document shape, migrate documents lazily (on read) or eagerly (batch job) to the new version. Maintain backward-compatible readers for older versions.
- **Partial normalisation over time**: If a JSONB field is queried frequently enough, extract it to a normalised column. For example, if `custom_attributes.veteran_status` is used in every equity report, add a `veteran_status` column to the students table. The JSONB column remains for less common attributes.
- **Multi-tenancy**: Use `institution_id` on all tables with RLS policies. JSONB columns naturally accommodate per-tenant schema variations without separate table structures.
- **Degree requirement versioning**: Each published requirement set is immutable (new version = new row). Students are linked to the requirement set version in effect at their catalog year. Historical audits always use the published version, not the current draft.
- **SIS data mapping**: The integration layer's JSONB config allows per-institution field mapping without code changes. A Banner integration might map `SPRIDEN_PIDM` to `sis_student_id`, while a Workday integration maps `Worker_ID`. Both use the same table structure.
- **Performance monitoring**: Use `pg_stat_user_tables` and `pg_stat_user_indexes` to monitor GIN index usage and bloat. Periodically REINDEX GIN indexes on high-write tables. Consider partial GIN indexes on frequently filtered JSONB paths.
- **Backup and recovery**: Standard PostgreSQL backup strategies (pg_basebackup, WAL archiving) cover both relational and JSONB data. No separate backup infrastructure needed.
