# SKU Reference Guide

This page is the complete reference for all SKU formats used across the Blue Delta Jeans Shopify store. As of the V4 audit (April 2026), there are **5,555 total SKUs** across 8 product categories. Every SKU listed here has been verified against the `master-sku-list.json` export.

Use this page when you need to understand what a specific SKU means, what each segment represents, or how a product category is structured. The Step 13 code parses these SKUs using `sku.split("-")` — understanding the segment breakdown is essential for debugging.

---

## Category Summary

| Category | Count | SKU Format | Segments | Pipeline | Thread Source |
| --- | --- | --- | --- | --- | --- |
| Online Pants (M-/W-) | 264 | `{G}-{Fabric_Name}-{Style}` | 3 | Pant | Line item property |
| POS Pants (M-/W-) | 2,920 | `{G}-{Fabric}-{Style}-{Thread}` | 4 | Pant | SKU segment |
| Shorts (M-) | 570 | `{G}-{Fabric_Name}-SHORT_{Len}-{Thread}` | 4 | Pant | SKU segment |
| Kentucky Derby (M-/W-) | 48 | `{G}-{DerbyFabric}-{Style}-{Thread}` | 4 | Pant | SKU segment |
| Online Belts (CB) | 63 | `{Leather}-{Width}-{Hardware}` | 3 | Belt | Line item property |
| POS Belts (CB) | 1,638 | `{Leather}-{W}-{HW}-[MONO-]{Thread}` | 4–5 | Belt | SKU segment |
| Shoes (SHOE-) | 76 | `SHOE-{Type}-{Gender}-{Size}` | 4 | Shoe | N/A |
| Video Gift Cards | 8 | `VID-GIFT-{Brand}-{Amount}` | 4 | Video Card | N/A |
| **Total** | **5,555** |  |  |  |  |

> **How to read SKU formats:** Curly braces `{X}` indicate variable values. Underscores within a segment are preserved by `split("-")` — they don't create additional segments. Square brackets `[X]` indicate optional segments.
> 

---

## Online Pants

**Count:** 264 SKUs (132 Men's + 132 Women's)

**Format:** `{Gender}-{FabricCode}_{FabricName}-{Style}`

**Segments:** 3 — Thread color and monogram come from SC Product Options (line item properties), NOT from the SKU.

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| Gender | SKU[0] | `M` (Men) or `W` (Women) | 2 |
| FabricCode_FabricName | SKU[1] | Fabric identifier — code + underscore + color name | 34 unique values |
| Style | SKU[2] | Cut/fit of the pant | Men: `STRAIGHT`, `BOOT`, `FASHBOOT`, `SKINNY`. Women: `STRAIGHT`, `BOOT`, `FLARE`, `SKINNY` |

### Fabric Codes (34 total, organized by fabric type)

**Raw Denim (RW) — 8 fabrics:**

| Code | Color Name |
| --- | --- |
| `RW26_NATURALINDIGO` | Natural Indigo |
| `RW28_CHARCOAL` | Charcoal |
| `RW29_DARKINDIGO` | Dark Indigo |
| `RW30_SMOOTHINDIGO` | Smooth Indigo |
| `RW32_STEELGRAY` | Steel Gray |
| `RW33_CASTGRAY` | Cast Gray |
| `RW34_SUPERBLACK` | Super Black |
| `RW38_POSTMAN` | Postman |

**Cotton Chino / Jones Heritage (CC/JH) — 16 fabrics:**

| Code | Color Name |
| --- | --- |
| `CC01_DARKBLUE` | Dark Blue (Cashiers) |
| `CC02_GRAPHITE` | Graphite (Cashiers) |
| `CC03_FOREST` | Forest (Cashiers) |
| `CC04_BROWN` | Buckskin Brown (Cashiers) — see SKU Inconsistencies |
| `CC05_WHITE` | White (Cashiers) |
| `JH01_BANANAOLIVE` | Banana Olive |
| `JH04_DARKTAN` | Dark Tan |
| `JH05_TAN` | Tan |
| `JH06_STONE` | Stone |
| `JH07_LIGHTGRAY` | Light Gray |
| `JH08_MIDGRAY` | Mid Gray |
| `JH09_DULLBLUE` | Dull Blue |
| `JH11_POWDERBLUE` | Powder Blue |
| `JH12_MAROON` | Maroon |
| `JH25_BLACK` | Black |
| `JH26_BLACK` | Black (alternate code — same color as JH25) |

