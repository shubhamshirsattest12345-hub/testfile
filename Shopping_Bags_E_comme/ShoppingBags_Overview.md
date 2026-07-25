# Shopping Bags E-commerce Platform — Overview

**Dedicated B2B & B2C platform for customizable shopping bags**
Version 1.0 • July 2026

---

## 1. What the Application Does

A dedicated e-commerce platform for **customizable shopping bags** serving two audiences on one system:

- **B2C (retail):** shoppers configure individual bags — size, material, colour, handle type — upload a logo or artwork with a live print preview, and buy at retail pricing.
- **B2B (business):** verified business accounts place **bulk orders** through a dedicated portal with tiered pricing slabs, quote requests, negotiated rates, purchase-order/invoice payment terms, and one-click reorder.

A dedicated **real-time inventory service** keeps stock availability accurate at every step — including large-quantity validation for bulk orders — so neither channel can oversell.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **Next.js** (React) — SSR/ISR storefront, B2B portal, admin dashboard |
| Core backend | **Django** + Django REST Framework |
| Real-time microservice | **FastAPI** (async) — live inventory |
| Database | **PostgreSQL** (row-level locking for stock integrity) |
| Cache / messaging | Redis (stock cache, cart sessions, Celery broker, Pub/Sub) |
| Background jobs | Celery workers |
| File storage | S3-compatible object storage + CDN |
| External | Payment gateway (Stripe/Razorpay), shipping API, email/SMS, GST/tax service |

## 3. Architecture in Brief

The platform uses a **hybrid backend** behind an NGINX gateway:

1. **Client Layer (Next.js):** three surfaces — the SEO-optimised B2C storefront with the bag customizer, the B2B portal, and the admin dashboard with a live inventory monitor.
2. **Application Layer:**
   - **Django/DRF** owns core commerce: auth & B2B account verification, product catalog and customization options, orders/quotes/checkout, the tiered pricing engine, the customization (logo/print-proof) service, and Celery workers for emails and invoices.
   - **FastAPI** owns real-time inventory: an async live-stock API, WebSocket push of stock changes to storefront and admin, bulk availability checks for B2B quantities, and warehouse sync jobs. Django and FastAPI communicate over internal REST.
3. **Data Layer:** PostgreSQL is the single source of truth (with row-level locking on stock rows); Redis provides the hot stock cache, cart sessions, the Celery broker, and the inventory-event Pub/Sub channel; S3 + CDN hold product images, customer logo uploads, and generated proofs/invoices.
4. **External Services:** payment gateway, shipping & logistics API, email/SMS, and GST/tax compliance.

**Key idea:** commerce logic and inventory truth are separated. Django never guesses stock — every availability answer comes from the FastAPI service, which reads a Redis hot cache backed by PostgreSQL row-level locks, and pushes every change live over WebSockets.

## 4. Application Flow in Brief

1. A visitor lands on the Next.js storefront and the flow branches by **customer type**.
2. **B2C path:** select a bag → customize (size/colour/material) → upload logo with live preview → add to cart at retail pricing.
3. **B2B path:** log in to the verified business portal → configure a bulk order or quote (quantities, branding, delivery schedule) → the pricing engine applies tiered slabs and negotiated rates.
4. Both paths converge on the **FastAPI real-time inventory check**. In-stock quantities are **reserved with a hold**; shortfalls trigger a lead-time/backorder offer the customer can accept or decline.
5. Checkout (Django Orders API) → payment via gateway (with a retry loop on failure) → order confirmed → **inventory decremented and broadcast live** → Celery sends invoice and emails → print proofs → production → dispatch with tracking.
6. A reorder loop supports repeat purchases and one-click B2B reorders.

## 5. What Makes the Design Work

- **Right tool per job:** Django's mature ORM/admin for commerce; FastAPI's async performance for high-frequency stock reads and WebSocket push.
- **No overselling:** stock holds at checkout + PostgreSQL row-level locking + a single inventory authority.
- **B2B is first-class, not bolted on:** separate portal, tiered pricing engine, quotes, PO/invoice terms, bulk availability validation.
- **Async everything non-critical:** invoices, emails, print-proof rendering, and warehouse sync run on Celery so checkout stays fast.

---

*See `ShoppingBags_Detailed.md` for the complete component-by-component and step-by-step reference.*
