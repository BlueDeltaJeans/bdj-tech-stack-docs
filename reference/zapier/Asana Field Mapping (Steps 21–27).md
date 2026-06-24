# Asana Field Mapping (Steps 21–27)

> ## ✅ STATUS / UPDATE (June 2026) — MIGRATION COMPLETE: six measurement fields written by Skynet; VT input mapping live at zap v51
>
> **The Bold Metrics → Skynet migration is DONE and live.** The storefront Virtual Tailor no longer calls Bold Metrics; Skynet computes the six measurements server-side post-order and writes them into Asana. The "Virtual Tailor Measurement Fields" mapping below is *legacy* — it accurately describes how the six measurement fields were filled for orders placed **before** the migration, and it's still the right reference for those historical tasks. It is **stale for NEW online orders.**
>
> What changed (current state = DONE/live):
>
> - **The 6 measurement fields are no longer populated by Zapier from `5. Extras *`.** Bold Metrics was removed from the storefront cart, so those `note_attributes` arrive empty (see [Step 5 — Order Parser](./Step%205%20—%20Order%20Parser.md) §STATUS). On new orders the Step 21 tokens `5. Extras Hip Circum` / `Jean Inseam` / `Knee Circum` / `Thigh Circum` / `U Crotch` / `Waist Average` resolve to **blank**.
> - **They are now WRITTEN BY SKYNET *after* task creation.** Skynet detects the VT order, computes the six measurements server-side, and writes them directly into the Asana custom fields via the Asana API (`PUT /tasks/:id` with a `custom_fields` GID map). So at task-creation time these fields are empty; they fill in a few seconds later out-of-band from Zapier.
> - **Five `VT *` input fields on the Pant Pipeline carry the raw Virtual Tailor inputs** (these are *inputs*, distinct from the measurement *outputs*). The Zapier mapping that populates them from `5. BDJUserData *` is **COMPLETE and live** — each order now populates all five `VT *` Asana input fields. These bug fixes plus later work are deployed: **zap ID 281794942 is at VERSION 51** (the older "V4 fixes pending as v43" framing is stale).
> - **Exact-display-name matching matters.** Skynet reads Asana custom fields **by exact display name, never by GID**. Renaming any of these fields — including the **trailing period in `Waist Avg.`** — silently breaks Skynet's parser and write-back. Do not "tidy up" field names.
>
> 📄 Full input/output field map, GIDs, and Bold Metrics request/response contract: [Bold Metrics Migration — Impact on the Zapier Pipeline](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) · Ground truth: `Measurement-Calculator/bold-metrics-skynet-migration-context.md` (§8 input gap list, §9 response→Asana mapping, §10 GID reference).

This page documents the exact field mappings from Zapier to Asana for all four production pipelines. Every mapping was manually transcribed from screenshots of the actual Zapier step configuration, then cross-referenced against live zap run data to confirm Asana custom field GIDs and actual values.

When you see a mapping like `13. Properties Fabric`, it means: "Step 13's output → the Properties object → the Fabric key." When you see `5. Extras Hip Circum`, it means: "Step 5's output → the Extras object → the Hip Circum key."

Fields with **multiple tokens** listed use Zapier's fallback behavior — Zapier tries each token in order and uses the first one that has a value. This is how online vs. POS data sources coexist: the same Asana field can pull from a line item property (online) or a SKU-parsed value (POS) depending on which token has data.

---

## Shared Fields — All Pipelines

These fields are configured identically in Steps 21, 23, 25, and 27:

### Task Name

Constructed from 6 tokens concatenated together:

| Token | Source | Example Value |
| --- | --- | --- |
| Order Number | `4. Response Data Order Name` | `#100001` |
| *(space)* | Literal |  |
| First Name | `4. Response Data Order Customer First Name` | `Jordan` |
| *(space)* | Literal |  |
| Last Name | `4. Response Data Order Customer Last Name` | `Sample` |
| *(space)* | Literal |  |
| Loop Iteration | `10. Loop Iteration` | `1` |
| `/` | Literal |  |
| Total Iterations | `10. Loop Total Iterations` | `1` |
| Rush Suffix | `5. RUSH` | `| RUSH ROYAL` (or empty) |

