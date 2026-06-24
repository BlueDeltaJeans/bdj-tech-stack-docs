# Step 13 — Line Item Processor

Step 13 is the most complex step in the pipeline. It runs inside the loop (once per line item) and contains all the product-specific parsing logic. Its job: take a raw line item JSON string, determine what product type it is based on the SKU, and extract every field that the Asana task needs — fabric, style, thread color, monogram, hardware, sizing, due date, and more.

The complexity comes from the fact that the same type of data (e.g., thread color) lives in different places depending on the product type and sales channel. Online pants carry thread as a line item property. POS pants encode it in the SKU. Online belts carry both thread and monogram as properties. POS belts encode monogram as a `MONO` SKU segment and thread as the last segment. Step 13 has to handle all of these paths correctly.

| Detail | Value |
| --- | --- |
| **Pipeline Position** | Step 13 of 27 (inside loop, after 15-sec delay, before gender lookup) |
| **Type** | JavaScript Code |
| **App** | CodeCLIAPI@1.0.1 |
| **Zap Step ID** | 281794951 |
| **Custom Title** | "Run Javascript: DO NOT" |
| **Runtime** | ~85–86 MB memory, ~195–204ms duration |
| **Bugs in this step** | 7 (BUG-02 CRITICAL, BUG-03 HIGH, BUG-07 LOW, NEW-01 HIGH, NEW-02 LOW, NEW-03 MEDIUM, NEW-04 HIGH) |

---

## Inputs

Step 13 receives 5 input variables from upstream steps:

| Input | Source | Example (Online) | Example (POS) |
| --- | --- | --- | --- |
| `LI` | Step 10 loop (current item) | `'{"sku":"W-RW26_NATURALINDIGO-FLARE","quantity":1,"properties":[{"name":"Thread Color","value":"Light Pink"}],...}'` | `'{"sku":"M-RW29-STRAIGHT-BLACK","quantity":1,"properties":[],...}'` |
| `DateCreated` | Step 7 (formatted date) | `"2026-03-31T10:42:08-05:00"` | Same format |
| `CartToken` | Step 5 output | `":censored:24:37b9e3afb5:"` (truthy) | `""` (falsy) |
| `Tags` | Step 5 output | `"Andy, Denim"` | `"Denim"` |
| `Source` | Step 4 API response | `"web"` | `"pos"` |

The `LI` input is a **stringified JSON** string — Step 5 ran `JSON.stringify()` on each line item, and now Step 13 must `JSON.parse()` it back into an object. This round-trip is a Zapier limitation (the Loop step requires string arrays).

---

## Shared Logic — Runs for All Products

Before the product-specific branches, four things happen for every line item regardless of type:

### JSON Parsing (Lines 7–11)

```jsx
let LI;
try {
  LI = JSON.parse(inputData.LI);
} catch (error) {
  throw new Error("Invalid JSON input for LI");
}
```

