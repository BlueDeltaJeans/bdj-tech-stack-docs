# Bug Tracker & Fix Plan

This page documents every bug found during the V4 code audit, their fixes, and the deployment plan. The audit analyzed `step-5.js` (97 lines) and `step-13.js` (147 lines) against all 5,555 current SKUs and the 4 Asana field mapping transcriptions (Steps 21, 23, 25, 27).

**Audit date:** April 4, 2026
**Files audited:** step-5.js, step-13.js
**Fixed files produced:** step-5-v4.js (3 fixes), step-13-v4.js (8 fixes)
**Test cases:** 22 tests with 71 assertions (in v4-test-cases.md)
**Status:** All fixes written and tested. **Not yet deployed** to the live Zapier automation.

---

## Bug Summary

| # | Title | Severity | Step | Status | Fix Summary |
| --- | --- | --- | --- | --- | --- |
| BUG-01 | Duplicate items only create 1 Asana task | **CRITICAL** | 5 | Fixed in V4 code | Expand line items by quantity |
| BUG-02 | POS belt monogram not detected from SKU | **CRITICAL** | 13 | Fixed in V4 code | Segment-count belt detection |
| BUG-03 | POS belt thread misread (NO_THREAD, MONO) | **HIGH** | 13 | Fixed in V4 code | Segment-count belt detection |
| NEW-01 | Shoe SKU format changed, parser broken | **HIGH** | 13 | Fixed in V4 code | Parse gender+size from SKU segments |
| NEW-04 | Belt POS detection via fragile Source check | **HIGH** | 13 | Fixed in V4 code | Replace with SKU segment count |
| BUG-05 | CustomerOrdersCount always 0 | **MEDIUM** | 5 | Fixed in V4 code | Add numberOfOrders fallback |
| NEW-03 | FLATBLACK not normalized to FLAT_BLACK | **MEDIUM** | 13 | Fixed in V4 code | Normalize in belt hardware extraction |
| BUG-06 | Carriage return in BDJ User Data | **LOW** | 5 | Fixed in V4 code | Strip \r before split |
| BUG-07 | Case-sensitive tag matching | **LOW** | 13 | Fixed in V4 code | Lowercase comparison |
| NEW-02 | Ryder dead code still in Step 13 | **LOW** | 13 | Fixed in V4 code | Removed all Ryder branches |
| BUG-04 | Ryder compound Thread/Logo value | **N/A** | 13 | Removed | Ryder products deleted from Shopify |

---

## Critical Bugs

### BUG-01: Duplicate Items Only Create 1 Asana Task

| Detail | Value |
| --- | --- |
| **Severity** | CRITICAL — Production tasks are being lost |
| **Step** | 5 (step-5.js) |
| **Affected SKUs** | ALL 5,555 — any item with quantity > 1 |
| **Test cases** | TEST-S5-01, TEST-S5-02, TEST-S5-03 |

**The problem:** When a customer orders 2 of the exact same product (same SKU, same variant, same customizations), Shopify creates a single line item with `quantity: 2` instead of two separate line item objects. Step 5's `.filter().map()` chain creates one LI array entry per line item object, not per unit. The loop in Step 10 runs once, and only 1 Asana task is created. The second physical item is never tracked in production.

This works fine when products differ (different SKU, variant, or properties) because Shopify creates separate line item objects for each.

**Current code:**

```jsx
let LI = (Order.line_items || []).filter(item =>
  item.sku?.startsWith("W-") ||
  item.sku?.startsWith("M-") ||
  item.sku?.startsWith("CB") ||
  item.sku?.startsWith("VID-GIFT-") ||
  item.sku?.startsWith("SHOE-")
).map(item => JSON.stringify(item));
```

**Fixed code (step-5-v4.js):**