**Result:** `#100001 Jordan Sample 1/1 | RUSH ROYAL`

### Due On

| Token | Source | Example Value |
| --- | --- | --- |
| Order Date | `4. Response Data Order Created At` | `2026-01-21T13:40:00-06:00` |
| Due Days Offset | `13. Properties Due Days` | `14 d` |

Zapier adds the due days offset to the order date to calculate the Asana due date.

### Event (tags)

| Token | Source | Example Value |
| --- | --- | --- |
| Event Tags | `13. Event Tags` | `Ella,RUSH ROYAL` |

### Order Type

| Token | Source | Example Value |
| --- | --- | --- |
| Order Type | `5. Order Type` | `Online` |

### Quantity

| Token | Source | Example Value |
| --- | --- | --- |
| Quantity | `13. LI Quantity` | `1` |

### Note

| Token | Source | Example Value |
| --- | --- | --- |
| Note | `4. Response Data Order Note` | *(empty if no note)* |

---

## Step 21 — Pant Pipeline

**Project:** Auto | Pant Pipeline (GID: `1206657933205972`)
**Products:** Men's Pants, Women's Pants, Shorts, Kentucky Derby Pants
**Total fields:** 49 (6 shared + 43 pant-specific)

### Order Information Fields

| Asana Field | Asana GID | Source Token(s) | Example | Notes |
| --- | --- | --- | --- | --- |
| New Customer | `1206649008312472` | `5. Customer New` | `Yes` | BUG-05: always "Yes" in current code |
| Gender | `1112754700909040` | `14. Output` | `1112754700909042` | Enum GID from Step 14 lookup |
| Order Type | `1206649285610127` | `5. Order Type` | `Online` |  |
| Product Type | `1203386148481388` | `13. Tags` | `Denim` | First product tag from filtered tags |
| Quantity | `1206648951199301` | `13. LI Quantity` | `1` |  |
| Event (tags) | `1203340633343053` | `13. Event Tags` | `Ella,RUSH ROYAL` | Comma-separated |

### Production Flag Fields

| Asana Field | Asana GID | Source Token(s) | Notes |
| --- | --- | --- | --- |
| POF | `1206660699367056` | `5. Extras Pattern On File` (×2) | "Pattern On File" — token listed twice in config |
| White Glove | `1208695148089782` | `5. Extras White Glove` | Premium handling flag |
| Note | `1206648966139310` | `4. Response Data Order Note` | Free-text order note |
| Print PDF | `1207446201916950` | *(not in transcription)* | May be set by a different mechanism |

### Garment Specification Fields

| Asana Field | Asana GID | Source Token(s) | Fallback Behavior | Example |
| --- | --- | --- | --- | --- |
| Fabric | `1203263757944175` | `13. Properties Fabric` | Single source | `RW29_DARKINDIGO` |
| Style | `1206671503498614` | `13. Properties Style` | Single source | `STRAIGHT` |
| Thread | `1206671503498616` | `13. Properties Thread` → `13. Properties Thread Color` | **Fallback chain.** Online: Thread is undefined, falls to Thread Color (from SC Product Options). POS: Thread has value from SKU[3]. | `Navy` |
| Monogram | `1206660694454917` | `13. Properties Monogram` → `13. Properties Lettering` | **Fallback chain.** Belts: Monogram from properties. Pants: Lettering from SC Product Options monogram add-on. | `tft` |
| Monogram Thread | `1206706696879461` | `13. Properties Color` | Single source — monogram thread color from SC Product Options | `Match Thread` |
| Fit | `1206671503498618` | `5. Extras Jean Fit` → `13. Properties Fit` | **Fallback chain.** Virtual Tailor orders: from note attributes. Manual: from line item property. | `Tailored` |
| Shoe Type | `1207059370633718` | `5. Extras Shoe Type` | Customer's common shoe type from Virtual Tailor form — NOT a shoe product field | `Chelsea Boots` |
| Pockets | `1206671503498620` | `13. Properties Pockets` → `13. Properties Front Pockets` → `13. Properties Front Pockets` | **Triple fallback.** Third token is duplicate of second. Online women's pants set this via SC Product Options. | `Fake` |
| Break | `1206671503498622` | `5. BDJ User Data Break` → `13. Properties Break` | **Fallback chain.** Virtual Tailor: from BDJ User Data. Manual: from line item property. |  |
| Waist Ride | `1206671503499680` | `5. BDJ User Data Waist Ride` → `13. Properties Waist` → `13. Properties Waist Height` | **Triple fallback.** VT: from BDJ User Data. POS: from Properties.Waist. Online: from SC Product Options "Waist Height". | `Mid` |

