# Enterprise AI Policy Chatbot — Overview

**A grounded conversational agent for internal policy inquiries**
Next.js • OpenAI SDK • Netlify • Version 1.0 • July 2026

---

## 1. What the Application Does

Employees ask policy questions in natural language — *"How many WFH days am I allowed?"*, *"What's the parental-leave process?"* — and get answers **grounded in the company's actual policy documents, with citations** to the source sections.

- Built with the **Next.js framework**: the chat UI, the admin view, and the API layer live in a single codebase.
- The **OpenAI SDK** powers both halves of the AI: **embeddings** to index policy documents and match questions to relevant sections, and **chat completions** (streamed) to compose grounded answers.
- Deployed on **Netlify**: the app is served from the edge CDN, and the Next.js API routes run as serverless functions with Git-triggered CI/CD, deploy previews, and instant rollback.
- Honesty is a feature: if no policy content matches the question above a relevance threshold, the bot says so and **escalates to HR** instead of guessing — and the question is logged as a potential policy gap.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Framework | **Next.js** (React) — chat UI + admin view + API routes |
| AI | **OpenAI SDK** — embeddings (indexing + query) and streamed chat completions |
| Knowledge | Policy docs chunked by section → vector store with metadata (policy, section, version, effective date) |
| Auth | Enterprise SSO (Okta / Azure AD) → session JWT, employee/admin roles |
| Hosting | **Netlify** — edge CDN + serverless functions, env secrets, atomic deploys, rollback |
| Feedback | 👍/👎 per answer, unanswered-question queue, HR escalation tickets |

## 3. Architecture in Brief

1. **Client (Netlify Edge CDN):** the employee chat interface (conversation UI, suggested questions, answers with citations, feedback buttons, escalate-to-HR) and the policy admin view (document upload/update, re-index trigger, unanswered-question review, analytics).
2. **Application (Netlify Serverless Functions):** Next.js API routes —
   - **`/api/chat` RAG pipeline ★:** embed the query → top-k semantic retrieval of policy chunks → grounded OpenAI completion with citations.
   - **`/api/ingest` knowledge pipeline:** parse → chunk by section → embed → upsert vectors with metadata.
   - **Auth middleware** (SSO session + roles), **Guardrails ★** (policy scope only, no legal/HR advice, low-confidence → escalate), the **OpenAI SDK client** (streaming, timeouts, retries), and the **conversation & feedback logger**.
3. **Data & AI:** the vector store (retrieval), the versioned policy document source, the transcript/feedback/escalation logs, and the OpenAI API.
4. **Deployment:** Git push → Netlify build (API routes packaged as functions; `OPENAI_API_KEY` and SSO config as environment secrets) → edge CDN + auto-scaling functions → atomic deploys with one-click rollback.

**Key idea:** the bot never answers from the model's general knowledge — it answers from *retrieved policy text or not at all*. The relevance threshold turns "I'm not sure" into a designed outcome (honest refusal + HR escalation + a logged policy gap) rather than a hallucination.

## 4. Application Flow in Brief

**Ingestion (admin, on upload/update):** policy documents → parse & chunk by section (with version metadata) → OpenAI embeddings → vector store. The knowledge base is refreshed without touching application code.

**Runtime loop:** employee opens the chatbot → SSO check (redirect to login if needed) → asks a question → `/api/chat` validates the session → embeds the query and retrieves top-k policy chunks → **relevance gate**: match found → streamed OpenAI completion grounded in the chunks → answer rendered **with policy citations**; no match → "not covered" + HR escalation + policy-gap log → 👍/👎 feedback (👎 opens an escalation ticket) → next question or end.

## 5. What Makes the Design Work

- **Citations build trust:** every answer links to the policy section it came from, so employees can verify — critical for HR-adjacent questions.
- **The gap loop improves the product:** unanswered and thumbs-down questions form a queue that tells the policy team exactly what's missing or unclear.
- **One codebase, zero servers:** Next.js on Netlify means the chat UI, the RAG backend, and the deployment pipeline are one repository with no infrastructure to babysit — functions scale with usage and cost nothing idle.
- **Enterprise-fit by default:** SSO-gated access, role separation for admins, PII-minimised logging, and secrets kept in the platform rather than the repo.

---

*See `PolicyChatbot_Detailed.md` for the complete component-by-component and step-by-step reference.*
