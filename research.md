# Project 427 – Student Advising Platform

_Research date: 2026-05-02_

---

## 1. Problem Statement

Student retention and graduation rates are among the most important operational and reputational metrics for higher education institutions. Research consistently shows that students who receive timely, accurate academic advising are significantly more likely to persist to graduation. Yet traditional advising models are resource-constrained: advisor-to-student ratios at many institutions exceed 1:500, and reactive advising (students seek help only in crisis) means that early intervention opportunities are missed until a student's situation has deteriorated significantly.

The core information problem is that a student's academic trajectory—current progress toward degree requirements, standing in individual courses, enrollment in the right sequence of courses, transfer credit evaluation, and eligibility for graduation—is scattered across multiple systems (SIS, LMS, financial aid, transfer articulation engine, advisor notes) that do not communicate well with each other. An advisor trying to understand a student's full picture must manually assemble data from several platforms, making proactive monitoring of an entire caseload practically impossible.

Simultaneously, early alert systems that could surface at-risk students (based on grade trends, attendance, or missed assignments) remain immature or siloed at many institutions, and referral pathways to supplemental support services (tutoring, mental health, career services) are informal and poorly tracked.

---

## 2. Existing Landscape

- **Ellucian Degree Works** – The most widely deployed degree audit and academic planning solution in the US, serving approximately 3,000 customers across 50 countries and over 21 million students as part of Ellucian's broader higher education ERP ecosystem. Degree Works provides audit compliance checking, what-if analysis, and transfer articulation, and is deeply embedded in institutions running Banner or Colleague.
- **Stellic** – A modern degree management platform designed to transform the holistic student experience. Stellic offers advisor intervention tools when students fall behind, more efficient scheduling sessions, and transfer rule management, with an emphasis on UX that traditional audit tools lack.
- **Salesforce Agentforce Education** – Salesforce's higher education CRM that combines AI, analytics, and SIS functionality in a unified platform. Positioned for large institutions seeking enterprise-grade case management, advising workflows, and cross-departmental coordination.
- **EAB Navigate** – (formerly known as SSC Campus) A widely adopted student success platform that combines predictive analytics, appointment scheduling, case notes, and early alert workflows. Navigate is positioned specifically around retention outcomes and is used at hundreds of institutions.
- **Civitas Learning** – Focuses on predictive analytics for student success, using institution-specific historical data to build risk models and identify students who need outreach before grades deteriorate.
- **Advisor.AI** – A newer entrant applying AI to advising workflows, with its Pathways product using an evidence-based four-stage framework to guide learner engagement and personalized academic planning.
- **uAchieve Planner (CollegeSource)** – An academic planning tool that leverages existing audit data to generate personalized term-by-term recommendations leading to graduation.
- **Workday Student Advising** – The advising module within the Workday Student SIS, offering degree audit, planning, and advisor collaboration tools for institutions adopting the Workday ecosystem.

---

## 3. Key Functional Requirements

A competitive Student Advising Platform must deliver:

1. **Degree audit** – Real-time checking of a student's completed and in-progress coursework against degree requirements, generating a compliant/deficient report by requirement category, with support for multiple active degree programs, minors, and concentrations.
2. **What-if analysis** – Tools enabling advisors and students to model how a change in major, minor, or catalog year would affect degree progress, without committing the change to the official record.
3. **Academic planning** – Term-by-term course planning tools that apply degree audit logic to propose graduation pathways, flag prerequisite conflicts, and account for course availability constraints (not all courses offered every term).
4. **Transfer credit evaluation** – Automated or semi-automated mapping of transfer credits from other institutions against the receiving institution's course equivalency rules, with exception workflow routing for unevaluated courses.
5. **Early alert system** – Configurable triggers for at-risk signals (grade below threshold, excessive absences recorded in LMS, missing assignments, financial hold, no advising contact in N days) that generate advisor tasks or automated outreach to students.
6. **Appointment scheduling and case notes** – Integrated advisor scheduling with calendar sync, session preparation summaries (student context pulled from all connected systems), and structured case note entry linked to the student's advising record.
7. **Caseload management** – Advisor dashboards showing active caseload, pending early alerts, outstanding tasks, and students approaching critical milestones (probation deadlines, graduation application windows), with workload balancing tools.
8. **Referral tracking** – Structured referral workflows to supplemental services (tutoring, writing center, mental health, disability services, financial aid), with outcome tracking to close the referral loop.
9. **Student self-service** – A student-facing portal where students can view their degree progress, plan upcoming terms, check graduation eligibility, schedule advising appointments, and complete assigned intake forms.
10. **Analytics and outcomes reporting** – Institution-level reporting on retention rates, time to graduation, advising contact frequency, early alert response rates, and disaggregated equity metrics by demographic subgroup.