```jsx
const isProductionSKU = (sku) =>
  sku?.startsWith("W-") ||
  sku?.startsWith("M-") ||
  sku?.startsWith("CB") ||
  sku?.startsWith("VID-GIFT-") ||
  sku?.startsWith("SHOE-");

let LI = [];
(Order.line_items || []).filter(item => isProductionSKU(item.sku)).forEach(item => {
  const qty = item.quantity || 1;
  for (let i = 0; i < qty; i++) {
    LI.push(JSON.stringify({ ...item, quantity: 1 }));
  }
});
```

**How it works:** After filtering for production SKUs, the fix loops `item.quantity` times for each line item, creating individual entries with `quantity: 1`. An order with 3 identical pants now produces 3 LI entries → 3 loop iterations → 3 Asana tasks.

**Verification:** Order with `M-RW29_DARKINDIGO-STRAIGHT` qty 3 → `LIcount` should be `3`, LI array should have 3 entries each with `"quantity":1`.

---

### BUG-02: POS Belt Monogram Not Detected From SKU

| Detail | Value |
| --- | --- |
| **Severity** | CRITICAL — POS belt monograms are silently lost |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | 819 POS belt SKUs with MONO segment (all 5-segment belts) |
| **Test cases** | TEST-S13-06, TEST-S13-07 |

**The problem:** Step 13 sets `Monogram` from a fallback chain of line item property names (`Properties.Monogram ?? Properties.monogram ?? Properties.Mono ?? Properties.mono ?? "NONE"`). POS belt orders have no line item properties — SC Product Options doesn't work in POS. All four fallback checks return `undefined`, so Monogram defaults to `"NONE"`. But POS belt SKUs with monogram have a `MONO` segment (e.g., `CB6-1-BRASS-MONO-BLACK`). The code never checks for this.

Every POS belt customer who requests a monogram gets `"NONE"` in their Asana task. Production staff don't see that a monogram was ordered.

**Current code:**

```jsx
Monogram:
  Properties.Monogram ??
  Properties.monogram ??
  Properties.Mono ??
  Properties.mono ??
  "NONE",
```

**Fixed code (step-13-v4.js) — part of the full belt rewrite:**

```jsx
if (beltSegments === 3) {
  // Online: monogram from line item properties
  Properties.Monogram =
    Properties.Monogram ?? Properties.monogram ??
    Properties.Mono ?? Properties.mono ?? "NONE";
} else if (beltSegments === 4) {
  // POS, no monogram
  Properties.Monogram = "NONE";
} else if (beltSegments === 5) {
  // POS, with monogram (MONO is SKU[3])
  Properties.Monogram = "YES";
}
```

**How it works:** Instead of checking line item properties for all belts, the fix uses SKU segment count. 5-segment SKUs always have `MONO` in position 3 → `Monogram = "YES"`. 4-segment SKUs have no MONO → `"NONE"`. 3-segment online SKUs still use the property fallback chain.

**Verification:** `CB6-1-BRASS-MONO-BLACK` → `Properties.Monogram` should be `"YES"`. `CB6-1-BRASS-BLACK` → should be `"NONE"`.

---

## High-Severity Bugs

### BUG-03: POS Belt Thread Color Misread

| Detail | Value |
| --- | --- |
| **Severity** | HIGH — Thread field gets incorrect values |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | All 1,638 POS belt SKUs. Critical subset: ~126 with NO_THREAD |
| **Test cases** | TEST-S13-08, TEST-S13-09, TEST-S13-18 |

**The problem (3 sub-issues):**

**3a:** POS belt SKUs with `NO_THREAD` (e.g., `CB6-1-BRASS-NO_THREAD`) set `Properties["Thread Color"] = "NO_THREAD"` literally. The Asana Belt-Stitching field should show blank/empty, not the string "NO_THREAD".

**3b:** For `CB6-1-BRASS-MONO-NO_THREAD`, `SKU[SKU.length - 1]` gives `"NO_THREAD"` — same issue as 3a but in the 5-segment pattern.

**3c:** The code uses `Source === "pos"` to decide whether to parse thread from the SKU. This depends on `inputData.Source` being correctly set upstream, which is fragile.

**Current code:**

