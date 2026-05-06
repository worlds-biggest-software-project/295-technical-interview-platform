# Technical Interview Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for live coding, system design whiteboarding, and automated evaluation in technical hiring.

Technical Interview Platform is a collaborative workspace for engineering hiring teams to run live coding sessions, system design discussions, and structured assessments. It targets engineering hiring managers, technical recruiters, and developer relations teams who need a transparent, AI-augmented alternative to opaque commercial SaaS tools. The platform combines real-time coding, video, whiteboarding, and automated evaluation into a single workflow.

---

## Why Technical Interview Platform?

- Incumbent pricing is opaque and steep at scale — CoderPad and iMocha offer enterprise-only quotes, while HackerEarth starts at ~$169/mo and per-interview credit models make cost forecasting difficult.
- System design whiteboarding remains the weakest feature area across all incumbents, with most platforms shipping basic diagram boards rather than first-class architecture tools.
- No open-source alternative exists in this space; all nine major solutions reviewed (CoderPad, Codility, iMocha, HireHunch, HackerEarth, CodeInterview.io, InCruiter, TechInterview.live, Playcode.io) are proprietary SaaS.
- Automated evaluation is fragmented — Codility leads on correctness scoring but lacks soft-skills assessment, while InCruiter claims AI scoring with limited public proof points.
- Bias detection in interviewer behaviour, adaptive difficulty, and AI-generated problem variants are absent from every reviewed incumbent.

---

## Key Features

### Collaborative Live Interview Workspace

- Real-time collaborative code editor supporting 10+ major languages (Python, JavaScript, Java, Go, C++)
- Live HD video conferencing with screen sharing and speaker detection
- Collaborative whiteboard (Excalidraw or custom) for system design discussions
- Code execution and immediate output via integrated compiler/REPL
- Session recording with playback for asynchronous review

### Evaluation and Scoring

- Structured evaluation rubrics, interviewer notes, and feedback forms
- Automated code evaluation for correctness and time complexity
- AI-generated interview summaries covering technical and soft-skills dimensions
- Multi-interviewer panel support for up to 5 simultaneous interviewers
- Candidate anonymisation option to reduce bias during review

### Assessment Library and Async Modes

- Pre-built problem library spanning multiple skill levels
- Asynchronous take-home interview mode with submission and review workflow
- AI-generated problem variants at different difficulty levels to prevent answer leakage
- Natural-language problem search for matching questions to hiring needs

### Enterprise, Security, and Integration

- ATS integration with Workday, Greenhouse, Lever, iCIMS, and Workable via REST and webhooks
- SOC 2 Type II compliance and end-to-end encryption
- AI proctoring with tab-switching, dual-face/voice, and device detection
- Interview history and audit trails for compliance
- Role-based access for interviewer, candidate, and observer roles

---

## AI-Native Advantage

The platform embeds AI throughout the interview lifecycle rather than bolting it onto a legacy assessment engine. A real-time AI co-interviewer surfaces follow-up questions based on the candidate's code choices, while automated post-interview reports score code quality, problem-solving approach, and edge-case coverage with specific evidence drawn from the session transcript. A bias detection layer analyses interviewer behaviour patterns — talk time ratio, question consistency across demographic groups — to flag potential structured-interview violations, a capability absent from every reviewed incumbent. Adaptive difficulty calibration and AI-generated isomorphic problem variants address answer leakage and skill differentiation in ways fixed-difficulty platforms cannot.

---

## Tech Stack & Deployment

The platform is browser-based with no candidate-side installation, following the deployment pattern shared by every reviewed incumbent. Expected integration surfaces include REST and webhook APIs for ATS handoff (Workday, Greenhouse, Lever, iCIMS, Workable), Excalidraw or Mermaid-compatible diagram export for post-interview documentation, and Google Meet-style video integration. Compliance targets are SOC 2 Type II and WCAG 2.1 accessibility, both increasingly required by enterprise procurement.

---

## Market Context

The global technical assessment and interviewing software market is estimated at approximately $800 million in 2026, growing at 12–15% CAGR. Incumbent pricing ranges from free tiers (CodeInterview.io, Playcode) to per-interview credits (~$15–$30/session) to enterprise seat licensing ($5,000–$50,000+/year). Primary buyers are engineering hiring managers and recruiting coordinators at mid-size to large technology companies, technical recruiters at staffing agencies, and HR technology buyers seeking to standardise hiring workflows.

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
