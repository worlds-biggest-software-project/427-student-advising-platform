# Student Advising Platform — Feature & Functionality Survey

> Candidate #427 · Researched: 2026-05-07

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| EAB Navigate360 | Student success CRM + early alert | Commercial (SaaS) | https://eab.com/solutions/navigate360/ |
| Ellucian Degree Works | Degree audit + academic planning | Commercial (SaaS, ERP-bundled) | https://www.ellucian.com/solutions/ellucian-degree-audit-planning |
| Stellic | Degree management + advising | Commercial (SaaS) | https://www.stellic.com/ |
| Civitas Learning | Predictive analytics + student success | Commercial (SaaS) | https://www.civitaslearning.com/ |
| Salesforce Education Cloud | CRM + AI advising + degree planning | Commercial (SaaS) | https://www.salesforce.com/education/cloud/ |
| Advisor.AI | AI-native advising workflows | Commercial (SaaS) | https://joinadvisorai.com/ |
| CollegeSource uAchieve | Degree audit + transfer articulation | Commercial (SaaS) | https://collegesource.com/ |
| Workday Student | SIS-embedded advising + degree audit | Commercial (SaaS, SIS-bundled) | https://www.workday.com/en-us/products/student/advising.html |
| FlightPath Academics | Degree audit + advising (open source) | Open source (AGPL-3.0) + commercial | https://getflightpath.com/ |

---

## Feature Analysis by Solution

### EAB Navigate360

**Core features**
- Appointment scheduling and calendar management for advisors and students
- Early alert system: faculty flag at-risk students, alerts route to advisor task queues
- Student self-service portal: appointments, resource access, To-Do lists, Student Journeys
- Caseload management dashboard with workload metrics and pending alert tracking
- Predictive analytics for identifying at-risk students before mid-term
- AI-driven course recommendations based on academic progress
- Bulk outreach campaigns (email/SMS) to student cohorts
- Case notes and interaction history linked to student profiles
- Reporting and student success analytics dashboard for institutional monitoring

**Differentiating features**
- Network data effect: aggregate data from hundreds of partner institutions powers risk models
- "Student Journeys" structured advising sequence workflows
- Native community college CRM positioning — purpose-built for high caseload, low-resource environments

**UX patterns**
- Advisor-centric dashboard with filtered alert queues
- Staff and faculty early alert submission uses simplified, non-technical forms
- Mobile-friendly student portal for self-service

**Integration points**
- SIS integrations: Banner, Colleague, PeopleSoft, Workday (via data import/export and APIs)
- LMS integrations: Canvas, Brightspace, Blackboard (grade and attendance data)
- Supports LTI for LMS embed
- Bulk data export API for institution data consumers
- SSO via SAML and OIDC

**Known gaps**
- No real-time SIS sync; time-lapse in data posted to Navigate has been a consistent complaint
- Lacks modern generative AI capabilities (no LLM-powered agents, AI content generation, or chatbot advising)
- Degree audit and academic planning not natively included — institutions must integrate Degree Works or uAchieve separately
- Data privacy concern: EAB aggregates institutional data for cross-institution research; some institutions object to data use beyond their campus
- Standardised platform may not accommodate highly unique institutional workflows
- Requires institutions to build and maintain custom SIS connectors, which is technically intensive

**Licence / IP notes**
- Proprietary SaaS; no open-source component. Institutional data governance agreements required.

---

### Ellucian Degree Works

**Core features**
- Degree audit: real-time compliance checking against published degree requirements
- What-if analysis: model impact of major/minor/catalog year changes without committing to records
- Smart Plan: automated graduation pathway calculation, self-correcting for plan changes
- Transfer credit articulation and equivalency mapping against degree requirements
- Credential Discovery: real-time analysis comparing completed coursework to all available degree programs (declared and undeclared)
- Student-facing audit display: at-a-glance progress, mobile-responsive
- Advisor notes and exception/substitution workflow for manual overrides
- Integration with Banner and Colleague SIS at the data layer

**Differentiating features**
- Deepest legacy integration with Banner/Colleague SIS — works within the Ellucian ecosystem natively
- Credential Discovery surfaces unrealised graduation eligibility across all programs
- Over 40 years of accumulated degree audit rule sophistication

**UX patterns**
- Audit displayed as structured checklist by requirement category
- What-if worksheets allow side-by-side comparison of degree progress under different scenarios
- Smart Plan generates a recommended term-by-term plan automatically