```jsx
const isPOS = Source === "pos";
if (isRyder || isPOS) {
  Properties["Thread Color"] = SKU[SKU.length - 1];
}
```

**Fixed code (step-13-v4.js) — part of the full belt rewrite:**

```jsx
if (beltSegments === 4) {
  const threadSeg = SKU[3];
  Properties["Thread Color"] = threadSeg === "NO_THREAD" ? "" : threadSeg;
} else if (beltSegments === 5) {
  const threadSeg = SKU[4];
  Properties["Thread Color"] = threadSeg === "NO_THREAD" ? "" : threadSeg;
}
```

**How it works:** Uses segment count instead of Source check. Maps `"NO_THREAD"` to empty string. For 4-segment SKUs, thread is at position 3. For 5-segment SKUs, thread is at position 4 (because position 3 is `MONO`).

**Verification:** `CB6-1-BRASS-NO_THREAD` → Thread Color should be `""` (empty). `CB6-1-BRASS-BLACK` → should be `"BLACK"`.

---

### NEW-01: Shoe SKU Format Changed — Old Parser Broken

| Detail | Value |
| --- | --- |
| **Severity** | HIGH — Gender and size fields are wrong for all shoe orders |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | All 76 shoe SKUs |
| **Test cases** | TEST-S13-12, TEST-S13-13 |

**The problem:** The shoe code was written for the old SKU format `SHOE-CLASSIC` (2 segments) where gender and size were parsed from `variant_title` (e.g., `"Men / 10"`). The SKUs were updated to `SHOE-CLASSIC-M-10` (4 segments) with gender and size in the SKU itself. The old `variant_title` parsing produces empty or wrong values for the new format.

**Current code:**

```jsx
Properties.Shoe = SKU[1];
if (LI.variant_title?.includes("/")) {
  let [gender, size] = LI.variant_title.split("/").map(item => item.trim());
  Properties.Gender = gender || "";
  Properties.Size = size || "";
}
```

**Fixed code (step-13-v4.js):**

```jsx
Properties.Shoe = SKU[1]; // "CLASSIC" or "CUSTOM"

if (SKU.length >= 4) {
  // New format: SHOE-{Type}-{Gender}-{Size}
  Properties.Gender = SKU[2] || "";
  Properties.Size = SKU[3] || "";
} else if (LI.variant_title?.includes("/")) {
  // Legacy fallback
  let [gender, size] = LI.variant_title.split("/").map(s => s.trim());
  Properties.Gender = gender || "";
  Properties.Size = size || "";
}
```

**Verification:** `SHOE-CLASSIC-M-10` → Gender: `"M"`, Size: `"10"`, Shoe: `"CLASSIC"`.

---

### NEW-04: Belt POS Detection Uses Fragile Source Check

| Detail | Value |
| --- | --- |
| **Severity** | HIGH — Belt thread/monogram parsing depends on unreliable upstream data |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | All 1,638 POS belt SKUs |
| **Test cases** | TEST-S13-18 |

**The problem:** The belt code uses `Source === "pos"` to decide whether to parse thread from the SKU. This relies on `inputData.Source` being correctly passed from upstream. If Source is empty or wrong, POS belt thread colors won't be parsed at all.

A more reliable approach: determine Online vs POS from the SKU structure itself. Online belts always have 3 segments. POS belts always have 4 or 5 segments. The SKU never lies.

**Fix:** Replaced entirely by the segment-count detection in the BUG-02/BUG-03 belt rewrite. The `Source` variable is no longer used in belt processing at all.

**Verification:** `CB6-1-BRASS-BLACK` with `Source = ""` (empty) → Thread Color should still be `"BLACK"` (detected from segment count, not Source).

---

## Medium-Severity Bugs

### BUG-05: CustomerOrdersCount Always 0

| Detail | Value |
| --- | --- |
| **Severity** | MEDIUM — "New Customer" is always wrong for returning customers |
| **Step** | 5 (step-5.js) |
| **Affected** | Every order from a returning customer |
| **Test cases** | TEST-S5-04, TEST-S5-05, TEST-S5-06 |