---

## 4. Technical Challenges

- **SIS integration depth** – Degree audit logic requires deep, real-time integration with the institution's Student Information System (Banner, Colleague, PeopleSoft, Workday). Enrollment, grade posting, and hold data must be current to avoid giving students inaccurate audit results. Legacy SIS architectures often expose limited real-time APIs.
- **Degree requirement modeling** – Institutional catalog requirements are extraordinarily complex: exception rules, substitution logic, residency requirements, GPA thresholds for majors, transfer credit caps, and co-requisites make academic rule authoring and maintenance one of the hardest problems in the domain. Small catalog errors produce incorrect audits that erode advisor trust and student confidence.
- **Predictive model validity** – Early alert risk models trained on institution-specific historical data perform well within that institution but require re-training and validation for each new customer deployment. Generic risk models trained on aggregate data are less actionable. Model fairness—avoiding disparate impact on students from historically marginalized groups—requires ongoing monitoring.
- **Advisor adoption** – If advisors perceive the platform as surveillance or as generating excessive workload (false-positive alert fatigue), adoption suffers and the intervention value is lost. UX design must prioritize advisor efficiency and alert precision.
- **Data privacy** – Student academic records are protected under FERPA. Any analytics or risk modeling using student data requires appropriate institutional oversight and, for certain disclosures, student consent.
- **Multi-system orchestration** – A complete advising picture requires integrating SIS, LMS (Canvas, Blackboard), financial aid, library systems, and student services platforms—each with different APIs, update frequencies, and data governance policies.

---

## 5. Market Opportunity

US higher education institutions enroll approximately 19 million students and spend heavily on retention initiatives because the financial impact of student attrition is severe—estimated at $16.5 billion annually in lost tuition revenue across the sector. Federal accountability metrics (graduation rates, transfer outcomes) create institutional pressure to demonstrate advising effectiveness.

The student success technology market, inclusive of advising, early alert, and degree planning, was estimated at approximately $2 billion in 2025 and is growing at 12–15% annually, driven by intensifying competition for students, demographic enrollment declines in traditional-age cohorts (requiring institutions to improve retention of enrolled students), and growing adoption of online and hybrid programs that extend the need for digital advising tools to non-traditional student populations. Community colleges, where retention challenges are most acute, represent a large and relatively under-served segment.

---

## Sources

- [Stellic | Degree Management and Student Success Platform](https://www.stellic.com/)
- [Unifying Campus Technology Solutions – Ellucian](https://www.ellucian.com/)
- [Degree Audit & Planning Software – Ellucian](https://www.ellucian.com/solutions/ellucian-degree-audit-planning)
- [20 Best Academic Advising Software for 2026 – Research.com](https://research.com/software/best-academic-advising-software)
- [Best Academic Advising Software: User Reviews from April 2026 – G2](https://www.g2.com/categories/academic-advising)
- [Advisor.AI | Transform Student Experience](https://joinadvisorai.com/)
- [Academic Advising and Planning Software – Workday](https://www.workday.com/en-us/products/student/advising.html)
- [FlightPath Academics | Advising and Student Success Software](https://flightpathacademics.com/)