**Integration points**
- Deep native integration with Ellucian Banner and Colleague SIS
- Ellucian Ethos (integration platform) provides REST/JSON APIs for non-Ellucian SIS connections
- SIS data sync for enrollment, grades, holds, and program of study

**Known gaps**
- Interface described as dated and complex; steep learning curve for new advisors
- Primarily a degree audit and planning tool — lacks early alert, scheduling, and caseload management out of the box
- Limited standalone value outside the Ellucian SIS ecosystem; non-Banner/Colleague institutions face complex integration
- No native AI-powered risk scoring or intervention workflows

**Licence / IP notes**
- Proprietary SaaS; part of Ellucian's ERP product suite. Licensing typically bundled with Banner/Colleague.

---

### Stellic

**Core features**
- Real-time degree audit connected directly to SIS with live enrollment, grade, and requirement data
- Multi-term academic planner with prerequisite conflict detection and course availability awareness
- Advisor intervention tools with analytics surfacing students falling behind
- Advisor notes, chat, and alert functionalities embedded in the platform
- Registration integration: students plan and register from the same interface (no portal switching)
- Unified platform for advisors, faculty, and registrars working from the same data
- Transfer rule management
- Institution-level analytics: program-level patterns, at-risk cohorts

**Differentiating features**
- Single platform combining degree audit, planning, and registration eliminates portal fragmentation
- Modern UX explicitly designed to reduce advisor burnout from administrative overload
- Positioning as a Degree Works replacement with better UX and modern SIS integration

**UX patterns**
- Clean, consumer-grade UI with progressive disclosure of complexity
- Student-centered design: visual progress indicators, roadmap views
- Automated transactional tasks freed up for meaningful advising conversations

**Integration points**
- SIS integrations via REST APIs supporting Banner, Workday, PeopleSoft, and others
- LMS integration for grade and engagement data

**Known gaps**
- Smaller market footprint than EAB Navigate or Ellucian; fewer third-party integration case studies available
- Early alert capabilities are present but less mature than dedicated early alert platforms like Navigate or Civitas
- No native predictive risk scoring beyond trend-based flags

**Licence / IP notes**
- Proprietary SaaS. No open-source component disclosed.

---

### Civitas Learning

**Core features**
- Institution-specific predictive risk models: weekly persistence and completion probability scores (0%–100%) for every enrolled student
- AI-powered adaptable analytics combining predictive and generative AI
- Advisor workflow tools: smart to-do lists, intervention recommendations, outreach automation
- Initiative Analysis with propensity score matching to evaluate program effectiveness
- SIS, LMS, CRM, and financial aid data integration for unified student risk picture
- Equity monitoring and disaggregated reporting by demographic subgroup
- Proactive outreach workflows triggered by risk score changes

**Differentiating features**
- Institution-specific model training (not generic cross-institution models) for higher prediction accuracy
- Propensity score matching for program evaluation is unusually sophisticated for the category
- Strongest emphasis on equity metrics and disaggregated outcomes reporting

**UX patterns**
- Advisor dashboard centred on risk-ranked student lists with recommended actions
- Generative AI used to summarise student context and recommend next steps

**Integration points**
- Data integrations with Banner, Colleague, Workday, PeopleSoft, Canvas, Blackboard, and Brightspace
- API-based data ingestion; hands-on implementation support included

**Known gaps**
- Focused narrowly on analytics and workflows; no degree audit, academic planning, or appointment scheduling
- Requires integration with a separate advising or degree audit platform to be fully functional
- Model re-training and fairness monitoring for each new institution is resource-intensive

**Licence / IP notes**
- Proprietary SaaS. No open-source component. Institutional data governance agreement required.

---

### Salesforce Education Cloud

**Core features**
- Advising Support Agent (AI): generates comprehensive student summaries, identifies best-fit resources, automates next steps
- Advising Assistant: AI agent synthesises academic data to prepare advisors for student meetings and proactively alerts to struggling students
- Intelligent Degree Planning: personalised degree plans with real-time progress display and customisable Pathway Templates
- Student Goals Agent: AI-powered career and life goal recommendations
- Skills Generator: AI-powered curriculum design for workforce readiness
- Enterprise CRM case management: advising records, appointments, referrals, and case notes at institutional scale
- Cross-departmental coordination tools (financial aid, student affairs, career services)
- Agentforce AI framework: configurable AI agents for education workflows

