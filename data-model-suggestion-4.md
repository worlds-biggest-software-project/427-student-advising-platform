# Data Model Suggestion 4: Hybrid Relational + Knowledge Graph (PostgreSQL + Neo4j)

> Project: Student Advising Platform (427) | Generated: 2026-05-25

---

## Summary

A polyglot persistence architecture that combines PostgreSQL for transactional student records and advising workflows with a Neo4j property graph database for the academic knowledge domain: course prerequisites, degree requirement pathways, transfer credit equivalencies, and AI-powered pathway optimisation. This approach recognises that the academic catalog and degree planning domain is fundamentally a graph problem, where courses, requirements, prerequisites, and student progress form a densely connected network that relational databases handle awkwardly but graph databases handle naturally.

The key insight is that degree audit, what-if analysis, prerequisite validation, academic plan generation, and credential discovery are all graph traversal operations. Finding whether a student satisfies a complex nested requirement tree, detecting prerequisite conflicts in a multi-term plan, or discovering unrealised graduation eligibility across all programs --- these are path-finding and pattern-matching queries that graph databases execute orders of magnitude more efficiently than recursive SQL CTEs.

Research from institutions like Rice University and publications on curriculum graphs (e.g., the CAPIRE model) confirm that modelling curricula as directed graphs of prerequisite relationships enables structural feature engineering for student outcome prediction, bottleneck course identification, and personalised pathway generation.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  (API Gateway, Auth, Business Logic)                     │
└──────────┬──────────────────────────────┬────────────────┘
           │                              │
           ▼                              ▼
┌──────────────────────┐    ┌──────────────────────────────┐
│     PostgreSQL       │    │          Neo4j               │
│  (System of Record)  │    │   (Academic Knowledge Graph) │
│                      │    │                              │
│ • Students           │    │ • Course nodes               │
│ • Enrollments        │    │ • Prerequisite edges         │
│ • Appointments       │    │ • Requirement tree           │
│ • Case notes         │    │ • Program pathway graph      │
│ • Early alerts       │    │ • Transfer equivalency edges │
│ • Risk scores        │    │ • Student progress overlay   │
│ • Referrals          │    │ • Corequisite relationships  │
│ • Users / Auth       │    │ • Course attribute tags      │
│ • Integration state  │    │                              │
└──────────────────────┘    └──────────────────────────────┘
           │                              │
           └──────────┬───────────────────┘
                      ▼
           ┌──────────────────┐
           │   Sync Service   │
           │ (CDC / Event Bus)│
           └──────────────────┘
```

---

## PostgreSQL: Transactional Data Model

The PostgreSQL schema handles all operational, transactional, and PII-bearing data. This follows standard normalised patterns similar to Suggestion 1, so only key tables are shown.

```sql
-- Students (system of record for PII)
CREATE TABLE students (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    sis_student_id  VARCHAR(50) NOT NULL,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    date_of_birth   DATE,
    gender          VARCHAR(20),
    ethnicity       VARCHAR(50),
    first_generation BOOLEAN DEFAULT FALSE,
    pell_recipient  BOOLEAN DEFAULT FALSE,
    graph_node_id   VARCHAR(100),              -- reference to Neo4j node for this student
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (institution_id, sis_student_id)
);

-- Enrollments (system of record)
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
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    UNIQUE (student_id, section_id)
);

-- Early alerts, appointments, case notes, referrals, risk scores
-- (same structure as Suggestion 1 — normalised relational tables)

-- Advising appointments with AI briefing
CREATE TABLE appointments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id      UUID NOT NULL REFERENCES students(id),
    advisor_id      UUID NOT NULL REFERENCES advisors(id),
    scheduled_start TIMESTAMPTZ NOT NULL,
    scheduled_end   TIMESTAMPTZ NOT NULL,
    appointment_type VARCHAR(30) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    ai_briefing     JSONB,                     -- AI-generated context from graph + relational data
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Neo4j: Academic Knowledge Graph

The graph database models the academic domain as an interconnected network of courses, requirements, programs, and student progress.

### Node Types

