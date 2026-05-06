# Standards & API Reference

> Project: Technical Interview Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### W3C & IETF Standards

**WebRTC (W3C Recommendation / IETF RFC suite)**
- **URL:** https://www.w3.org/TR/webrtc/ · https://datatracker.ietf.org/doc/html/rfc8825
- WebRTC (Web Real-Time Communication) is the foundational standard for peer-to-peer audio, video, and data communication in browser-based applications. It underpins all live video conferencing features in a technical interview platform. The IETF published over 40 related RFCs through the RTCWEB working group (finalised January 2021), including RFC 8825 (overview) and RFC 8834 (media transport via RTP). Any platform building native video conferencing should conform to these specifications.

**RFC 6455 — The WebSocket Protocol**
- **URL:** https://www.rfc-editor.org/rfc/rfc6455
- Defines the WebSocket protocol, which provides full-duplex communication over a single TCP connection via an HTTP upgrade handshake. WebSockets are the standard transport layer for real-time collaborative code editing (e.g., syncing keystrokes, cursor positions, and code state across participants). All major collaborative coding platforms rely on WebSocket connections for low-latency synchronisation.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7231
- The authoritative specification for HTTP/1.1 request methods, status codes, and headers. Any REST API exposed by the platform for ATS integration, interview creation, or report retrieval must conform to HTTP/1.1 semantics. Relevant when designing webhook delivery (POST requests) and API authentication (Authorization headers).

**RFC 7617 — The 'Basic' HTTP Authentication Scheme**
- **URL:** https://datatracker.ietf.org/doc/html/rfc7617
- Specifies the HTTP Basic authentication scheme used by many interview platform APIs as an API-key authentication mechanism (API key passed in Authorization header). Relevant for the platform's own API key authentication design.

**WCAG 2.2 (ISO/IEC 40500:2025)**
- **URL:** https://www.w3.org/TR/WCAG22/ · https://webaim.org/standards/wcag/checklist
- Web Content Accessibility Guidelines version 2.2, published October 2023 and approved as ISO/IEC 40500:2025. Enterprise procurement teams increasingly require WCAG 2.2 Level AA compliance for interview tooling to accommodate candidates with disabilities (visual impairments, motor disabilities, cognitive differences). Nine new success criteria were added in 2.2 relative to 2.1, focusing on low vision, cognitive/learning disabilities, and touch-screen accessibility. Code editors, whiteboards, and video interfaces all require careful attention to keyboard navigability, colour contrast, and screen-reader compatibility.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OAuth 2.1 (draft)**
- **URL:** https://www.rfc-editor.org/rfc/rfc6749 · https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1
- OAuth 2.0 is the industry-standard authorisation framework. As of 2026, OAuth 2.1 (consolidating best practices) recommends the Authorisation Code Grant with PKCE (Proof Key for Code Exchange) for all clients. Relevant for SSO integration with enterprise identity providers (Okta, Azure AD, Google Workspace) and for ATS OAuth-based authentication flows (Greenhouse, Lever, Workday all support OAuth).

**OpenID Connect (OIDC)**
- **URL:** https://openid.net/connect/ · https://auth0.com/docs/authenticate/protocols/openid-connect-protocol
- OIDC is an identity layer built on top of OAuth 2.0 that adds user authentication (ID tokens) alongside OAuth 2.0 authorisation (access tokens). Technical interview platforms must implement OIDC for enterprise SSO (single sign-on) integration, allowing recruiters and interviewers to log in via their company's identity provider. The OpenID Foundation's certified implementation list provides guidance on compliant libraries and providers.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- **URL:** https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- The de-facto security certification required by enterprise HR and legal teams before approving interview tooling. SOC 2 Type II audits assess the operating effectiveness of security controls across five Trust Services Criteria (Security, Availability, Processing Integrity, Confidentiality, and Privacy) over a defined period (typically 3–12 months). Competing platforms including Hirevue, Workable, and X0PA AI all carry SOC 2 Type II certification. Any platform targeting enterprise buyers must obtain this certification.

