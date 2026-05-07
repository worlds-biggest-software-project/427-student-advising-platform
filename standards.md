# Standards & API Reference

> Project: Student Advising Platform · Generated: 2026-05-07

---

## Industry Standards & Specifications

### Privacy & Data Governance

**FERPA — Family Educational Rights and Privacy Act (US, 20 U.S.C. § 1232g)**
- URL: https://studentprivacy.ed.gov/ferpa
- The foundational US federal law governing the confidentiality of student education records. Applies to any institution receiving federal funding. Prohibits disclosure of personally identifiable information (PII) from education records without student or parent consent, with enumerated exceptions. Any student advising platform that stores, processes, or transmits student records (grades, enrollment status, advising notes, risk scores) must be deployed as a FERPA-compliant "school official" under the legitimate educational interest exception. All data governance agreements, audit logs, and role-based access controls must align with FERPA requirements.

**GDPR — General Data Protection Regulation (EU, Regulation 2016/679)**
- URL: https://gdpr.eu/
- Applies to any platform used by EU institutions or processing data of EU students. Requires lawful basis for processing, data minimisation, purpose limitation, consent or legitimate interest grounds, and documented Data Processing Agreements (DPAs) with vendors. Relevant for institutions with international campuses or study-abroad programmes feeding advising data into the platform.

**GLBA — Gramm-Leach-Bliley Act (US, applicable to Title IV financial aid data)**
- URL: https://www.ftc.gov/business-guidance/privacy-security/gramm-leach-bliley-act
- Federal Trade Commission regulations apply to higher education institutions handling financial information for Title IV federal aid. GLBA Safeguards Rule requires a written information security programme protecting student financial data. Advising platforms that ingest financial aid data (scholarship eligibility, hold status, FAFSA-derived risk signals) must comply with GLBA safeguards.

**COPPA — Children's Online Privacy Protection Act (US, for dual-enrolled minors)**
- URL: https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa
- Relevant where platforms serve students under 13 in dual-enrolment programmes. Requires parental consent for data collection from children.

---

### Interoperability & Integration Standards

**1EdTech Edu-API v1.0 (Candidate Final, 2025)**
- URL: https://www.1edtech.org/standards/edu-api | https://www.imsglobal.org/spec/eduapi/v1p0
- The emerging higher-education REST/JSON standard for data exchange between Student Information Systems and teaching-and-learning tools. Edu-API is designed as the "OneRoster for Higher Education," superseding the older Learning Information Services (LIS) standard with modern REST patterns, OAuth 2.0 security, and JSON payloads. Oracle PeopleSoft Campus Solutions achieved the first Edu-API certification in October 2025. A student advising platform implementing Edu-API natively gains interoperability with any certified SIS without proprietary connectors — a significant architectural advantage over current platforms that rely on bespoke data feeds.

**1EdTech LTI Advantage v1.3**
- URL: https://www.imsglobal.org/lti-advantage-overview | https://www.imsglobal.org/spec/lti/v1p3
- Learning Tools Interoperability v1.3 is the standard for securely embedding and integrating educational tools within Learning Management Systems. Built on OAuth 2.0 and JSON Web Tokens (RFC 7519). LTI Advantage includes three service extensions: Names and Role Provisioning Services (roster sync), Deep Linking, and Assignment and Grade Services. An advising platform surfaced as an LTI tool within Canvas or Blackboard can deliver grade-signal data directly without a separate SIS connector.

**1EdTech OneRoster v1.2**
- URL: https://www.imsglobal.org/activity/oneroster
- Standard for sharing class rosters and related data between SIS and other platforms, supporting both CSV batch transfer and REST API modes. Primarily adopted in K-12 but relevant for advising platforms targeting community college and dual-enrolment contexts where K-12 SIS systems feed higher-ed platforms.

**1EdTech Caliper Analytics v1.2**
- URL: https://www.1edtech.org/standards/caliper
- Standard for emitting and consuming learning event data across LMS, SIS, and advising tools. Defines a common vocabulary for learning activities (page views, assignment submissions, grade events) and a standard event format for Learning Record Stores (LRS). An advising platform consuming Caliper events from Canvas or Blackboard can build rich student engagement signals for early alert models without bespoke LMS connectors.

