# Restaurant Online Web Application — Overview

**Highly scalable restaurant platform unifying online ordering and real-time table reservations**
Client: A. Centauri Software • Version 1.0 • July 2026

---

## 1. What the Application Does

A restaurant management platform that unifies **two customer journeys** in one system:

- **Online food ordering** — browse the live menu, build a cart with modifiers, and choose **pickup** (with a dynamic prep-time slot) or **delivery** (with zone check, fee, and rider tracking).
- **Real-time table reservations** — pick a date, time, and party size against a **live availability engine**; confirmed bookings block the table, send reminders, and release automatically on no-shows.

The customer web app, the restaurant dashboard (with a live **kitchen display system** and floor plan), and the admin console all operate on the same live state — an online booking, a walk-in seated by the host, and an order moving through the kitchen never contradict each other.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **Next.js** (React) — customer app, restaurant dashboard + KDS, admin console |
| Core backend | **Django** + DRF (**Python**) — menu, orders, payments, loyalty |
| Real-time service | **FastAPI** (async) — table availability, seat holds, order-status stream, ETAs, delivery tracking |
| Database | PostgreSQL — exclusion constraint against double-booked tables, partitioned orders, read replicas |
| Cache / messaging | Redis — availability cache, seat holds (TTL), Pub/Sub events, cart sessions, Celery broker |
| Workers | Celery — receipts, reservation reminders, reports |
| Files | S3 + CDN — dish photography, receipts |
| External | Payment gateway, delivery/logistics API, SMS/email/WhatsApp, maps/geocoding |

## 3. Architecture in Brief

1. **Client Layer (Next.js):** customer web app (menu, cart, orders, reservations, live tracking), restaurant dashboard + KDS (live order queue, prep status, floor plan and walk-ins), admin console (menus, pricing, locations, staff roles, promotions, analytics).
2. **Application Layer (Python hybrid):** NGINX gateway routing to —
   - **Django/DRF core:** auth & staff roles, menu & catalog with instant 86'ing (sold-out) propagation, the order lifecycle state machine, payments (checkout, deposits, refunds), loyalty & promotions, Celery workers.
   - **FastAPI real-time service:** the **table availability engine** (live slots + TTL seat holds), the **order-status WebSocket stream** connecting kitchen events to the customer's tracker, the **prep-time/ETA engine** driven by kitchen load, and delivery tracking relay.
3. **Data Layer:** PostgreSQL as the source of truth — a range **exclusion constraint** makes double-booking a table physically impossible; orders partitioned by month; replicas for reporting. Redis holds the live world: per-slot availability cache, seat holds, and all event Pub/Sub.
4. **External Services:** payment gateway (orders + reservation deposits), delivery/logistics API for rider dispatch and tracking, SMS/email/WhatsApp for confirmations and reminders, maps/geocoding for delivery zones.

**Key idea:** the same availability truth serves everyone. Whether a slot request comes from the website, a host seating a walk-in, or a no-show release, it resolves through one FastAPI engine backed by one PostgreSQL constraint — so overbooking can't happen by design rather than by discipline.

## 4. Application Flow in Brief

**Ordering path:** browse menu → pickup or delivery? → (delivery: address + zone/fee/ETA · pickup: prep-time slot) → checkout with coupons/loyalty → payment (retry on failure) → order hits the KDS (accepted → preparing → ready, streamed live to the customer) → counter handover with order code, or rider dispatch with live map tracking → completed, receipt + loyalty points.

**Reservation path:** select date/time/party size → real-time availability check → full? suggest nearby slots → available? **seat hold (TTL)** → confirm with guest details (optional peak-slot deposit) → SMS/WhatsApp confirmation + reminders → guest arrives? seated on the floor plan → dine, settle, table released — or grace-window **no-show release** frees the table.

Both paths converge on feedback and a loyalty nudge, with a return loop for reorders and rebookings.

## 5. What Makes the Design Work

- **No double-booking, structurally:** the PostgreSQL exclusion constraint is the last line of defence under any concurrency; Redis holds are just the fast path.
- **One status stream, three audiences:** the same kitchen events power the KDS, the customer tracker, and ops dashboards.
- **Honest ETAs:** prep-time quotes respond to actual kitchen load rather than a fixed constant.
- **Unified customer identity:** ordering and dining share one profile, so loyalty, history, and promotions work across both journeys.

---

*See `RestaurantOnline_Detailed.md` for the complete component-by-component and step-by-step reference.*