**ISO/IEC 27001 — Information Security Management**
- **URL:** https://www.iso.org/standard/27001
- The international standard for information security management systems (ISMS). Commonly paired with SOC 2 for enterprise sales; platforms including Workable carry both ISO 27001 and SOC 2 Type II. Relevant to the platform's overall data governance framework, particularly for storing candidate session recordings and evaluation data.

**OWASP Application Security Verification Standard (ASVS)**
- **URL:** https://owasp.org/www-project-application-security-verification-standard/
- OWASP ASVS is a secure-coding and verification standard applied during development and testing to ensure implementations meet agreed-upon security controls. Particularly relevant for the code execution sandbox (preventing arbitrary code from escaping the sandbox), the session recording storage (protecting candidate PII), and proctoring capabilities. The OWASP Top 10 Proactive Controls provide complementary developer guidance.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- **URL:** https://gdpr.eu/ · https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=celex%3A32016R0679
- The EU's comprehensive data privacy regulation governing the collection, use, storage, and transfer of personal data of individuals within the EU/EEA. Technical interview platforms collect highly sensitive candidate data (video recordings, code submissions, behavioural analysis). GDPR compliance requires lawful basis for processing, data minimisation, right to erasure, and data processing agreements (DPAs) with all sub-processors. Competing platforms including eSkill, Workable, and HackerEarth explicitly market GDPR compliance.

**EEOC Uniform Guidelines on Employee Selection Procedures (29 CFR Part 1607)**
- **URL:** https://www.eeoc.gov/laws/guidance/questions-and-answers-clarify-and-provide-common-interpretation-uniform-guidelines
- US federal guidelines that govern the design and validation of employment selection procedures to prevent discriminatory outcomes. Technical assessment instruments used in hiring must be validated to demonstrate job-relatedness and non-adverse-impact on protected classes. Platforms including eSkill and HackerEarth explicitly market EEOC compliance. AI-powered scoring systems must be particularly carefully designed to pass EEOC scrutiny.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- **URL:** https://spec.openapis.org/oas/v3.1.0.html · https://www.openapis.org/
- The standard format for describing RESTful APIs in YAML or JSON, enabling automatic code generation, interactive documentation, and contract testing. OAS 3.1 achieves full JSON Schema Draft 2020-12 alignment. Any public API exposed by the platform (for ATS integration, interview lifecycle management, or reporting) should be documented using OAS 3.1. AI agents and LLM-based tools increasingly rely on OpenAPI specifications as the bridge between natural language and structured API calls.

**JSON Schema (Draft 2020-12)**
- **URL:** https://json-schema.org/specification
- The authoritative standard for describing and validating JSON data structures. Directly relevant to interview report schemas, problem library data structures, candidate evaluation payloads, and webhook event bodies. OAS 3.1 and JSON Schema are now fully aligned, allowing schema definitions to be shared across validation, documentation, and code generation.

**HR Open Standards (HR-JSON / HR-XML)**
- **URL:** https://www.hropenstandards.org/
- The HR Open Standards Consortium (formerly HR-XML Consortium, founded 1999) is the only independent non-profit organisation dedicated to HR data exchange specifications. Their 4.0 suite provides HR-JSON and HR-XML data vocabularies for ATS integration, candidate screening, and interview data exchange. The Consortium has initiated specific Screening and Interviewing data exchange standards within their HR-JSON 4.0 framework — directly applicable to interview platform data models. Adopting HR Open Standards schemas for candidate data and evaluation reports reduces integration friction with ATS and HRIS systems.

---

### Real-Time Collaboration Protocols

**Operational Transformation (OT)**
- **URL:** https://dl.acm.org/doi/10.1145/3375186 · https://www.tiny.cloud/blog/real-time-collaboration-ot-vs-crdt/
- OT is a foundational algorithm family for concurrent collaborative text editing, invented in the late 1980s and used in Google Docs. It transforms concurrent operations to maintain consistency with a central server coordinating operation ordering. Best suited when reliable server infrastructure is available and compact data representation is required. Understanding OT is essential for implementing the real-time collaborative code editor.

