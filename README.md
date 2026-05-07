# Student Advising Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for degree audit, academic planning, and proactive student intervention that gives advisors a unified picture of every student without vendor lock-in.

Higher education advisors are responsible for guiding students to graduation, yet they routinely manage caseloads of 500+ students using data scattered across SIS, LMS, financial aid, and support-service systems that do not talk to each other. The Student Advising Platform consolidates degree audit, multi-term planning, early alerts, and referral tracking into a single open-source system, augmented by AI capabilities that automate the manual data assembly and pattern recognition advisors currently perform by hand.

---

## Why Student Advising Platform?

- **Advisor caseloads are unsustainable.** At many institutions, advisor-to-student ratios exceed 1:500. Reactive advising means at-risk students are identified only after their situation has deteriorated significantly. Scalable, AI-assisted proactive monitoring is needed to close this gap.
- **Data is fragmented across siloed systems.** Assembling a complete student picture requires manually pulling information from SIS, LMS, financial aid, and student services platforms. No open-source tool unifies this data automatically for each advising session.
- **Incumbent solutions impose vendor lock-in.** Ellucian Degree Works is tightly coupled to Banner/Colleague. Workday Student advising only works within the Workday SIS. Salesforce Education Cloud requires expensive Salesforce expertise and licensing. Institutions without these ecosystems face complex, costly integrations.
- **Existing platforms lack genuine AI advising capabilities.** EAB Navigate has no generative AI features. Degree Works has no AI-powered risk scoring. No platform yet delivers LLM-powered conversational advising that can answer nuanced student questions about degree requirements in natural language.
- **Community colleges are underserved.** The institutions with the most acute retention challenges and highest caseload ratios have the fewest resources for enterprise SaaS licensing. An open-source, self-hostable option with AI-assisted advising at scale is conspicuously absent from the market.

---

## Key Features

### Degree Audit and Academic Planning

- Real-time compliance checking of completed and in-progress coursework against configurable degree requirements, with support for multiple programs, minors, and concentrations
- What-if analysis to model major, minor, or catalog year changes without committing to the official record
- Multi-term course planning with prerequisite conflict detection and course availability awareness
- Transfer credit articulation with equivalency rules, exception workflows, and prospective student preview
- Credential Discovery to surface unrealised graduation eligibility across all declared and undeclared programs

### Early Alert and Intervention

- Configurable at-risk triggers based on grade thresholds, LMS attendance and assignment data, financial holds, and advising contact gaps
- Faculty-facing alert submission forms that route to advisor task queues
- Predictive risk scoring using institution-specific ML models with explainability output
- Structured referral workflows to tutoring, mental health, disability services, and financial aid, with outcome tracking to close the referral loop

### Advisor Workflow and Caseload Management

- Caseload dashboard showing assigned students, pending alerts, outstanding tasks, and approaching milestones (probation deadlines, graduation windows)
- Appointment scheduling with calendar sync and student-facing booking
- Structured case notes linked to student advising records
- Bulk outreach campaigns via email and SMS to filtered student cohorts
- AI-generated pre-meeting briefings that automatically assemble student context from all connected systems

### Student Self-Service

- Student-facing portal for viewing degree progress, planning upcoming terms, and checking graduation eligibility
- Appointment booking and intake form completion
- Natural-language Q&A chatbot for degree requirement and course selection questions (backlog)

### Analytics and Reporting

- Institution-level reporting on retention rates, time to graduation, advising contact frequency, and early alert response rates
- Equity-disaggregated metrics by demographic subgroup
- Program effectiveness evaluation using propensity score matching

---

## AI-Native Advantage

Current advising platforms bolt analytics onto legacy architectures or require separate products for risk scoring. This platform is designed from the ground up with AI at the core: ML-based risk models replace rigid threshold triggers, LLM-generated briefings eliminate the manual data assembly that consumes advisor time before every meeting, AI-assisted catalog rule authoring can generate degree requirement rules from uploaded catalog PDFs, and AI-suggested transfer credit equivalency mappings with human-in-the-loop confirmation replace fully manual review. The result is a system where advisors spend time on meaningful student conversations rather than administrative data wrangling.

---

## Tech Stack & Deployment

- **Deployment modes:** Self-hosted, cloud, or hybrid, giving institutions full control over student data governance (FERPA compliance)
- **SIS integration:** Live data feed support for Banner, Colleague, PeopleSoft, and Workday Student via configurable connectors
- **LMS integration:** Canvas and Blackboard grade and attendance data ingestion
- **Authentication:** SAML 2.0 and OIDC for institutional identity providers
- **Standards alignment:** 1EdTech Edu-API and LTI Advantage integration planned for interoperability with the broader EdTech ecosystem
- **API surface:** Published OpenAPI 3.x specification for institutional data consumers and third-party integrations; MCP server for LLM tool-use agents (backlog)

---

## Market Context

US higher education institutions enroll approximately 19 million students and lose an estimated $16.5 billion annually in tuition revenue to student attrition. The student success technology market (advising, early alert, degree planning) was estimated at approximately $2 billion in 2025 and is growing at 12-15% annually, driven by demographic enrollment declines that force institutions to improve retention of enrolled students. Primary buyers are provost offices, enrollment management divisions, and student affairs leadership at four-year universities and community colleges.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
