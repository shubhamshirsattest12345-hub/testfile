# Rent Property Listing Web Application — Overview

**High-volume property rental marketplace with a credit-based lead model**
Client: A. Centauri Software • Version 1.0 • July 2026

---

## 1. What the Application Does

A property rental marketplace connecting **renters** with **property listers** (owners and agents) at high traffic volume. Its commercial core is a **credit-based lead generation and consumption model**:

- Renters browse and search listings freely; owner contact details are **masked**.
- When a renter submits an inquiry, it becomes a **lead** — deduplicated, quality-scored, and delivered to the lister's inbox with contacts still hidden.
- The lister spends **wallet credits** to unlock a lead and reveal the renter's contact details.
- Credits are bought as **packs** through integrated payment gateways; every credit movement is recorded in an append-only ledger.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **Next.js** (React) — renter app, lister portal, admin console |
| Backend | **FastAPI** (async Python), NGINX + Uvicorn, Pydantic validation, JWT/OTP auth |
| Database | **PostgreSQL**, deliberately optimised (see below) |
| Cache / queues | Redis — search cache, sessions, OTP, rate limits, worker queues, lead dedupe |
| Workers | Celery/ARQ — lead scoring, listing expiry, image processing, digests |
| Files | S3-compatible storage + CDN (property photos, documents, invoices) |
| Payments | Stripe / Razorpay — credit packs, webhooks, refunds, GST invoices |
| Other external | Maps/geocoding API, email/SMS/WhatsApp, KYC/verification provider |

## 3. Architecture in Brief

1. **Client Layer (Next.js):** three surfaces — the renter/seeker app (search, map, galleries, inquiry forms), the lister portal (listings, credit wallet, lead inbox), and the admin console (moderation, credit plans, lead-quality and fraud review).
2. **Application Layer (FastAPI):** a single async backend behind an NGINX/Uvicorn gateway with JWT middleware and rate limiting. Seven services: Auth & Profiles, Listing, Search, **Lead Service** (capture, dedupe, scoring, contact masking), **Credit Wallet Service** (atomic debits, ledger), Payment, and Notification — plus background workers.
3. **Data Layer:** PostgreSQL as the source of truth with an explicit **database optimization** story: PostGIS geo indexes, composite/partial indexes, monthly partitioning of the leads table, PgBouncer pooling, read replicas for search traffic, and row-level locking on wallet debits. Redis absorbs hot reads; S3 + CDN serve photos.
4. **External Services:** payment gateway, maps/geocoding, email/SMS/WhatsApp, KYC verification.

**Key idea:** the lead is the product. The Lead Service and Credit Wallet Service form one transactional loop — a lead unlock is an atomic debit + reveal, so a lister can never be charged without receiving the contact, and never receive it without being charged.

## 4. Application Flow in Brief

1. Users sign in (JWT/OTP) and the flow branches by **role**.
2. **Renter path:** search with geo + filters → view listing (contacts masked) → submit inquiry → **lead created** (deduped, scored).
3. **Lister path:** post a listing (moderated, then live) → hold a credit wallet (starter or purchased credits) → wait for inquiries.
4. Paths converge when a lead lands: the lister is **notified** (email/SMS/WhatsApp) and decides whether to unlock.
5. **Credit gate:** sufficient balance → **atomic debit** → contact revealed → lister contacts renter and schedules a visit. Insufficient balance → **buy a credit pack** (payment with retry loop, webhook-confirmed) → back to the gate.
6. If the property rents, the listing is marked rented/delisted and the lead marked converted; otherwise the listing stays live for the next lead.

## 5. What Makes the Design Work

- **Monetisation is transactionally safe:** debit-and-reveal happens under a row-level lock with a ledger entry — no double-charges, no free unlocks, fully auditable.
- **Built for read volume:** replicas + Redis caching keep the search path fast while the primary handles writes.
- **Lead quality is protected:** dedupe (renter × listing), scoring, and admin fraud review keep paid leads worth paying for.
- **Async where it counts:** FastAPI's async I/O suits the traffic profile (many small search/lead requests), and workers keep slow jobs out of request paths.

---

*See `RentProperty_Detailed.md` for the complete component-by-component and step-by-step reference.*