**CRDTs (Conflict-free Replicated Data Types)**
- **URL:** https://arxiv.org/pdf/1810.02137 · https://github.com/yjs/yjs
- CRDTs are a class of distributed data structures that allow concurrent modifications on different replicas to converge automatically without coordination. The Yjs CRDT library is widely used for collaborative editors (including VS Code Live Share). CRDTs are preferred for offline-capable or peer-to-peer scenarios. Platforms like Figma switched from OT to CRDTs. An open-source technical interview platform should evaluate Yjs (MIT licence) as the collaborative editing foundation.

---

### Code Execution & Sandbox Standards

**Judge0 API (Open Source)**
- **URL:** https://ce.judge0.com/ · https://github.com/judge0/judge0
- Judge0 is the dominant open-source sandboxed code execution system, used as the backend for many coding platforms. It supports 60+ programming languages, sandboxed compilation and execution with configurable time and memory limits, and custom compiler options. Judge0 is used by competitive programming platforms, e-learning platforms, and candidate assessment tools. An open-source technical interview platform should evaluate Judge0 CE (Community Edition) as the code execution backend.

---

## Similar Products — Developer Documentation & APIs

### CoderPad

- **Description:** Industry-standard collaborative coding interview platform supporting 40+ languages, digital whiteboard, take-home assessments, and AI-in-context tools.
- **API Documentation:** https://api.interview.coderpad.io/ (Interview API) · https://api.screen.coderpad.io/ (Screen API)
- **Developer Guide:** https://coderpad.io/resources/docs/
- **Standards:** REST/JSON; API key authentication via Authorization header; webhook support for ATS integration
- **Authentication:** API Key (enterprise plans only)
- **Notable:** API access restricted to Enterprise tier; supports programmatic pad creation and interview lifecycle management; exposes interview transcript retrieval endpoints. Integrates with Greenhouse, Lever, Workday, and iCIMS via REST/Webhooks.

---

### HackerEarth

- **Description:** Developer-focused live coding interview platform (FaceCode) with 40,000+ pre-built problems, integrated HD video, ATS integrations, and AI interview summaries.
- **API Documentation:** https://www.hackerearth.com/docs/wiki/developers/v4/ (Code Compilation API v4)
- **Assessment API:** https://www.hackerearth.com/recruit/api
- **Developer Guide:** https://help.hackerearth.com/hc/en-us/articles/12033582123289-how-to-integrate-with-hackerearth-
- **Standards:** REST/JSON; API key authentication; webhook callbacks for assessment completion
- **Authentication:** API Key (client-secret based)
- **Notable:** Code Compilation API supports 40+ languages in a single API call (compile + execute). Candidate invite API allows controlling email dispatch and post-test redirect. Assessment reports delivered via webhook after candidate completion. Integrates with Greenhouse (documented), Lever, iCIMS, Workable, and others.

---

### Codility

- **Description:** Technical assessment platform with automated evaluation engine, performance complexity scoring, anti-plagiarism detection, and live interview IDE.
- **API Documentation:** https://codility.com/api-documentation/
- **Developer Guide:** https://support.codility.com/hc/en-us/articles/360043824513-Using-Codility-s-API
- **Standards:** REST/JSON; token-based API authentication via registered application
- **Authentication:** Application registration → access token (Bearer token in Authorization header)
- **Notable:** Integration user concept for service-account-style API access. Supports programmatic task creation, candidate management, and result export. Integrates with Workable (documented).

---

### iMocha

- **Description:** Broad skills assessment platform with 690+ pre-built IT skill tests, AI scoring, advanced proctoring, and skills intelligence analytics.
- **API Documentation:** https://developer.imocha.io/
- **Developer Guide:** https://developer.imocha.io/api/getting-started
- **Standards:** REST API; key-based authentication
- **Authentication:** API Key (requested via support)
- **Notable:** REST-styled API for retrieving test lists, inviting candidates, and retrieving candidate test reports. Integrates with Workable and other ATS platforms.

---

### Greenhouse (ATS)

