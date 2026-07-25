# Creative Delivery Web Application Platform — Overview

**Highly scalable, real-time courier marketplace with multi-tiered delivery**
Client: A. Centauri Software • Version 1.0 • July 2026

---

## 1. What the Application Does

A real-time courier service platform connecting **customers** who need packages delivered with **independent contractor couriers**. Deliveries come in **multiple tiers** — Express, Same-Day, Scheduled — priced instantly from distance, tier rates, and demand. A dispatch engine matches every booking to the best nearby online courier within seconds, and live GPS tracking keeps the customer, courier, and operations team synchronised from pickup through proof of delivery.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **React** — customer app, courier/contractor app, ops/admin console |
| Core backend | **Django** + DRF (**Python**) — accounts, bookings, pricing, payments, ratings |
| Real-time service | **FastAPI** (async) — dispatch/matching, WebSocket tracking, ETA, presence |
| Database | PostgreSQL + PostGIS — geo indexes, partitioned delivery history, read replicas |
| Cache / messaging | Redis — GEO sets of live couriers, Pub/Sub, offer locks, Celery broker |
| Workers | Celery — invoices, payout batches, notifications |
| Files | S3 + CDN — POD photos/signatures, KYC docs, invoices |
| External | Payment gateway (+ payout rails), maps/routing API, push/SMS/email, KYC checks |

## 3. Architecture in Brief

1. **Client Layer (React):** the customer app (booking, tier/quote selection, live tracking map), the courier app (go online, job offers, navigation, GPS streaming, proof of delivery, earnings), and the ops console (live fleet map, dispatch overrides, contractor onboarding, disputes).
2. **Application Layer (Python hybrid):** an NGINX gateway routes between two backends —
   - **Django/DRF** for the transactional core: accounts & contractor KYC onboarding, the order/booking lifecycle state machine, the pricing & tier engine, payments & payouts, ratings & disputes, and Celery workers.
   - **FastAPI** for everything real-time: the **dispatch/matching engine** (Redis GEO nearest-courier search with ranked offer waves), **live tracking** WebSockets (courier GPS in, customer map out), ETA/route computation, and courier presence.
3. **Data Layer:** PostgreSQL + PostGIS as the source of truth (deliveries partitioned by month, read replicas for history/analytics); Redis holding the live world (online-courier GEO sets, Pub/Sub fan-out, offer locks); S3 + CDN for proof-of-delivery media and documents.
4. **External Services:** payment gateway for customer charges and contractor payout rails, maps/routing/geocoding, push/SMS/email, and KYC/background checks.

**Key idea:** the durable system of record (Django + PostgreSQL) is separated from the live world (FastAPI + Redis). Courier positions and job offers live in Redis for sub-second matching; every state change that matters commercially — assignment, pickup, delivery, capture, payout — is written back to PostgreSQL through the order state machine.

## 4. Application Flow in Brief

1. **Customer path:** enter pickup & drop → choose tier (Express/Same-Day/Scheduled) → instant quote → confirm & authorise payment.
2. **Courier path:** one-time contractor onboarding (KYC, vehicle docs) → go online → GPS streams into the Redis GEO presence set.
3. **Dispatch:** the matching engine finds and ranks nearby couriers, sending a locked, time-limited **job offer** to the best one; timeouts rebroadcast to the next courier in waves.
4. **Execution:** courier navigates to pickup → OTP-verified handover → status `IN_TRANSIT` → **live WebSocket tracking** with refreshed ETAs → drop-off.
5. **Completion:** successful handover captures **proof of delivery** (photo/signature/OTP → S3); payment is captured and the courier's payout accrues; both sides rate each other. Failed handovers route through an exception flow (retry / return to sender / support).
6. **Loops:** the customer books again; the courier stays online for the next offer.

## 5. What Makes the Design Work

- **Sub-second matching at scale:** Redis GEO queries against live courier positions avoid touching PostgreSQL on the hot dispatch path.
- **No double-assignment:** offer locks guarantee a job is offered to exactly one courier at a time, with clean wave escalation on timeout.
- **Financial safety:** authorise-at-booking / capture-on-delivery means customers only pay for completed deliveries; payouts accrue against verified POD.
- **Ops visibility:** the same tracking stream that powers the customer map powers the fleet view and dispatch overrides.

---

*See `CreativeDelivery_Detailed.md` for the complete component-by-component and step-by-step reference.*