**Performance (PF) — 10 fabrics:**

| Code | Color Name |
| --- | --- |
| `PF01_WHITE` | White |
| `PF02_LIGHTPINK` | Light Pink |
| `PF03_LIGHTTAN` | Light Tan |
| `PF04_LIGHTGRAY` | Light Gray |
| `PF05_LIGHTBLUE` | Light Blue |
| `PF10_BLACK` | Black |
| `PF11_TAN` | Tan |
| `PF28_DARKBLUE` | Dark Blue |
| `PF29_DARKGRAY` | Dark Gray |
| `PF56_MIDBLUE` | Mid Blue |

### Examples

| SKU | Decoded |
| --- | --- |
| `M-RW29_DARKINDIGO-STRAIGHT` | Men's, Raw Denim Dark Indigo, Straight |
| `W-PF03_LIGHTTAN-BOOT` | Women's, Performance Light Tan, Boot |
| `M-JH01_BANANAOLIVE-FASHBOOT` | Men's, Jones Heritage Banana Olive, Fashion Boot |
| `W-CC01_DARKBLUE-FLARE` | Women's, Cashiers Dark Blue, Flare |

---

## POS Pants

**Count:** 2,920 SKUs (1,460 Men's + 1,460 Women's)

This count includes standard POS pants (2,888) plus POS Derby pants (32) that use the same 4-segment structure.

**Format:** `{Gender}-{FabricCode}-{Style}-{ThreadColor}`

**Segments:** 4 — Thread color is the last segment. No monogram in POS pant SKUs.

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| Gender | SKU[0] | `M` or `W` | 2 |
| FabricCode | SKU[1] | Fabric code WITHOUT underscore color name | 23 unique values |
| Style | SKU[2] | Cut/fit | Men: `STRAIGHT`, `BOOT`, `FASHBOOT`, `SKINNY`. Women: `STRAIGHT`, `BOOT`, `FLARE`, `SKINNY` |
| ThreadColor | SKU[3] | Thread stitching color | 20 values (including 2 anomalies) |

### Key Difference from Online

POS pant fabric codes **do not include the underscore-separated color name**. Online uses `RW29_DARKINDIGO`, POS uses just `RW29`. This is how you can visually distinguish an Online vs POS pant SKU at a glance — but Step 13 doesn't need to make this distinction because the `M-`/`W-` branch handles both by reading `SKU[1]` as the fabric code regardless.

### POS Fabric Codes (23 total)

**Standard fabrics (19):** `RW26`, `RW28`, `RW29`, `RW30`, `RW32`, `RW33`, `RW34`, `RW38`, `JH01`, `JH04`, `JH05`, `JH06`, `JH07`, `JH08`, `JH09`, `JH11`, `JH12`, `JH16`, `JH25`

**Derby fabrics (4 — POS-specific derby products):** `RW29_DERBY`, `RW29_DERBY_OAKS`, `JH07_DERBY`, `JH07_DERBY_OAKS`

> **Note:** `JH16` and `JH25` appear in POS but not in Online. CC-prefixed fabrics (Cashiers Collection) appear in Online but not as separate POS fabric codes — Cashiers POS products use `CC01_DARKBLUE` etc. with the full underscore format as separate products.
> 

### Thread Colors (20 values)

`BABYBLUE`, `BLACK`, `BROWN`, `DARKGRAY`, `DARKPINK`, `DARKTAN`, `GRAY`, `GREEN`, `LIGHTPINK`, `LIGHTTAN`, `MAROON`, `MIDTAN`, `NAVY`, `OLDGOLD`, `ORANGE`, `PURPLE`, `RED`, `SNOWWHITE`

Plus 2 anomalies: `DARK` (truncated — should be DARKGRAY, 1 SKU), `MIDTTAN` (typo — should be MIDTAN, 2 SKUs). See Data Anomalies section.

### Examples