**xAPI (Experience API / Tin Can) v2.0**
- URL: https://xapi.com/ | https://adlnet.gov/projects/xapi/
- ADL-maintained standard for tracking learning experiences beyond the LMS. xAPI statements (actor → verb → object) sent to a Learning Record Store provide a flexible basis for recording advising session interactions, completed milestones, and referral outcomes. Complements Caliper for platforms wanting to capture broader student activity including non-LMS touchpoints.

**Ed-Fi Data Standard v4.x**
- URL: https://www.ed-fi.org/
- Open, community-governed data standard for educational data interoperability. Ed-Fi defines a broader operational data model for student information across the education ecosystem and is commonly adopted at the state education agency level. Relevant for advising platforms deployed in state-system or system-office contexts requiring state-level reporting alignment.

---

### Web, API & Data Model Standards

**OpenAPI Specification v3.2**
- URL: https://spec.openapis.org/oas/v3.2.0.html | https://www.openapis.org/
- The de-facto standard for describing REST APIs in a machine-readable YAML or JSON format. Any student advising platform exposing a public or partner API should publish an OpenAPI 3.x specification to enable automated SDK generation, contract testing, and developer tooling. Enables institutional IT teams to build custom integrations and third-party tooling vendors to certify compatibility without bespoke documentation.

**JSON:API v1.1**
- URL: https://jsonapi.org/
- A specification for building REST APIs with JSON that provides conventions for resource representation, relationships, filtering, sorting, pagination, and error handling. Reduces bikeshedding in API design and improves client-side predictability.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation protocol for delegated API access. Required for all SIS, LMS, and identity provider integrations. The platform should implement OAuth 2.0 authorization code flow for user-delegated access and client credentials flow for server-to-server data ingestion.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Authentication layer built on OAuth 2.0. Required for SSO integration with institutional identity providers (Shibboleth, Azure AD, Okta). OIDC ID tokens provide verified claims about authenticated users including institution affiliation, role, and student ID, enabling role-based access control for students, advisors, faculty, and administrators.

**SAML 2.0 — Security Assertion Markup Language**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- Legacy SSO protocol still widely deployed in higher education through institutional identity federations (InCommon in the US, eduGAIN internationally). Many universities operate a Shibboleth Identity Provider supporting SAML 2.0. The platform must support SAML 2.0 Service Provider mode as well as OIDC to maximise institutional compatibility.

**InCommon Federation**
- URL: https://www.incommon.org/
- The US higher education identity federation operated by Internet2. Membership provides standardised SAML 2.0 federated SSO across thousands of US universities and research institutions. InCommon certification of the advising platform enables one-click SSO deployment at member institutions.

---

### Accessibility Standards

**WCAG 2.2 (Web Content Accessibility Guidelines) — ISO/IEC 40500:2025**
- URL: https://www.w3.org/TR/WCAG22/ | https://www.w3.org/WAI/standards-guidelines/wcag/
- Published as a W3C Recommendation in October 2023 and approved as ISO/IEC 40500:2025 in 2025. WCAG 2.2 Level AA is now the legally required standard for ADA Section 508 (US), the European Accessibility Act (EAA, in force June 2025), and many state-level digital accessibility laws. The advising platform, including both the advisor dashboard and the student self-service portal, must meet WCAG 2.2 Level AA. The 9 new success criteria in WCAG 2.2 (vs. 2.1) include focus appearance, target size, and accessible authentication — all applicable to advising portal UX patterns.

**Section 508 of the Rehabilitation Act (US)**
- URL: https://www.section508.gov/
- US federal accessibility law requiring electronic and information technology used by federal agencies to be accessible. Applies to public institutions receiving federal funds. WCAG 2.2 Level AA conformance satisfies Section 508 Technical Standards.

---

### Security Standards

**NIST SP 800-171 Rev. 3 — Protecting Controlled Unclassified Information**
- URL: https://csrc.nist.gov/publications/detail/sp/800-171/rev-3/final
- The US Department of Education has designated most student financial aid data (Federal Tax Information used in FAFSA processing) as Controlled Unclassified Information (CUI) and has urged institutions to align with NIST 800-171 controls. As advising platforms commonly ingest financial hold and aid status data, alignment with NIST 800-171 families (access control, incident response, audit and accountability, system and communications protection) is expected by research universities and institutions with federal contracts.