**Differentiating features**
- Full Salesforce platform extensibility: custom objects, flows, and automations applicable to education
- Agentforce AI agents are the most sophisticated AI advising automation in the market as of 2026
- Integrates academic advising with CRM enrollment management and career services in a single platform

**UX patterns**
- Enterprise Salesforce UX: highly configurable, but steep learning curve for non-Salesforce institutions
- AI agent interactions presented in natural language within advisor and student interfaces

**Integration points**
- REST and SOAP APIs (standard Salesforce API surface)
- Education Cloud objects exposed via Salesforce API (Appointment, Alert, SuccessPlan, PersonEducation, AcademicTermEnrollment)
- Native data integrations with Banner, Workday, and other SIS via AppExchange connectors or MuleSoft
- LTI support for LMS integration
- SSO via SAML, OAuth 2.0, OIDC

**Known gaps**
- Very high implementation cost and complexity; requires Salesforce expertise or consultants
- Degree audit is a newer, less mature feature compared to Degree Works or uAchieve
- Early alert is present but less operationally focused than EAB Navigate
- Not purpose-built for advising; academic advising workflows are a module within a broader CRM

**Licence / IP notes**
- Proprietary SaaS; Salesforce licensing model. Education Cloud is a vertically packaged product layer on the Salesforce platform.

---

### Advisor.AI

**Core features**
- Automated academic plan generation: transforms student interests and goals into structured multi-year plans in minutes
- Integrated career, course, and resource planning within a single pathway
- Customisable milestones: students visualise progress, track achievements, and set new goals
- Degree planning assistance: instant clarity on course options, requirements, credit policies, and registration steps
- Four-stage advising framework (Explore, Prepare, Connect, Optimise) embedded in workflows
- Appreciative advising methodology integration for student-centred conversations
- Scaling capabilities for large caseloads (targeting 10,000+ advisors across 10+ states by 2026)

**Differentiating features**
- AI-native platform built from the ground up around advising workflows (not an adaptation of a CRM or SIS)
- Evidence-based Appreciative Advising theoretical framework embedded as a product feature
- Fastest plan generation in the category: structured multi-year pathways generated in minutes from initial intake

**UX patterns**
- Student-friendly visual roadmap for degree/career pathway
- Advisor-facing tools for managing and customising generated plans at scale
- Progressive four-stage workflow guides students through exploration to goal achievement

**Integration points**
- SIS integration for degree requirement data and enrollment status
- Career services system integration for job and internship alignment
- Integration details are limited in public documentation

**Known gaps**
- Relatively new platform with limited publicly available integration documentation
- Early alert and predictive risk scoring not prominently featured
- Degree audit compliance checking depth unknown compared to Degree Works or uAchieve
- Analytics and institutional reporting capabilities not well documented publicly

**Licence / IP notes**
- Proprietary SaaS. Proprietary advising framework; no open-source component.

---

### CollegeSource uAchieve

**Core features**
- Degree audit: fully hosted, cloud-based compliance checking against degree requirements
- Transfer articulation: storage of course and program-specific equivalencies, statewide transfer rules, and articulation agreements
- Transfer course handling: grade and credit translation from transfer institution values to home institution values
- Exception handling, multi-major/minor, and manual substitution workflows
- Degree Discovery: identifies unrealised graduation eligibility across all programs for enrolled students
- Student-facing modern display with mobile-responsive at-a-glance progress
- Transferology integration: prospective student transfer credit preview tool
- Real-time web-based access for students and advisors

**Differentiating features**
- Industry-leading transfer articulation engine — purpose-built for complex multi-institution transfer scenarios
- Transferology network: prospective students can preview transfer credit equivalencies before applying
- 40+ years of degree audit rule engine development

**UX patterns**
- Structured requirement checklist with at-a-glance completion status
- Mobile-responsive modern display updated from legacy interfaces

**Integration points**
- SIS integrations via standard data feeds (Banner, PeopleSoft, and others)
- Transferology external-facing tool for prospective student transfer previews
- APIs for data exchange with institutional systems

**Known gaps**
- Focused narrowly on degree audit and transfer articulation; lacks early alert, scheduling, caseload tools
- Requires integration with a student success or advising workflow platform for complete advising functionality
- AI capabilities not a prominent feature of the platform

**Licence / IP notes**
- Proprietary SaaS. No open-source component.

---

### Workday Student