| SKU | Decoded |
| --- | --- |
| `M-RW29-STRAIGHT-BLACK` | Men's, Dark Indigo Denim, Straight, Black thread |
| `W-JH01-BOOT-GREEN` | Women's, Banana Olive Chino, Boot, Green thread |
| `M-JH05-SKINNY-TONAL` | Men's, Tan Chino, Skinny, Tonal thread |
| `M-RW29_DERBY-STRAIGHT-BLACK` | Men's, Kentucky Derby Denim, Straight, Black thread |
| `W-JH07_DERBY_OAKS-FLARE-GRAY` | Women's, Derby Oaks Light Gray Chino, Flare, Gray thread |

---

## Shorts

**Count:** 570 SKUs (Men's only — no women's shorts currently)

**Format:** `M-{FabricCode}_{FabricName}-SHORT_{Length}-{ThreadColor}`

**Segments:** 4 — The underscore in `SHORT_5` keeps it as one segment when split on `-`. Thread color is in the last segment, like POS pants. All shorts use Performance fabric.

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| Gender | SKU[0] | Always `M` | 1 |
| FabricCode_FabricName | SKU[1] | Performance fabric with underscore name | 10 fabrics |
| SHORT_{Length} | SKU[2] | Inseam length in inches | `SHORT_5`, `SHORT_7`, `SHORT_9` |
| ThreadColor | SKU[3] | Thread stitching color | 19 colors |

### Fabrics (10 — all Performance)

`PF01_WHITE`, `PF02_LIGHTPINK`, `PF03_LIGHTTAN`, `PF04_LIGHTGRAY`, `PF05_LIGHTBLUE`, `PF10_BLACK`, `PF11_TAN`, `PF28_DARKBLUE`, `PF29_DARKGRAY`, `PF56_MIDBLUE`

### Thread Colors (19)

`TONAL`, `BABYBLUE`, `BLACK`, `BROWN`, `DARKGRAY`, `DARKPINK`, `DARKTAN`, `GRAY`, `GREEN`, `LIGHTPINK`, `LIGHTTAN`, `MAROON`, `MIDTAN`, `NAVY`, `OLDGOLD`, `ORANGE`, `PURPLE`, `RED`, `SNOWWHITE`

> **Note:** Shorts include `TONAL` (matches fabric color) which POS pants do not have as a standard thread option. Shorts do NOT have the `DARK` or `MIDTTAN` anomalies found in POS pants.
> 

### Examples

| SKU | Decoded |
| --- | --- |
| `M-PF03_LIGHTTAN-SHORT_5-TONAL` | Men's, Light Tan Performance, 5" inseam, Tonal thread |
| `M-PF10_BLACK-SHORT_7-NAVY` | Men's, Black Performance, 7" inseam, Navy thread |
| `M-PF01_WHITE-SHORT_9-RED` | Men's, White Performance, 9" inseam, Red thread |

### How Shorts Work in Step 13

Shorts trigger the `M-` branch (they start with `M-`). The code sets `Style = SKU[2]` which becomes `"SHORT_5"`, `"SHORT_7"`, or `"SHORT_9"`. The underscore keeps the style and length together as one value. This flows to the Asana "Style" field, and production staff know that `SHORT_5` means a 5-inch inseam short. No special handling is needed.

---

## Kentucky Derby Pants

**Count:** 48 SKUs (24 Men's + 24 Women's)

Derby pants are event-specific products for the Kentucky Derby. There are three fabric variants across two events (Kentucky Derby and Derby Oaks):

**Performance Hot Pink (PFDERBY) — 16 SKUs:**
Format: `{G}-PFDERBY_{Color}-{Style}-TONAL`

**Raw Denim Derby — 16 SKUs:**
Format: `{G}-RW29_DERBY-{Style}-BLACK` and `{G}-RW29_DERBY_OAKS-{Style}-BLACK`

**Light Gray Chino Derby — 16 SKUs:**
Format: `{G}-JH07_DERBY-{Style}-GRAY` and `{G}-JH07_DERBY_OAKS-{Style}-GRAY`

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| Gender | SKU[0] | `M` or `W` | 2 |
| DerbyFabric | SKU[1] | Derby-specific fabric code | 6 values (see below) |
| Style | SKU[2] | Cut/fit | Men: `STRAIGHT`, `BOOT`, `FASHBOOT`, `SKINNY`. Women: `STRAIGHT`, `BOOT`, `FLARE`, `SKINNY` |
| ThreadColor | SKU[3] | Thread stitching | `TONAL`, `BLACK`, or `GRAY` |

