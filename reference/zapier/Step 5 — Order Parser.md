# Step 5 — Order Parser

Step 5 is the first of two JavaScript code steps in the pipeline. It sits between the Shopify API re-fetch (Step 4) and the line item count filter (Step 6). Its job is to take the raw Shopify order JSON and parse it into every structured component that downstream steps need: customer info, order type, rush status, line items, note attributes, Virtual Tailor measurements, and the customer's body profile.

Everything after Step 5 — the loop, the line item processor, the path routing, the Asana task creation — depends on this step's output. If Step 5 parses something wrong, every Asana task created from that order will have incorrect data.

> ## ⚠️ STATUS / UPDATE (June 2026) — Bold Metrics moved off the cart into Skynet
>
> **The measurement story below describes the *legacy* (pre-migration) behavior. It is still accurate for historical orders, but it is now stale for NEW online orders.**
>
> The Bold Metrics call that used to run on the storefront **cart** has been moved into **Skynet** (the `BlueDeltaJeans/Measurement-Calculator` backend), where it fires **once per real order, post-purchase**. Consequences for *this* step:
>
> - **The six measurement note attributes are now EMPTY on NEW online orders.** `Extras["Hip Circum"]`, `Extras["Jean Inseam"]`, `Extras["Knee Circum"]`, `Extras["Thigh Circum"]`, `Extras["U Crotch"]`, and `Extras["Waist Average"]` no longer arrive in `note_attributes` because Bold Metrics is no longer called at the cart. Step 5 still *parses* them the same way — it just gets blanks. (Older orders placed before the cutover still carry these values; that's why the historical description is annotated, not deleted.)
> - **`BDJUserData{}` (the VT form inputs) is now the load-bearing data.** The customer's raw Virtual Tailor inputs (Gender, Age, Height, Weight, Shoe Size, Bra Size, Inseam, Jean Fit, etc.) parsed in §7 below are what Skynet uses to compute measurements. Preserving these durably is the whole point of the migration (it fixes a cart-side race condition that previously dropped them).
> - **Measurements are computed *after* the order, by Skynet, and written directly into Asana.** Skynet detects a VT order (inputs present + measurement fields empty), calls Bold Metrics with the inputs, and writes the six measurements into the Asana custom fields via the Asana API. They no longer flow Shopify → Zapier → Asana.
> - **No Step 5 code change is required for this migration** — Step 5's parsing logic is unchanged. What changed is *what the upstream order payload contains* (empty measurement attributes) and the *separate Zapier mapping* (owned by Caleb) that populates five NEW `VT *` input fields in Asana.
>
> **Current state (as of June 2026): migration COMPLETE / all live.** The Bold Metrics removal from the storefront and the Skynet write-back are deployed and live; the Bold Metrics call no longer fires on the main storefront Virtual Tailor path (removed via PR #83, merged 2026-06-19) and the GemPages "quick-tailor" page no longer fires it on the live site either. The Zapier mapping that populates the five `VT *` Asana input fields on each new order is **DONE and live** — the zap is at **version 51** and every new order now populates the five `VT *` input fields. (Repo nuance: the committed GemPages quick-tailor snippet *exports* still contain the old key string and lag the live page — they are stale and should be purged — but the live site no longer fires Bold Metrics.)
>
> 📄 Full cross-system breakdown: [Bold Metrics Migration — Impact on the Zapier Pipeline](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) · Ground truth: `Measurement-Calculator/bold-metrics-skynet-migration-context.md` · Vendor background: [BlueDelta-Brain → Suppliers & Vendors → Bold Metrics](../domain/domain-context.md)

| Detail | Value |
| --- | --- |
| **Pipeline Position** | Step 5 of 27 (after API re-fetch, before LIcount filter) |
| **Type** | JavaScript Code |
| **App** | CodeCLIAPI@1.0.1 |
| **Zap Step ID** | 281794945 |
| **Action ID** | `01929fad-d3dd-62c2-52ed-7868d5fcc691` |
| **Custom Title** | "Run Javascript: DO NOT DELETE!!!" |
| **Runtime** | ~85–86 MB memory, ~200–216ms duration |
| **Bugs in this step** | 3 (BUG-01 CRITICAL, BUG-05 MEDIUM, BUG-06 LOW) |

---

## Inputs

Step 5 receives two input variables, configured in the Zapier step's input mapping:

### `inputData.RAW`

**Zapier token:** `{{4__response__body}}`

The complete HTTP response body from Step 4's Shopify API call. This is a JSON string containing the full order object with all nested data. The string is typically 3,000–10,000 characters depending on the number of line items and note attributes.

The code parses this with `JSON.parse(inputData.RAW)` and expects the structure `{ order: { ... } }`.

**Example (condensed):**

```json
{"order":{"id":7544208490787,"cart_token":":censored:24:37b9e3afb5:","tags":"Andy, Denim","source_name":"web","note":null,"note_attributes":[{"name":"Virtual Tailor","value":"Yes"},{"name":"Hip Circum","value":"41.83"},{"name":"BDJ User Data","value":"Gender: Female\r\nAge: 51\r\n..."}],"line_items":[{"sku":"W-RW26_NATURALINDIGO-FLARE","quantity":1,"properties":[{"name":"Thread Color","value":"Light Pink"}]}],"customer":{"first_name":"Erin","last_name":"Test"}}}
```

### `inputData.TAGS`

**Zapier token:** `{{=gives['281795060']['response']['data']['order']['tags']}}`

The order's tags string. This uses Zapier's advanced reference syntax to pull tags from a specific upstream step's response data. The value is a comma-separated string like `"Andy, Denim"` or `"RUSH ROYAL, Ella, Performance"`.

Note: This is a separate input from the tags inside `RAW` — they should contain the same data, but using the advanced reference syntax ensures the latest tag values are captured (tags may have been added during the 1.5-minute delay in Step 3).

---

## Full Code — Current (Live in Zapier)

This is the code currently running in production. Bugs are marked with inline comments.

```jsx
// Input variable: TAGS
let TAGS = inputData.TAGS || "";

// Uppercase for consistent matching
TAGS = TAGS.toString().toUpperCase();

// Default
let RUSH = "";

// Check for specific matches
if (TAGS.includes("RUSH ROYAL")) {
  RUSH = " | RUSH ROYAL";
} else if (TAGS.includes("RUSH COBALT")) {
  RUSH = " | RUSH COBALT";
}

let RAW;
try {
  RAW = JSON.parse(inputData.RAW);
} catch (error) {
  throw new Error("Invalid JSON in inputData.RAW");
}

// Ensure RAW has an order object
if (!RAW.order) throw new Error("Invalid data structure: Missing 'order' object");

let Order = RAW.order; // Extract order object

// Determine Order Type
let OrderType = Order.cart_token ? "Online" : "Direct";

// Customer Details
let CustomerOrdersCount = Order.customer?.orders_count || 0;  // ⚠️ BUG-05: always 0
let CustomerNew = CustomerOrdersCount > 1 ? "No" : "Yes";     // ⚠️ BUG-05: always "Yes"

// Extract and Filter Line Items
// ⚠️ BUG-01: Does not expand quantity > 1 items — creates only 1 LI entry per line item object
let LI = (Order.line_items || []).filter(item =>
  item.sku?.startsWith("W-") ||
  item.sku?.startsWith("M-") ||
  item.sku?.startsWith("CB") ||
  item.sku?.startsWith("VID-GIFT-") ||
  item.sku?.startsWith("SHOE-")
).map(item => JSON.stringify(item));

let LIcount = LI.length;

// Function to Clean Up Note Attributes
function cleanNoteAttributes(NA) {
  if (!Array.isArray(NA)) return [];

  return NA.map(attr => ({
    name: attr.name,
    value: attr.name === "__aftersell_tamper_proof" ? "" : attr.value
  }));
}

// Extract and Clean Note Attributes
let NA = cleanNoteAttributes(Order.note_attributes);
let NAcount = NA.length;

// Convert Note Attributes to Extras Object
let Extras = {};
NA.forEach(attr => {
  if (attr.name) Extras[attr.name] = attr.value;
});

// BDJ User Data Extraction
let BDJUserData = {};
if (typeof Extras["BDJ User Data"] === "string") {
  // ⚠️ BUG-06: split("\n") leaves \r residue from Windows \r\n line endings
  Extras["BDJ User Data"].split("\n").forEach(line => {
    let [key, ...valueParts] = line.split(": ");
    let value = valueParts.join(": "); // Handles cases where value contains `:`
    if (key && value) BDJUserData[key.trim()] = value.trim();
  });
  delete Extras["BDJ User Data"];
}
let BDJUserDataCount = Object.keys(BDJUserData).length;

// Extract Order Note
let Note = Order.note || "";

// Final Output
output = [{
  RUSH,
  NAcount,
  LIcount,
  CustomerOrdersCount,
  BDJUserDataCount,
  CustomerNew,
  OrderType,
  Extras,
  BDJUserData,
  Note,
  LI,
  NA
}];
```

---

## Code Walkthrough

### 1. RUSH Detection (Lines 1–14)

```jsx
let TAGS = inputData.TAGS || "";
TAGS = TAGS.toString().toUpperCase();

if (TAGS.includes("RUSH ROYAL")) {
  RUSH = " | RUSH ROYAL";
} else if (TAGS.includes("RUSH COBALT")) {
  RUSH = " | RUSH COBALT";
}
```

Blue Delta has two rush tiers for priority production. The code uppercases the tags string first so the check is case-insensitive, then looks for the specific rush phrases.

**RUSH ROYAL** is the highest priority — checked first so it takes precedence if both tags somehow exist. **RUSH COBALT** is the standard rush tier.

The output is a suffix string (e.g., `" | RUSH ROYAL"`) that gets appended to the Asana task name. If no rush tag exists, `RUSH` is an empty string and the task name has no suffix.

**Example from live run (order #100002):** Tags = `"Andy, Denim, RUSH COBALT"` → uppercased to `"ANDY, DENIM, RUSH COBALT"` → matches `"RUSH COBALT"` → `RUSH = " | RUSH COBALT"` → Asana task name: `#100002 Alex Rivera 1/1 | RUSH COBALT`

---

### 2. JSON Parsing & Validation (Lines 16–25)

```jsx
let RAW;
try {
  RAW = JSON.parse(inputData.RAW);
} catch (error) {
  throw new Error("Invalid JSON in inputData.RAW");
}

if (!RAW.order) throw new Error("Invalid data structure: Missing 'order' object");

let Order = RAW.order;
```

Parses the raw API response string into a JavaScript object and validates that it contains an `order` property. If the JSON is malformed or the structure is unexpected, the step throws an error and the zap run fails (which is the correct behavior — better to fail loudly than create garbage Asana tasks).

---

### 3. Order Type Detection (Line 28)

```jsx
let OrderType = Order.cart_token ? "Online" : "Direct";
```

The simplest and most consequential line in the step. If the Shopify order has a `cart_token` value, it came from the online storefront checkout → `"Online"`. If not, it came from POS, a draft order, or the API → `"Direct"`.

This matters for two things downstream:

1. **Due dates** — Online pants get 14-day due dates, Direct/POS pants get 30 days
2. **Data sources** — Online orders carry thread/monogram as line item properties, POS orders encode them in the SKU

Zapier censors the cart token in run logs (`:censored:24:37b9e3afb5:`) but the JavaScript truthiness check still works — a censored non-empty string is truthy.

**Live run verification:** Order #114265 had `cart_token: ":censored:24:37b9e3afb5:"` → `OrderType = "Online"`. Correct.

---

### 4. Customer New/Returning Detection (Lines 31–32)

```jsx
let CustomerOrdersCount = Order.customer?.orders_count || 0;
let CustomerNew = CustomerOrdersCount > 1 ? "No" : "Yes";
```

Attempts to read the customer's total order count to determine if this is their first purchase. If `orders_count > 1`, the customer is returning (`"No"`). Otherwise, they're new (`"Yes"`).

> **BUG-05 (MEDIUM):** The Shopify REST API's `/orders/{id}.json` endpoint does NOT include `orders_count` in the nested customer object. This field is always `undefined`, which defaults to `0` via the `|| 0` fallback. Since `0 > 1` is false, `CustomerNew` is always `"Yes"` — even for customers with 50+ previous orders.
> 
> 
> The trigger data (Step 1) does have `numberOfOrders` via GraphQL, but the code reads from the REST response (Step 4's `RAW`), not the trigger.
> 
> **V4 fix (deployed, live at zap v51):** Add `numberOfOrders` as a fallback: `Order.customer?.orders_count ?? Order.customer?.numberOfOrders ?? 0`
> 

---

### 5. Line Item Filtering (Lines 35–43)

```jsx
let LI = (Order.line_items || []).filter(item =>
  item.sku?.startsWith("W-") ||
  item.sku?.startsWith("M-") ||
  item.sku?.startsWith("CB") ||
  item.sku?.startsWith("VID-GIFT-") ||
  item.sku?.startsWith("SHOE-")
).map(item => JSON.stringify(item));

let LIcount = LI.length;
```

This is the gatekeeper. It filters the order's line items to only keep production-qualifying products — items that need a physical production task in Asana. The five SKU prefixes correspond to:

| Prefix | Products |
| --- | --- |
| `W-` | Women's pants (all fabrics), women's shorts, women's derby pants |
| `M-` | Men's pants (all fabrics), men's shorts, men's derby pants |
| `CB` | All belts (online and POS, all leather types) |
| `SHOE-` | All shoes (Classic and Custom) |
| `VID-GIFT-` | Video gift cards |

Everything else is discarded: shipping protection line items, tips, discount codes, regular (non-video) gift cards, merchandise (hats, jackets), and any other non-production items.

Each qualifying item is **stringified** via `JSON.stringify()` because Zapier's Loop step (Step 10) requires an array of strings, not objects. Step 13 later parses each item back with `JSON.parse()`.

> **BUG-01 (CRITICAL):** When a customer orders 2 of the exact same product (same SKU, same variant, same customizations), Shopify creates a single line item with `quantity: 2` instead of two separate line item objects. The `.filter().map()` chain creates one LI array entry per line item object, not per unit. The loop in Step 10 runs once, and only 1 Asana task is created — the second physical item is never tracked.
> 
> 
> This works fine when products differ (different SKU, different variant, different properties) because Shopify creates separate line item objects for each.
> 
> **V4 fix (deployed, live at zap v51):** Expand items with `quantity > 1` into multiple entries:
> 
> ```jsx
> let LI = [];
> (Order.line_items || []).filter(item => isProductionSKU(item.sku)).forEach(item => {
>   const qty = item.quantity || 1;
>   for (let i = 0; i < qty; i++) {
>     LI.push(JSON.stringify({ ...item, quantity: 1 }));
>   }
> });
> ```
> 

---

### 6. Note Attributes Cleaning (Lines 46–62)

```jsx
function cleanNoteAttributes(NA) {
  if (!Array.isArray(NA)) return [];
  return NA.map(attr => ({
    name: attr.name,
    value: attr.name === "__aftersell_tamper_proof" ? "" : attr.value
  }));
}

let NA = cleanNoteAttributes(Order.note_attributes);
let NAcount = NA.length;

let Extras = {};
NA.forEach(attr => {
  if (attr.name) Extras[attr.name] = attr.value;
});
```

Shopify's `note_attributes` is an array of `{name, value}` objects attached to the order. This section does three things:

**First,** it cleans the array by blanking the `__aftersell_tamper_proof` attribute. AfterSell (a post-purchase upsell app) writes a long hash string to this attribute. Blanking it keeps the data clean without removing the attribute entirely.

**Second,** it counts the attributes (`NAcount`) for informational purposes.

**Third,** it converts the array into a flat key-value object called `Extras`. This is the object that downstream steps use to access Virtual Tailor measurements and other order-level custom data.

> **⚠️ Stale for NEW online orders (June 2026):** The example below is from a *pre-migration* order (#114265, March 2026), when the cart still wrote Bold Metrics measurements into `note_attributes`. For orders placed **after** the Bold Metrics → Skynet cutover, the six measurement keys (`Hip Circum`, `Jean Inseam`, `Knee Circum`, `Thigh Circum`, `U Crotch`, `Waist Average`) are **absent/empty** in `Extras` — Skynet now computes them post-order and writes them straight into Asana. `Virtual Tailor`, `Jean Fit`, `Shoe Type`, and the POS body-measurement keys are unaffected. See the §STATUS callout at the top of this page and the [migration bridge doc](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md).

**Example `Extras` object from live run (order #114265 — *pre-migration*, measurement keys still present):**

```json
{
  "Virtual Tailor": "Yes",
  "Hip Circum": "41.83",
  "Jean Inseam": "30.00",
  "Knee Circum": "15.68",
  "Thigh Circum": "24.06",
  "U Crotch": "29.65",
  "Waist Average": "34.52",
  "Jean Fit": "Tailored"
}
```

These values map directly to Asana custom fields in Step 21 (Pant Pipeline):

- `Extras["Hip Circum"]` → Asana "Hip Circum" field
- `Extras["Jean Inseam"]` → Asana "Jean Inseam" field
- `Extras["Waist Average"]` → Asana "Waist Avg." field
- etc.

> **⚠️ Historical mapping (pre-migration).** This six-field mapping reflects how Bold Metrics measurements *used to* flow from `Extras` into Asana. **For NEW online orders these `Extras` keys are empty**, so the Step 21 mapping receives blanks; **Skynet writes the six measurement custom fields (`Waist Avg.`, `Hip Circum`, `Thigh Circum`, `Knee Circum`, `U Crotch`, `Jean Inseam`) directly via the Asana API after task creation.** The Zapier-side `Extras` → Asana mapping is now a no-op for these fields on new orders. POS/in-person measurements (below) and the VT inputs in `BDJUserData` are unaffected.

For in-person measured orders (POS), `Extras` also contains detailed body measurement fields: Seat Down, Seat Around, Thigh Upper/Middle/Lower (Down and Around), Outseam, Knee Up/Around, Calf Up/Around, Front Rise, Full Rise, Leg Opening, Measured By, Photo 1, Photo 2. These are empty for online/Virtual Tailor orders.

---

### 7. BDJ User Data Parsing (Lines 65–74)

```jsx
let BDJUserData = {};
if (typeof Extras["BDJ User Data"] === "string") {
  Extras["BDJ User Data"].split("\n").forEach(line => {
    let [key, ...valueParts] = line.split(": ");
    let value = valueParts.join(": ");
    if (key && value) BDJUserData[key.trim()] = value.trim();
  });
  delete Extras["BDJ User Data"];
}
let BDJUserDataCount = Object.keys(BDJUserData).length;
```

One of the note attributes is called `"BDJ User Data"` and contains a newline-delimited block of key-value pairs from the Virtual Tailor form on the storefront. It looks like this in the raw Shopify data:

> **⚠️ Now load-bearing (June 2026).** Post Bold Metrics → Skynet migration, the VT *inputs* parsed here are the durable record of what the customer entered — Skynet reads these (plus five NEW `VT *` Asana fields populated by a separate Zapier mapping that is now **DONE and live at zap v51**) to call Bold Metrics and compute measurements after the order. Pre-migration, only *some* of these inputs were preserved (and the cart consumed the rest), which is exactly the data-loss problem the migration fixes. The parser code below is unchanged; its output simply matters more now.

```
Gender: Female
Age: 51
Height: 70
Weight: 155
Shoe Size: 10
Bra Size: 34B
Inseam: 30
Jean Fit: Tailored
Common Shoe: undefined
Style: (see SKU)

Notes:
```

The parser splits on `\n`, then splits each line on `:`  (colon-space) to extract key-value pairs. The `valueParts.join(": ")` handles cases where the value itself contains a colon. The `if (key && value)` check filters out empty lines and the trailing `"Notes: "` line (where value is empty).

After parsing, the raw `"BDJ User Data"` string is deleted from `Extras` (since it's been expanded into the `BDJUserData` object).

**Example `BDJUserData` object from live run:**

```json
{
  "Gender": "Female",
  "Age": "51",
  "Height": "70",
  "Weight": "155",
  "Shoe Size": "10",
  "Bra Size": "34B",
  "Inseam": "30",
  "Jean Fit": "Tailored",
  "Common Shoe": "undefined",
  "Style": "(see SKU)"
}
```

These values map to Asana custom fields:

- `BDJUserData["Age"]` → Asana "Age" field
- `BDJUserData["Weight"]` → Asana "Weight" field
- `BDJUserData["Body"]` → Asana "Body" field (when present)
- `BDJUserData["Break"]` → Asana "Break" field (when present)
- `BDJUserData["Waist Ride"]` → Asana "Waist Ride" field (when present)

> **BUG-06 (LOW):** The Shopify API returns `BDJ User Data` with Windows-style `\r\n` line endings. The code splits on `\n`, which leaves `\r` at the end of each value. So `BDJUserData["Gender"]` is actually `"Female\r"` not `"Female"`. The `.trim()` call on the value handles most cases, but the key side also gets trimmed — so this may not cause visible issues. Still, it's sloppy.
> 
> 
> **V4 fix (deployed, live at zap v51):** Strip `\r` before splitting: `Extras["BDJ User Data"].replace(/\r/g, "").split("\n")`
> 

---

### 8. Order Note Extraction (Line 77)

```jsx
let Note = Order.note || "";
```

Extracts the free-text order note. This is the note field that staff or customers can add to the order in Shopify. Maps to the Asana "Note" field in Step 21.

From the live runs, this was `null` for both test orders (no note entered), so `Note` defaulted to `""`.

---

## Output Schema

Every field Step 5 outputs and where it's consumed downstream:

| Field | Type | Example | Consumed By |
| --- | --- | --- | --- |
| `RUSH` | string | `" | RUSH COBALT"` or `""` | Steps 21/23/25/27: Asana task name suffix |
| `NAcount` | number | `9` | Informational only |
| `LIcount` | number | `1` | Step 6: filter (must be > 0 to continue) |
| `CustomerOrdersCount` | number | `0` (bugged) | Informational only |
| `BDJUserDataCount` | number | `10` | Informational only |
| `CustomerNew` | string | `"Yes"` (bugged) or `"No"` | Step 21: Asana "New Customer" field |
| `OrderType` | string | `"Online"` or `"Direct"` | Steps 21/23/25/27: Asana "Order Type" field |
| `Extras` | object | `{"Virtual Tailor":"Yes","Hip Circum":"41.83",...}` | Step 21: Asana measurement fields (Hip Circum, Jean Inseam, Knee Circum, Thigh Circum, U Crotch, Waist Average, Waist Around, Seat Down/Around, all Thigh fields, Outseam, Knee Up/Around, Calf Up/Around, Front/Full Rise, Leg Opening, Measured By, Photo 1/2), Pattern On File, White Glove, Jean Fit, Shoe Type. Step 23: Waist Average, Waist Around, Pattern On File, White Glove. **⚠️ June 2026: the 6 VT/Bold Metrics measurement keys (Hip Circum, Jean Inseam, Knee Circum, Thigh Circum, U Crotch, Waist Average) are now EMPTY for new online orders — Skynet writes those Asana fields post-order. POS in-person measurement keys are unaffected.** |
| `BDJUserData` | object | `{"Gender":"Female","Age":"51",...}` | Step 21: Asana Age, Weight, Body, Break, Waist Ride fields |
| `Note` | string | `""` or `"Please rush this"` | Steps 21/23/25/27: Asana "Note" field |
| `LI` | string[] | `['{"sku":"W-RW26_NATURALINDIGO-FLARE",...}']` | Step 10: Loop input (iterated, each item passed to Step 13) |
| `NA` | object[] | `[{"name":"Virtual Tailor","value":"Yes"},...]` | Informational only (raw note attributes before conversion) |

---

## Full Code — V4 Fixed (Deployed, live at zap v51)

Three fixes applied: BUG-01 (quantity expansion), BUG-05 (customer count fallback), BUG-06 (carriage return stripping). Changes marked with `// FIX` comments. These fixes are deployed and live as of zap v51 (June 2026).

```jsx
// ============================================================
// Step 5 — Order Parser (V4)
// Fixes: BUG-01 (duplicate items), BUG-05 (CustomerOrdersCount),
//        BUG-06 (carriage return residue)
// ============================================================

// Input variable: TAGS
let TAGS = inputData.TAGS || "";

// Uppercase for consistent matching
TAGS = TAGS.toString().toUpperCase();

// Default
let RUSH = "";

// Check for specific matches
if (TAGS.includes("RUSH ROYAL")) {
  RUSH = " | RUSH ROYAL";
} else if (TAGS.includes("RUSH COBALT")) {
  RUSH = " | RUSH COBALT";
}

let RAW;
try {
  RAW = JSON.parse(inputData.RAW);
} catch (error) {
  throw new Error("Invalid JSON in inputData.RAW");
}

// Ensure RAW has an order object
if (!RAW.order) throw new Error("Invalid data structure: Missing 'order' object");

let Order = RAW.order; // Extract order object

// Determine Order Type
let OrderType = Order.cart_token ? "Online" : "Direct";

// Customer Details
// FIX BUG-05: Check both REST (orders_count) and GraphQL (numberOfOrders) field names
let CustomerOrdersCount = Order.customer?.orders_count
  ?? Order.customer?.numberOfOrders
  ?? 0;
let CustomerNew = CustomerOrdersCount > 1 ? "No" : "Yes";

// Valid SKU prefixes for production pipeline
const isProductionSKU = (sku) =>
  sku?.startsWith("W-") ||
  sku?.startsWith("M-") ||
  sku?.startsWith("CB") ||
  sku?.startsWith("VID-GIFT-") ||
  sku?.startsWith("SHOE-");

// FIX BUG-01: Expand items with quantity > 1 into individual LI entries
// Each entry gets quantity: 1 so the downstream loop creates one Asana task per unit
let LI = [];
(Order.line_items || []).filter(item => isProductionSKU(item.sku)).forEach(item => {
  const qty = item.quantity || 1;
  for (let i = 0; i < qty; i++) {
    LI.push(JSON.stringify({ ...item, quantity: 1 }));
  }
});

let LIcount = LI.length;

// Function to Clean Up Note Attributes
function cleanNoteAttributes(NA) {
  if (!Array.isArray(NA)) return [];

  return NA.map(attr => ({
    name: attr.name,
    value: attr.name === "__aftersell_tamper_proof" ? "" : attr.value
  }));
}

// Extract and Clean Note Attributes
let NA = cleanNoteAttributes(Order.note_attributes);
let NAcount = NA.length;

// Convert Note Attributes to Extras Object
let Extras = {};
NA.forEach(attr => {
  if (attr.name) Extras[attr.name] = attr.value;
});

// BDJ User Data Extraction
// FIX BUG-06: Strip \r before splitting on \n to handle Windows line endings
let BDJUserData = {};
if (typeof Extras["BDJ User Data"] === "string") {
  Extras["BDJ User Data"].replace(/\r/g, "").split("\n").forEach(line => {
    let [key, ...valueParts] = line.split(": ");
    let value = valueParts.join(": ");
    if (key && value) BDJUserData[key.trim()] = value.trim();
  });
  delete Extras["BDJ User Data"];
}
let BDJUserDataCount = Object.keys(BDJUserData).length;

// Extract Order Note
let Note = Order.note || "";

// Final Output
output = [{
  RUSH,
  NAcount,
  LIcount,
  CustomerOrdersCount,
  BDJUserDataCount,
  CustomerNew,
  OrderType,
  Extras,
  BDJUserData,
  Note,
  LI,
  NA
}];
```

---

## Bugs in This Step

| # | Title | Severity | Status | Quick Summary |
| --- | --- | --- | --- | --- |
| BUG-01 | Duplicate items collapsed | **CRITICAL** | Fixed & deployed (zap v51) | Items with `quantity > 1` only create 1 LI entry instead of N entries. Production tasks are lost. |
| BUG-05 | CustomerOrdersCount always 0 | **MEDIUM** | Fixed & deployed (zap v51) | `orders_count` doesn't exist in the REST response. `CustomerNew` is always "Yes." |
| BUG-06 | Carriage return residue | **LOW** | Fixed & deployed (zap v51) | `\r\n` line endings in BDJ User Data leave `\r` in parsed values. |

For full bug analysis with affected SKU counts, root cause, and test cases, see the **Bug Tracker & Fix Plan** page.

---

## Live Run Data

### Order #114265 — Erin Test (Women's Raw Denim, Online, No Rush)

**Step 5 output:**

```
RUSH: ""
NAcount: 9
LIcount: 1
CustomerOrdersCount: 0        ← BUG-05: should be 1
CustomerNew: "Yes"             ← correct for this order (first purchase)
BDJUserDataCount: 10
OrderType: "Online"
Extras: {
  "Virtual Tailor": "Yes",
  "Hip Circum": "41.83",
  "Jean Inseam": "30.00",
  "Knee Circum": "15.68",
  "Thigh Circum": "24.06",
  "U Crotch": "29.65",
  "Waist Average": "34.52",
  "Jean Fit": "Tailored"
}
BDJUserData: {
  "Gender": "Female", "Age": "51", "Height": "70", "Weight": "155",
  "Shoe Size": "10", "Bra Size": "34B", "Inseam": "30",
  "Jean Fit": "Tailored", "Common Shoe": "undefined", "Style": "(see SKU)"
}
Note: ""
LI: [1 item — "W-RW26_NATURALINDIGO-FLARE"]
```

**Runtime:** 86 MB memory, 216ms duration.

---

### Order #100002 — Alex Rivera (Men's Raw Denim, Online, RUSH COBALT, Monogram)

**Step 5 output:**

```
RUSH: " | RUSH COBALT"
NAcount: 10
LIcount: 1
CustomerOrdersCount: 0        ← BUG-05
CustomerNew: "Yes"
BDJUserDataCount: 10
OrderType: "Online"
Extras: {
  "Virtual Tailor": "Yes",
  "Hip Circum": "46.20",
  "Jean Inseam": "30.00",
  "Knee Circum": "17.21",
  "Thigh Circum": "27.32",
  "U Crotch": "29.14",
  "Waist Average": "41.78",
  "Jean Fit": "Tailored",
  "Shoe Type": "Chelsea Boots"
}
BDJUserData: {
  "Gender": "Male", "Age": "40", "Height": "72", "Weight": "260",
  "Shoe Size": "12", "Waist": "38", "Inseam": "30",
  "Jean Fit": "Tailored", "Common Shoe": "Chelsea Boots", "Style": "(see SKU)"
}
Note: ""
LI: [1 item — "M-RW26_NATURALINDIGO-STRAIGHT" with monogram properties]
```

**Notable:** This order had `"Shoe Type": "Chelsea Boots"` in Extras (from note attributes) — this maps to the Asana "Shoe Type" field in Step 21, which shows the customer's preferred shoe type (not a shoe product).

**Runtime:** 86 MB memory, 211ms duration.

---

*This page documents step-5.js. The zap is now live at **v51** (June 2026), with the V4 bug fixes plus the VT* input-field mapping deployed. See the Bug Tracker page for history.*