```cypher
// Course node
CREATE (c:Course {
    courseId: 'uuid-cs201',
    institutionId: 'uuid-inst-1',
    subjectCode: 'CS',
    courseNumber: '201',
    title: 'Data Structures',
    creditsMin: 3.0,
    creditsMax: 3.0,
    isActive: true,
    attributes: ['writing_intensive'],
    typicallyOffered: ['fall', 'spring']
})

// Program node
CREATE (p:Program {
    programId: 'uuid-prog-cs-bs',
    institutionId: 'uuid-inst-1',
    code: 'CS-BS',
    name: 'B.S. Computer Science',
    programType: 'major',
    degreeLevel: 'bachelor'
})

// Requirement node (part of the degree requirement tree)
CREATE (r:Requirement {
    requirementId: 'uuid-req-cs-core',
    programId: 'uuid-prog-cs-bs',
    catalogYear: '2025-2026',
    name: 'Computer Science Core',
    requirementType: 'group',
    logicOperator: 'AND',
    minCredits: 36,
    minGpa: 2.5,
    sortOrder: 2
})

// Term node
CREATE (t:Term {
    termId: 'uuid-fall2025',
    institutionId: 'uuid-inst-1',
    name: 'Fall 2025',
    termType: 'fall',
    startDate: '2025-08-25',
    endDate: '2025-12-15'
})

// Student node (lightweight — PII stays in PostgreSQL)
CREATE (s:Student {
    studentId: 'uuid-student-1',        -- matches PostgreSQL students.id
    institutionId: 'uuid-inst-1',
    sisStudentId: 'S12345',
    currentGpa: 3.45,
    totalCredits: 75,
    riskLevel: 'low'
    // NO PII (name, email, DOB) stored in Neo4j
})
```

### Relationship Types

```cypher
// Course prerequisite relationships
(cs201:Course)-[:REQUIRES {minGrade: 'C', isCorequisite: false}]->(cs101:Course)
(cs201:Course)-[:REQUIRES {minGrade: 'C', isCorequisite: false}]->(math151:Course)

// Corequisite (must be taken concurrently)
(chem201:Course)-[:COREQUISITE]->(chem201L:Course)

// Program contains requirement groups (tree structure)
(program:Program)-[:HAS_REQUIREMENT {catalogYear: '2025-2026'}]->(genEd:Requirement)
(program:Program)-[:HAS_REQUIREMENT {catalogYear: '2025-2026'}]->(core:Requirement)
(genEd:Requirement)-[:HAS_CHILD {sortOrder: 1}]->(writingReq:Requirement)
(genEd:Requirement)-[:HAS_CHILD {sortOrder: 2}]->(mathReq:Requirement)

// Requirement can be satisfied by specific courses
(writingReq:Requirement)-[:SATISFIED_BY {isRequired: true}]->(eng101:Course)
(writingReq:Requirement)-[:SATISFIED_BY {isRequired: true}]->(eng102:Course)
(mathReq:Requirement)-[:SATISFIED_BY {isRequired: false}]->(math151:Course)
(mathReq:Requirement)-[:SATISFIED_BY {isRequired: false}]->(stat200:Course)

// Student enrollment and completion
(student:Student)-[:COMPLETED {
    grade: 'A',
    gradePoints: 4.0,
    creditsEarned: 3,
    termId: 'uuid-fall2024',
    completedAt: '2024-12-15'
}]->(cs101:Course)

(student:Student)-[:ENROLLED {
    termId: 'uuid-spring2025',
    status: 'in_progress'
}]->(cs201:Course)

(student:Student)-[:PLANNED {
    termId: 'uuid-fall2025',
    planId: 'uuid-plan-1'
}]->(cs301:Course)

// Transfer credit equivalency
(extCourse:ExternalCourse)-[:EQUIVALENT_TO {
    confidence: 0.95,
    evaluatedBy: 'uuid-advisor-1',
    effectiveFrom: '2020-08-01'
}]->(cs101:Course)

// Student declared program
(student:Student)-[:DECLARED {
    catalogYear: '2025-2026',
    declaredDate: '2024-01-15',
    isPrimary: true
}]->(program:Program)

// Course offered in a term (availability graph)
(cs201:Course)-[:OFFERED_IN {
    sectionCount: 3,
    totalCapacity: 120,
    deliveryModes: ['in_person', 'online']
}]->(fall2025:Term)
```

---

## Graph-Powered Operations

### 1. Degree Audit (Requirement Satisfaction Check)