### Derby Fabric Codes

| Code | Fabric | Event |
| --- | --- | --- |
| `PFDERBY_HOTPINK` | Performance Hot Pink | Kentucky Derby |
| `PFDERBY_OAKS_HOTPINK` | Performance Hot Pink | Derby Oaks |
| `RW29_DERBY` | Raw Denim Dark Indigo | Kentucky Derby |
| `RW29_DERBY_OAKS` | Raw Denim Dark Indigo | Derby Oaks |
| `JH07_DERBY` | Cotton Chino Light Gray | Kentucky Derby |
| `JH07_DERBY_OAKS` | Cotton Chino Light Gray | Derby Oaks |

### Examples

| SKU | Decoded |
| --- | --- |
| `M-PFDERBY_HOTPINK-STRAIGHT-TONAL` | Men's, Hot Pink Performance, Straight, Tonal |
| `W-PFDERBY_OAKS_HOTPINK-FLARE-TONAL` | Women's, Derby Oaks Hot Pink, Flare, Tonal |
| `M-RW29_DERBY-BOOT-BLACK` | Men's, Derby Dark Indigo Denim, Boot, Black |
| `W-JH07_DERBY_OAKS-SKINNY-GRAY` | Women's, Derby Oaks Light Gray Chino, Skinny, Gray |

### How Derby Pants Work in Step 13

All Derby SKUs trigger the `M-` or `W-` branch because they start with those prefixes. The underscore in `PFDERBY_HOTPINK`, `RW29_DERBY`, etc. keeps the fabric code as one segment. Thread is in SKU[3] as expected. No special handling needed — they're parsed identically to POS pants.

---

## Online Belts

**Count:** 63 SKUs (1 Shopify product with 63 variants)

**Format:** `{LeatherCode}-{Width}-{Hardware}`

**Segments:** 3 — Thread color and monogram come from SC Product Options (line item properties).

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| LeatherCode | SKU[0] | Leather type | `CB1`, `CB2`, `CB4`, `CB5`, `CB6`, `CB9`, `CB10` (7 types) |
| Width | SKU[1] | Belt width in inches | `1`, `1.25`, `1.5` |
| Hardware | SKU[2] | Buckle finish | `BRASS`, `NICKEL`, `FLAT_BLACK` |

### Leather Legend

| Code | Leather Name | Color |
| --- | --- | --- |
| CB1 | Dark Brown Leather | Dark Brown |
| CB2 | Mid Brown Leather | Mid Brown |
| CB4 | Navy Leather | Navy |
| CB5 | Football Leather | Football (textured) |
| CB6 | Black Leather | Black |
| CB9 | Red Leather | Red |
| CB10 | Light Brown Leather | Light Brown |

> **Inactive codes in the lookup table but not in current products:** CB3 (Light Brown — duplicate of CB10), CB7 (Natural Derby Leather), CB8 (English Tan Leather), CB11 (Gray). These are in the Step 13 `LeatherNames` object but have 0 SKUs in the current catalog.
> 

### Examples

| SKU | Decoded |
| --- | --- |
| `CB6-1.5-BRASS` | Black Leather, 1.5" width, Antique Brass |
| `CB1-1.25-FLAT_BLACK` | Dark Brown Leather, 1.25" width, Flat Black |
| `CB9-1-NICKEL` | Red Leather, 1" width, Nickel |

---

## POS Belts

**Count:** 1,638 SKUs (21 Shopify products: 7 leathers × 3 widths, each with 78 variants)

**Format:** 4 or 5 segments depending on monogram and thread selections.

### Pattern Breakdown

| Pattern | Format | Segments | Count | Meaning |
| --- | --- | --- | --- | --- |
| Thread, no mono | `{Leather}-{W}-{HW}-{Thread}` | 4 | 756 | Customer selected a thread color, no monogram |
| No thread, no mono | `{Leather}-{W}-{HW}-NO_THREAD` | 4 | 63 | Customer selected no stitching, no monogram |
| Thread + mono | `{Leather}-{W}-{HW}-MONO-{Thread}` | 5 | 756 | Has monogram stamped + thread stitching |
| No thread + mono | `{Leather}-{W}-{HW}-MONO-NO_THREAD` | 5 | 63 | Has monogram stamped, no stitching |

