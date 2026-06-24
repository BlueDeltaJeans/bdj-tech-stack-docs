# Architecture Overview

> This page documents the complete architecture of the Blue Delta Jeans order pipeline: a 27-step Zapier automation that processes incoming Shopify orders into individual Asana production tasks. Every step described here has been verified against the Zapier export JSON, live zap run logs, and Asana output data. If something isn't documented here, it hasn't been verified.
> 

> ## ⚠️ STATUS / UPDATE (June 2026) — Bold Metrics no longer runs on the storefront
>
> **Several measurement statements on this page describe the pre-migration architecture and are now stale.** They remain accurate for historical orders.
>
> **What's live now:** The Bold Metrics measurement call has been **removed from the Shopify storefront cart** and moved into **Skynet** (the `BlueDeltaJeans/Measurement-Calculator` backend). It now fires **once per real order, post-purchase**, on Skynet's Asana-webhook pipeline — *not* on the cart, and *not* inside this Zapier zap. Net effect on this pipeline:
>
> - Online orders now carry the **Virtual Tailor form inputs** in `note_attributes` (not pre-computed measurements). Step 5 still parses `Extras{}` and `BDJUserData{}`; the six measurement keys in `Extras{}` are now **empty** on new orders.
> - The six Bold Metrics measurement fields in the **Auto | Pant Pipeline** (`Waist Avg.`, `Hip Circum`, `Thigh Circum`, `Knee Circum`, `U Crotch`, `Jean Inseam`) are now **written by Skynet via the Asana API after task creation** — they are *not* set by Zapier's `5. Extras *` mapping for new orders.
> - A **separate Zapier mapping (owned by Caleb)** routes the VT inputs into **five NEW `VT *` Asana fields** that Skynet reads to build the Bold Metrics request. This mapping is **DONE and live** — the zap is at **version 51** and every new order now populates the five `VT *` input fields.
>
> **What this means for the diagram/narrative below:** the Zapier steps themselves are unchanged (no step was added or removed for this migration). What changed is *outside* the 27 steps — the storefront stopped calling Bold Metrics (main Virtual Tailor path removed via PR #83, merged 2026-06-19; the GemPages "quick-tailor" page no longer fires it on the live site either), and Skynet picks up the measurement computation after Asana task creation.
>
> 📄 Full cross-system breakdown: [Bold Metrics Migration — Impact on the Zapier Pipeline](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) · Ground truth: `Measurement-Calculator/bold-metrics-skynet-migration-context.md` · Vendor background: [BlueDelta-Brain → Suppliers & Vendors → Bold Metrics](../domain/domain-context.md)

The pipeline follows a **3-phase ETL flow**:

**Extract (Steps 1–6)** — Trigger on new order, wait for post-purchase processing, re-fetch the complete order from the Shopify API, parse it into structured data, and filter for production-qualifying items.

**Transform (Steps 7–14)** — Track order/item counts, loop through each line item, parse SKU-specific production data, and look up gender display values.

**Load (Steps 15–27)** — Route each item by product type through Zapier Paths and create an Asana task in the correct pipeline project with fully mapped custom fields.

---

## Data Flow Diagram

```html
PHASE 1: ORDER INGESTION
═══════════════════════════════════════════════════════════

  [Step 1]  Shopify: New Order
            Trigger fires on every new order. Provides basic order 
            data but missing nested fields (line item properties, 
            note attributes). That's why Step 4 exists.
                │
                ▼
  [Step 2]  Filter: Order ID Exists
            Guard rail — checks that the order ID is present.
            Stops malformed or empty trigger events.
                │
                ▼
  [Step 3]  Delay: 1.5 Minutes
            Intentional pause. Gives AfterSell (post-purchase app) 
            time to add note attributes, and gives staff time to add 
            tags like "RUSH ROYAL" before the order is fetched.
                │
                ▼
  [Step 4]  Shopify API: GET Order by ID
            Raw HTTP request to the Shopify Admin REST API.
            Re-fetches the COMPLETE order with all nested data:
            line_items[].properties, note_attributes, tags, etc.
                │
                ▼
  [Step 5]  Code: Order Parser (step-5.js)
            The first JavaScript step. Parses the raw API response.
            Outputs: LI[] (filtered line items), OrderType, RUSH,
            CustomerNew, Extras{}, BDJUserData{}, Note
                │
                ▼
  [Step 6]  Filter: LIcount > 0
            Only continues if at least one production-qualifying
            line item exists (SKU starts with W-, M-, CB, SHOE-,
            or VID-GIFT-). Orders with only non-production items
            (tips, shipping protection, regular gift cards) stop here.

PHASE 2: ORDER TRACKING & LINE ITEM LOOP
═══════════════════════════════════════════════════════════

  [Step 7]  Formatter: Text Split — Date
            Extracts clean date from the order's created_at
            timestamp. Example: "2026-03-31T10:42:08-05:00" → "2026-03-31"
                │
                ▼
  [Step 8]  Zapier Tables: Find Record
            Looks up a tracking record in a Zapier Table that stores
            daily processing counters. Used for volume monitoring.
                │
                ▼
  [Step 9]  Zapier Tables: Increment Value — Orders
            Increments the order counter (field f3) in the tracking
            table. Example from live run: 10 → 11
                │
                ▼
 Loop:
 ╔═════════════════════════════════════════════════════════════════════════════╗
 ║                                                                             ║
 ║  [Step 10]  Loop: Create Loop From Line Items                               ║
 ║             Takes the LI[] array from Step 5 and iterates. Everything       ║
 ║             below runs ONCE PER LINE ITEM. An order with 3 pants runs       ║
 ║             Steps 11–27 three times.                                        ║
 ║                 │                                                           ║
 ║                 ▼                                                           ║
 ║  [Step 11]  Zapier Tables: Increment Value — Items                          ║
 ║             Increments the line item counter (f4).                          ║
 ║             Example from live run: 10 → 11                                  ║
 ║                 │                                                           ║
 ║                 ▼                                                           ║
 ║  [Step 12]  Delay: 15 Seconds (Queue-Based)                                 ║
 ║             Spaces out loop iterations to prevent Asana API rate            ║
 ║             limiting on multi-item orders. Queue name: "LineItemsX"         ║
 ║                 │                                                           ║
 ║                 ▼                                                           ║
 ║  [Step 13]  Code: Line Item Processor (step-13.js)                          ║
 ║             The second JavaScript step. Parses the current line item's      ║
 ║             SKU to determine product type, then extracts all fields:        ║
 ║             fabric, style, thread, monogram, etc.                           ║
 ║                 │                                                           ║
 ║                 ▼                                                           ║
 ║  [Step 14]  Formatter: Gender Lookup Table                                  ║
 ║             Converts gender code to Asana enum GID:                         ║
 ║             "M" → 1112754700909041 (aqua badge)                             ║
 ║             "F" → 1112754700909042 (pink "W" badge)                         ║
 ║             Also supports "Youth" → yellow badge                            ║
 ║                                                                             ║
 ║                                                                             ║
 ║  PHASE 3: ROUTING & ASANA TASK CREATION                                     ║
 ║  ═══════════════════════════════════════                                    ║
 ║                 │                                                           ║
 ║                 ▼                                                           ║
 ║  [Step 15]  Paths: Split Into Paths                                         ║
 ║             Primary router. Branches into 3 paths                           ║
 ║             based on ProductType from Step 13.                              ║
 ║                 │                                                           ║
 ║         ┌───────┴──────────────────┬─────────────────────────┐              ║
 ║         ▼                          ▼                         ▼              ║
 ║   PANTS & BELTS                  SHOES                  VIDEO CARDS         ║
 ║         │                          │                         │              ║
 ║         ▼                          ▼                         ▼              ║
 ║    [Step 16]                  [Step 24]                 [Step 26]           ║
 ║    Path Conditions            Path Conditions           Path Conditions     ║
 ║         │                          │                         │              ║
 ║         ▼                          ▼                         ▼              ║
 ║    [Step 17]                  [Step 25]                 [Step 27]           ║
 ║    Format Order Date          Asana: Create             Asana: Create       ║
 ║         │                     Shoe Task                 Video Card Task     ║
 ║         ▼                                                                   ║
 ║    [Step 18]                                                                ║
 ║    Format Due Date                                                          ║
 ║         │                                                                   ║
 ║         ▼                                                                   ║
 ║    [Step 19]                                                                ║
 ║    Paths: Pants vs. Belts                                                   ║
 ║         │                                                                   ║
 ║      ┌──┴───────────────────────┐                                           ║
 ║      ▼                          ▼                                           ║
 ║    PANTS                      BELTS                                         ║
 ║      │                          │                                           ║
 ║      ▼                          ▼                                           ║
 ║  [Step 20]                  [Step 22]                                       ║
 ║  Path Conditions            Path Conditions                                 ║
 ║      │                          │                                           ║
 ║      ▼                          ▼                                           ║
 ║  [Step 21]                  [Step 23]                                       ║
 ║  Asana Pant Task            Asana Belt Task                                 ║
 ║                                                                             ║
 ╚═════════════════════════════════════════════════════════════════════════════╝
```

<aside>

### Expand All Toggles

Cmd + Option + T

</aside>

## Phase 1: Order Ingestion (Steps 1–6)

### Step 1 — Shopify: New Order (Trigger)

| Detail | Value |
| --- | --- |
| **Type** | Trigger |
| **App** | ShopifyCLIAPI@6.1.1 |
| **Action** | `new_order_hook_v2` |
| **Auth** | Shopify API credentials for `blue-delta-jeans.myshopify.com` |

This is the entry point. Every new order in the Blue Delta Shopify store fires this trigger. Zapier's native Shopify integration provides a basic order payload, but it's **incomplete** — critical nested objects like `line_items[].properties` (custom options from SC Product Options), `note_attributes` (Virtual Tailor data — see migration note below), and deep customer data are either missing or truncated.

> **⚠️ June 2026:** "Virtual Tailor measurements" in `note_attributes` now means the VT form **inputs**, not pre-computed Bold Metrics measurements. Bold Metrics was moved off the cart into Skynet, which computes measurements post-order. See the §STATUS callout at the top of this page.

This is why the pipeline doesn't rely on the trigger data for production fields. Instead, Step 4 re-fetches the complete order from the Shopify Admin API.

**Key output used downstream:** `1__id` — the Shopify order ID (e.g., `7544208490787`). This is the only field from Step 1 that the rest of the pipeline depends on.

---

### Step 2 — Filter: Order ID Exists

| Detail | Value |
| --- | --- |
| **Type** | Filter |
| **Condition** | `1__id` must exist (`iexist` match) |

A simple guard rail. If the trigger fires without a valid order ID (malformed webhook, test event, etc.), execution stops here. In practice this almost never blocks — it's a safety net.

**Verified from live run:** `7544208490787 — Exists` → passed.

---

### Step 3 — Delay: 1.5 Minutes

| Detail | Value |
| --- | --- |
| **Type** | Delay |
| **App** | DelayCLIAPI@1.1.1 |
| **Duration** | 1.5 minutes (90 seconds) |

This intentional pause exists for three reasons:

1. **AfterSell processing** — The AfterSell post-purchase upsell app adds note attributes and metadata to the order after creation. Without this delay, Step 4 might fetch the order before AfterSell has finished writing its data.
2. **Staff tagging** — Store staff at events may manually add tags to the order shortly after it's placed (e.g., "RUSH ROYAL", "Andy", event names). The delay gives them a window to do this before the order is processed.
3. **Shopify finalization** — Shopify's webhook fires immediately on order creation, but some order data (payment confirmation, fraud analysis, inventory reservation) may still be processing. The delay ensures the API re-fetch in Step 4 returns a fully settled order.

---

### Step 4 — Shopify API: GET Order by ID

| Detail | Value |
| --- | --- |
| **Type** | Action (Raw HTTP Request) |
| **App** | ShopifyCLIAPI@6.1.1 (`_zap_raw_request` action) |
| **Method** | GET |
| **Custom Title** | "GET Shopify API - Find Order by ID" |

**Endpoint:**

```
GET https://blue-delta-jeans.myshopify.com/admin/api/2024-10/orders/{{1__id}}.json
```

This is the most important data-fetching step. It makes a direct HTTP call to the Shopify Admin REST API to get the **complete** order object with all nested data that the trigger doesn't provide:

- `order.line_items[].properties` — Custom line item properties from SC Product Options (thread color, monogram, waist height, front pockets, etc.)
- `order.note_attributes` — Order-level custom data including Virtual Tailor data, BDJ User Data (customer profile), and AfterSell metadata. **⚠️ June 2026:** historically this carried Bold Metrics body *measurements*; post-migration it carries the VT form *inputs* (the six measurement keys are now empty on new orders — Skynet computes them post-order). See the §STATUS callout at the top.
- `order.tags` — Full comma-separated tag string (e.g., "Andy, Denim, RUSH COBALT")
- `order.cart_token` — Present for online orders, absent for POS/draft orders. This is how the pipeline detects order type.
- `order.source_name` — Where the order originated ("web", "pos", etc.)
- `order.customer` — Customer record (but notably missing `orders_count` — see BUG-05)
- `order.note` — Free-text order note

**API version note:** The URL specifies `2024-10` but Shopify's response header shows `x-shopify-api-version: 2025-04` — Shopify auto-upgrades within the same major version. The URL should be updated periodically as Shopify deprecates older versions (~1 year after release).

**Rate limit from live run:** `1/400` — well within Shopify's limits.

The full response body is passed to Step 5 as `inputData.RAW`.

---

### Step 5 — Code: Order Parser (step-5.js)

| Detail | Value |
| --- | --- |
| **Type** | JavaScript Code |
| **App** | CodeCLIAPI@1.0.1 |
| **Zap Step ID** | 281794945 |
| **Action ID** | `01929fad-d3dd-62c2-52ed-7868d5fcc691` |
| **Custom Title** | "Run Javascript: DO NOT DELETE!!!" |
| **Runtime** | ~85-86 MB memory, ~200-216ms duration |

This is the first of two JavaScript code steps and handles all order-level parsing. It receives the raw API response from Step 4 and transforms it into the structured data that every downstream step depends on.

**Inputs:**

| Input | Source | Example |
| --- | --- | --- |
| `RAW` | `{{4__response__body}}` | Full JSON string of the order API response |
| `TAGS` | `{{=gives['281795060']['response']['data']['order']['tags']}}` | `"Andy, Denim"` |

**What it does (in order):**

1. **RUSH detection** — Uppercases tags, checks for "RUSH ROYAL" (highest priority) or "RUSH COBALT" (standard rush). Outputs a suffix string like `" | RUSH ROYAL"` for the Asana task name.
2. **Order type detection** — Checks `Order.cart_token`. If present → `OrderType = "Online"` (website checkout). If absent → `OrderType = "Direct"` (POS, draft order, or API-created). This determines due date calculations downstream.
3. **Customer new/returning** — Reads `Order.customer.orders_count` to determine if first order. **BUG-05:** This field doesn't exist in the REST API response, so `CustomerNew` is currently always "Yes." Fixed in V4 code with a `numberOfOrders` fallback.
4. **Line item filtering** — Filters `Order.line_items` to only keep items with production-qualifying SKU prefixes: `W-` (women's pants), `M-` (men's pants/shorts/derby), `CB` (belts), `SHOE-` (shoes), `VID-GIFT-` (video cards). Everything else (shipping protection, tips, merchandise, regular gift cards) is discarded. Each qualifying item is stringified via `JSON.stringify()` because Zapier's Loop step requires an array of strings. **BUG-01:** Items with `quantity > 1` are not expanded — a single line item with qty 3 produces only 1 loop iteration instead of 3. Fixed in V4 code.
5. **Note attributes cleaning** — Converts `Order.note_attributes` array into a flat object (`Extras`). Blanks the `__aftersell_tamper_proof` attribute value (a long hash that clutters the data). **⚠️ June 2026:** the six Bold Metrics measurement keys in `Extras` (`Hip Circum`, `Jean Inseam`, `Knee Circum`, `Thigh Circum`, `U Crotch`, `Waist Average`) are now **empty** on new online orders — Skynet computes and writes those Asana fields post-order.
6. **BDJ User Data extraction** — Parses the `"BDJ User Data"` note attribute, which is a newline-delimited `key: value` block from the storefront's Virtual Tailor form. Contains customer body profile data: Gender, Age, Height, Weight, Shoe Size, Bra Size, Inseam, Jean Fit, Common Shoe, Style. **⚠️ June 2026:** these VT *inputs* are now load-bearing — Skynet feeds them (plus five new `VT *` Asana fields) to Bold Metrics to compute measurements post-order. **BUG-06:** Uses `\r\n` line endings from Windows, leaving `\r` residue in values. Fixed in V4 code.

**Outputs:**

| Field | Type | Example | Used By |
| --- | --- | --- | --- |
| `RUSH` | string | `" | RUSH ROYAL"` or `""` | Asana task name (Step 21/23/25/27) |
| `LIcount` | number | `1` | Step 6 filter |
| `CustomerNew` | string | `"Yes"` or `"No"` | Asana "New Customer" field |
| `OrderType` | string | `"Online"` or `"Direct"` | Asana "Order Type" field, due date calc |
| `Extras` | object | `{"Virtual Tailor":"Yes","Hip Circum":"41.83",...}` | Asana measurement fields. **⚠️ June 2026: 6 measurement keys now empty on new orders — Skynet writes those fields post-order** |
| `BDJUserData` | object | `{"Gender":"Female","Age":"51",...}` | Asana Age/Weight/Body fields. **⚠️ June 2026: VT inputs now feed Skynet's Bold Metrics call + 5 new `VT *` fields** |
| `Note` | string | `"Please rush this"` or `""` | Asana "Note" field |
| `LI` | string[] | `['{"sku":"W-RW26_NATURALINDIGO-FLARE",...}']` | Step 10 loop input |
| `NA` | object[] | `[{"name":"Virtual Tailor","value":"Yes"},...]` | Reference |
| `NAcount` | number | `9` | Reference |
| `CustomerOrdersCount` | number | `0` (bugged) | Reference |
| `BDJUserDataCount` | number | `10` | Reference |

For the full annotated code, see the **Step 5 — Order Parser** page.

---

### Step 6 — Filter: LIcount > 0

| Detail | Value |
| --- | --- |
| **Type** | Filter |
| **Condition** | `5__LIcount` (Number) Greater than `0` |

Stops execution if no production-qualifying line items remain after filtering. For example, if a customer only purchased a standard (non-video) gift card, a hat, or shipping protection, all SKUs would be filtered out in Step 5 and `LIcount` would be `0`.

**Verified from live run:** `1 > 0` → passed.

---

## Phase 2: Order Tracking & Line Item Loop (Steps 7–14)

### Step 7 — Formatter: Text Split — Date

| Detail | Value |
| --- | --- |
| **Type** | Formatter (Text transform) |
| **Output** | `"2026-03-31"` |

Splits the order's `created_at` ISO timestamp to extract just the date portion. This clean date is used in Step 13 for due date calculations (`DateCreated.split("T")[0]`) and in the Asana date formatting steps (17–18).

---

### Step 8 — Zapier Tables: Find Record

| Detail | Value |
| --- | --- |
| **Type** | Zapier Tables lookup |
| **Table ID** | `01KBG6F9HBKQM28ZA0E3XJTNPK` |

Finds a tracking record in a Zapier Table used for daily pipeline volume monitoring. This table has a simple structure:

| Field | Name (inferred) | Example Value | Purpose |
| --- | --- | --- | --- |
| f1 | Record ID | `01KMYWZ9XV8GAFGVNPW033FB9H` | Primary key |
| f2 | Created At | `2026-03-30T08:15:12Z` | When record was created |
| f3 | Order Count | `10` → incremented to `11` in Step 9 | Running total of orders processed |
| f4 | Line Item Count | `10` → incremented to `11` in Step 11 | Running total of items processed |
| f5 | Reset Date | `2026-03-31T07:00:00Z` | Daily reset at 2 AM Central |

---

### Step 9 — Zapier Tables: Increment Value — Orders

Increments field `f3` (Order Count) in the tracking table. From the live run: `10 → 11`.

---

### Step 10 — Loop: Create Loop From Line Items

| Detail | Value |
| --- | --- |
| **Type** | Looping by Zapier |
| **Input** | The `LI` array from Step 5 |

This is the architectural pivot point of the entire pipeline. It takes the array of stringified line items and creates a loop — **every step from 11 through 27 runs once per line item**. An order with 5 pairs of pants creates 5 loop iterations, each producing its own Asana task.

**Loop output fields (from live run):**

| Field | Value | Purpose |
| --- | --- | --- |
| `loop_iteration` | `1` | Current item number (used in Asana task name as "X/Y") |
| `loop_total_iterations` | `1` | Total items in order (used in Asana task name as "X/Y") |
| `loop_iteration_is_last` | `true` | Whether this is the final iteration |
| `LI` | `'{"sku":"W-RW26_NATURALINDIGO-FLARE",...}'` | The current line item (stringified JSON) |
| `loop_id` | `6d1b1494-946a-45e4-99cc-fff03d18fb92` | Unique ID for this loop execution |
| `loop_id_and_iteration` | `6d1b1494-...-0001` | Unique ID for this specific iteration |

**Important for BUG-01:** If a customer orders 2 identical pants (same SKU, same options), Shopify creates a single line item with `quantity: 2`. Step 5 currently creates only 1 LI entry for this, so the loop runs once and only 1 Asana task is created. The V4 fix expands `quantity > 1` items into multiple LI entries.

---

### Step 11 — Zapier Tables: Increment Value — Line Items

Increments field `f4` (Line Item Count) in the tracking table. From the live run: `10 → 11`.

---

### Step 12 — Delay: 15 Seconds (Queue-Based)

| Detail | Value |
| --- | --- |
| **Type** | Delay (Queue-based) |
| **Duration** | 0.25 minutes (15 seconds) |
| **Queue Name** | `LineItemsX` |
| **Zap Step ID** | 281794957 |

Adds a 15-second pause between loop iterations. This serves two purposes:

1. **Rate limiting prevention** — Avoids hitting Asana's API rate limits when processing orders with many line items. Without this, a 5-item order would fire 5 rapid Asana API calls.
2. **Execution ordering** — Helps ensure tasks are created in sequence in Asana, so "Sam Taylor 1/5" appears before "Sam Taylor 2/5."

---

### Step 13 — Code: Line Item Processor (step-13.js)

| Detail | Value |
| --- | --- |
| **Type** | JavaScript Code |
| **App** | CodeCLIAPI@1.0.1 |
| **Zap Step ID** | 281794951 |
| **Custom Title** | "Run Javascript: DO NOT" |
| **Runtime** | ~85-86 MB memory, ~195-204ms duration |

This is the second JavaScript code step and the most complex. It runs inside the loop (once per line item) and is responsible for parsing the SKU, determining the product type, and extracting every product-specific field that Asana needs.

**Inputs:**

| Input | Source | Example |
| --- | --- | --- |
| `LI` | Current loop item (Step 10) | Stringified JSON of one line item |
| `DateCreated` | Step 7 output | `"2026-03-31T10:42:08-05:00"` |
| `CartToken` | Step 5 output | `:censored:24:37b9e3afb5:` (online) or empty (POS) |
| `Tags` | Step 5 output | `"Andy, Denim"` |
| `Source` | Step 4 output | `"web"` or `"pos"` |

**What it does:**

1. Parses the line item JSON back from string (`JSON.parse`)
2. Splits the SKU on  to get segment array (`LI.sku.split("-")`)
3. Converts line item properties into a flat Properties object
4. Routes by SKU prefix into product-specific parsing branches:
    - `W-` → Women's Pants (Online 3-seg, POS 4-seg, Shorts, Derby)
    - `M-` → Men's Pants (same variants as above)
    - `CB` → Belts (Online 3-seg, POS 4-seg and 5-seg)
    - `SHOE-` → Shoes (4-seg with gender and size in SKU)
    - `VID-GIFT-` → Video Gift Cards (4-seg)
5. Calculates due date based on product type and CartToken presence
6. Processes order tags into Product Tags vs. Event Tags
7. Detects alteration flag

**Key output:** `Properties` object containing all parsed fields (ProductType, Gender, Fabric, Style, Thread, Monogram, DueDate, etc.), plus `Tags`, `EventTags`, `Alteration`, and the raw `SKU` array.

For the full annotated code with product-specific parsing logic, see the **Step 13 — Line Item Processor** page.

---

### Step 14 — Formatter: Gender Lookup Table

| Detail | Value |
| --- | --- |
| **Type** | Formatter (Utilities — Lookup Table) |

Converts the gender code from Step 13 into an Asana custom field enum option GID. Asana's Gender field (GID `1112754700909040`) is an enum type, so the Zapier step needs to send the GID of the enum option, not the text value.

| Input Code | Asana Enum GID | Asana Display | Badge Color |
| --- | --- | --- | --- |
| `M` | `1112754700909041` | M | Aqua/teal |
| `F` | `1112754700909042` | W | Pink |
| *(Youth)* | `1209345887831124` | Youth | Yellow |

Note that `F` maps to `W` (Women's) in the Asana display — the lookup converts the internal code to the display-friendly enum option.

---

## Phase 3: Routing & Asana Task Creation (Steps 15–27)

### Step 15 — Paths: Split Into Paths (Primary Router)

The primary routing step. Examines the `ProductType` value from Step 13's Properties object and branches into one of three paths:

| Path | Condition | Steps |
| --- | --- | --- |
| **Pants & Belts** | ProductType = "Pant" or "Belt" | Steps 16–23 |
| **Shoes** | ProductType = "Shoe" | Steps 24–25 |
| **Video Cards** | ProductType = "Video Card" | Steps 26–27 |

**Verified from live run (order #114265):** Took the "Pants & Belts" path. Shoes and Video Cards paths showed "Path did not run as the rule conditions were not met" and "Did not attempt to send new Task to Asana."

---

### Pants & Belts Path (Steps 16–23)

**Step 16 — Path Conditions**

Two conditions, both must match:

1. ProductType `contains` "pant" (case-insensitive)
2. Alteration `contains` "false"

<aside>

> **Important discovery:** The Pants & Belts path checks that Alteration is `false`. This means orders tagged "alteration" in Shopify are **excluded from the standard pipeline**. They presumably need a separate workflow or manual handling.
> 
</aside>

---

**Step 17 — Formatter: Date/Time — Format Order Date**

Formats the order creation date for Asana. Takes the ISO timestamp and converts it to Asana's expected date format.

---

**Step 18 — Formatter: Date/Time — Format Due Date**

Formats the calculated due date from Step 13 (e.g., `2026-04-14`) for Asana.

---

**Step 19 — Paths: Pants vs. Belts (Sub-Router)**

Sub-routes within the Pants & Belts path based on ProductType:

| Sub-Path | Condition | Destination |
| --- | --- | --- |
| Pants | ProductType contains "pant", Alteration contains "false" | Step 21 |
| Belts | ProductType contains "belt", Alteration contains "false" | Step 23 |

---

**Step 20 — Path Conditions: Pants**

Same conditions as Step 16 (ProductType contains "pant", Alteration is false). Verified from live run.

---

**Step 21 — Asana: Create Task → Auto | Pant Pipeline**

Creates a task in the **Auto | Pant Pipeline** Asana project (GID: `1206657933205972`). Maps up to 49 custom fields including fabric, style, thread, monogram, all Virtual Tailor measurements, and customer body profile data.

> **⚠️ June 2026 (live):** The six Virtual Tailor / Bold Metrics measurement fields are no longer filled by Zapier at task creation — they arrive empty and are **written by Skynet via the Asana API a few seconds later** (post-order Bold Metrics call). Zapier additionally maps the VT *inputs* into five new `VT *` fields that Skynet reads; this mapping is **DONE and live at zap v51**. See the §STATUS callout at the top and the [Asana Field Mapping](./Asana%20Field%20Mapping%20%28Steps%2021–27%29.md) page.

For the complete field mapping, see the **Asana Field Mapping (Steps 21–27)** page.

---

**Step 22 — Path Conditions: Belts**

Matches when ProductType contains "belt" and Alteration is false.

---

**Step 23 — Asana: Create Task → Auto | Belt Pipeline**

Creates a task in the **Auto | Belt Pipeline** project. Maps 17 custom fields including leather type, width, hardware, stitching (thread color), and monogram.

---

### Shoes Path (Steps 24–25)

**Step 24 — Path Conditions: Shoes**

Matches when ProductType = "Shoe".

---

**Step 25 — Asana: Create Task → Auto | Shoe Pipeline**

Creates a task in the **Auto | Shoe Pipeline** project. Maps 15 custom fields including shoe type (Classic/Custom), gender, size, and customization options (Laces, Swoosh, Back Tab/Eyelets, Toe Cap/Back Heel, Toe Box/Mid Panel). All shoe customization data comes from line item properties that are manually entered from paper order forms at events.

---

### Video Cards Path (Steps 26–27)

**Step 26 — Path Conditions: Video Cards**

Matches when ProductType = "Video Card".

---

**Step 27 — Asana: Create Task → Auto | Video Card Pipeline**

Creates a task in the **Auto | Video Card Pipeline** project. Maps 7 custom fields — the simplest of the four pipelines. Includes Product Type, Order Type, Event Tags, Quantity, and Note.

---

## Order Type Detection

The pipeline uses the `CartToken` field from the Shopify order to distinguish between two types of orders:

**Online orders** have a `CartToken` value — this is set automatically when a customer checks out through the Blue Delta website. The token is censored by Zapier in run logs (`:censored:24:37b9e3afb5:`) but JavaScript truthiness checks still work because a censored non-empty string is truthy.

**Direct/POS orders** have no `CartToken` — orders created via the Shopify POS app at events, draft orders created in the admin, or API-created orders don't go through the storefront checkout and therefore have no cart token.

This distinction matters in two ways:

1. **Due date calculation** — Online pants get 14-day due dates; POS pants get 30 days (more time because POS customers are measured in person and patterns may need more refinement).
2. **Data source for customization** — Online orders carry thread color and monogram as line item properties (from SC Product Options). POS orders encode this data in the SKU itself because SC Product Options doesn't work in the Shopify POS app. See the **Online vs POS Product Architecture** page for the full breakdown.

**Code (Step 5):**

```jsx
let OrderType = Order.cart_token ? "Online" : "Direct";
```

---

## Due Date Rules

Due dates are calculated in Step 13 by adding a product-specific number of days to the order creation date. The calculation uses `CartToken` presence to determine the offset for pants.

| Product | Online (has CartToken) | Direct/POS (no CartToken) |
| --- | --- | --- |
| **Pants & Shorts** | 14 days | 30 days |
| **Belts** | 30 days | 30 days |
| **Shoes** | 45 days | 45 days |
| **Video Gift Cards** | 5 days | 5 days |

**Code (Step 13):**

```jsx
const setDueDate = (days) => {
  let dueDate = new Date(DateCreated);
  dueDate.setDate(dueDate.getDate() + days);
  return dueDate.toISOString().split("T")[0];
};

// For pants:
DueDays: CartToken ? 14 : 30,
DueDate: setDueDate(CartToken ? 14 : 30),
```

**Example from live run:** Order created `2026-03-31`, online order (CartToken present) → Due date = `2026-04-14` (14 days later).

---

## Asana Task Name Format

The Asana task name is constructed in the Asana steps (21/23/25/27) by combining multiple Zapier tokens:

```
#{OrderNumber} {FirstName} {LastName} {LoopIteration}/{TotalIterations}{RushSuffix}
```

| Token | Source | Example |
| --- | --- | --- |
| Order Number | Step 4: `Response Data Order Name` | `#114265` |
| First Name | Step 4: `Response Data Order Customer First Name` | `Erin` |
| Last Name | Step 4: `Response Data Order Customer Last Name` | `Test` |
| Loop Iteration | Step 10: `Loop Iteration` | `1` |
| Total Iterations | Step 10: `Loop Total Iterations` | `1` |
| Rush Suffix | Step 5: `RUSH` | `| RUSH COBALT` or empty |

**Examples from live runs:**

- `#114265 Erin Test 1/1` — Single-item online order, no rush
- `#100002 Alex Rivera 1/1 | RUSH COBALT` — Single-item order with rush priority
- `#100003 Sam Taylor 5/5` — Fifth item in a 5-item order

---

## Key System Dependencies

### Shopify API Version

The Step 4 API request URL specifies version `2024-10`, but Shopify's response header shows it's serving `2025-04`. Shopify auto-upgrades within a stable release but deprecates versions approximately 1 year after release. The URL should be updated periodically to avoid eventual breakage.

### 1.5-Minute Delay (Step 3)

The delay depends on AfterSell's post-purchase processing completing within 90 seconds. If AfterSell's processing time increases, or if a new post-purchase app is added, the delay may need to be extended.

### Bold Metrics / Skynet (measurement computation) — ⚠️ June 2026

**Bold Metrics no longer runs on the storefront (migration COMPLETE).** The Virtual Tailor body-measurement call was moved off the Shopify storefront into **Skynet** (`BlueDeltaJeans/Measurement-Calculator`), which fires Bold Metrics **once per order, post-purchase**, on its Asana-webhook pipeline and writes the six measurement custom fields directly into the Pant Pipeline task via the Asana API. The main Virtual Tailor path removal was merged to the live theme via PR #83 (merged 2026-06-19), and the GemPages "quick-tailor" page no longer fires Bold Metrics on the live site either. (Repo nuance: the committed GemPages quick-tailor snippet *exports* still contain the old key string and lag the live page — they are stale and should be purged — but the live site no longer fires.) This Zapier zap no longer transports computed measurements; it transports the VT **inputs** (in `note_attributes` / `BDJUserData`). A **separate Zapier mapping (owned by Caleb)** populates five new `VT *` Asana input fields Skynet reads — that mapping is **DONE and live at zap v51** (every new order populates the five `VT *` fields). If Skynet, the Bold Metrics credential (a company-wide key), or the Asana write-back breaks, new online orders will land in Asana with **empty measurement fields**. Skynet matches Asana fields **by exact display name** (e.g. the trailing period in `Waist Avg.`), so renaming any measurement or `VT *` field silently breaks it. See [Bold Metrics Migration — Impact on the Zapier Pipeline](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) and `Measurement-Calculator/bold-metrics-skynet-migration-context.md`.

### SC Product Options

The entire Online vs. POS product architecture depends on SC Product Options being the mechanism for passing thread color and monogram on online orders. If this app is removed or replaced, the line item properties format would change and Step 13's parsing logic would need updating.

> **SC Product Options vs. Bold Metrics — not the same thing.** SC Product Options is the storefront app that captures **product customization choices** (thread color, monogram/lettering, front pockets, waist height) as Shopify **line item properties** — parsed by Step 13 into `Properties{}`. Bold Metrics is the **AI body-measurement vendor** behind the Virtual Tailor; it produces **order-level body measurements** that historically arrived in `note_attributes` (parsed by Step 5 into `Extras{}`) and are now computed post-order by Skynet. Different app, different data path (line item properties vs. note attributes), different Zapier step (13 vs. 5).

### Zapier Tables Counter

Steps 8, 9, and 11 use a Zapier Tables record for volume tracking. If migrating away from Zapier, this tracking data needs to be replicated or the counter steps removed. The table resets daily at 2 AM Central (07:00 UTC).

### Zapier's "paused" Export Field

The `zap-export.json` shows `"paused": true` on Steps 2–6, but live runs confirm all steps execute successfully. **Do NOT rely on the export's `paused` field** to determine whether steps are active. Always verify against live run history.

---

## Step Quick Reference

| Step | Type | App | Name | Key Output |
| --- | --- | --- | --- | --- |
| 1 | Trigger | Shopify | New Order | Order ID |
| 2 | Filter | Zapier | Order ID exists | Pass/fail |
| 3 | Delay | Zapier | 1.5 min wait | — |
| 4 | API Call | Shopify | GET order by ID | Full order JSON |
| 5 | **Code** | Zapier | **Order Parser** | LI[], OrderType, RUSH, Extras, BDJUserData |
| 6 | Filter | Zapier | LIcount > 0 | Pass/fail |
| 7 | Formatter | Zapier | Split date | `"2026-03-31"` |
| 8 | Tables | Zapier | Find tracking record | Counter record |
| 9 | Tables | Zapier | Increment orders | f3 count |
| 10 | **Loop** | Zapier | **Loop line items** | Current LI, iteration count |
| 11 | Tables | Zapier | Increment items | f4 count |
| 12 | Delay | Zapier | 15 sec queue | — |
| 13 | **Code** | Zapier | **Line Item Processor** | Properties{}, Tags, EventTags |
| 14 | Formatter | Zapier | Gender lookup | Asana enum GID |
| 15 | Paths | Zapier | Route by product type | Branch selection |
| 16 | Paths | Zapier | Pants & Belts (not alteration) | Branch condition |
| 17 | Formatter | Zapier | Format order date | Asana date |
| 18 | Formatter | Zapier | Format due date | Asana date |
| 19 | Paths | Zapier | Pants vs. Belts | Branch selection |
| 20 | Paths | Zapier | Pants condition | Branch condition |
| 21 | **Asana** | Asana | **Create Pant task** | Task GID |
| 22 | Paths | Zapier | Belts condition | Branch condition |
| 23 | **Asana** | Asana | **Create Belt task** | Task GID |
| 24 | Paths | Zapier | Shoes condition | Branch condition |
| 25 | **Asana** | Asana | **Create Shoe task** | Task GID |
| 26 | Paths | Zapier | Video Cards condition | Branch condition |
| 27 | **Asana** | Asana | **Create Video Card task** | Task GID |