```cypher
// Find all unsatisfied requirements for a student in their declared program
MATCH (s:Student {studentId: $studentId})-[:DECLARED]->(p:Program)
MATCH (p)-[:HAS_REQUIREMENT]->(req:Requirement)
MATCH path = (req)-[:HAS_CHILD*0..5]->(leaf:Requirement)
MATCH (leaf)-[:SATISFIED_BY]->(course:Course)
OPTIONAL MATCH (s)-[completed:COMPLETED]->(course)
WITH leaf, course, completed,
     CASE WHEN completed IS NOT NULL 
          AND (leaf.minGrade IS NULL OR completed.grade >= leaf.minGrade)
     THEN true ELSE false END AS satisfied
WITH leaf, 
     collect({course: course, completed: completed IS NOT NULL, satisfied: satisfied}) AS courses,
     count(CASE WHEN satisfied THEN 1 END) AS satisfiedCount
RETURN leaf.name, leaf.requirementType, leaf.logicOperator, 
       satisfiedCount, size(courses) AS totalCourses,
       courses
ORDER BY leaf.sortOrder
```

### 2. Prerequisite Conflict Detection (Plan Validation)

```cypher
// Check if a planned course schedule has prerequisite violations
MATCH (s:Student {studentId: $studentId})-[:PLANNED {termId: $termId}]->(planned:Course)
MATCH (planned)-[:REQUIRES]->(prereq:Course)
WHERE NOT EXISTS {
    MATCH (s)-[:COMPLETED]->(prereq)
}
AND NOT EXISTS {
    MATCH (s)-[:ENROLLED]->(prereq)
    WHERE s.enrolledTermEndDate < $termStartDate
}
RETURN planned.subjectCode + ' ' + planned.courseNumber AS course,
       prereq.subjectCode + ' ' + prereq.courseNumber AS missingPrerequisite
```

### 3. What-If Analysis (Hypothetical Program Change)

```cypher
// Model a major change: what would the student's degree progress look like
// under a different program without modifying any real data?
MATCH (s:Student {studentId: $studentId})-[completed:COMPLETED]->(course:Course)
MATCH (newProgram:Program {programId: $newProgramId})
      -[:HAS_REQUIREMENT {catalogYear: $catalogYear}]->(req:Requirement)
MATCH path = (req)-[:HAS_CHILD*0..5]->(leaf:Requirement)
MATCH (leaf)-[:SATISFIED_BY]->(reqCourse:Course)
WITH s, leaf, reqCourse,
     EXISTS { MATCH (s)-[:COMPLETED]->(reqCourse) } AS isSatisfied
WITH leaf,
     count(CASE WHEN isSatisfied THEN 1 END) AS met,
     count(reqCourse) AS total
RETURN leaf.name, met, total,
       CASE WHEN met >= total THEN 'satisfied' ELSE 'incomplete' END AS status
```

### 4. Credential Discovery (Unrealised Graduation Eligibility)

```cypher
// Find programs where the student has unknowingly satisfied all or nearly all requirements
MATCH (s:Student {studentId: $studentId})
MATCH (p:Program {institutionId: s.institutionId, isActive: true})
WHERE NOT EXISTS { MATCH (s)-[:DECLARED]->(p) }
MATCH (p)-[:HAS_REQUIREMENT]->(req:Requirement)
MATCH path = (req)-[:HAS_CHILD*0..5]->(leaf:Requirement)
MATCH (leaf)-[:SATISFIED_BY]->(reqCourse:Course)
WITH p, leaf, reqCourse,
     EXISTS { MATCH (s)-[:COMPLETED]->(reqCourse) } AS isSatisfied
WITH p,
     count(CASE WHEN isSatisfied THEN 1 END) AS totalMet,
     count(reqCourse) AS totalRequired
WITH p, totalMet, totalRequired,
     toFloat(totalMet) / totalRequired AS completionRatio
WHERE completionRatio >= 0.85  // At least 85% of requirements met
RETURN p.name, p.code, totalMet, totalRequired,
       round(completionRatio * 100, 1) AS percentComplete
ORDER BY completionRatio DESC
```

### 5. Optimal Graduation Pathway (AI-Assisted Planning)