**Total:** 756 + 63 + 756 + 63 = **1,638**

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| LeatherCode | SKU[0] | Same 7 codes as online belts | `CB1`, `CB2`, `CB4`, `CB5`, `CB6`, `CB9`, `CB10` |
| Width | SKU[1] | Belt width in inches | `1`, `1.25`, `1.5` |
| Hardware | SKU[2] | Buckle finish | `BRASS`, `NICKEL`, `FLATBLACK` (note: no underscore!) |
| MONO | SKU[3] (5-seg only) | Monogram flag | `MONO` (literal string) |
| Thread/NO_THREAD | SKU[3] or SKU[4] | Thread color or no-thread marker | 13 values (see below) |

### POS Belt Thread Colors (12 colors + NO_THREAD)

`BLACK`, `DARKBROWN`, `GRAY`, `GREEN`, `LIGHTBLUE`, `LIGHTBROWN`, `NAVY`, `NO_THREAD`, `ORANGE`, `PINK`, `PURPLE`, `RED`, `WHITE`

> **Important:** `NO_THREAD` is a literal SKU segment meaning "no stitching." The V4 code maps this to an empty string so the Asana Belt-Stitching field shows blank instead of the literal text "NO_THREAD."
> 

### Hardware Naming

POS belts use `FLATBLACK` (no underscore). Online belts use `FLAT_BLACK` (with underscore). The V4 code normalizes `FLATBLACK` → `FLAT_BLACK` so Asana shows consistent values. See SKU Inconsistencies below.

### How Step 13 Detects POS Belt Patterns (V4)

The V4 code uses segment count to determine the pattern:

```
3 segments → Online belt → thread and monogram from line item properties
4 segments → POS belt, no monogram → thread from SKU[3] (or "" if NO_THREAD)
5 segments → POS belt, with monogram → thread from SKU[4] (or "" if NO_THREAD), monogram = YES
```

### Examples

| SKU | Pattern | Decoded |
| --- | --- | --- |
| `CB6-1-BRASS-BLACK` | 4-seg, thread | Black, 1", Brass, Black thread, no monogram |
| `CB6-1-BRASS-NO_THREAD` | 4-seg, no thread | Black, 1", Brass, no stitching, no monogram |
| `CB6-1-BRASS-MONO-BLACK` | 5-seg, mono+thread | Black, 1", Brass, monogram YES, Black thread |
| `CB6-1-BRASS-MONO-NO_THREAD` | 5-seg, mono only | Black, 1", Brass, monogram YES, no stitching |
| `CB10-1.5-FLATBLACK-NAVY` | 4-seg, thread | Light Brown, 1.5", Flat Black (normalized), Navy thread |

---

## Shoes

**Count:** 76 SKUs (38 Classic + 38 Custom)

**Format:** `SHOE-{Type}-{Gender}-{Size}`

**Segments:** 4 — Gender and size are encoded in the SKU (updated in V4 from the old format where they were parsed from `variant_title`).

**Channel:** POS/event only — no shoe products on the website. Nike partnership pending.

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| SHOE | SKU[0] | Fixed prefix | Always `SHOE` |
| Type | SKU[1] | Shoe model | `CLASSIC`, `CUSTOM` |
| Gender | SKU[2] | `M` (Men) or `W` (Women) | 2 |
| Size | SKU[3] | Shoe size (half sizes supported) | 19 sizes: `5` through `14` |

### Size Range

Both men's and women's shoes cover sizes 5 to 14 in half-size increments (5, 5.5, 6, 6.5, ... 13.5, 14) = 19 sizes per gender per type.

### Customization

Shoe customization (Laces, Swoosh, Back Tab/Eyelets, Toe Cap/Back Heel, Toe Box/Mid Panel) is entered manually by staff from paper order forms as line item properties. These are NOT in the SKU.

### Examples

| SKU | Decoded |
| --- | --- |
| `SHOE-CLASSIC-M-10` | Classic shoe, Men's, Size 10 |
| `SHOE-CUSTOM-W-7.5` | Custom shoe, Women's, Size 7.5 |
| `SHOE-CLASSIC-M-5.5` | Classic shoe, Men's, Size 5.5 |

---

## Video Gift Cards

**Count:** 8 SKUs (1 Shopify product with 8 variants)

**Format:** `VID-GIFT-{Brand}-{Amount}`