**NIST Privacy Framework v1.0**
- URL: https://www.nist.gov/privacy-framework
- Voluntary framework for managing privacy risk, compatible with GDPR and CCPA. Provides a structured approach to identifying, governing, controlling, communicating, and protecting personal data — appropriate for advising platform privacy impact assessments.

**OWASP Application Security Verification Standard (ASVS) 4.0**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- De-facto standard for web application security controls. ASVS Level 2 provides a realistic target for a SaaS platform handling sensitive student records. Covers authentication, session management, access control, input validation, cryptography, error handling, and logging requirements.

---

### Model Context Protocol (MCP)

**Anthropic Model Context Protocol (MCP) — Draft Specification**
- URL: https://modelcontextprotocol.io/
- Open protocol for exposing structured context and tools to LLM agents. An MCP server exposing the advising platform's degree audit data, student profiles, and intervention history would allow AI agents (Claude, GPT-4, Gemini) to answer complex natural language advising questions using real institutional data. Particularly relevant for the AI-native advising chatbot and automated pre-meeting briefing generation use cases. MCP is increasingly adopted by higher-education AI tooling vendors (ibl.ai MCP architecture guide for higher education was published in 2026).

---

## Similar Products — Developer Documentation & APIs

### Salesforce Education Cloud
- **Description:** Enterprise CRM and AI advising platform for higher education, including degree planning, advising workflows, and Agentforce AI agents.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.edu_cloud_dev_guide.meta/edu_cloud_dev_guide/edu_cloud_intro.htm
- **SDKs/Libraries:** Salesforce JavaScript SDK, Python SDK, Apex (native), REST/SOAP clients in all major languages via standard Salesforce APIs
- **Developer Guide:** https://developer.salesforce.com/developer-centers/education-cloud
- **Standards:** REST/JSON, SOAP, OpenAPI (Salesforce API Explorer), OAuth 2.0, SAML 2.0, OIDC
- **Authentication:** OAuth 2.0 (authorization code, client credentials, JWT bearer), SAML 2.0 SSO

### Canvas LMS (Instructure)
- **Description:** Widely deployed LMS providing course management, grade data, and student engagement signals critical for early alert systems in advising platforms.
- **API Documentation:** https://canvas.instructure.com/doc/api/
- **Developer Portal:** https://developerdocs.instructure.com/services/canvas
- **SDKs/Libraries:** No official SDK; community Python library `canvasapi`; REST with standard HTTP client libraries
- **Developer Guide:** https://community.canvaslms.com/t5/Developers-Group/Canvas-APIs-Getting-started-the-practical-ins-and-outs-gotchas/ba-p/263685
- **Standards:** REST/JSON, LTI Advantage v1.3, OAuth 2.0 (RFC 6749), SAML 2.0, FERPA-compliant data handling
- **Authentication:** OAuth 2.0 (authorization code with 1-hour token expiry and refresh); Developer Keys issued per institution

### Ellucian Ethos (Banner/Colleague Integration Platform)
- **Description:** Ellucian's cloud integration platform providing REST/JSON access to Banner and Colleague SIS data — the primary integration pathway for the market-dominant Ellucian SIS ecosystem.
- **API Documentation:** https://www.ellucian.com/blog/data-integration-guide-institutions
- **Integration Documentation:** https://tray.ai/documentation/connectors/service/ellucian-ethos (third-party reference)
- **SDKs/Libraries:** REST/JSON client; community connectors available for MuleSoft, Workato, Tray.ai
- **Standards:** REST/JSON, OAuth 2.0 (API key authentication), 1EdTech Edu-API (in development)
- **Authentication:** API Key (Ellucian Ethos API key per integration)

### EAB Navigate360
- **Description:** Leading student success CRM with early alert, appointment scheduling, and predictive analytics for higher education.
- **API Documentation:** https://help.navigate360.com/ (partner login required for full API docs)
- **Developer Reference:** EAB Systems API — https://www.eabsystems.com/api.html
- **Integration Methods:** REST API (partner data extract), LTI v1.1/v1.3 for LMS embed, bulk data export
- **Standards:** REST/JSON, LTI, SAML 2.0, OIDC
- **Authentication:** API Key for data export endpoints; SAML/OIDC for SSO

