# Online vs POS Product Architecture

> This page explains the single most important architectural decision in the Blue Delta Jeans pipeline: **why two sets of products exist in Shopify**, and how that decision affects every line of code in Step 13. If you're editing the Zapier code, debugging a production task, or building the replacement database — read this page first.
> 

---

## The Core Problem

<aside>

Blue Delta Jeans sells custom products with extensive personalization options. A single pair of men's pants has 8 fabric colors × 4 styles × 19 thread colors = **608 possible combinations**. Shopify limits every product to **100 variants maximum**. That's a hard platform limit — there's no way around it.

</aside>

The solution for the website: use **SC Product Options** (a Shopify app) to move thread color and monogram out of the variant matrix and into line item properties. This means the Shopify product only needs fabric color × style = 32 variants (well under 100), and the thread color is captured as a custom field at checkout.

The problem: **SC Product Options does not work in the Shopify POS app.** POS orders need every customization option encoded in the variant itself, because the POS app can only select from existing variants — it can't prompt for custom fields.

The workaround: create a **separate set of POS-specific products** where each product is split by the first variation option (one product per fabric color), freeing up variant slots to add thread color as a 3rd option.

This is why the Step 13 code has different parsing logic for every product type depending on how many segments the SKU has — it's detecting whether the item came from the online store or POS based on the SKU structure.

---

## How SC Product Options Works

SC Product Options (formerly Bold Product Options) is a Shopify app that adds custom input fields to the product page on the online storefront. When a customer selects their thread color or enters monogram text, those values are attached to the line item as **properties** — key-value pairs that travel with the item through the Shopify API and into Zapier.

### What SC Product Options Produces

