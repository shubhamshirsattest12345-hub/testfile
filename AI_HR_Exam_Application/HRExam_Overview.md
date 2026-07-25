# AI HR Exam Application — Overview

**A multi-agent AI system on Django for automated HR testing and reporting**
Employees • Freshers • Interview Candidates • Version 1.0 • July 2026

---

## 1. What the Application Does

A complete assessment platform that automates the HR testing lifecycle with a **pipeline of five AI agents**:

- **HR admins** define an assessment — role, skills, difficulty, duration, question mix — for one of three audiences: **current employees** (skills/compliance testing over the SSO roster), **freshers** (aptitude + fundamentals for campus batches), or **interview candidates** (role-specific screening with per-candidate links).
- The **agent pipeline** generates the questions, reviews and calibrates them, and — after HR approval — the exam is published.
- Candidates take a **timed, autosaving exam** (SSO for employees; single-use tokenized links for externals).
- On submission, agents **grade everything** (MCQs auto-scored; subjective and coding answers graded against LLM rubrics), **analyse integrity** (flagged attempts go to human review), and **compile insight reports** — scores, rankings, skill gaps, and hire/train recommendations — delivered to dashboards and pushed to HRIS/ATS.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Backend | **Django** + DRF (Python) — the application backbone |
| Multi-agent AI | 5 LLM-powered agents orchestrated as **Celery pipelines** over the LLM API |
| Identity | **Enterprise SSO** (SAML/OIDC — Azure AD/Okta), auto-provisioning, RBAC + org hierarchy, audit trail |
| External access | Single-use, expiring **tokenized exam links** for freshers & interviewees |
| Database | PostgreSQL — question bank, assessments, partitioned attempts, results, integrity events |
| Cache / queues | Redis — live exam timers, autosave buffer, Celery broker, token single-use locks |
| Files | S3 — report PDFs, code artifacts, integrity evidence |
| External | LLM API (OpenAI/Claude), IdP, email, HRIS/ATS webhooks |

## 3. Architecture in Brief

1. **Client Layer:** the exam portal (all three audiences), the HR admin console (assessment creation, AI-question approval, integrity reviews), and the manager/recruiter dashboard (results, rankings, skill-gap reports, exports).
2. **Application Layer (Django + DRF):** an NGINX-fronted core in two halves plus the centrepiece —
   - **Enterprise Identity & User Management:** SSO with employee auto-provisioning, RBAC over an org hierarchy, and tokenized external-candidate access.
   - **Assessment & Exam Session Engine:** audience blueprints, question bank, scheduling/invitations; server-side timers, answer autosave, question shuffling, auto-submit on expiry.
   - **Multi-Agent AI Orchestrator ★:** five agents — Question Generator, Reviewer/Calibration, Evaluation, Integrity/Proctoring, Reporting & Insights — run as Celery pipelines against the LLM API.
3. **Data Layer:** PostgreSQL as truth (including the audit log), Redis for live exam state and queues, S3 for reports and evidence.
4. **External:** LLM API, enterprise IdP, email/notifications, HRIS/ATS integration.

**Key idea:** AI does the labour, humans keep the authority. Every agent output passes a human gate where it matters — HR approves generated questions before publication, and integrity flags route to human review before an attempt is voided — so automation accelerates the process without removing accountability.

## 4. Application Flow in Brief

1. **Create:** HR defines the assessment → the audience decision selects the employee / fresher / interview blueprint.
2. **Generate:** Question Generator drafts items → Reviewer/Calibration agent validates and tunes → **HR approval gate** (edit/regenerate loop) → published to the question bank.
3. **Invite:** employees get SSO links from the org roster; externals get single-use tokens. The identity gate rejects used/expired tokens.
4. **Examine:** timed session with autosave, shuffling, and integrity-signal logging → submit or auto-submit on expiry.
5. **Grade:** Evaluation Agent scores MCQs automatically and grades subjective/code answers by rubric.
6. **Verify:** Integrity Agent flags suspicious attempts → HR review accepts or voids with evidence attached.
7. **Report:** Reporting & Insights Agent compiles scores, rankings, skill gaps, and hire/train recommendations → dashboards, email, HRIS/ATS → loop to the next cycle or batch.

## 5. What Makes the Design Work

- **Three audiences, one system:** blueprints and access methods differ (SSO roster vs token links), but the pipeline, grading, and reporting are shared — no parallel tools to maintain.
- **Agents check agents, humans check agents:** the Reviewer catches the Generator's mistakes; HR approval and integrity review keep humans in the loop at the two decisions that carry consequences.
- **Fairness at exam time:** server-side timers, shuffled questions, single-use tokens, and integrity signals make results comparable and defensible.
- **Reports that end in a decision:** output isn't just a score — it's a ranked, skill-gapped, recommendation-bearing report pushed straight into the HR systems where the decision happens.

---

*See `HRExam_Detailed.md` for the complete component-by-component and step-by-step reference.*