**Core features**
- Academic Progress Report (degree audit): real-time requirement tracking updated as students register and complete courses
- Academic Plans: interactive multi-term graduation planning tool with course eligibility and availability awareness
- Academics Hub: unified student portal for planning, registration, hold resolution, and advisor contact
- Real-time requirement status: requirements update to "In Progress" on registration and "Satisfied" on course completion
- Advisor collaboration tools embedded within the Workday SIS
- Student holds management and resolution workflows

**Differentiating features**
- Tight SIS-native integration: degree audit, planning, and registration all operate on the same underlying data model with no synchronisation lag
- Single-system architecture eliminates data duplication across advising and SIS modules

**UX patterns**
- Workday consumer-grade UX with consistent design across HR, Finance, and Student modules
- Students plan and register within the same interface as all other Workday Student interactions

**Integration points**
- Native to Workday Student SIS — no external integration required for core advising and degree audit
- Workday REST APIs and report-as-a-service for external system integration
- SSO via SAML and OIDC aligned with Workday Identity

**Known gaps**
- Only available to institutions using Workday Student as their SIS; no standalone deployment
- Advising features are less mature than purpose-built advising platforms
- Early alert system is limited compared to EAB Navigate or Civitas
- Predictive analytics and AI-powered risk scoring not a prominent feature

**Licence / IP notes**
- Proprietary SaaS; Workday Student licensing. Advising module bundled with SIS.

---

### FlightPath Academics / FlightPath (Open Source)

**Core features**
- Open-source, web-based degree audit and academic advising platform
- Degree audit with prerequisite, minimum grade, elective, and co-requisite rule handling
- Transfer credit equivalency mapping with advisor substitution and exception support
- Semester-by-semester course plan generation tailored to student goals and transfer credits
- AI-driven early alerts and analytics for at-risk students (added functionality)
- SMS and email communication tools built into the platform
- Prospective student transfer credit preview (similar to Transferology)

**Differentiating features**
- Only open-source, self-hostable option in the category with full degree audit capability
- Complete control over data governance and hosting for privacy-sensitive institutions
- Commercial support available through FlightPath Academics alongside the open-source core

**UX patterns**
- Web-based interface accessible to advisors and students
- Advisor workflow-centric design for equivalency and exception management

**Integration points**
- SIS integration via data import/export
- Open-source codebase allows custom integration development
- Email and SMS outreach built in

**Known gaps**
- Community size and active maintenance pace are risks compared to commercial vendors
- AI and predictive analytics capabilities are limited compared to Civitas or EAB
- UX less polished than commercial SaaS alternatives
- Limited enterprise support and SLA guarantees without paid commercial contract

**Licence / IP notes**
- Open source: AGPL-3.0 licence for the core platform (getflightpath.com). FlightPath Academics (flightpathacademics.com) is a commercial managed service layer on top of the open-source core. AGPL copyleft terms apply to any modifications deployed to users.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Degree audit: real-time compliance checking of coursework against declared degree requirements
- What-if analysis: modelling impact of major, minor, or catalog year changes
- Academic planning: term-by-term course planning with prerequisite and availability awareness
- Student self-service portal: degree progress, plan view, appointment booking
- Appointment scheduling: advisor calendar management with student-facing booking
- Advisor case notes: structured interaction logging linked to student records
- SIS integration: live data feeds for enrollment, grades, holds, and program of study
- Early alert: faculty-submitted flags and automated at-risk triggers routed to advisor queues
- SSO authentication: SAML or OIDC integration with institutional identity providers
- Caseload management dashboard: advisor-facing view of assigned students with alert status

### Differentiating Features
- AI-generated advising summaries and meeting preparation (Salesforce Agentforce, Civitas)
- Generative AI-powered plan creation from student interests in minutes (Advisor.AI)
- Institution-specific predictive risk models with weekly re-scoring (Civitas)
- Propensity score matching for programme effectiveness evaluation (Civitas)
- Credential Discovery — unrealised graduation eligibility surfaced across all programmes (Ellucian, uAchieve)
- Network effect risk models trained on cross-institution data (EAB Navigate)
- Native SIS-embedded advising eliminating synchronisation lag (Workday Student)
- Open-source, self-hostable architecture (FlightPath)
- Integrated registration within the same planning interface (Stellic)
- Appreciative Advising evidence-based framework baked into UX (Advisor.AI)