```cypher
// Generate a graduation pathway: find remaining courses, respect prerequisites,
// optimise for course availability and shortest time to completion
MATCH (s:Student {studentId: $studentId})-[:DECLARED]->(p:Program)
MATCH (p)-[:HAS_REQUIREMENT]->()-[:HAS_CHILD*0..5]->(leaf:Requirement)
      -[:SATISFIED_BY]->(needed:Course)
WHERE NOT EXISTS { MATCH (s)-[:COMPLETED]->(needed) }
  AND NOT EXISTS { MATCH (s)-[:ENROLLED]->(needed) }

// Find prerequisite chains for remaining courses
OPTIONAL MATCH prereqChain = (needed)-[:REQUIRES*1..5]->(prereq:Course)
WHERE NOT EXISTS { MATCH (s)-[:COMPLETED]->(prereq) }

// Check course availability in upcoming terms
OPTIONAL MATCH (needed)-[:OFFERED_IN]->(futureTerm:Term)
WHERE futureTerm.startDate > date()

RETURN needed.subjectCode + ' ' + needed.courseNumber AS course,
       needed.title,
       needed.creditsMin AS credits,
       collect(DISTINCT prereq.subjectCode + ' ' + prereq.courseNumber) AS unmetPrereqs,
       collect(DISTINCT futureTerm.name) AS availableTerms,
       length(prereqChain) AS prereqDepth
ORDER BY prereqDepth DESC, needed.subjectCode, needed.courseNumber
```

### 6. Bottleneck Course Identification (Institutional Analytics)

```cypher
// Find courses that are prerequisites for the most other courses
// (these are institutional bottleneck courses where failures cascade)
MATCH (c:Course {institutionId: $institutionId})
MATCH (dependent:Course)-[:REQUIRES*1..]->(c)
WITH c, count(DISTINCT dependent) AS dependentCount
ORDER BY dependentCount DESC
LIMIT 20
RETURN c.subjectCode + ' ' + c.courseNumber AS course,
       c.title,
       dependentCount AS coursesBlocked
```

---

## Data Synchronisation

### PostgreSQL to Neo4j Sync

```
PostgreSQL (source of truth)  ──CDC──>  Sync Service  ──>  Neo4j (derived graph)

Events synced:
• New course created        → CREATE Course node
• Enrollment recorded       → CREATE ENROLLED relationship
• Grade posted              → Convert ENROLLED to COMPLETED relationship
• Requirement set published → Rebuild requirement subgraph
• Transfer credit evaluated → CREATE EQUIVALENT_TO relationship
• Program declared          → CREATE DECLARED relationship
```

```python
# Sync service pseudocode
class GraphSyncService:
    def on_enrollment_created(self, event):
        self.neo4j.run("""
            MATCH (s:Student {studentId: $studentId})
            MATCH (c:Course {courseId: $courseId})
            MERGE (s)-[e:ENROLLED {termId: $termId}]->(c)
            SET e.status = 'in_progress', e.enrolledAt = $enrolledAt
        """, **event)

    def on_grade_posted(self, event):
        self.neo4j.run("""
            MATCH (s:Student {studentId: $studentId})-[e:ENROLLED]->(c:Course {courseId: $courseId})
            DELETE e
            MERGE (s)-[comp:COMPLETED]->(c)
            SET comp.grade = $grade, comp.gradePoints = $gradePoints,
                comp.creditsEarned = $creditsEarned, comp.termId = $termId
        """, **event)

    def on_requirement_set_published(self, event):
        # Rebuild the entire requirement subgraph for this program + catalog year
        self.neo4j.run("MATCH (r:Requirement {programId: $programId, catalogYear: $catalogYear}) DETACH DELETE r", **event)
        self._build_requirement_graph(event['program_id'], event['catalog_year'], event['requirements'])
```

---

## Pros

- **Natural graph operations**: Degree audit, prerequisite validation, what-if analysis, credential discovery, and pathway generation are all graph traversal problems. Cypher queries express these operations in 5-10 lines vs. 100+ lines of recursive SQL with CTEs.
- **Performance for deep traversals**: Neo4j performs constant-time relationship traversals regardless of dataset size. A prerequisite chain 8 levels deep is resolved in milliseconds, whereas recursive SQL CTEs degrade with depth and dataset size.
- **AI-native architecture**: The knowledge graph provides structured context for LLM-powered advising agents. An MCP server can expose graph traversal results as tool outputs, enabling AI agents to answer complex multi-step questions ("What courses should I take next term if I drop CHEM 201 and add a minor in Data Science?") by querying the graph.
- **Curriculum analytics**: Graph centrality metrics (betweenness, PageRank) identify bottleneck courses, program overlap, and structural vulnerabilities in the curriculum --- analyses that are impractical in relational databases.
- **Visual comprehension**: Neo4j's built-in visualisation and tools like Neo4j Bloom allow advisors, registrars, and curriculum committees to visually explore and understand degree requirement structures, prerequisite networks, and student progress.
- **Transfer credit network**: Transfer equivalency rules across institutions form a natural bipartite graph. Graph queries can find transitive equivalencies (Course A at Institution X = Course B at Institution Y = Course C at Institution Z) that are difficult to discover in relational tables.
- **Separation of concerns**: PII and transactional data (FERPA-sensitive) stays in PostgreSQL with RLS and audit logging. The knowledge graph contains only academic structure and anonymised progress data.