**The problem:** Step 5 reads `Order.customer?.orders_count || 0`. The Shopify REST API's `/orders/{id}.json` endpoint does NOT include `orders_count` in the nested customer object. It's always `undefined`, defaulting to `0`. Since `0 > 1` is false, `CustomerNew` is always `"Yes"`.

The trigger data (Step 1) has `numberOfOrders` via GraphQL, but the code reads from the REST response (Step 4's `RAW`), not the trigger.

**Current code:**

```jsx
let CustomerOrdersCount = Order.customer?.orders_count || 0;
```

**Fixed code (step-5-v4.js):**

```jsx
let CustomerOrdersCount = Order.customer?.orders_count
  ?? Order.customer?.numberOfOrders
  ?? 0;
```

**Note:** This fix adds a fallback to `numberOfOrders` but it may still not work if the REST response doesn't include either field name. The definitive fix would be to pass `numberOfOrders` from the Step 1 trigger data as a separate input to Step 5, but that requires changing the Zapier step configuration (not just the code). The current fix is the best we can do within the code step alone.

---

### NEW-03: FLATBLACK Not Normalized to FLAT_BLACK

| Detail | Value |
| --- | --- |
| **Severity** | MEDIUM — Inconsistent hardware values in Asana |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | 546 POS belt SKUs with FLATBLACK hardware |
| **Test cases** | TEST-S13-10, TEST-S13-11 |

**The problem:** POS belt SKUs use `FLATBLACK` (no underscore), online belt SKUs use `FLAT_BLACK` (with underscore). The code passes the raw SKU segment, so the Asana Belt-Hardware field shows different values for the same hardware depending on order source.

**Fixed code (step-13-v4.js):**

```jsx
let hardware = SKU[2];
if (hardware === "FLATBLACK") {
  hardware = "FLAT_BLACK";
}
```

**Verification:** `CB6-1-FLATBLACK-BLACK` → Hardware should be `"FLAT_BLACK"` (normalized).

---

## Low-Severity Bugs

### BUG-06: Carriage Return Residue in BDJ User Data

| Detail | Value |
| --- | --- |
| **Severity** | LOW — Values may have trailing \r characters |
| **Step** | 5 (step-5.js) |
| **Test cases** | TEST-S5-07 |

**The problem:** Shopify sends `BDJ User Data` with Windows-style `\r\n` line endings. Step 5 splits on `\n`, leaving `\r` at the end of each value. The `.trim()` call handles most cases, but it's sloppy.

**Current code:**

```jsx
Extras["BDJ User Data"].split("\n").forEach(line => {
```

**Fixed code (step-5-v4.js):**

```jsx
Extras["BDJ User Data"].replace(/\r/g, "").split("\n").forEach(line => {
```

---

### BUG-07: Case-Sensitive Tag Matching

| Detail | Value |
| --- | --- |
| **Severity** | LOW — Product tags could be miscategorized |
| **Step** | 13 (step-13.js) |
| **Test cases** | TEST-S13-16, TEST-S13-17 |

**The problem:** `TagsProduct` uses title case (`"Denim"`, `"Cashiers"`) but Shopify tags can be any case. A tag of `"denim"` (lowercase) would not match `"Denim"` and would end up in EventTags instead of Tags. Similarly, `Tags.includes("alteration")` only matches lowercase.

**Current code:**

```jsx
let TagsProduct = ["Cashiers", "Denim", "Vintage", "Performance", "Chino", "Jhino"];
let Alteration = Tags.includes("alteration");
```

**Fixed code (step-13-v4.js):**

```jsx
let TagsProduct = ["cashiers", "denim", "vintage", "performance", "chino", "jhino"];
let tagsLower = Tags.map(tag => tag.toLowerCase());
let Alteration = tagsLower.includes("alteration");
let EventTags = Tags.filter((_, i) => !TagsProduct.includes(tagsLower[i]));
Tags = Tags.filter((_, i) => TagsProduct.includes(tagsLower[i]));
```

The fix compares lowercase versions but preserves original casing in the output. `"DENIM"` matches `"denim"` but outputs as `"DENIM"`.

---

### NEW-02: Ryder Dead Code Still in Step 13

| Detail | Value |
| --- | --- |
| **Severity** | LOW — Non-functional code, no runtime impact |
| **Step** | 13 (step-13.js) |
| **Affected SKUs** | 0 (Ryder products deleted from Shopify) |

**The problem:** Three code blocks reference Ryder products that no longer exist:

1. `SKU[0].replace("_RYDER", "")` — strips Ryder from leather code
2. `const isRyder = LI.sku.includes("_RYDER-")` — checks for Ryder SKU pattern
3. `if (isRyder) { Properties.Lettering = ... }` — Ryder-only lettering logic

These never execute (0 Ryder SKUs exist) but clutter the code and confuse future maintainers.

**Fix:** All Ryder references are removed in the V4 belt rewrite. `LeatherCode` is now `SKU[0]` without the `.replace()` call.

---

### BUG-04: Ryder Compound Thread/Logo Value — REMOVED

| Detail | Value |
| --- | --- |
| **Severity** | N/A — Bug no longer exists |
| **Reason** | Ryder products deleted from Shopify in March 2026 |

This bug was documented in V3 (Ryder pant SKUs had compound values like `BLACK_THREAD_GRAY_LOGO` in SKU[3]). Since all Ryder products have been removed from Shopify, this bug is no longer applicable and requires no fix.

---

## Resolved Shopify Data Bugs

These product data bugs were fixed by Caleb directly in the Shopify admin (March 2026). They are documented here for the historical record.

| # | Bug | What Was Fixed | Date |
| --- | --- | --- | --- |
| DATA-01 | 5 online belt variants had wrong widths in SKU | Fixed SKUs to show correct 1" or 1.25" width | March 2026 |
| DATA-02 | 3 POS belt variants had "ORNAGE" typo | Fixed to "ORANGE" in CB10 Brass Mono variants | March 2026 |
| DATA-03 | 1 POS pant variant had "CCC04" triple-C | Fixed to "CC04" in Buckskin Brown Cashiers Skinny/Red | March 2026 |
| DATA-04 | $200 video gift card SKU showed $100 | Fixed `VID-GIFT-PERSONAL-100` → `VID-GIFT-PERSONAL-200` | March 2026 |

---

## SKU Data Anomalies (Not Yet Fixed)

These were found during the V4 SKU audit. They are Shopify catalog issues, not code bugs. They should be corrected in Shopify when convenient.

| # | Anomaly | SKU | Should Be | Count |
| --- | --- | --- | --- | --- |
| ANOM-01 | `SKINY` typo | `W-JH26_BLACK-SKINY` | `W-JH26_BLACK-SKINNY` | 1 |
| ANOM-02 | `MIDTTAN` typo | `W-JH04-STRAIGHT-MIDTTAN`, `W-JH04-BOOT-MIDTTAN` | `MIDTAN` | 2 |
| ANOM-03 | `DARK` truncated | `W-RW26-SKINNY-DARK` | `DARKGRAY` (title says "Dark Gray") | 1 |
| ANOM-04 | Tonal duplicate SKUs | Various POS SKUs | "Tonal" and named-color variants share identical SKUs | ~151 pairs |
| ANOM-05 | JH25 vs JH26 both "Black" | `JH25_BLACK`, `JH26_BLACK` | May be intentional (different batches?) | 8 |

---

## Deployment Checklist

Follow these steps to deploy the V4 fixes to the live Zapier automation:

### Step 1: Deploy step-5-v4.js

1. Open the Zapier zap editor (Zap ID: `281794942`)
2. Navigate to Step 5 ("Run Javascript: DO NOT DELETE!!!")
3. Copy the entire contents of `step-5-v4.js` from the project files
4. Replace the existing code block completely
5. Save the step

### Step 2: Deploy step-13-v4.js

1. Navigate to Step 13 ("Run Javascript: DO NOT")
2. Copy the entire contents of `step-13-v4.js` from the project files
3. Replace the existing code block completely
4. Save the step

### Step 3: Test with Real Orders

Run the following test orders through the live zap and verify the Asana task fields match expected values:

| Test | SKU to Use | What to Verify |
| --- | --- | --- |
| Online pant | `W-RW29_DARKINDIGO-FLARE` | Due date = 14 days. Thread from line item property. |
| POS pant | `M-CC01_DARKBLUE-STRAIGHT-NAVY` | Due date = 30 days. Thread = "NAVY" from SKU. |
| Online belt (3-seg) | `CB6-1.5-BRASS` | Thread from line item property. Monogram from property or "NONE". |
| POS belt + thread (4-seg) | `CB6-1-BRASS-BLACK` | Thread = "BLACK". Monogram = "NONE". |
| POS belt NO_THREAD (4-seg) | `CB6-1-BRASS-NO_THREAD` | Thread = "" (empty, NOT "NO_THREAD"). Monogram = "NONE". |
| POS belt MONO + thread (5-seg) | `CB6-1-BRASS-MONO-BLACK` | Thread = "BLACK". Monogram = "YES". |
| POS belt MONO + NO_THREAD (5-seg) | `CB6-1-BRASS-MONO-NO_THREAD` | Thread = "" (empty). Monogram = "YES". |
| POS belt FLATBLACK (4-seg) | `CB6-1-FLATBLACK-BLACK` | Hardware = "FLAT_BLACK" (normalized). |
| Shoe (new format) | `SHOE-CLASSIC-M-10` | Gender = "M". Size = "10". Shoe = "CLASSIC". |
| Video gift card | `VID-GIFT-BDJ-500` | ProductType = "Video Card". Denomination = "500". |
| Quantity > 1 order | Any SKU with quantity: 2 | **2 Asana tasks created** (not 1). Task names show "1/2" and "2/2". |

### Step 4: Publish

Once all tests pass, publish the zap. This creates **Zap version v43**.

### Rollback Plan

If something goes wrong after deployment:

1. Open the Zapier zap editor
2. Click the version history / undo button
3. Revert to v42 (the pre-V4 version)
4. Republish

The original `step-5.js` and `step-13.js` files are preserved in the project files and in the Zapier Pipeline Notion page under Resources.

---

## Fix Coverage by SKU Category

This matrix shows which fixes affect which product categories:

| Fix | Pants (3,184) | Shorts (570) | Derby (48) | Belts (1,701) | Shoes (76) | Video Cards (8) |
| --- | --- | --- | --- | --- | --- | --- |
| BUG-01 (qty expansion) | Affected | Affected | Affected | Affected | Affected | Affected |
| BUG-02 (belt monogram) | — | — | — | **819 SKUs** | — | — |
| BUG-03 (belt thread) | — | — | — | **1,638 SKUs** | — | — |
| NEW-01 (shoe format) | — | — | — | — | **76 SKUs** | — |
| NEW-04 (belt Source) | — | — | — | **1,638 SKUs** | — | — |
| BUG-05 (customer count) | Affected | Affected | Affected | Affected | Affected | Affected |
| NEW-03 (FLATBLACK) | — | — | — | **546 SKUs** | — | — |
| BUG-06 (carriage return) | Affected | Affected | Affected | Affected | Affected | Affected |
| BUG-07 (tag case) | Affected | Affected | Affected | Affected | Affected | Affected |
| NEW-02 (Ryder removal) | — | — | — | 0 SKUs (dead code) | — | — |

---

*This bug tracker reflects the state of the V4 code audit completed April 4, 2026. Once the V4 fixes are deployed and verified, update the Status column of each bug to "Deployed" and record the deployment date. The fixed code files (step-5-v4.js, step-13-v4.js) and the full CODEFIXES.md implementation reference are available in the project files.*