**Segments:** 4 (note: `split("-")` produces `["VID", "GIFT", "BDJ", "500"]` — the useful data starts at position 2)

| Segment | Position | Description | Values |
| --- | --- | --- | --- |
| VID | SKU[0] | Fixed prefix part 1 | Always `VID` |
| GIFT | SKU[1] | Fixed prefix part 2 | Always `GIFT` |
| Brand | SKU[2] | Video content type | `BDJ` (Blue Delta Story), `PERSONAL` (Upload My Own) |
| Amount | SKU[3] | Dollar denomination | `200`, `300`, `450`, `500` |

### All 8 SKUs

| SKU | Amount | Video Type |
| --- | --- | --- |
| `VID-GIFT-BDJ-200` | $200 | The Blue Delta Story |
| `VID-GIFT-PERSONAL-200` | $200 | Upload My Own |
| `VID-GIFT-BDJ-300` | $300 | The Blue Delta Story |
| `VID-GIFT-PERSONAL-300` | $300 | Upload My Own |
| `VID-GIFT-BDJ-450` | $450 | The Blue Delta Story |
| `VID-GIFT-PERSONAL-450` | $450 | Upload My Own |
| `VID-GIFT-BDJ-500` | $500 | The Blue Delta Story |
| `VID-GIFT-PERSONAL-500` | $500 | Upload My Own |

---

## SKU Inconsistencies Between Channels

These are known naming differences between Online and POS SKUs. They will **not** be fixed in Shopify — the V4 Zapier code normalizes them instead.

### Inconsistency #1: Belt Hardware

| Hardware | Online SKU Value | POS SKU Value | Normalization |
| --- | --- | --- | --- |
| Antique Brass | `BRASS` | `BRASS` | None needed |
| Nickel | `NICKEL` | `NICKEL` | None needed |
| Flat Black | `FLAT_BLACK` | `FLATBLACK` | V4 code: `FLATBLACK` → `FLAT_BLACK` |

**Affected SKUs:** 21 online (use `FLAT_BLACK`) + 546 POS (use `FLATBLACK`). After V4 normalization, all Asana tasks will show `FLAT_BLACK`.

### Inconsistency #2: Cashiers CC04 Fabric Code

| Color | Online Code | POS Code | Status |
| --- | --- | --- | --- |
| Buckskin Brown | `CC04_BROWN` | Not found in master list | Monitor — may not need normalization |

The V3 documentation noted that POS Cashiers used `CC04_BUCKSKINBROWN`, but the V4 master SKU list audit found **0 instances** of `CC04_BUCKSKINBROWN`. The POS Cashiers products may use a different code or may not have been exported. This should be monitored in production — if both codes appear in real orders, add normalization to Step 13.

---

## Data Anomalies Found in V4 Audit

These are errors in the current Shopify product data that were identified during the V4 SKU audit. They have NOT been fixed yet.

| # | Anomaly | SKU | Expected | Severity | Count |
| --- | --- | --- | --- | --- | --- |
| 1 | `SKINY` typo | `W-JH26_BLACK-SKINY` | Should be `SKINNY` | Medium | 1 SKU |
| 2 | `MIDTTAN` typo | `W-JH04-STRAIGHT-MIDTTAN`, `W-JH04-BOOT-MIDTTAN` | Should be `MIDTAN` | Medium | 2 SKUs |
| 3 | `DARK` truncated | `W-RW26-SKINNY-DARK` | Should be `DARKGRAY` (title says "Dark Gray") | Medium | 1 SKU |
| 4 | Tonal duplicates | Various POS SKUs | "Tonal (matches fabric)" and named-color variants produce identical SKUs | Low | ~151 pairs |
| 5 | JH25 vs JH26 | `JH25_BLACK` and `JH26_BLACK` | Both map to "Black" — may be intentional (different material batches?) | Info | 8 SKUs |

Anomalies #1–3 should be corrected in Shopify when convenient. Anomaly #4 is a Shopify variant display issue that doesn't affect the pipeline (identical SKUs produce identical Asana tasks). Anomaly #5 may be intentional.

---

*All data on this page was verified against `master-sku-list.json` (5,555 entries) exported April 2026. Individual counts were cross-referenced against the per-category spreadsheet exports in the project files. If products are added or removed from Shopify, this page should be updated to match.*