### Virtual Tailor Measurement Fields (from Step 5 Extras)

> **⚠️ MIGRATION UPDATE (June 2026): these 6 fields are now WRITTEN BY SKYNET after task creation — NOT by the Zapier `5. Extras *` tokens.**
> Bold Metrics no longer runs on the storefront cart, so the source note attributes arrive empty and the Zapier tokens below resolve to blank on new online orders. **Skynet** computes the measurements post-order (calls Bold Metrics with the customer's VT inputs) and writes these six custom fields directly via the Asana API. The mapping table below is retained as the **historical (pre-migration)** reference and still applies to older tasks. Skynet matches every field **by exact display name** — note the **trailing period in `Waist Avg.`** (do not rename). See the §STATUS callout at the top of this page.

These come from the order's note attributes, **historically** populated by the Virtual Tailor (Bold Metrics AI) for online orders. *(Pre-migration mapping — for new orders, see the migration note above.)*

| Asana Field | Asana GID | Source Token | June 2026 status |
| --- | --- | --- | --- |
| Hip Circum | `1206671503499682` | `5. Extras Hip Circum` | ⚠️ Now written by Skynet post-creation (token resolves blank) |
| Jean Inseam | `1206671503499684` | `5. Extras Jean Inseam` | ⚠️ Now written by Skynet post-creation |
| Knee Circum | `1206671503499686` | `5. Extras Knee Circum` | ⚠️ Now written by Skynet post-creation |
| Thigh Circum | `1206671503499688` | `5. Extras Thigh Circum` | ⚠️ Now written by Skynet post-creation (from `thigh_circum_proximal`) |
| U Crotch | `1206671503499690` | `5. Extras U Crotch` | ⚠️ Now written by Skynet post-creation |
| Waist Avg. | `1206671503499692` | `5. Extras Waist Average` | ⚠️ Now written by Skynet post-creation; **exact name incl. trailing period** |

### NEW — Virtual Tailor *Input* Fields (June 2026 migration)

> **These are the *inputs* Skynet uses to compute the measurements — not the measurement *outputs* above.** They were created on the Pant Pipeline for the migration. Three pre-existing fields (`Gender`, `Age`, `Weight`) plus the **five `VT *` fields** below carry the raw Virtual Tailor form answers. The **Zapier mapping (owned by Caleb) that populates them from `5. BDJUserData *`** on each new order is **COMPLETE and live (zap v51)** — each order now populates all five `VT *` fields. Skynet reads them **by exact display name**.

| Asana Field (input) | Asana GID | Source (VT form input) | Bold Metrics param (legacy) | Status |
| --- | --- | --- | --- | --- |
| Gender | `1112754700909040` | `5. BDJ User Data Gender` (enum) | `gender` (m/w) | pre-existing |
| Age | `1206671503499694` | `5. BDJ User Data Age` | `age` | pre-existing |
| Weight | `1206671503499698` | `5. BDJ User Data Weight` | `weight` | pre-existing |
| VT Height | `1215612213755174` | `5. BDJ User Data Height` (inches) | `height` | live (v51) |
| VT Shoe Size | `1215612213755176` | `5. BDJ User Data Shoe Size` | `shoe_size_us` | live (v51) |
| VT Waist | `1215612213755178` | `5. BDJ User Data Waist` (men) | `waist_circum_preferred` | live (v51) |
| VT Inseam | `1215612213755180` | `5. BDJ User Data Inseam` (men) | `jean_inseam` | live (v51) |
| VT Bra Size | `1215851864495064` | `5. BDJ User Data Bra Size` (women) | `bra_size` | live (v51) |

> **Don't confuse inputs with outputs:** `VT Waist` (input) ≠ `Waist Avg.` (output); `VT Inseam` (input) ≠ `Jean Inseam` (output). The `VT *` fields feed the Bold Metrics request; Skynet writes the measurement fields above from the response. The full request/response mapping (including the `waist_circum_preferred` and `thigh_circum_proximal` source picks) is in the [migration bridge doc](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md) and `bold-metrics-skynet-migration-context.md` (§8–§10).

### BDJ User Data Fields (from Step 5 BDJUserData)

These come from the parsed "BDJ User Data" note attribute — the customer's body profile from the Virtual Tailor form.

| Asana Field | Asana GID | Source Token |
| --- | --- | --- |
| Age | `1206671503499694` | `5. BDJ User Data Age` |
| Body | *(not confirmed)* | `5. BDJ User Data Body` |
| Weight | `1206671503499698` | `5. BDJ User Data Weight` |

### In-Person Measurement Fields (from Step 5 Extras)

These are populated for POS/in-person measured orders where a tailor takes physical measurements. They are empty for online Virtual Tailor orders.

| Asana Field | Asana GID | Source Token |
| --- | --- | --- |
| Waist Around | `1206889563216920` | `5. Extras Waist Around` |
| Seat Down | `1206890038906314` | `5. Extras Seat Down` |
| Seat Around | `1206889231694800` | `5. Extras Seat Around` |
| Thigh Upper Down | `1206889694672039` | `5. Extras Thigh Upper Down` |
| Thigh Upper Around | `1206890039124414` | `5. Extras Thigh Upper Around` |
| Thigh Middle Down | `1206889231694802` | `5. Extras Thigh Middle Down` |
| Thigh Middle Around | `1206889572274478` | `5. Extras Thigh Middle Around` |
| Thigh Lower Down | `1206889573928500` | `5. Extras Thigh Lower Down` |
| Thigh Lower Around | `1206889699211441` | `5. Extras Thigh Lower Around` |
| Outseam | `1206889231694804` | `5. Extras Outseam` |
| Knee Up | `1206889231694806` | `5. Extras Knee Up` |
| Knee Around | `1206890052338070` | `5. Extras Knee Around` |
| Calf Up | `1206890051647572` | `5. Extras Calf Up` |
| Calf Around | `1206889585835253` | `5. Extras Calf Around` |
| Front Rise | `1206890057965690` | `5. Extras Front Rise` |
| Full Rise | `1206889589197926` | `5. Extras Full Rise` |
| Leg Opening | `1206889587561906` | `5. Extras Leg Opening` |

### Meta Fields

| Asana Field | Asana GID | Source Token |
| --- | --- | --- |
| Measured By | `1206889231694808` | `5. Extras Measured By` |
| Photo 1 | `1206889231694810` | `5. Extras Photo 1` |
| Photo 2 | `1206889231694812` | `5. Extras Photo 2` |

---

## Step 23 — Belt Pipeline

**Project:** Auto | Belt Pipeline (GID: `1206657932919233`)
**Products:** All belts (Online 3-segment + POS 4/5-segment)
**Total fields:** 17 (6 shared + 11 belt-specific)

### Order Information Fields

| Asana Field | Source Token(s) | Notes |
| --- | --- | --- |
| New Customer | `5. Customer New` | Same as Pant Pipeline |
| Gender | `14. Output` | Enum GID from Step 14 lookup |
| Order Type | `5. Order Type` |  |
| Event (tags) | `13. Event Tags` |  |
| Quantity | `13. LI Quantity` |  |

### Production Flag Fields

| Asana Field | Source Token(s) | Notes |
| --- | --- | --- |
| POF | `5. Extras Pattern On File` | Single token (vs. double in Pant) |
| White Glove | `5. Extras White Glove` |  |
| Note | `4. Response Data Order Note` |  |

### Belt-Specific Fields

| Asana Field | Source Token(s) | Fallback Behavior | Example |
| --- | --- | --- | --- |
| Belt - Leather | `13. Properties Leather Name` | Single source — from LeatherNames lookup table | `Black Leather` |
| Belt - Width | `13. Properties Width` | Single source — from SKU[1] | `1.5` |
| Belt - Hardware | `13. Properties Hardware` | Single source — from SKU[2] (V4 normalizes FLATBLACK → FLAT_BLACK) | `BRASS` |
| Belt - Stitching | `13. Properties Thread Color` | Single source — Online: from SC Product Options. POS: from SKU segment. V4 handles NO_THREAD → empty. | `Navy` |
| Monogram | `13. Properties Monogram` → `13. Properties Lettering` | **Fallback chain.** Same as Pant Pipeline. Online: from SC Product Options. POS V4: "YES" or "NONE" from MONO detection. |  |

### Waist Measurements

| Asana Field | Source Token |
| --- | --- |
| Waist Avg. | `5. Extras Waist Average` |
| Waist Around | `5. Extras Waist Around` |

> **Note:** The Belt Pipeline receives waist measurements for sizing reference but does NOT receive the full set of in-person body measurements (thigh, knee, calf, rise, etc.) that the Pant Pipeline gets. This makes sense — belts only need waist size.
> 
> **⚠️ June 2026:** For VT/online belt orders, `5. Extras Waist Average` is now empty (Bold Metrics moved to Skynet). The Skynet migration's scope is the **Pant Pipeline** (GID `1206657933205972`) — Belt/Shoe/Video pipelines are explicitly out of scope, and there is currently no Skynet write-back for the Belt `Waist Avg.` field. `Waist Around` (in-person/POS) is unaffected. See the §STATUS callout at the top of this page.
> 

---

## Step 25 — Shoe Pipeline

**Project:** Auto | Shoe Pipeline (GID: `1206648505149980`)
**Products:** All shoes (Classic + Custom, POS/event only)
**Total fields:** 15 (6 shared + 9 shoe-specific)

### Order Information Fields

| Asana Field | Source Token(s) | Notes |
| --- | --- | --- |
| New Customer | `5. Customer New` |  |
| Gender | `14. Output` | Enum GID from Step 14 lookup |
| Order Type | `5. Order Type` |  |
| Event (tags) | `13. Event Tags` |  |
| Quantity | `13. LI Quantity` |  |

### Shoe-Specific Fields

| Asana Field | Source Token(s) | Example | Notes |
| --- | --- | --- | --- |
| Shoe | `13. Properties Shoe` | `CLASSIC` | Shoe type from SKU[1] |
| Size | `13. Properties Size` | `10` | V4: from SKU[3]. Old code: from variant_title. |
| Laces | `13. Properties Laces` | `White` | From line item properties (manually entered from paper form) |
| Swoosh | `13. Properties Swoosh` | `Red` | From line item properties |
| Back Tab | Eyelets | `13. Properties Back Tab Eyelets` |  | From line item properties |
| Toe Cap | Back Heel | `13. Properties Toe Cap Back Heel` |  | From line item properties |
| Toe Box | Mid Panel | `13. Properties Toe Box Mid Panel` |  | From line item properties |
| Note | `4. Response Data Order Note` |  |  |

> **Note:** The Shoe Pipeline does NOT have a Product Type field — unlike the other pipelines. All items in this pipeline are shoes by definition.
> 

> **Note:** Gender is mapped from Step 14's lookup (same as other pipelines), but the Gender value in the shoe branch comes from the SKU (`SKU[2]` = `"M"` or `"W"`) rather than from the pant branch's SKU[0] or hardcoded value. The lookup table handles both `M` and `W` identically to the pant pipeline.
> 

---

## Step 27 — Video Card Pipeline

**Project:** Auto | Video Card Pipeline (GID: `1206657933205969`)
**Products:** Video Gift Cards only
**Total fields:** 7 (6 shared + 1 video-specific)

This is the simplest pipeline. Video gift cards don't require production measurements, customization options, or gender — just basic order tracking.

### All Fields

| Asana Field | Source Token(s) | Example | Notes |
| --- | --- | --- | --- |
| Product Type | `13. Properties Product Type` | `Video Card` | Only pipeline that maps ProductType directly |
| Order Type | `5. Order Type` | `Online` |  |
| Event (tags) | `13. Event Tags` | `Ella,RUSH ROYAL` |  |
| Quantity | `13. LI Quantity` | `1` |  |
| Note | `4. Response Data Order Note` |  |  |

> **Note:** The Video Card Pipeline does NOT have New Customer or Gender fields — unlike the other three pipelines.
> 

---

## Multi-Token Fallback Chains

This is the debugging reference. When a value shows up wrong or blank in an Asana task, this table tells you exactly which sources Zapier checks and in what order.

| Asana Field | Pipeline | Token 1 (tried first) | Token 2 (fallback) | Token 3 (fallback) | When Each Wins |
| --- | --- | --- | --- | --- | --- |
| **Thread** | Pant | `13. Properties Thread` | `13. Properties Thread Color` | — | Token 1 wins for POS pants (SKU[3] has value). Token 2 wins for online pants (Thread is undefined, Thread Color set by SC Product Options). |
| **Monogram** | Pant, Belt | `13. Properties Monogram` | `13. Properties Lettering` | — | Token 1 wins for online belts (SC Product Options sets "Monogram"). Token 2 wins for online pants with monogram add-on (SC Product Options sets "Lettering"). For POS belts V4: Properties.Monogram = "YES" or "NONE" (from segment detection). |
| **Fit** | Pant | `5. Extras Jean Fit` | `13. Properties Fit` | — | Token 1 wins for Virtual Tailor orders (Jean Fit in note attributes). Token 2 wins for manual/POS orders (Fit as line item property). |
| **Pockets** | Pant | `13. Properties Pockets` | `13. Properties Front Pockets` | `13. Properties Front Pockets` | Token 2/3 win for online women's pants (SC Product Options sets "Front Pockets"). Token 3 is a duplicate of Token 2 in the config. |
| **Waist Ride** | Pant | `5. BDJ User Data Waist Ride` | `13. Properties Waist` | `13. Properties Waist Height` | Token 1 wins if customer specified in VT form. Token 3 wins for online women's pants (SC Product Options sets "Waist Height"). Token 2 may win for POS if staff sets "Waist" property. |
| **Break** | Pant | `5. BDJ User Data Break` | `13. Properties Break` | — | Token 1 wins if customer specified in VT form. Token 2 for manual entry. |
| **POF** | Pant | `5. Extras Pattern On File` | `5. Extras Pattern On File` | — | Same token twice — likely a config duplication, not an intentional fallback. |

### How Fallbacks Work in Practice

**Online pant order (e.g., order #100002 Alex Rivera):**

- Thread: `Properties.Thread` = undefined (3-seg SKU) → falls to `Properties["Thread Color"]` = `"Green"` (from SC Product Options) → Asana shows "Green" ✓
- Monogram: `Properties.Monogram` = undefined → falls to `Properties.Lettering` = `"RBJ"` (from SC Product Options monogram add-on) → Asana shows "RBJ" ✓
- Monogram Thread: `Properties.Color` = `"Match Thread"` → Asana shows "Match Thread" ✓
- Fit: `Extras["Jean Fit"]` = `"Tailored"` → Asana shows "Tailored" ✓

**POS pant order:**

- Thread: `Properties.Thread` = `"BLACK"` (from SKU[3]) → first token wins, Asana shows "BLACK" ✓
- Monogram: `Properties.Monogram` = undefined, `Properties.Lettering` = undefined → Asana shows blank (no monogram for POS pants) ✓

**Online belt order:**

- Belt - Stitching: `Properties["Thread Color"]` = `"Navy"` (from SC Product Options) → Asana shows "Navy" ✓
- Monogram: `Properties.Monogram` = `"ABC"` (from SC Product Options) → first token wins, Asana shows "ABC" ✓

**POS belt order (V4 code):**

- Belt - Stitching: `Properties["Thread Color"]` = `"BLACK"` (from SKU segment detection) → Asana shows "BLACK" ✓
- Monogram: `Properties.Monogram` = `"YES"` (from MONO segment detection) → Asana shows "YES" ✓

---

## Asana Custom Field GID Reference

All GIDs confirmed from live zap run data (orders #114265 and #100002) and, for the previously-uncaptured pipelines, from a live Asana API schema capture (June 2026). Use these when making direct Asana API calls or building the replacement database.

> **Shared vs. per-project field GIDs (important nuance):** Many fields are workspace-shared and reuse **one** `custom_field` GID across projects — e.g. Gender `1112754700909040` (enum), Order Type `1206649285610127`, Event (tags) `1203340633343053`, Quantity `1206648951199301`, Note `1206648966139310`, New Customer `1206649008312472`, Product Type `1203386148481388`, Print PDF `1207446201916950`, Monogram `1206660694454917`, Waist Around `1206889563216920`. **But some same-named fields are per-project DUPLICATES with DIFFERENT GIDs** — e.g. "Waist Avg." (Pant `1206671503499692`, Belt `1206660342181401`) and "White Glove" (Pant `1208695148089782`, Belt `1210137215275442`). This is exactly why Skynet matches Asana fields by **exact display name, not GID** — and why renaming silently breaks it.

### Gender Enum Options

| Option | GID | Display | Color |
| --- | --- | --- | --- |
| M | `1112754700909041` | M | Aqua |
| W | `1112754700909042` | W | Pink |
| Youth | `1209345887831124` | Youth | Yellow |

### Pant Pipeline Custom Fields

| GID | Field Name | Type |
| --- | --- | --- |
| `1206649008312472` | New Customer | text |
| `1112754700909040` | Gender | enum |
| `1203340633343053` | Event (tags) | text |
| `1206649285610127` | Order Type | text |
| `1203386148481388` | Product Type | text |
| `1206648951199301` | Quantity | text |
| `1206660699367056` | POF | text |
| `1208695148089782` | White Glove | text |
| `1206648966139310` | Note | text |
| `1207446201916950` | Print PDF | text |
| `1203263757944175` | Fabric | text |
| `1206671503498614` | Style | text |
| `1206671503498616` | Thread | text |
| `1206660694454917` | Monogram | text |
| `1206706696879461` | Monogram Thread | text |
| `1206671503498618` | Fit | text |
| `1207059370633718` | Shoe Type | text |
| `1206671503498620` | Pockets | text |
| `1206671503498622` | Break | text |
| `1206671503499680` | Waist Ride | text |
| `1206671503499682` | Hip Circum | text |
| `1206671503499684` | Jean Inseam | text |
| `1206671503499686` | Knee Circum | text |
| `1206671503499688` | Thigh Circum | text |
| `1206671503499690` | U Crotch | text |
| `1206671503499692` | Waist Avg. | text |
| `1206671503499694` | Age | text |
| `1206671503499698` | Weight | text |
| `1215612213755174` | VT Height | text |
| `1215612213755176` | VT Shoe Size | text |
| `1215612213755178` | VT Waist | text |
| `1215612213755180` | VT Inseam | text |
| `1215851864495064` | VT Bra Size | text |
| `1206889563216920` | Waist Around | text |
| `1206890038906314` | Seat Down | text |
| `1206889231694800` | Seat Around | text |
| `1206889694672039` | Thigh Upper Down | text |
| `1206890039124414` | Thigh Upper Around | text |
| `1206889231694802` | Thigh Middle Down | text |
| `1206889572274478` | Thigh Middle Around | text |
| `1206889573928500` | Thigh Lower Down | text |
| `1206889699211441` | Thigh Lower Around | text |
| `1206889231694804` | Outseam | text |
| `1206889231694806` | Knee Up | text |
| `1206890052338070` | Knee Around | text |
| `1206890051647572` | Calf Up | text |
| `1206889585835253` | Calf Around | text |
| `1206890057965690` | Front Rise | text |
| `1206889589197926` | Full Rise | text |
| `1206889587561906` | Leg Opening | text |
| `1206889231694808` | Measured By | text |
| `1206889231694810` | Photo 1 | text |
| `1206889231694812` | Photo 2 | text |

> **Note:** All custom fields were created by user "JK" (GID: `1200862872026994`), the previous developer. All fields are type `text` except Gender (which is `enum`). All fields have `privacy_setting: public_with_guests` and `default_access_level: admin`.
> 
> **✅ June 2026 additions:** The five `VT *` fields (`VT Height` `1215612213755174`, `VT Shoe Size` `1215612213755176`, `VT Waist` `1215612213755178`, `VT Inseam` `1215612213755180`, `VT Bra Size` `1215851864495064`) were created by Caleb for the Bold Metrics → Skynet migration. They hold Virtual Tailor **inputs** (read by Skynet by exact display name) and are distinct from the measurement **outputs** Skynet writes back. The Zapier mapping that populates them from `5. BDJUserData *` is complete and live (zap v51). See the [migration bridge doc](./Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md).
> 

### Belt Pipeline Custom Fields

**Project:** Auto | Belt Pipeline (GID: `1206657932919233`)

| GID | Field Name | Type | Notes |
| --- | --- | --- | --- |
| `1206649008312472` | New Customer | text | shared w/ Pant |
| `1112754700909040` | Gender | enum | shared (same enum options) |
| `1206660699367056` | POF | text | shared w/ Pant |
| `1206649285610127` | Order Type | text | shared |
| `1210137215275442` | White Glove | text | **per-project duplicate** (Pant uses `1208695148089782`) |
| `1203340633343053` | Event (tags) | text | shared |
| `1206648951199301` | Quantity | text | shared |
| `1206660342181401` | Waist Avg. | text | **per-project duplicate** (Pant uses `1206671503499692`) |
| `1206889563216920` | Waist Around | text | shared w/ Pant |
| `1206660686733050` | Belt - Leather | text |  |
| `1206660695392880` | Belt - Width | text |  |
| `1206660696647112` | Belt - Hardware | text |  |
| `1210023972860730` | Belt - Stitching | text |  |
| `1206660694454917` | Monogram | text | shared |
| `1206648966139310` | Note | text | shared |
| `1207446201916950` | Print PDF | text | shared |

### Shoe Pipeline Custom Fields

**Project:** Auto | Shoe Pipeline (GID: `1206648505149980`)

| GID | Field Name | Type | Notes |
| --- | --- | --- | --- |
| `1206649008312472` | New Customer | text | shared |
| `1112754700909040` | Gender | enum | shared |
| `1203386148481388` | Product Type | text | shared |
| `1206649285610127` | Order Type | text | shared |
| `1203340633343053` | Event (tags) | text | shared |
| `1206648951199301` | Quantity | text | shared |
| `1206649434581930` | Shoe | text |  |
| `1206657371264334` | Size | text |  |
| `1206649011400238` | Laces | text |  |
| `1206649336008535` | Swoosh | text |  |
| `1206649338146406` | Back Tab \| Eyelets | text |  |
| `1206649001881160` | Toe Cap \| Back Heel | text |  |
| `1206649000374109` | Toe Box \| Mid Panel | text |  |
| `1206648966139310` | Note | text | shared |

### Video Card Pipeline Custom Fields

**Project:** Auto | Video Card Pipeline (GID: `1206657933205969`)

| GID | Field Name | Type | Notes |
| --- | --- | --- | --- |
| `1203386148481388` | Product Type | text | shared |
| `1206649285610127` | Order Type | text | shared |
| `1203340633343053` | Event (tags) | text | shared |
| `1206648951199301` | Quantity | text | shared |
| `1206648966139310` | Note | text | shared |

---

*Field mappings on this page were transcribed from Zapier step configuration screenshots (Steps 21, 23, 25, 27) and verified against live zap run data from orders #114265 (Erin Test, March 31 2026) and #100002 (Alex Rivera, March 27 2026). Pant Pipeline Asana GIDs were extracted from the Step 21 Create Task response in the zap run logs. The Belt (`1206657932919233`), Shoe (`1206648505149980`), and Video Card (`1206657933205969`) project GIDs and their per-field custom-field GIDs — previously not captured in available run data — were filled in from a live Asana API schema capture (June 2026; team "PRODUCTION ORDER PIPELINE" `5333978630773`, workspace `2357765184667`). Note that some same-named fields are per-project duplicates with different GIDs (see the shared-vs-per-project callout above), which is why Skynet matches by exact display name.*