- **Description:** Leading Applicant Tracking System (ATS) with a rich developer ecosystem; the most common ATS that technical interview platforms must integrate with.
- **API Documentation:** https://developers.greenhouse.io/
- **Harvest API (primary):** https://developers.greenhouse.io/harvest.html
- **Assessment API:** https://developers.greenhouse.io/ (Assessment API section)
- **Webhooks:** https://developers.greenhouse.io/webhooks.html
- **Standards:** REST/JSON (Harvest API); webhook POST events; OAuth 2.0 for partner integrations
- **Authentication:** API Key (Harvest API); OAuth 2.0 (partner integrations)
- **Notable:** Assessment API allows third-party coding platforms to be triggered when a candidate reaches a predetermined stage; assessment status is updated back in Greenhouse when candidate completes the test. Webhook events cover candidate and interview lifecycle. The ScheduledInterview object represents interview sessions with full panel and time data.

---

### Lever (ATS)

- **Description:** Modern ATS with REST API and webhook support; popular with mid-size technology companies.
- **API Documentation:** https://hire.lever.co/developer/documentation
- **Developer Guide:** https://www.getknit.dev/blog/lever-api-directory
- **Standards:** REST/JSON; OAuth 2.0; HTTPS-only webhooks (mandatory encryption)
- **Authentication:** OAuth 2.0
- **Notable:** ScheduledInterview object represents interview sessions; Create Interview endpoint supports panel management. Webhook signatures use a signing token for authenticity verification. All webhook traffic must use HTTPS. CoderPad, HackerRank, and Qualified.io all document Lever integration.

---

### Workday (HRIS/ATS)

- **Description:** Enterprise HR platform and ATS; the most common enterprise system that large-company hiring workflows must integrate with.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/restapi/index.html (Workday REST API)
- **Integration Guide:** https://www.merge.dev/blog/workday-api-integration
- **Standards:** SOAP Web Services (broadest coverage); REST API (modern, partial coverage); OAuth 2.0
- **Authentication:** OAuth 2.0 (preferred) or Integration System User (ISU)
- **Notable:** Three API surfaces: SOAP (broadest), REST (modern), and Reports-as-a-Service. Unified API platforms (Merge, Knit, Unified.to) provide abstraction layers to reduce Workday integration complexity. Candidate data model supports application details, attachments, and personal information.

---

### Judge0 (Code Execution Backend)

- **Description:** Open-source sandboxed code execution system used as the execution backend for many coding platforms; the reference implementation for online code execution.
- **API Documentation:** https://ce.judge0.com/
- **GitHub:** https://github.com/judge0/judge0
- **RapidAPI:** https://rapidapi.com/judge0-official/api/judge0-ce
- **Standards:** REST/JSON; submission-based execution model
- **Authentication:** API Key (RapidAPI managed instance) or self-hosted (no auth by default)
- **Notable:** 60+ language support; configurable time and memory limits; custom compiler options and stdin/stdout; attributes 1–20 for submission creation, 21–33 for execution results. Research paper: "Robust and Scalable Online Code Execution System." MIT-licenced Community Edition available for self-hosting. Widely used in competitive programming and assessment platforms.

---

## Notes

**Emerging Standards:**
- **OpenID4VP / OpenID4VCI:** The OpenID Foundation is launching verifiable credential programs in 2026, potentially relevant for future candidate credential verification use cases (e.g., certified skill badges).
- **OpenAPI 4.0 ("Moonwalk"):** In early development; promises richer JSON Schema integration and support for emerging API paradigms including AI agent-to-API interactions.
- **HR Open Standards Interviewing Spec:** The HR Open Standards Consortium has initiated a specific Interviewing data exchange standard within their 4.0 HR-JSON suite, which may become the canonical data model for interview result exchange between platforms and ATS systems as it matures.

**Key Integration Complexity:**
Workday's API surface remains the most complex integration target due to its combination of SOAP, REST, and Reports-as-a-Service surfaces. Unified API abstraction platforms (Merge, Knit, Kombo) can reduce this burden by providing a single normalised API across multiple ATS/HRIS providers — worth evaluating for the MVP ATS integration layer rather than building individual ATS connectors.

**Code Execution Security:**
Judge0 has known sandbox escape vulnerabilities in certain configurations (see: https://tantosec.com/blog/judge0/). Any production deployment of Judge0 or a custom code execution sandbox must be run in an isolated network environment (e.g., per-submission microVM or gVisor container) and reviewed against OWASP ASVS Level 2 controls.
