# Appendix — Notion Source Context

> **Confidential — shared for proposal evaluation.**
>
> **What this is.** This appendix compiles the substance of several internal Blue Delta planning documents and meeting notes (originally hosted in Notion) so this documentation bundle is fully self-contained and requires no Notion access. These are **point-in-time summaries from roughly May–June 2026** — design docs, audit decisions, feature-ideation lists, and meeting notes — reproduced for context. They are **not live documents** and may have been superseded by later decisions. Customer PII has been scrubbed; only synthetic examples appear. No secret values are reproduced.
>
> For the structured, current treatment of the Jeanius initiative — including the draft data model and the integration contract a data-writer must conform to — see [`../05-jeanius.md`](../05-jeanius.md). This appendix is the underlying raw context behind that document.

Contents:

1. [Jeanius / Genius 2.0 — design, scope, audit, ideation, dev-planning](#1-jeanius--genius-20)
2. [Tom James order-entry automation — the canonical "tool that writes data into orders"](#2-tom-james-order-entry-automation)
3. [Database / agency-handoff preference](#3-database--agency-handoff-preference)
4. [Tech-stack inventory, order flow & pain points (OpenTeams discovery)](#4-tech-stack-inventory-order-flow--pain-points)
5. [Bold Metrics → Skynet migration brief (now complete)](#5-bold-metrics--skynet-migration-brief)

---

## 1. Jeanius / Genius 2.0

Jeanius (working name "Genius 2.0") is the planned rebuild of the legacy **Solid Commerce "Genius"** iPad event order-entry system. See [`../05-jeanius.md`](../05-jeanius.md) for the consolidated version of this material.

### 1.1 Phase 1 design doc + functional scope

**Why rebuild instead of patch.** The current Genius system carries roughly **13,000 hand-built SKUs that are out of sync**. Rebuilding is judged to beat patching the existing system.

**Real stakeholders.** The pattern team — **Johnson, Bud, and Andy** — are the true stakeholders for this work (not Blake, as had been assumed earlier).

**v1 product scope.** Pants, belts, and gift cards.

**User flows (v1):**
- Event rep entering an order on behalf of a customer.
- Retail walk-in.
- Corporate CSV → QR pre-fill (a corporate list is imported ahead of an event and customers get a pre-filled QR to scan).
- QR lobby self-serve (customer self-registers in the lobby via QR).

**Order submission.** Orders submit **directly to Shopify**, riding the **same Shopify Flow tagging + Zapier→Asana bridge** that web orders use. In effect, **Jeanius becomes a Shopify sales channel** rather than a separate order silo. (This is the key architectural decision — it means a Jeanius order is indistinguishable downstream from a web order and inherits the existing pipeline.)

**Operational requirements:**
- Admin control of **required fields per event**.
- An `info+{n}@bluedelta.com` email workaround to handle duplicate customer emails (Shopify treats email as a customer key).
- Pre-event **CSV import**.
- Event sizes to support: ~200, ~400–500, and ~1,200 attendees.

**Measurement unification.** Merge three measurement sources into **one universal customer profile**:
- Genius **15** hand measurements,
- Virtual Tailor **5** inputs,
- **Bold Metrics** outputs,

…keyed by a unique identifier (**"BDID"**), with **Skynet-style discrepancy flagging** across the merged values. Two governance rules: **FIT is per-customer-account**; **BREAK is per-product**.

**Platform.** iPad primary. **Offline mode required.**

**Integrations.** Shopify Admin/Order API, Bold Metrics, Skynet.

**Dependency note.** Jeanius cannot be built in isolation from the custom-database project and the Klaviyo project; they are intertwined.

### 1.2 Jeanius Rebuild Audit (May 14, 2026) — decisions

- Real stakeholders confirmed: **Johnson + Bud + Andy** (not Blake).
- Unify measurements into **one universal profile** with **Skynet flagging**.
- **Fit per-account, Break per-product** — with an open question on whether **Break** is used by patterning.
- **Jeanius = Shopify sales channel.**
- v1: **pants, belts, gift cards.**
- **iPad primary; offline required.**
- **Admin control of required fields.**
- **No event-size cap.**
- Measurement counts: **Genius 15, Virtual Tailor 5, base 16.**
- **Actions:** hold a working session with Johnson + Bud; confirm patterning fields; scope Shopify field workarounds; build the design doc + QR pre-registration; decide whether to migrate vs. rebuild the **Solid Commerce Product Options POS-monogram** capability.

### 1.3 Feature ideation

Grouped wishlist (aspirational — not all v1):

- **Speed / line-management:** pre-event import, QR check-in, returning-customer lookup, skip-ahead, queue dashboard.
- **Measurement intelligence:** photo-assisted measurement, confidence flags, voice input, group linking.
- **Customer experience:** second screen, swatches, recommender, AR, instant confirmation.
- **Team coordination:** multi-rep sync, lead dashboard, metrics, help flag.
- **Data integrity:** offline-first, validation, bulk edit before the Shopify push, event reporting.
- **Inventory:** live fabric availability, lead times, substitution suggestions.

Proposes a **persistent customer record (possibly Supabase)** that follows the person across events / retail / B2B and feeds Shopify, with **per-rep login + audit log + GDPR deletion**.

### 1.4 Development Planning Meeting (May 19, 2026)

- Build a **modular system** that routes to **Asana today, a database tomorrow**.
- Genius acts as a **portal/router for ALL orders**, including **Tom James**.
- Need a **unique customer ID (BDID)**.
- Support **group orders**; **track who measured each order**.
- **v1:** real-time Skynet validation; required-field enforcement; event-wide customization assignment (pocket bags, monograms, bag prints, heat-transfer labels) with **retroactive add**; manager view; event templates; QR pre-fill.
- **v2:** returning-customer autofill; voice input; offline native app.
- **AWS:** S3 for pattern storage; **Textract** OCR for Tom James PDFs.
- **Cost note:** Zapier is now **> $500/mo** (n8n floated as an alternative).
- **Shopify variant limits:** 100 (old theme), 2,048 (new theme) — intend to keep under **~1,000**.

---

## 2. Tom James order-entry automation

> This is the **canonical example of "a tool that writes data into Jeanius-created orders."** The integration contract in [`../05-jeanius.md`](../05-jeanius.md) (the order-data shape, idempotency, and "must-not-override" rules) is written so that an external writer like this one conforms cleanly to the pipeline.

**Source:** Esther / Tom James order-entry automation notes (May 22, 2026).

A full-stack contractor is building an order-entry automation that **parses Tom James orders and writes them into the pipeline** — a multi-purpose tool intended eventually to **replace Asana with a real database**.

**Input characteristics.** Tom James (TJ) orders arrive as **scanned PDFs** — a mix of typed and handwritten content.

**Flow (today):**

1. Extract the PDF.
2. **AWS Textract** OCR → JSON.
3. Land in a small AWS DB / S3, **deduplicating by order number**.
4. Push into **Asana** (the present endpoint).

**Future direction:**

- A **full custom AWS database** (orders / inventory / customers).
- **Database-first**, with **Asana demoted to a severable endpoint**.
- A new **electronic order-entry app** replaces the paper process.

**Surrounding facts captured in the same notes:**

- **Asana is a makeshift database for everything** today.
- There is **no customer database**.
- Zapier requires **manual SKU fixes** across the ~13,000 SKUs.
- Catalog scale: **~23 online products / 953 variants** plus **~91 POS products / ~7,000 variants**.
- There is a plan to **merge the online + POS product sets within ~6 months**. (See [`../04-website-vs-pos-products.md`](../04-website-vs-pos-products.md) for the online-vs-POS split.)

---

## 3. Database / agency-handoff preference

**Source:** Vendor Call (June 23, 2026) — agency/database handoff preference.

**Stated priorities, in order:**

1. **Tom James automation** — nearly ready.
2. **Genius rebuild** — frontend mostly specced in Figma; a React component library is being built; help is needed on **backend/API hookup to Shopify**, possible **POS/payments (Stripe)**, and a **security layer**.
3. **Database** — an internal developer knows the intended table/column structure and has used **Supabase at small scale**; the team wants **production-scale support**.

**Already completed (as of the call):**

- **Bold Metrics integration shipped** (see §5).
- **Skynet API integration** — the site-facing API was removed, runs are now automatic, and a cart race condition was fixed.
- **Figma component library.**
- **Obsidian knowledge base on GitHub.**

**Critical preference (important for any engaging agency):** the team wants to **design the database schema internally and hand off only the build**. Their reasoning is that outside agencies lack internal context — especially for database work — so schema design stays in-house and the implementation/build is the work to be contracted.

---

## 4. Tech-stack inventory, order flow & pain points

**Source:** OpenTeams discovery (June 18, 2026). OpenTeams is the prospective automation/AI agency.

### 4.1 Full stack inventory

| System | Role / notes |
| --- | --- |
| **Genius** | Solid Commerce iPad event order entry; being rebuilt; ~13,000 SKUs out of sync. |
| **Shopify** | Central order database. |
| **Zapier** | Bridges Shopify → Asana. |
| **Asana** | Production database (orders, **not** customers); becomes manual at the shipping step. |
| **Skynet** | Proprietary measurement engine; ~30,000 patterns; maps body measurements → **18 Optitex inputs**; integrates Bold Metrics. (See [`../reference/skynet/01-overview.md`](../reference/skynet/01-overview.md).) |
| **Bold Metrics** | ML body-measurement vendor; ~**$20–25K/yr**; moved into Skynet. |
| **Optitex** | Manual CAD patterning. |
| **Tom James** | B2B; manual entry into Asana. |
| **AfterShip** | Shipment tracking. |
| **Gladly** | Customer support. |
| **Klaviyo** | Email/marketing. |
| **QuickBooks** | Accounting. |

### 4.2 Order flow

1. Order entry → **Shopify** (or, for Tom James, **TJ → Asana** directly).
2. **Zapier** → **Asana**.
3. **Skynet** validates and **posts back**.
4. The design team **manually reads Asana into Optitex** — **the biggest bottleneck**.
5. Cut and ship; an operator **copies the order number from Asana to Shopify**.

### 4.3 Pain points

- The **manual Asana → Optitex** step (the primary bottleneck).
- **Fragmented data** across systems.
- **Costly Bold Metrics.**
- **Low rebuy rates:** golf/casino ~**9%**, corporate ~**6%**, versus **Tom James ~40%**.
- Product promise: a **365-day fit guarantee.**

---

## 5. Bold Metrics → Skynet migration brief

> **Status: COMPLETE.** This migration has shipped; the Zapier pipeline is at **v51**. The brief below is preserved for context on what changed and why.

**Goal.** Move the Bold Metrics call **off the storefront cart** and into **Skynet's backend** so it fires **once, post-purchase** — instead of repeatedly during browsing. Real need was ~**1,000 calls/mo**, but billing was running ~**3,000–4,000**; the change also fixes a **cart race condition**.

**The Bold Metrics request (shape, not credentials).** A `GET` request with `client_id=bluedelta` plus a secret `user_key`. Inputs: `gender`, `age`, `height`, `weight`, `shoe_size_us`, and `sleeve_type=ARS`. Men additionally send `waist` and `inseam`; women additionally send `bra_size`. *(The actual key value is not reproduced here.)*

**Mapping.** The **6 returned dimensions** map to specific Asana fields. (See [`../reference/zapier/Bold Metrics Migration — Impact on the Zapier Pipeline.md`](../reference/zapier/Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) and [`../03-virtual-tailor.md`](../03-virtual-tailor.md).)

**Guards.** Idempotency guards ensure the call fires once and isn't duplicated.

**Storefront change.** The public **Quick Tailor** page was removed and rebuilt as a **Skynet tab**.

> **Known tech-debt (carried forward):** a hard-coded Bold Metrics key remains in a legacy GemPages quick-tailor export snippet and is **slated for rotation**. Key rotation is pending; the exact file/value pairing is intentionally not documented here.