### Workday Student
- **Description:** SIS with native advising and degree audit modules; API access to student enrollment, academic progress, and planning data.
- **API Documentation:** https://community.workday.com/t5/Workday-APIs/ct-p/workday_apis (Workday Community login required)
- **Developer Guide:** Workday REST API and Report-as-a-Service (RAAS) documentation via Workday Community portal
- **SDKs/Libraries:** REST client libraries; Workday Studio for integration development
- **Standards:** REST/JSON, SOAP/WSDL (legacy), OAuth 2.0, SAML 2.0, OpenID Connect
- **Authentication:** OAuth 2.0 (client credentials for API access), SAML 2.0 SSO

### 1EdTech Edu-API Reference Implementation
- **Description:** The emerging REST standard for SIS-to-learning-tool data exchange in higher education; Oracle PeopleSoft is the first certified SIS (October 2025).
- **Specification:** https://www.imsglobal.org/spec/eduapi/v1p0
- **Overview:** https://www.1edtech.org/standards/edu-api
- **Developer Documentation:** https://docs.oracle.com/cd/G11181_01/cs92pbr35/eng/cs/lscc/UnderstandingEduApi.html (PeopleSoft reference implementation)
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI 3.x
- **Authentication:** OAuth 2.0 (client credentials flow for M2M, authorization code for user-delegated access)

### Civitas Learning Student Impact Platform
- **Description:** AI-powered analytics and student success workflows platform providing predictive risk scoring and intervention tools.
- **API Documentation:** Not publicly available; integration is handled via implementation engagement
- **Integration Methods:** Data ingestion APIs (SIS, LMS, CRM feeds); outbound workflow webhooks
- **Standards:** REST/JSON; FERPA-compliant data handling
- **Authentication:** API key and institutional data governance agreements

### FlightPath (Open Source)
- **Description:** Open-source AGPL-3.0 degree audit and advising platform with SIS integration and transfer credit mapping.
- **Source Code:** https://github.com/flipflipshift/flightpath (historical); main repo at https://getflightpath.com/
- **Documentation:** https://getflightpath.com/node/1003
- **SDKs/Libraries:** PHP-based; can be extended via source code modification (AGPL-3.0 copyleft applies)
- **Standards:** REST/JSON APIs for SIS data import; web-based interface
- **Authentication:** Institutional SSO via SAML; local authentication

---

## Notes

**Edu-API adoption timeline**: As of May 2026, Edu-API is in Candidate Final status with only Oracle PeopleSoft certified. Ellucian Banner and Colleague certifications have not yet been announced. An AI-native advising platform should implement Edu-API support now to be positioned as a first-mover when the major SIS vendors certify, while also maintaining legacy data feed support (Banner REST API, Colleague Ethos, Workday RAAS) for near-term deployments.

**MCP for advising context**: The MCP architecture is an emerging pattern in higher education AI tooling (documented by ibl.ai in their 2026 higher education MCP guide). A student advising platform implementing an MCP server exposing degree audit state, student risk profiles, and advising history would enable a new generation of LLM-powered advising agents that can answer multi-step questions ("What courses should I take next term to stay on track for May graduation if I drop CHEM 201?") using real institutional data without screen-scraping.

**FERPA and AI model training**: Using student records to train or fine-tune AI risk models requires careful FERPA analysis. The general consensus in the field is that training on de-identified or aggregated data is permissible, but using personally identifiable student records to train commercial models without appropriate data processing agreements and oversight constitutes a FERPA violation. The platform's AI architecture should use privacy-preserving techniques (differential privacy, federated learning, or strict de-identification pipelines) for any model training use cases.

**Data portability**: No widely adopted student advising data portability standard exists as of 2026. Student degree plans, advising notes, and interaction histories are locked in proprietary formats across all commercial vendors. An open-source platform publishing its data model as a JSON Schema or OpenAPI 3.x schema would represent a meaningful differentiation and could seed a community standard.
