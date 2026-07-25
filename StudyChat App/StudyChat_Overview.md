# StudyChat App — Overview

**Real-time collaborative study & AI-assisted chat platform**
Version 1.0 • July 2026

---

## 1. What the Application Does

StudyChat is a real-time collaborative study platform that combines peer-to-peer group chat with an integrated AI study assistant. Students create or join topic-based study rooms, exchange messages and study material in real time, and can address questions directly to an AI assistant (`@AI`) for explanations, summaries, and practice quizzes.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Web frontend | React.js + Vite, Tailwind CSS, Redux Toolkit, Socket.IO client |
| Mobile frontend | React Native (iOS / Android), offline cache, push notifications |
| Backend | Node.js + Express.js behind an API gateway (JWT, rate limiting, CORS) |
| Real-time | Socket.IO with Redis Pub/Sub adapter |
| Database | MongoDB Atlas via Mongoose ODM |
| Cache / sessions | Redis |
| File storage | AWS S3 + CloudFront CDN |
| AI | LLM API (OpenAI / Claude) |
| Notifications | Firebase Cloud Messaging (push), SendGrid/SMTP (email) |

## 3. Architecture in Brief

The system is organised into **four layers**:

1. **Client Layer** — a React web app and a React Native mobile app. Both talk to the backend over HTTPS (REST) for standard operations and secure WebSockets (Socket.IO) for live messaging.
2. **Application Layer** — a Node.js + Express backend fronted by an API gateway that enforces JWT verification, rate limiting, and CORS. Behind it sit five domain services (Auth, Chat, Study Room, AI Assistant, Notification) plus the Socket.IO real-time engine.
3. **Data Layer** — MongoDB Atlas as the primary store, Redis for sessions/cache/presence and as the Socket.IO scaling adapter, and S3 for file attachments.
4. **External Services** — the LLM API that powers the AI assistant, Firebase Cloud Messaging for push, and SendGrid/SMTP for email.

**Key idea:** every message travels through one pipeline — persist to MongoDB, broadcast via Socket.IO — whether it comes from a human peer or from the AI assistant. The AI is just another participant whose replies are produced by the AI Assistant Service calling the LLM API.

## 4. Application Flow in Brief

1. User opens the app; a session/JWT check decides between the dashboard and the login/registration screen (with retry on failed authentication).
2. From the dashboard the user joins or creates a **study room**; Socket.IO subscribes them to the room channel.
3. The user sends a message. A single decision routes it:
   - **Peer message** → saved to MongoDB → broadcast to room members via Socket.IO → offline members get an FCM push.
   - **@AI query** → AI Assistant Service builds a prompt, calls the LLM API, and injects the formatted reply into the same persist-and-broadcast pipeline.
4. The loop continues until the user logs out or closes the app.

## 5. What Makes the Design Work

- **One gateway, one entry point** — all traffic (REST and WebSocket) is authenticated and rate-limited in a single place.
- **Redis Pub/Sub adapter** — lets multiple Socket.IO server instances stay in sync, so the platform scales horizontally.
- **AI as a first-class participant** — no separate AI pipeline; assistant replies reuse the ordinary message flow, which keeps history, receipts, and notifications consistent.
- **Offline continuity** — push notifications and the mobile offline cache keep users in the loop when disconnected.

---

*See `StudyChat_Detailed.md` for the complete component-by-component and step-by-step reference.*