---

## Cons

- **Operational complexity**: Running two database engines (PostgreSQL + Neo4j) doubles the operational burden: separate backup strategies, monitoring, scaling, security hardening, and on-call procedures. Institutional IT teams may lack Neo4j expertise.
- **Data synchronisation**: Keeping PostgreSQL and Neo4j consistent requires a reliable sync service. Failures, ordering issues, or lag in the sync pipeline can produce stale or inconsistent graph state. This is the single biggest operational risk.
- **Two query languages**: Developers must be proficient in both SQL and Cypher. Code reviews, debugging, and onboarding require familiarity with both paradigms.
- **Neo4j licensing**: Neo4j Community Edition is open source (GPL), but Enterprise Edition (required for clustering, role-based access control, and online backup) is commercially licensed. This may conflict with the platform's open-source positioning.
- **Write throughput**: Neo4j's write throughput is lower than PostgreSQL's for bulk operations. Term-start enrollment loads (50,000+ enrollments in a day) require batched graph writes or a temporary write queue.
- **FERPA data separation**: Ensuring no PII leaks into the graph database requires disciplined data governance. Student nodes must contain only IDs and aggregated metrics, never names, emails, or demographic details.
- **Graph model evolution**: Changing the graph schema (adding new relationship types, node labels, or properties) requires data migration scripts in Cypher, which lack the maturity of SQL migration tooling (no equivalent of Flyway or dbmate).

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Transactional database | PostgreSQL 16+ (system of record for all PII and transactional data) |
| Graph database | Neo4j 5.x Community Edition (consider Memgraph as open-source alternative) |
| Sync service | Debezium CDC from PostgreSQL WAL to Kafka, consumed by a graph sync worker |
| Graph query layer | Neo4j Bolt driver (official drivers for Python, Node.js, Java, Go) |
| API layer | Unified GraphQL API that resolves fields from PostgreSQL or Neo4j as appropriate |
| Visualisation | Neo4j Bloom or custom D3.js graph visualisation for advisor-facing UIs |
| AI integration | MCP server exposing Cypher query results as tool outputs for LLM agents |
| Monitoring | Neo4j Ops Manager + Prometheus metrics for graph database health |

---

## Migration and Scaling Considerations

- **Phased adoption**: Start with PostgreSQL-only (Suggestion 1 or 3) and introduce Neo4j in a later phase for degree audit and planning. The graph is a derived read model, not the system of record, so it can be introduced without migrating existing data.
- **Memgraph as alternative**: Memgraph is an open-source (BSL), Cypher-compatible graph database with lower operational overhead than Neo4j. It runs as an in-memory graph with WAL persistence and has no enterprise licensing tier. Consider it for cost-sensitive deployments.
- **Graph data volume**: The academic knowledge graph is relatively small (tens of thousands of courses, hundreds of programs, a few hundred thousand prerequisite and requirement relationships per institution). Even with student progress overlaid, the graph fits comfortably in memory on a single Neo4j instance.
- **Multi-tenancy in Neo4j**: Use a `institutionId` property on all nodes and filter by it in every query. Alternatively, use Neo4j's multi-database feature (Enterprise Edition) for tenant isolation, or deploy separate Memgraph instances per tenant.
- **Sync reliability**: Implement idempotent sync handlers with a checkpoint table in PostgreSQL tracking the last successfully synced event per entity type. On sync failure, replay from the last checkpoint. Include a full graph rebuild capability for disaster recovery.
- **Performance testing**: Benchmark Cypher queries with realistic data volumes (5,000 courses, 200 programs, 50,000 students with full enrollment history) to validate sub-second response times for degree audit and what-if analysis.
- **Fallback strategy**: If Neo4j proves too operationally complex, the degree audit and pathway logic can fall back to PostgreSQL with recursive CTEs and materialised views. The graph is an optimisation, not a hard dependency.
- **Graph-enhanced AI briefings**: The knowledge graph provides rich structured context for AI pre-meeting briefings. An LLM can receive the student's position in the degree requirement graph, remaining course prerequisites, and optimal pathway options as structured input, producing more accurate and actionable briefings than text-only context retrieval.