Parses the stringified line item back into a JavaScript object. If the JSON is malformed (shouldn't happen unless Step 5 has a bug), the step throws an error.

### SKU Splitting (Line 15)

```jsx
let SKU = LI.sku.split("-");
```

This is the foundation of all product-type detection. Splitting on `-` (hyphen) produces a segment array that the downstream branches use to extract fields. Crucially, **underscores are NOT split** — this is why fabric codes like `RW26_NATURALINDIGO` and style values like `SHORT_5` stay as single segments.

**Examples:**

| SKU | `split("-")` Result | Segment Count |
| --- | --- | --- |
| `W-RW26_NATURALINDIGO-FLARE` | `["W", "RW26_NATURALINDIGO", "FLARE"]` | 3 |
| `M-RW29-STRAIGHT-BLACK` | `["M", "RW29", "STRAIGHT", "BLACK"]` | 4 |
| `M-PF03_LIGHTTAN-SHORT_5-TONAL` | `["M", "PF03_LIGHTTAN", "SHORT_5", "TONAL"]` | 4 |
| `M-PFDERBY_HOTPINK-STRAIGHT-TONAL` | `["M", "PFDERBY_HOTPINK", "STRAIGHT", "TONAL"]` | 4 |
| `CB6-1.5-BRASS` | `["CB6", "1.5", "BRASS"]` | 3 |
| `CB6-1-BRASS-MONO-BLACK` | `["CB6", "1", "BRASS", "MONO", "BLACK"]` | 5 |
| `SHOE-CLASSIC-M-10` | `["SHOE", "CLASSIC", "M", "10"]` | 4 |
| `VID-GIFT-BDJ-500` | `["VID", "GIFT", "BDJ", "500"]` | 4 |

### Properties Initialization (Lines 18–21)

```jsx
let Properties = LI.properties.reduce((acc, prop) => {
  acc[prop.name] = prop.value;
  return acc;
}, {});
```

Converts the line item's `properties` array (from SC Product Options or manual POS entry) into a flat key-value object. For **online orders**, this pre-populates Properties with customer-facing customization data like `"Thread Color"`, `"Lettering"`, `"Waist Height"`, etc. For **POS orders**, this array is typically empty or contains only basic metadata.

This is important: by the time the product-specific branches run, Properties may already contain thread color and monogram data from the storefront. The branches then add SKU-derived fields on top with `Object.assign()`, which merges without overwriting existing keys (unless the same key is explicitly set).

**Online example (from order #100002):**

```json
{
  "Thread Color": "Green",
  "Add Watch Pocket Monogram": "✓",
  "Lettering": "RBJ",
  "Color": "Match Thread",
  "_boldVariantNames": "Add Watch Pocket Monogram",
  "_boldVariantIds": "50222549664035",
  "_boldProductIds": "9641429893411",
  "_boldVariantPrices": "2500",
  "_boldBuilderId": "1912808081",
  "_boldOptionLocalStorageId": "8823112139043_-345214764"
}
```

**POS example:** `{}` (empty object — no SC Product Options properties)

### Due Date Helper (Lines 24–28)

```jsx
const setDueDate = (days) => {
  let dueDate = new Date(DateCreated);
  dueDate.setDate(dueDate.getDate() + days);
  return dueDate.toISOString().split("T")[0];
};
```

Takes a number of days and calculates the due date from the order creation date. Returns an ISO date string (e.g., `"2026-04-14"`). Each product branch calls this with its specific offset (14/30 for pants, 30 for belts, 45 for shoes, 5 for video cards).

Note: `DateCreated` is split from the ISO timestamp on line 2: `inputData.DateCreated.split("T")[0]` — but the `setDueDate` function creates `new Date(DateCreated)` from this, which may interpret the date as UTC midnight. In practice this hasn't caused off-by-one errors because the `.split("T")[0]` on the output truncates back to just the date.

---

## Women's Pants — `W-` Branch

**Current code (lines 31–41):**

```jsx
if (LI.sku.startsWith("W-")) {
  Object.assign(Properties, {
    ProductType: "Pant",
    Gender: "F",
    Fabric: SKU[1],
    Style: SKU[2],
    Thread: SKU[3],
    DueDays: CartToken ? 14 : 30,
    DueDate: setDueDate(CartToken ? 14 : 30),
  });
}
```

**Triggers on:** Any SKU starting with `W-` — women's pants (all fabrics), women's shorts, women's derby pants.

**SKU patterns handled:**

| Type | Example SKU | SKU[1] (Fabric) | SKU[2] (Style) | SKU[3] (Thread) |
| --- | --- | --- | --- | --- |
| Online pant (3-seg) | `W-RW26_NATURALINDIGO-FLARE` | `RW26_NATURALINDIGO` | `FLARE` | `undefined` |
| POS pant (4-seg) | `W-JH01-BOOT-GREEN` | `JH01` | `BOOT` | `GREEN` |
| Derby pant (4-seg) | `W-PFDERBY_HOTPINK-FLARE-TONAL` | `PFDERBY_HOTPINK` | `FLARE` | `TONAL` |

**Key behaviors:**

- `Gender` is hardcoded to `"F"` (not read from SKU)
- `Thread` is `SKU[3]` — this is `undefined` for online 3-segment SKUs and a color string for POS/derby 4-segment SKUs. The Asana step (Step 21) handles this with a dual-token fallback: it tries `Properties.Thread` first, then `Properties["Thread Color"]` (from the SC Product Options line item property). So online orders get thread from the property, POS orders get it from the SKU.
- Due date: 14 days for online (has CartToken), 30 days for POS/direct (no CartToken)

**No bugs in this branch.** The V4 code is identical to the current code.

**There are no women's shorts currently** — all 570 short SKUs use `M-` prefix (men's only). If women's shorts are added in the future, they would use `W-` and this branch would handle them correctly.

---

## Men's Pants — `M-` Branch

**Current code (lines 44–54):**

```jsx
if (LI.sku.startsWith("M-")) {
  Object.assign(Properties, {
    ProductType: "Pant",
    Gender: SKU[0],
    Fabric: SKU[1],
    Style: SKU[2],
    Thread: SKU[3],
    DueDays: CartToken ? 14 : 30,
    DueDate: setDueDate(CartToken ? 14 : 30),
  });
}
```

**Triggers on:** Any SKU starting with `M-` — men's pants (all fabrics), men's shorts, men's derby pants.

**SKU patterns handled:**

| Type | Example SKU | SKU[0] | SKU[1] (Fabric) | SKU[2] (Style) | SKU[3] (Thread) |
| --- | --- | --- | --- | --- | --- |
| Online pant (3-seg) | `M-RW29_DARKINDIGO-STRAIGHT` | `M` | `RW29_DARKINDIGO` | `STRAIGHT` | `undefined` |
| POS pant (4-seg) | `M-RW29-STRAIGHT-BLACK` | `M` | `RW29` | `STRAIGHT` | `BLACK` |
| Shorts (4-seg) | `M-PF03_LIGHTTAN-SHORT_5-TONAL` | `M` | `PF03_LIGHTTAN` | `SHORT_5` | `TONAL` |
| Derby (4-seg) | `M-PFDERBY_HOTPINK-STRAIGHT-TONAL` | `M` | `PFDERBY_HOTPINK` | `STRAIGHT` | `TONAL` |
| Derby Oaks (4-seg) | `M-PFDERBY_OAKS_HOTPINK-BOOT-TONAL` | `M` | `PFDERBY_OAKS_HOTPINK` | `BOOT` | `TONAL` |

**Key differences from Women's branch:**

- `Gender` is `SKU[0]` (which is `"M"`) instead of hardcoded. This is a subtle inconsistency — Women's hardcodes `"F"` but Men's reads from the SKU. Both work correctly because `SKU[0]` is always `"M"` for this branch, but the approaches are different.

**Shorts work without special handling** because `split("-")` on `M-PF03_LIGHTTAN-SHORT_5-TONAL` produces `["M", "PF03_LIGHTTAN", "SHORT_5", "TONAL"]` — the underscore in `SHORT_5` is preserved. So `Style = "SHORT_5"` and `Thread = "TONAL"`. The Asana task shows Style = "SHORT_5" which production staff understand means a 5-inch inseam short.

**Derby pants also work without special handling** — same reason. The underscore in `PFDERBY_HOTPINK` keeps the fabric code as one segment.

**No bugs in this branch.** The V4 code is identical to the current code.

**Live run verification (order #114265):** SKU `W-RW26_NATURALINDIGO-FLARE` → Gender: "F", Fabric: "RW26_NATURALINDIGO", Style: "FLARE", Thread: undefined (3-seg online). The Asana task's Thread field correctly showed "Light Pink" from the `Properties["Thread Color"]` fallback.

**Live run verification (order #100002):** SKU `M-RW26_NATURALINDIGO-STRAIGHT` → Gender: "M", Fabric: "RW26_NATURALINDIGO", Style: "STRAIGHT", Thread: undefined (3-seg online). Asana Thread field showed "Green" from the line item property.

---

## Shoes — `SHOE-` Branch

**Current code (lines 57–71) — CONTAINS BUG:**

```jsx
if (LI.sku.includes("SHOE-")) {
  Properties.ProductType = "Shoe";
  Properties.Quantity = LI.quantity;
  Properties.Shoe = SKU[1];
  if (LI.variant_title?.includes("/")) {
    let [gender, size] = LI.variant_title.split("/").map(item => item.trim());
    Properties.Gender = gender || "";
    Properties.Size = size || "";
  } else {
    Properties.Gender = "";
    Properties.Size = "";
  }
  Properties.DueDays = 45;
  Properties.DueDate = setDueDate(45);
}
```

**Triggers on:** Any SKU containing `SHOE-`.

> **NEW-01 (HIGH):** This code was written for the old SKU format (`SHOE-CLASSIC` — 2 segments) where gender and size were parsed from `variant_title` (e.g., `"Men / 10"`). The SKUs were updated to `SHOE-CLASSIC-M-10` (4 segments) with gender and size directly in the SKU. The old `variant_title` parsing may produce wrong results or empty strings for the new format.
> 

**Old SKU format (no longer in use):**

```
SHOE-CLASSIC          → SKU[1] = "CLASSIC", variant_title = "Men / 10"
```

**New SKU format (current):**

```
SHOE-CLASSIC-M-10     → SKU[1] = "CLASSIC", SKU[2] = "M", SKU[3] = "10"
SHOE-CUSTOM-W-7.5     → SKU[1] = "CUSTOM", SKU[2] = "W", SKU[3] = "7.5"
```

**V4 fixed code:**

```jsx
if (LI.sku.startsWith("SHOE-")) {
  Properties.ProductType = "Shoe";
  Properties.Quantity = LI.quantity;
  Properties.Shoe = SKU[1]; // "CLASSIC" or "CUSTOM"

  if (SKU.length >= 4) {
    // New format: SHOE-{Type}-{Gender}-{Size}
    Properties.Gender = SKU[2] || "";
    Properties.Size = SKU[3] || "";
  } else if (LI.variant_title?.includes("/")) {
    // Legacy fallback: variant_title = "Men / 10"
    let [gender, size] = LI.variant_title.split("/").map(s => s.trim());
    Properties.Gender = gender || "";
    Properties.Size = size || "";
  } else {
    Properties.Gender = "";
    Properties.Size = "";
  }

  Properties.DueDays = 45;
  Properties.DueDate = setDueDate(45);
}
```

The fix checks segment count first (4+ segments = new format, read from SKU) and falls back to variant_title parsing for any legacy orders that might still exist.

**Also fixed:** Changed `LI.sku.includes("SHOE-")` to `LI.sku.startsWith("SHOE-")` for consistency with all other branches. The `includes` check could theoretically match a SKU like `CUSTOM-SHOE-XYZ` — unlikely but sloppy.

**Shoe customization properties** (Laces, Swoosh, Back Tab/Eyelets, Toe Cap/Back Heel, Toe Box/Mid Panel) are entered manually by staff from paper order forms. They flow through the `LI.properties.reduce()` initialization into Properties and are mapped directly to Asana fields in Step 25 — no special handling needed in this branch.

**Properties output (new format):**

| Property | Source | Example |
| --- | --- | --- |
| `ProductType` | Hardcoded | `"Shoe"` |
| `Quantity` | `LI.quantity` | `1` |
| `Shoe` | `SKU[1]` | `"CLASSIC"` or `"CUSTOM"` |
| `Gender` | `SKU[2]` (V4) | `"M"` or `"W"` |
| `Size` | `SKU[3]` (V4) | `"10"`, `"7.5"` |
| `DueDays` | Hardcoded | `45` |
| `DueDate` | Calculated | `"2026-05-16"` |
| `Laces` | Line item property | `"White"` |
| `Swoosh` | Line item property | `"Red"` |

---

## Video Gift Cards — `VID-GIFT-` Branch

**Current code (lines 74–83):**

```jsx
if (LI.sku.startsWith("VID-GIFT-")) {
  Object.assign(Properties, {
    ProductType: "Video Card",
    Message: SKU[2],
    Denomination: SKU[3],
    Quantity: LI.quantity,
    DueDays: 5,
    DueDate: setDueDate(5),
  });
}
```

**Triggers on:** Any SKU starting with `VID-GIFT-`.

This is the simplest branch. All 8 video gift card SKUs have the same 4-segment format:

| SKU | SKU[2] (Message) | SKU[3] (Denomination) |
| --- | --- | --- |
| `VID-GIFT-BDJ-500` | `BDJ` | `500` |
| `VID-GIFT-PERSONAL-500` | `PERSONAL` | `500` |
| `VID-GIFT-BDJ-450` | `BDJ` | `450` |
| `VID-GIFT-PERSONAL-450` | `PERSONAL` | `450` |
| `VID-GIFT-BDJ-300` | `BDJ` | `300` |
| `VID-GIFT-PERSONAL-300` | `PERSONAL` | `300` |
| `VID-GIFT-BDJ-200` | `BDJ` | `200` |
| `VID-GIFT-PERSONAL-200` | `PERSONAL` | `200` |

`Message` indicates the video type: `"BDJ"` = The Blue Delta Story (pre-made brand video), `"PERSONAL"` = customer uploads their own video.

**Note:** The VID-GIFT prefix is checked with `startsWith("VID-GIFT-")` which means `SKU.split("-")` produces `["VID", "GIFT", "BDJ", "500"]`. So `SKU[0]` = `"VID"` and `SKU[1]` = `"GIFT"` — neither is used. The useful data starts at `SKU[2]`.

**No bugs in this branch.** V4 code is identical to current.

**Properties output:**

| Property | Source | Example |
| --- | --- | --- |
| `ProductType` | Hardcoded | `"Video Card"` |
| `Message` | `SKU[2]` | `"BDJ"` or `"PERSONAL"` |
| `Denomination` | `SKU[3]` | `"500"`, `"450"`, `"300"`, `"200"` |
| `Quantity` | `LI.quantity` | `1` |
| `DueDays` | Hardcoded | `5` |
| `DueDate` | Calculated | `"2026-04-05"` |

---

## Belts — `CB` Branch

This is the most complex branch and the one with the most bugs. It handles 5 different SKU patterns across Online and POS channels.

### Current Code (Lines 86–137) — CONTAINS BUGS

```jsx
if (LI.sku.startsWith("CB")) {
  Object.assign(Properties, {
    ProductType: "Belt",
    LeatherCode: SKU[0].replace("_RYDER", ""),   // ⚠️ NEW-02: Ryder dead code
    Width: SKU[1],
    Hardware: SKU[2],                             // ⚠️ NEW-03: FLATBLACK not normalized
    Monogram:                                      // ⚠️ BUG-02: Only checks properties, misses POS MONO
      Properties.Monogram ??
      Properties.monogram ??
      Properties.Mono ??
      Properties.mono ??
      "NONE",
    DueDays: 30,
    DueDate: setDueDate(30),
  });

  const isRyder = LI.sku.includes("_RYDER-");     // ⚠️ NEW-02: Ryder dead code
  const isPOS = Source === "pos";                   // ⚠️ NEW-04: Fragile Source check

  // ⚠️ BUG-03: Reads last segment blindly — fails for NO_THREAD and MONO-only
  if (isRyder || isPOS) {
    Properties["Thread Color"] = SKU[SKU.length - 1];
  }

  // ⚠️ NEW-02: Ryder dead code — products removed from Shopify
  if (isRyder) {
    Properties.Lettering =
      Properties.Monogram ??
      Properties.monogram ??
      Properties.Mono ??
      Properties.mono ??
      "";
  }

  const LeatherNames = {
    CB1: "Dark Brown Leather", CB2: "Mid Brown Leather", CB3: "Light Brown Leather",
    CB4: "Navy Leather", CB5: "Football Leather", CB6: "Black Leather",
    CB7: "Natural Derby Leather", CB8: "English Tan Leather", CB9: "Red Leather",
    CB10: "Light Brown Leather", CB11: "Gray",
  };

  Properties.LeatherName = LeatherNames[Properties.LeatherCode] || "";
}
```

### What's Wrong — Bug by Bug

**BUG-02 (CRITICAL): POS belt monogram not detected.** The `Monogram` property is set from a fallback chain of line item property names (`Monogram`, `monogram`, `Mono`, `mono`). POS belts have no line item properties — all four return `undefined`, so Monogram defaults to `"NONE"`. But POS belt SKUs with monogram have a `MONO` segment (e.g., `CB6-1-BRASS-MONO-BLACK`). The code never checks for this. All POS belt monograms are silently lost. **Affects 819 POS belt SKUs with MONO.**

**BUG-03 (HIGH): POS belt thread misread.** The code does `SKU[SKU.length - 1]` for POS belt thread. This fails for:

- `CB6-1-BRASS-NO_THREAD` → last segment is `"NO_THREAD"`, should be empty
- `CB6-1-BRASS-MONO-NO_THREAD` → last segment is `"NO_THREAD"`, should be empty
- Both produce the literal string `"NO_THREAD"` in the Asana Belt-Stitching field instead of an empty value.

**NEW-03 (MEDIUM): Hardware not normalized.** POS belts use `FLATBLACK` (no underscore), online belts use `FLAT_BLACK` (with underscore). The code passes the raw value. **Affects 546 POS belt SKUs.**

**NEW-04 (HIGH): Fragile POS detection.** The code uses `Source === "pos"` to detect POS orders. This depends on `inputData.Source` being correctly set from upstream. A more reliable approach is to check SKU segment count: 3 = online, 4+ = POS.

**NEW-02 (LOW): Ryder dead code.** Three code blocks reference Ryder products that no longer exist: `SKU[0].replace("_RYDER", "")`, `const isRyder = LI.sku.includes("_RYDER-")`, and the `if (isRyder)` lettering block.

### V4 Fixed Code — Complete Belt Rewrite

```jsx
if (LI.sku.startsWith("CB")) {
  const leatherCode = SKU[0];
  const beltSegments = SKU.length;

  // FIX NEW-03: Normalize hardware
  let hardware = SKU[2];
  if (hardware === "FLATBLACK") {
    hardware = "FLAT_BLACK";
  }

  Object.assign(Properties, {
    ProductType: "Belt",
    LeatherCode: leatherCode,
    Width: SKU[1],
    Hardware: hardware,
    DueDays: 30,
    DueDate: setDueDate(30),
  });

  // FIX BUG-02 + BUG-03 + NEW-04: Segment-count detection
  if (beltSegments === 3) {
    // Online: thread + monogram from line item properties
    Properties.Monogram =
      Properties.Monogram ?? Properties.monogram ??
      Properties.Mono ?? Properties.mono ?? "NONE";
  } else if (beltSegments === 4) {
    // POS, no monogram: CB6-1-BRASS-{Thread|NO_THREAD}
    const threadSeg = SKU[3];
    Properties["Thread Color"] = threadSeg === "NO_THREAD" ? "" : threadSeg;
    Properties.Monogram = "NONE";
  } else if (beltSegments === 5) {
    // POS, with monogram: CB6-1-BRASS-MONO-{Thread|NO_THREAD}
    const threadSeg = SKU[4];
    Properties["Thread Color"] = threadSeg === "NO_THREAD" ? "" : threadSeg;
    Properties.Monogram = "YES";
  }

  const LeatherNames = {
    CB1: "Dark Brown Leather", CB2: "Mid Brown Leather", CB3: "Light Brown Leather",
    CB4: "Navy Leather", CB5: "Football Leather", CB6: "Black Leather",
    CB7: "Natural Derby Leather", CB8: "English Tan Leather", CB9: "Red Leather",
    CB10: "Light Brown Leather", CB11: "Gray",
  };

  Properties.LeatherName = LeatherNames[leatherCode] || "";
}
```

### How the V4 Belt Code Works — Pattern by Pattern

| SKU Example | Segments | Thread Source | Thread Value | Monogram Value | Hardware |
| --- | --- | --- | --- | --- | --- |
| `CB6-1.5-BRASS` | 3 (online) | Line item property | *(from Properties)* | *(from Properties)* | `BRASS` |
| `CB6-1.5-FLAT_BLACK` | 3 (online) | Line item property | *(from Properties)* | *(from Properties)* | `FLAT_BLACK` |
| `CB6-1-BRASS-BLACK` | 4 (POS) | `SKU[3]` | `"BLACK"` | `"NONE"` | `BRASS` |
| `CB6-1-BRASS-NO_THREAD` | 4 (POS) | `SKU[3]` | `""` (empty) | `"NONE"` | `BRASS` |
| `CB6-1-FLATBLACK-BLACK` | 4 (POS) | `SKU[3]` | `"BLACK"` | `"NONE"` | `FLAT_BLACK` (normalized) |
| `CB6-1-BRASS-MONO-BLACK` | 5 (POS) | `SKU[4]` | `"BLACK"` | `"YES"` | `BRASS` |
| `CB6-1-BRASS-MONO-NO_THREAD` | 5 (POS) | `SKU[4]` | `""` (empty) | `"YES"` | `BRASS` |

### Leather Legend

The `LeatherNames` lookup table converts leather codes to human-readable names:

| Code | Leather Name | Notes |
| --- | --- | --- |
| CB1 | Dark Brown Leather |  |
| CB2 | Mid Brown Leather |  |
| CB3 | Light Brown Leather | Not in current product exports — may be legacy |
| CB4 | Navy Leather |  |
| CB5 | Football Leather |  |
| CB6 | Black Leather |  |
| CB7 | Natural Derby Leather | Not in current product exports — may be legacy |
| CB8 | English Tan Leather | Not in current product exports — may be legacy |
| CB9 | Red Leather |  |
| CB10 | Light Brown Leather | Same name as CB3 — intentional |
| CB11 | Gray | Not in current product exports — may be legacy |

Active leather codes in current products: CB1, CB2, CB4, CB5, CB6, CB9, CB10.

### Properties Output (Belt)

| Property | Source | Example (Online) | Example (POS) |
| --- | --- | --- | --- |
| `ProductType` | Hardcoded | `"Belt"` | `"Belt"` |
| `LeatherCode` | `SKU[0]` | `"CB6"` | `"CB6"` |
| `LeatherName` | Lookup table | `"Black Leather"` | `"Black Leather"` |
| `Width` | `SKU[1]` | `"1.5"` | `"1"` |
| `Hardware` | `SKU[2]` (normalized) | `"FLAT_BLACK"` | `"FLAT_BLACK"` |
| `Monogram` | Properties (online) or MONO detection (POS) | `"ABC"` or `"NONE"` | `"YES"` or `"NONE"` |
| `"Thread Color"` | Properties (online) or SKU (POS) | `"Navy"` | `"BLACK"` or `""` |
| `DueDays` | Hardcoded | `30` | `30` |
| `DueDate` | Calculated | `"2026-04-30"` | `"2026-04-30"` |

---

## Tag Processing

**Current code (lines 140–147) — CONTAINS BUG:**

```jsx
let Tags = inputData.Tags ? inputData.Tags.split(",").map(tag => tag.trim()) : [];
let TagsProduct = ["Cashiers", "Denim", "Vintage", "Performance", "Chino", "Jhino"];
let Alteration = Tags.includes("alteration");

let EventTags = Tags.filter(tag => !TagsProduct.includes(tag));
Tags = Tags.filter(tag => TagsProduct.includes(tag));
```

This section runs after all product-specific branches and splits the order's tags into two groups:

**Product Tags** — Tags that match the `TagsProduct` list. These describe the product category and appear in the Asana "Product Type" field. Example: `"Denim"`, `"Performance"`, `"Cashiers"`.

**Event Tags** — Everything else. These are event names, sales rep names, payment notes, and rush tiers. They appear in the Asana "Event (tags)" field. Example: `"Andy"`, `"Paid With Gift Card"`, `"RUSH COBALT"`.

**Alteration** — Boolean flag, true if the tag `"alteration"` is present. Used by Steps 16 and 20 to exclude alteration orders from the standard pipeline.

> **BUG-07 (LOW):** The matching is case-sensitive. `TagsProduct` uses title case (`"Denim"`), but Shopify tags could arrive in any case. A tag of `"denim"` (lowercase) would NOT match `"Denim"` and would incorrectly end up in EventTags. Similarly, `Tags.includes("alteration")` is lowercase-only — `"Alteration"` (capitalized) wouldn't match.
> 

**V4 fixed code:**

```jsx
let Tags = inputData.Tags ? inputData.Tags.split(",").map(tag => tag.trim()) : [];
let TagsProduct = ["cashiers", "denim", "vintage", "performance", "chino", "jhino"];

let tagsLower = Tags.map(tag => tag.toLowerCase());

let Alteration = tagsLower.includes("alteration");
let EventTags = Tags.filter((_, i) => !TagsProduct.includes(tagsLower[i]));
Tags = Tags.filter((_, i) => TagsProduct.includes(tagsLower[i]));
```

The fix compares lowercase versions for matching but preserves the original casing in the output arrays. So a tag of `"DENIM"` matches `"denim"` in the comparison but appears as `"DENIM"` in the Tags output.

**Live run verification (order #100002):** Tags input = `"Andy, Denim, Paid With Gift Card, RUSH COBALT"` → Product Tags: `["Denim"]`, Event Tags: `["Andy", "Paid With Gift Card", "RUSH COBALT"]`, Alteration: `false`.

---

## Output Schema

Everything Step 13 outputs and where it's consumed downstream:

| Field | Type | Example | Consumed By |
| --- | --- | --- | --- |
| `Source` | string | `"web"` or `"pos"` | Informational (V4 removes its use in belt detection) |
| `Alteration` | boolean | `false` | Steps 16/20: Path conditions (excludes alterations) |
| `Tags` | string[] | `["Denim"]` | Steps 21/23: Asana "Product Type" field |
| `EventTags` | string[] | `["Andy", "RUSH COBALT"]` | Steps 21/23/25/27: Asana "Event (tags)" field |
| `CartToken` | string | `":censored:..."` or `""` | Passed through (originally from Step 5) |
| `DateCreated` | string | `"2026-03-31"` | Passed through |
| `Properties` | object | `{ProductType, Gender, Fabric, Style, Thread, ...}` | Steps 21/23/25/27: All product-specific Asana fields |
| `SKU` | string[] | `["M", "RW29_DARKINDIGO", "STRAIGHT"]` | Informational / debugging |
| `LI` | object | Full parsed line item | Informational / debugging |

The `Properties` object is the big one — it contains every product-specific field that gets mapped to Asana custom fields. The exact contents depend on which branch ran (pants, belts, shoes, or video cards). See the per-branch Properties output tables above.

---

## Bugs in This Step

| # | Title | Severity | Branch | Status |
| --- | --- | --- | --- | --- |
| BUG-02 | POS belt monogram not detected from MONO in SKU | **CRITICAL** | Belt | Fixed in V4 |
| BUG-03 | POS belt thread misread for NO_THREAD and MONO patterns | **HIGH** | Belt | Fixed in V4 |
| NEW-01 | Shoe SKU format changed, old parser reads wrong segments | **HIGH** | Shoe | Fixed in V4 |
| NEW-04 | Belt POS detection via fragile Source check | **HIGH** | Belt | Fixed in V4 |
| NEW-03 | FLATBLACK not normalized to FLAT_BLACK | **MEDIUM** | Belt | Fixed in V4 |
| BUG-07 | Case-sensitive tag matching | **LOW** | Tags | Fixed in V4 |
| NEW-02 | Ryder dead code (products deleted) | **LOW** | Belt | Fixed in V4 |

For full bug analysis with affected SKU counts, root cause, test cases, and before/after code, see the **Bug Tracker & Fix Plan** page.

---

*This page documents step-13.js. The V4 fixes described here are **deployed** — the zap is live at **v51** (June 2026); the Bold Metrics → Skynet migration is complete. Historical note: this page was originally written against v42 with step-13-v4.js staged for deployment as v43; that work has since shipped (v43 → … → v51). Both the original and V4 code files remain in the project resources for reference; the snippets above are shown inline for readability. See the Bug Tracker page for the original deployment details.*