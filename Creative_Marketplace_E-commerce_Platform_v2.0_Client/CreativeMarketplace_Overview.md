# Creative Marketplace E-commerce Platform v2.0 — Overview

**Multi-vendor digital marketplace connecting independent creators with global buyers**
Client: A. Centauri Software • Version 2.0 • July 2026

---

## 1. What the Application Does

A multi-vendor marketplace for **digital products** — design assets, templates, fonts, audio, and similar creative work:

- **Creators** onboard once (KYC + payout account), upload products into a private asset vault, choose pricing and license tiers, and publish through a moderation + malware-scan pipeline into search.
- **Buyers** worldwide discover products through faceted search and CDN previews, purchase in their **own currency** with region-correct tax/VAT, and receive **secure digital delivery**: a license key plus signed, time-limited, download-limited URLs in their library.
- Every sale is **split at payment time** — platform fee vs creator earnings — recorded in an append-only ledger and settled through scheduled payout batches via Stripe Connect.

## 2. Technology Stack at a Glance

| Layer | Technology |
|---|---|
| Frontend | **Next.js** (React) — buyer marketplace, creator studio, admin console |
| Backend | **FastAPI** (async **Python**) — the entire service layer, NGINX + Uvicorn, JWT, Pydantic |
| Database | PostgreSQL — partitioned orders, licenses, append-only earnings ledger, read replicas |
| Search | OpenSearch — full-text + facets + trending, fed by indexing workers |
| Cache / queues | Redis — sessions, carts, hot product cache, download-limit counters, queues |
| Workers | ARQ/Celery — malware scans, preview generation, indexing, payout batches, email |
| Files | S3 private vault + CDN for public previews/thumbnails |
| External | Stripe Connect (split payments + payout rails), tax/VAT service, email, KYC/fraud & content scan |

## 3. Architecture in Brief

1. **Client Layer (Next.js):** buyer marketplace (discovery, previews, multi-currency checkout, download library), creator studio (uploads & versioning, storefront branding, sales analytics, earnings dashboard), admin console (moderation, vendor approvals, commission plans, disputes, fraud review).
2. **Application Layer (FastAPI):** one async backend with eight services — Auth & Identity, Vendor/Creator, Product Catalog, Search & Discovery, **Checkout & Split Payments ★**, **Digital Delivery & Licensing ★**, Payout, and Reviews & Notifications — plus background workers for upload scanning, preview generation, indexing, payouts, and email.
3. **Data Layer:** PostgreSQL as commercial truth (including the earnings ledger); OpenSearch as the discovery index; Redis for carts, hot caches, and **download-limit counters**; S3 split into a **private vault** (paid files) and public preview storage behind the CDN.
4. **External Services:** Stripe Connect for global checkout, split payments, and creator payout rails; a tax/VAT service for cross-border digital-goods compliance (EU VAT, GST); email; KYC/fraud and content scanning.

**Key idea:** in a digital marketplace the product *is* a file and the revenue *is* a split — so the two starred services are the core. Delivery converts a paid order into a license plus expiring signed URLs (the vault is never public), and the split is computed and ledgered at the moment of payment, so creator earnings are always reconstructible and refund clawbacks are just compensating ledger entries.

## 4. Application Flow in Brief

1. **Creator path:** onboard (KYC + Stripe Connect) → upload files, price, license terms → async malware scan + preview generation → moderation gate (rejected work loops back with feedback) → published and indexed into search.
2. **Buyer path:** browse/search with facets → product page with CDN previews and reviews → pick a license tier → cart → **multi-currency checkout** with region tax → payment (retry loop; credited only on webhook).
3. **Post-payment:** split recorded (fee + earnings ledger entry) → **license key issued + signed limited download URLs** → buyer downloads from their library (Redis-enforced limits).
4. **Quality loop:** files as described → verified-purchase review boosts ranking; problems → dispute/refund flow with earnings clawback.
5. **Settlement:** scheduled payout batches transfer net earnings to creators with a notice.
6. **Loops:** buyers browse again; creators upload the next product.

## 5. What Makes the Design Work

- **The vault stays sealed:** paid files live in private S3 and are only ever reachable through short-lived signed URLs tied to a valid license — no public bucket, no leaked permanent links.
- **Money is an append-only story:** the earnings ledger (sale, fee, refund clawback, payout) makes every creator balance auditable and dispute-proof.
- **Global by default:** multi-currency pricing and delegated tax/VAT computation make cross-border sales a first-class case, not a patch.
- **Trust compounds:** KYC'd creators, scanned uploads, moderated listings, and verified-purchase reviews keep marketplace quality — the real product — high.

---

*See `CreativeMarketplace_Detailed.md` for the complete component-by-component and step-by-step reference.*