When a customer purchases an online pant with a monogram add-on, the Shopify API returns a `properties` array on the line item that looks like this (from the Alex Rivera live order #100002):

```json
"properties": [
  {"name": "Thread Color", "value": "Green"},
  {"name": "Add Watch Pocket Monogram", "value": "✓"},
  {"name": "Lettering", "value": "RBJ"},
  {"name": "Color", "value": "Match Thread"},
  {"name": "_boldVariantNames", "value": "Add Watch Pocket Monogram"},
  {"name": "_boldVariantIds", "value": "50222549664035"},
  {"name": "_boldProductIds", "value": "9641429893411"},
  {"name": "_boldVariantPrices", "value": "2500"},
  {"name": "_boldBuilderId", "value": "1912808081"},
  {"name": "_boldOptionLocalStorageId", "value": "8823112139043_-345214764"}
]
```

### Properties Breakdown

**Customer-facing properties** (used by the pipeline):

| Property | What It Is | Example Value | Where It Goes in Asana |
| --- | --- | --- | --- |
| `Thread Color` | The thread stitching color selected by the customer | `"Green"` | Pant Pipeline → Thread field |
| `Add Watch Pocket Monogram` | Checkbox flag — did the customer add a monogram? ($25 add-on) | `"✓"` or `""` | Not directly mapped — acts as a flag |
| `Lettering` | The monogram text (up to 4 characters) | `"RBJ"` | Pant Pipeline → Monogram field |
| `Color` | The monogram thread color | `"Match Thread"` | Pant Pipeline → Monogram Thread field |

**Women's pants also include:**

| Property | What It Is | Example Value | Where It Goes in Asana |
| --- | --- | --- | --- |
| `Waist Height` | Rise height preference | `"High"`, `"Mid"`, `"Low"` | Pant Pipeline → Waist Ride field |
| `Front Pockets` | Pocket style | `"Functional"`, `"Fake"` | Pant Pipeline → Pockets field |

**Internal SC Product Options metadata** (ignore these):

Properties prefixed with `_bold` are internal app metadata. They track which option was selected, which variant it corresponds to, pricing, and session IDs. Step 13 doesn't filter these out — they end up in the Properties object harmlessly but are never mapped to Asana fields.

### What Happens With POS Orders

POS orders have **no SC Product Options properties**. The `properties` array on a POS line item is either empty or contains only basic Shopify POS metadata. All customization data must come from the SKU itself.

---

## The 100-Variant Math

Here's why the online products can't just encode thread color as a variant option like POS does:

### Online Pant Product (Men's Raw Denim)

| Option | Values | Count |
| --- | --- | --- |
| Style | Straight, Boot, Skinny, Fashion Boot | 4 |
| Fabric Color | Dark Indigo, Smooth Indigo, Natural Indigo, Postman, Steel Gray, Cast Gray, Charcoal, Super Black | 8 |
| **Total variants** | 4 × 8 | **32** ✓ (under 100 limit) |

If thread color were added as a 3rd option: 4 × 8 × 19 = **608 variants** — impossible in Shopify.

SC Product Options moves thread color out of the variant matrix, keeping the product at 32 variants.

### POS Pant Product (Dark Blue Cashiers — one of 5 Cashiers products)

| Option | Values | Count |
| --- | --- | --- |
| Style | Straight, Boot, Skinny, Fashion Boot | 4 |
| Thread Color | 19 colors | 19 |
| **Total variants** | 4 × 19 | **76** ✓ (under 100 limit) |

By splitting the single Cashiers product into 5 separate products (one per fabric color: Dark Blue, Graphite, Forest, Buckskin Brown, White), each product only needs style × thread = 76 variants. The fabric color is determined by which product the POS staff selects, not by a variant option.

### Online Belt Product (Custom Leather Belt — single product)

| Option | Values | Count |
| --- | --- | --- |
| Leather | Black, Dark Brown, Mid Brown, Light Brown, Navy, Red, Football | 7 |
| Width | 1", 1.25", 1.5" | 3 |
| Hardware | Brass, Nickel, Flat Black | 3 |
| **Total variants** | 7 × 3 × 3 | **63** ✓ |

Thread color and monogram come from SC Product Options. Adding thread (13 colors) × monogram (yes/no) would be 7 × 3 × 3 × 13 × 2 = **1,638 variants** — completely impossible.

### POS Belt Products (21 products — one per leather × width)

| Option | Values | Count |
| --- | --- | --- |
| Hardware | Brass, Nickel, Flat Black | 3 |
| Monogram | Mono, NoMono | 2 |
| Thread Color | 13 colors + NO THREAD | 14 |
| **Total variants** | 3 × 2 × (13 + 1) | **78** ✓ (under 100, fits because it's 1 leather × 1 width per product) |

21 products = 7 leathers × 3 widths. Each product is scoped to a single leather and width, freeing all variant slots for hardware × monogram × thread.

---

## Product Count Summary

| Product Type | Online Products | Online Variants | POS Products | POS Variants per Product | Total SKUs |
| --- | --- | --- | --- | --- | --- |
| **Pants — Raw Denim** | 2 (Men's + Women's) | ~64 each | 4 (M+W × RD Men, M+W × RD Women) | 76 each | ~592 |
| **Pants — Cotton Chino** | 2 (Men's + Women's) | ~40 each | Multiple per fabric | 76 each | ~720+ |
| **Pants — Performance** | 2 (Men's + Women's) | ~40 each | *(same pattern)* | 76 each | ~720+ |
| **Pants — Cashiers** | 2 (Men's + Women's) | 20 each | 5 per gender (10 total) | 76 each | ~800 |
| **Shorts** | 10 (Performance fabrics) | ~57 each | N/A | N/A | 570 |
| **Derby Pants** | 1 (M+W combined) | 16 | N/A | N/A | 48* |
| **Belts** | 1 | 63 | 21 (7 leathers × 3 widths) | 78 each | 1,701 |
| **Shoes** | 0 | 0 | 2 (Classic + Custom) | 38 each | 76 |
| **Video Gift Cards** | 1 | 8 | N/A | N/A | 8 |
| **Total** |  |  |  |  | **5,555** |
- Derby pants are event-specific and used across both channels. Shorts include thread in the SKU like POS products but are sold online.

---

## Thread Color Flow

This is the most critical table for understanding Step 13. It shows exactly where thread color comes from for every product type and channel combination.

### Online Products (Thread from SC Product Options)

| Product | SKU Example | SKU Segments | Thread Source |
| --- | --- | --- | --- |
| **Men's Pants** | `M-RW29_DARKINDIGO-STRAIGHT` | 3 (no thread) | Line item property: `"Thread Color"` |
| **Women's Pants** | `W-RW29_DARKINDIGO-FLARE` | 3 (no thread) | Line item property: `"Thread Color"` |
| **Online Belts** | `CB6-1.5-BRASS` | 3 (no thread) | Line item property: `"Thread Color"` |

Step 13 doesn't need to extract thread from the SKU for these — it's already in `Properties["Thread Color"]` from the `LI.properties.reduce()` call.

### POS Products (Thread from SKU)

| Product | SKU Example | SKU Segments | Thread Source |
| --- | --- | --- | --- |
| **POS Men's Pants** | `M-RW29-STRAIGHT-BLACK` | 4 | `SKU[3]` = `"BLACK"` → `Properties.Thread` |
| **POS Women's Pants** | `W-JH01-BOOT-GREEN` | 4 | `SKU[3]` = `"GREEN"` → `Properties.Thread` |
| **POS Belt (no mono)** | `CB6-1-BRASS-BLACK` | 4 | `SKU[3]` = `"BLACK"` → `Properties["Thread Color"]` |
| **POS Belt (no mono, no thread)** | `CB6-1-BRASS-NO_THREAD` | 4 | `SKU[3]` = `"NO_THREAD"` → mapped to `""` (empty) |
| **POS Belt (mono + thread)** | `CB6-1-BRASS-MONO-BLACK` | 5 | `SKU[4]` = `"BLACK"` → `Properties["Thread Color"]` |
| **POS Belt (mono, no thread)** | `CB6-1-BRASS-MONO-NO_THREAD` | 5 | `SKU[4]` = `"NO_THREAD"` → mapped to `""` (empty) |

### Products With Thread Always in SKU (Both Channels)

| Product | SKU Example | SKU Segments | Thread Source |
| --- | --- | --- | --- |
| **Shorts** | `M-PF03_LIGHTTAN-SHORT_5-TONAL` | 4 | `SKU[3]` = `"TONAL"` → `Properties.Thread` |
| **Derby Pants** | `M-PFDERBY_HOTPINK-STRAIGHT-TONAL` | 4 | `SKU[3]` = `"TONAL"` → `Properties.Thread` |

Shorts and Derby pants always have thread in the SKU regardless of channel because they were built after the POS product architecture was established.

### How Step 13 Handles the Dual Source (Pants)

For pants, the Asana field mapping (Step 21) maps the Thread field from **two** Zapier tokens as a fallback chain:

```
Asana "Thread" field ← Properties.Thread (from SKU, if it exists)
                      ← Properties["Thread Color"] (from line item property, if it exists)
```

This means:

- **Online 3-segment pants:** `Properties.Thread` is `undefined` (SKU has no 4th segment). Falls back to `Properties["Thread Color"]` = `"Green"` (from SC Product Options). Correct.
- **POS 4-segment pants:** `Properties.Thread` = `"BLACK"` (from SKU[3]). First token wins. Correct.

### How Step 13 Handles the Dual Source (Belts)

For belts, the V4 code uses SKU **segment count** to determine the source, eliminating the fragile `Source === "pos"` check:

```
3 segments → Online belt → Thread from Properties["Thread Color"] (line item property)
4 segments → POS belt (no mono) → Thread from SKU[3]
5 segments → POS belt (with mono) → Thread from SKU[4]
```

If `SKU[3]` or `SKU[4]` is `"NO_THREAD"`, the code maps it to an empty string.

---

## Monogram Flow

Monogram handling differs significantly by product type and channel.

### Pant Monogram (Online Only — $25 Add-On)

Pant monograms are an optional add-on through SC Product Options on the online store. When a customer adds a monogram, three properties appear on the line item:

| Line Item Property | Meaning | Example | Asana Field |
| --- | --- | --- | --- |
| `Add Watch Pocket Monogram` | Checkbox — was monogram added? | `"✓"` | Not directly mapped |
| `Lettering` | The initials (up to 4 characters) | `"RBJ"` | Monogram |
| `Color` | The monogram thread color | `"Match Thread"` | Monogram Thread |

When no monogram is added, `Lettering` and `Color` are empty strings and the Asana fields remain blank.

POS pants do not have a monogram option — there's no mechanism to enter custom text through the POS variant selector. If a POS pant customer wants a monogram, it would need to be handled manually.

**Verified from live run (order #100002, Alex Rivera):** `Lettering = "RBJ"`, `Color = "Match Thread"` → Asana Monogram = "RBJ", Asana Monogram Thread = "Match Thread". Working correctly.

### Belt Monogram — Online

Online belt monograms come from SC Product Options as a line item property. The property name varies (the storefront may use different casing), so Step 13 uses a fallback chain:

```jsx
Properties.Monogram ?? Properties.monogram ?? Properties.Mono ?? Properties.mono ?? "NONE"
```

The Asana step (Step 23) maps Monogram from two tokens: `Properties.Monogram` then `Properties.Lettering` — same fallback pattern as pants.

### Belt Monogram — POS

POS belts encode monogram as a `MONO` segment in the SKU. The V4 code detects this by segment count:

| SKU Pattern | Segments | Monogram Value |
| --- | --- | --- |
| `CB6-1-BRASS-BLACK` | 4 | `"NONE"` (no MONO segment) |
| `CB6-1-BRASS-NO_THREAD` | 4 | `"NONE"` (no MONO segment) |
| `CB6-1-BRASS-MONO-BLACK` | 5 | `"YES"` (MONO is SKU[3]) |
| `CB6-1-BRASS-MONO-NO_THREAD` | 5 | `"YES"` (MONO is SKU[3]) |

**Important limitation:** POS belt monograms only flag that a monogram was requested (`"YES"`) — the actual monogram text (the customer's initials) is not in the SKU. Production staff must get the lettering from the order note, the customer record, or a separate communication. This is a known gap in the POS workflow.

<aside>

> **BUG-02 (CRITICAL, fixed in V4):** The current live code does not detect `MONO` in POS belt SKUs at all. All POS belt monograms silently fall through to `"NONE"`. The V4 fix uses segment-count detection to correctly identify 5-segment SKUs as monogram orders.
> 
</aside>

---

## Shoe Products — POS/Event Only

Shoes are a special case in the product architecture. Blue Delta customizes Nike shoes as a premium offering at events. This is exclusively a POS product — there are no shoe products on the website because the Nike partnership has not yet been formally approved for online sales and marketing.

### How Shoe Orders Work

1. A customer at an event fills out a **paper order form** specifying their customization choices (laces color, swoosh color, toe cap, etc.)
2. A Blue Delta staff member creates the order in Shopify POS, selecting the correct shoe variant (Classic or Custom, gender, size)
3. The staff member **manually enters the customization choices as line item properties** in the POS order
4. The Zapier pipeline picks up the order and creates an Asana task with the shoe customization fields

### Shoe SKU Format (Updated in V4)

The shoe SKU format was updated to encode gender and size directly:

**Old format:** `SHOE-CLASSIC` (2 segments) — gender and size parsed from `variant_title` (e.g., `"Men / 10"`)

**New format:** `SHOE-CLASSIC-M-10` (4 segments) — gender and size in the SKU itself

| Segment | Description | Values |
| --- | --- | --- |
| `SHOE` | Fixed prefix | — |
| Type | Shoe model | `CLASSIC`, `CUSTOM` |
| Gender | `M` or `W` | 2 values |
| Size | Shoe size (half sizes) | `5`, `5.5`, `6`, ... `14` (19 sizes) |

76 total shoe SKUs: 2 types × 2 genders × 19 sizes.

### Shoe Customization Fields (From Paper Order Forms)

These fields are manually entered into Shopify POS as line item properties and flow through to Asana via Step 13's `LI.properties.reduce()`:

| Property | Asana Field (Step 25) |
| --- | --- |
| `Laces` | Laces |
| `Swoosh` | Swoosh |
| `Back Tab Eyelets` | Back Tab | Eyelets |
| `Toe Cap Back Heel` | Toe Cap | Back Heel |
| `Toe Box Mid Panel` | Toe Box | Mid Panel |

---

## SKU Inconsistencies Between Channels

Two naming inconsistencies exist between Online and POS SKUs. These are **intentional** and will **not** be fixed in Shopify because updating thousands of SKUs would be error-prone. Instead, the Zapier code (Step 13) normalizes these values so Asana tasks display consistent data regardless of order source.

### Inconsistency #1: Belt Hardware Naming

| Hardware | Online SKU | POS SKU |
| --- | --- | --- |
| Antique Brass | `BRASS` | `BRASS` |
| Nickel | `NICKEL` | `NICKEL` |
| Flat Black | `FLAT_BLACK` (underscore) | `FLATBLACK` (no underscore) |

**Impact:** 21 online belt SKUs use `FLAT_BLACK`, 546 POS belt SKUs use `FLATBLACK`. Without normalization, the Asana task's Belt-Hardware field shows different values for the same hardware.

**V4 fix (in step-13-v4.js):**

```jsx
let hardware = SKU[2];
if (hardware === "FLATBLACK") {
  hardware = "FLAT_BLACK";
}
```

### Inconsistency #2: Cashiers CC04 Fabric Code

| Fabric Color | Online SKU | POS SKU |
| --- | --- | --- |
| Dark Blue | `CC01_DARKBLUE` | `CC01_DARKBLUE` |
| Graphite | `CC02_GRAPHITE` | `CC02_GRAPHITE` |
| Forest | `CC03_FOREST` | `CC03_FOREST` |
| Buckskin Brown | `CC04_BROWN` | `CC04_BUCKSKINBROWN` |
| White | `CC05_WHITE` | `CC05_WHITE` |

**Impact:** 8 online Cashiers SKUs use `CC04_BROWN`, POS Cashiers SKUs use `CC04_BUCKSKINBROWN`. The Asana Fabric field would show different values for the same fabric.

**Status:** The V4 code audit found that `CC04_BUCKSKINBROWN` does not appear in the current master SKU list (the POS Cashiers Buckskin Brown products may use a different code or may not be in the export). This inconsistency should be monitored — if both codes appear in production orders, normalization should be added.

---

## How Step 13 Detects Online vs. POS

The V4 code uses **SKU segment count** as the primary detection method, not the `Source` field from upstream. This is more reliable because the SKU format is deterministic:

### Pants

| Segments | Channel | Example | How Code Knows |
| --- | --- | --- | --- |
| 3 | Online | `M-RW29_DARKINDIGO-STRAIGHT` | `SKU[3]` is `undefined` → thread from properties |
| 4 | POS (or Shorts/Derby) | `M-RW29-STRAIGHT-BLACK` | `SKU[3]` exists → thread from SKU |

The pants code doesn't need explicit Online/POS detection — the `M-` branch always sets `Properties.Thread = SKU[3]`, which is `undefined` for 3-segment online SKUs and a thread color string for 4-segment POS SKUs. The Asana step's dual-token fallback handles the rest.

### Belts

| Segments | Channel | Example | Detection |
| --- | --- | --- | --- |
| 3 | Online | `CB6-1.5-BRASS` | `beltSegments === 3` → thread and monogram from properties |
| 4 | POS (no mono) | `CB6-1-BRASS-BLACK` | `beltSegments === 4` → thread from SKU[3], monogram = NONE |
| 5 | POS (with mono) | `CB6-1-BRASS-MONO-BLACK` | `beltSegments === 5` → thread from SKU[4], monogram = YES |

This replaced the old `Source === "pos"` check, which was fragile because it depended on the `inputData.Source` field being correctly passed from upstream. The segment-count approach works purely from the SKU data with no external dependencies.

---

*This page explains the product architecture as of April 2026. If SC Product Options is replaced, if Shopify raises the variant limit, or if the POS app gains custom field support, this architecture will need to be revisited. The long-term plan is to build a real database to replace the Asana pipeline, at which point the Online/POS product split can potentially be simplified.*