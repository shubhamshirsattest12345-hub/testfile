# LinkedIn Profile Assistance Bot — Overview

**A grounded AI chat assistant for one professional profile**
React • FastAPI • Gemini • Terraform → Docker → Google Cloud Run • Version 1.0 • July 2026

---

## 1. What the Application Does

A **LinkedIn Personal Info Bot**: visitors chat with an assistant that answers questions about one person's professional profile — experience, projects, skills, education.

- The **React frontend** is a chat interface: message thread, input box, typing indicator, client-side history.
- The **FastAPI backend** exposes a **single `/chat` endpoint**. At startup it loads a **local portfolio JSON** file, and every request calls **Gemini** with a **strict system prompt** so the response is grounded *only* in that JSON.
- Questions outside the portfolio get a **polite, scoped refusal** ("I can only answer questions about this profile") — never an invented answer.
- **Infrastructure is fully codified with Terraform**, which builds/pushes the Docker image and deploys the backend on **Google Cloud Run**.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **React** SPA — chat UI, static build (CDN/static hosting) |
| Backend | **FastAPI** (Python) — single `POST /chat`, Pydantic, CORS, rate limiting |
| AI | **Google Gemini** (google-generativeai SDK) with timeout/retry + safety settings |
| Knowledge | `portfolio.json` bundled in the Docker image, loaded at startup |
| Secrets | GCP **Secret Manager** → `GEMINI_API_KEY` env var (never in code/image) |
| IaC | **Terraform** — APIs, Artifact Registry, Secret Manager, IAM, Cloud Run |
| Runtime | **Docker** container on **Cloud Run** — HTTPS, scale-to-zero autoscaling, revisions |

## 3. Architecture in Brief

Four lean layers:

1. **Client:** the React chat SPA, keeping conversation history in client state and sending `{ message, history }` to the backend.
2. **Application (Cloud Run):** one FastAPI container with six components — Request Handling (Pydantic/CORS/rate limit), **Portfolio Data Loader** (JSON parsed once at startup), **Grounded Prompt Builder ★** (system prompt + JSON + question), **Gemini Client** (SDK, timeouts, retries), **Grounding Guardrails ★** (scope enforcement → refusal), and the Response Composer (`{ reply }`, clean error mapping).
3. **AI & Secrets:** the Gemini API outside the container boundary; Secret Manager injecting the API key at deploy time.
4. **Infrastructure:** the Terraform pipeline — provision → build/push Docker image to Artifact Registry → deploy Cloud Run with secret wiring and least-privilege IAM.

**Key idea:** the bot's entire knowledge is one JSON file, and the architecture enforces that honestly. Grounding isn't a hope — it's a strict system prompt at generation time plus guardrails on the way out, so the assistant is precise about the profile and silent about everything else.

## 4. Application Flow in Brief

**Deployment (one-time / per release):** `terraform apply` → Docker image (FastAPI + portfolio.json) built and pushed to Artifact Registry → Cloud Run deploys it with `GEMINI_API_KEY` from Secret Manager → backend loads the portfolio at startup, ready.

**Runtime loop:** visitor opens the chat → types a question → `POST /chat` → request validation (invalid → clean 4xx message in the UI) → grounded prompt built (system prompt + portfolio JSON + question) → Gemini called with timeout/retry (API failure → graceful "try again") → **grounding decision**: answerable from the portfolio → grounded answer; out of scope → polite scoped refusal → reply rendered in the thread → next question, or close the chat.

## 5. What Makes the Design Work

- **Hallucination is designed out, not patched out:** the model only ever sees the portfolio JSON as its knowledge, and off-topic paths terminate in a refusal template.
- **One endpoint, zero database:** the entire knowledge source ships inside the image — nothing to migrate, back up, or drift; updating the profile is a rebuild + redeploy.
- **Reproducible infrastructure:** `terraform apply` recreates the whole environment; Cloud Run revisions give instant rollback.
- **Costs match traffic:** scale-to-zero means a personal profile bot costs nearly nothing while idle and scales automatically if a post goes viral.

---

*See `LinkedInBot_Detailed.md` for the complete component-by-component and step-by-step reference.*