### Underserved Areas / Opportunities
- **Real-time SIS synchronisation**: most non-SIS-bundled platforms suffer data lags that undermine audit accuracy
- **Genuine AI conversation with students**: no platform yet delivers LLM-powered conversational advising that can answer nuanced questions about degree requirements or course selection in natural language
- **Holistic student context**: cross-system data (SIS + LMS + financial aid + mental health + career services) assembled automatically for each advising session rather than manually
- **Referral loop closure**: referrals to support services (tutoring, mental health, financial aid) are tracked inconsistently; outcome tracking is rarely automated
- **Transfer student journey**: end-to-end support from prospective transfer credit evaluation through post-enrolment planning remains fragmented across tools
- **Equity-aware advising nudges**: tools that surface equity-relevant context (first-generation, Pell recipient, under-represented groups) to advisors in actionable ways without algorithmic bias
- **Proactive degree completion coaching for community colleges**: high advisor-to-student ratios mean most community college students receive inadequate proactive advising; scalable AI tools are needed
- **Open-source interoperability**: no open-source platform with mature 1EdTech Edu-API / LTI Advantage integration exists

### AI-Augmentation Candidates
- Manual meeting preparation (assembling student context from SIS, LMS, financial aid) → AI-generated pre-meeting briefings
- Rule-based early alert triggers (GPA threshold, missed assignments) → ML-based risk scoring with explanatory factors
- Manual degree requirement rule authoring → AI-assisted rule creation and validation from catalog PDFs
- Transfer credit manual equivalency review → AI-suggested equivalency mapping with human-in-the-loop confirmation
- Generic outreach messaging to at-risk cohorts → LLM-generated personalised outreach tailored to student context
- Post-advising session documentation → AI-generated draft case notes from session summary
- Course sequence optimisation → AI-generated graduation pathways optimised for course availability, prerequisites, and student preferences

---

## Legal & IP Summary

No patent concerns were identified in public sources for the core functional categories (degree audit, academic planning, early alert, appointment scheduling). FlightPath is published under AGPL-3.0, which imposes copyleft obligations on any modifications deployed to users — this licence is incompatible with a proprietary product strategy unless the open-source component is used purely as a dependency without modification. All commercial platforms (EAB Navigate, Ellucian Degree Works, Salesforce Education Cloud, Civitas, Stellic, Advisor.AI, uAchieve, Workday Student) are proprietary SaaS products. An AI-native open-source advising platform would need to implement its own degree audit rule engine, planning tools, and analytics from scratch or build on open foundations (e.g., the AGPL-licensed FlightPath core with appropriate copyleft compliance) to avoid licence conflicts.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Degree audit engine: real-time requirement checking against configurable institution catalog rules
- Academic planner: multi-term course planning with prerequisite validation and course availability flags
- Student self-service portal: audit view, plan view, appointment booking
- Advisor caseload dashboard: assigned students, pending alerts, outstanding tasks
- Early alert system: configurable triggers, faculty submission forms, advisor task routing
- SIS integration layer: live data feed support for Banner, Colleague, PeopleSoft, and Workday Student
- LMS integration: Canvas and Blackboard grade and attendance ingestion for risk signals
- SSO integration: SAML 2.0 and OIDC for institutional identity providers
- Case notes: structured interaction logging per student per session

**Should-have (v1.1)**
- AI-generated pre-meeting briefings: automatic assembly of student context from all connected systems
- Transfer credit articulation: equivalency rules, exception workflows, prospective student preview
- What-if analysis: major/minor/catalog year change modelling without committing to records
- Predictive risk scoring: institution-specific ML model with explainability output
- Referral tracking: structured referral workflows to support services with outcome tracking
- Bulk outreach campaigns: email and SMS to filtered student cohorts
- Analytics and outcomes reporting: retention rates, advising contact rates, equity disaggregation

**Nice-to-have (backlog)**
- Credential Discovery: surface unrealised graduation eligibility across all programs
- LLM-powered conversational student chatbot: natural language Q&A on degree requirements and course selection
- AI-assisted catalog rule authoring: generate degree requirement rules from uploaded catalog PDFs
- AI-generated case note drafts: post-session note generation from advisor summary input
- Appointment preparation agent: autonomous AI agent preparing advisor briefings without manual trigger
- Open API layer: published OpenAPI 3.x specification for institutional data consumers and third-party integrations
- MCP server: Model Context Protocol server exposing advising context to LLM tool-use agents
