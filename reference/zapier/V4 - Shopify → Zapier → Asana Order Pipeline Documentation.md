# V4 - Shopify → Zapier → Asana: Order Pipeline Documentation

> **System:** Shopify → Zapier → Asana Order Pipeline
**Zap:** "Orders - PRODUCTS: Shopify > Code > Looping > Code > Paths > Asana"
**Version:** v51 (live)
**Zap ID:** 281794942
**Maintained by:** Caleb (Marketing & Tech)
**Last updated:** June 2026
>
> ✅ **Status (June 2026): up to date.** The zap is live at **v51**. The V4 audit bug-fixes (see the Bug Status Summary below) are **deployed**, and the **Bold Metrics → Skynet migration is complete** — the 5 new `VT *` Asana input fields are mapped and populated on every order, and Skynet computes the 6 measurement fields server-side, post-order. Treat the "v42 / pending as v43" version notes and the "ready to deploy" statuses below as **historical context**. See [Bold Metrics Migration — Impact on the Zapier Pipeline](Bold%20Metrics%20Migration%20—%20Impact%20on%20the%20Zapier%20Pipeline.md).
> 

---

## Pages

[Architecture Overview](./Architecture%20Overview.md)

[Online vs POS Product Architecture](./Online%20vs%20POS%20Product%20Architecture.md)

[Step 5 — Order Parser](./Step%205%20%E2%80%94%20Order%20Parser.md)

[Step 13 — Line Item Processor](./Step%2013%20%E2%80%94%20Line%20Item%20Processor.md)

[SKU Reference Guide](./SKU%20Reference%20Guide.md)

[Asana Field Mapping (Steps 21–27)](./Asana%20Field%20Mapping%20(Steps%2021%E2%80%9327).md)

[Bug Tracker & Fix Plan](./Bug%20Tracker%20&%20Fix%20Plan.md)

## Key Resources & Files

[zap-export.json](./zap-export%20json.md)

[step-5.js](./step-5%20js.md)

[step-13.js](./step-13%20js.md)

[Zapier > Asana Field Mapping (Step 21-27)](./Asana%20Field%20Mapping%20(Steps%2021%E2%80%9327).md)

[Auto | Pant Pipeline](../../appendix/notion-source-context.md) 

[Auto | Belt Pipeline](../../appendix/notion-source-context.md) 

[Auto | Shoe Pipeline](../../appendix/notion-source-context.md) 

[Auto | Video Card Pipeline](../../appendix/notion-source-context.md) 

[BDJ Product Data](../../appendix/notion-source-context.md) 

## Archive

V3 - Order Pipeline Documentation (archived Notion page — not included in this bundle)

V2 - Order Pipeline Documentation (archived Notion page — not included in this bundle)

V1 - Order Pipeline Documentation (archived Notion page — not included in this bundle)

## System Overview

Every order placed on the Blue Delta Jeans Shopify store — whether through the website or the POS app at events — triggers a Zapier automation that does three things:

**1. Extract** — The automation re-fetches the complete order from the Shopify Admin API (because Zapier's native trigger data is incomplete), then parses it into structured components: customer info, order tags, note attributes (Virtual Tailor measurements), and a filtered list of production-qualifying line items.

**2. Transform** — Each qualifying line item is processed individually through a loop. A JavaScript code step parses the SKU to determine product type (Pant, Belt, Shoe, or Video Card) and extracts all product-specific fields: fabric, style, thread color, monogram, hardware, sizing, and more. The logic differs for Online vs. POS orders because customization data comes from different sources depending on the channel.

**3. Load** — Based on the product type, the line item is routed to the correct Asana project where a task is created with up to 49 mapped custom fields. Production staff use these Asana tasks as their work orders.

---

## Bug Status Summary

| # | Bug | Severity | Status |
| --- | --- | --- | --- |
| BUG-01 | Identical products with qty > 1 only create 1 Asana task | **CRITICAL** | Fixed in `step-5-v4.js` — ready to deploy |
| BUG-02 | POS belt monogram (`MONO` in SKU) not detected | **CRITICAL** | Fixed in `step-13-v4.js` — ready to deploy |
| BUG-03 | POS belt thread misread for `NO_THREAD` and `MONO`-only SKUs | **HIGH** | Fixed in `step-13-v4.js` — ready to deploy |
| NEW-01 | Shoe SKU format changed, old parser reads wrong segments | **HIGH** | Fixed in `step-13-v4.js` — ready to deploy |
| NEW-04 | Belt POS detection relies on fragile `Source` field instead of SKU structure | **HIGH** | Fixed in `step-13-v4.js` — ready to deploy |
| BUG-05 | `CustomerOrdersCount` always 0 — "New Customer" always shows "Yes" | **MEDIUM** | Fixed in `step-5-v4.js` — ready to deploy |
| NEW-03 | POS belt hardware `FLATBLACK` not normalized to `FLAT_BLACK` | **MEDIUM** | Fixed in `step-13-v4.js` — ready to deploy |
| BUG-06 | Carriage return `\r` residue in BDJ User Data values | **LOW** | Fixed in `step-5-v4.js` — ready to deploy |
| BUG-07 | Tag matching is case-sensitive (could miscategorize tags) | **LOW** | Fixed in `step-13-v4.js` — ready to deploy |
| NEW-02 | Ryder dead code still in Step 13 (products deleted from Shopify) | **LOW** | Fixed in `step-13-v4.js` — ready to deploy |
| BUG-04 | Ryder pant compound Thread/Logo value | **N/A** | Removed — Ryder products deleted from Shopify |

## V3 → V4 Changelog

### Shopify Changes (Done by Caleb, March–April 2026)

These changes were made directly in the Shopify admin before the V4 code audit:

- **7,000+ SKUs audited and corrected** — Fixed width mismatches on 5 online belt variants, corrected "ORNAGE" typo on 3 POS belt variants, fixed "CCC04" triple-C typo on 1 POS pant variant, corrected $200 video gift card denomination SKU from `VID-GIFT-PERSONAL-100` to `VID-GIFT-PERSONAL-200`
- **Ryder products removed** — 2025 Ryder Cup Edition Jean and Belt deleted from Shopify (0 SKUs remaining)
- **Shoe SKUs updated** — New format `SHOE-CLASSIC-M-10` encodes gender and size directly in the SKU. Previously was `SHOE-CLASSIC` with gender/size parsed from the Shopify variant title
- **POS belt "NO THREAD" standardized** — Variants where the customer selects no thread stitching now use explicit `NO_THREAD` segment in the SKU (e.g., `CB6-1-BRASS-NO_THREAD`) instead of omitting the thread segment entirely
- **New product types identified** — Shorts (570 SKUs, format `M-PF03_LIGHTTAN-SHORT_5-TONAL`) and Kentucky Derby Pants (48 SKUs, format `M-PFDERBY_HOTPINK-STRAIGHT-TONAL`) were documented for the first time

### Code Audit Findings (V4)

The V4 code audit analyzed `step-5.js` (97 lines) and `step-13.js` (147 lines) against all 5,555 current SKUs and the 4 Asana field mapping transcriptions. It produced:

- **11 bugs documented** — 2 critical, 3 high, 2 medium, 4 low/removed
- **`step-5-v4.js`** — Complete fixed version with 3 bug fixes (quantity expansion, customer count fallback, carriage return stripping)
- **`step-13-v4.js`** — Complete fixed version with 8 bug fixes (belt rewrite with segment-count detection, shoe SKU parser rewrite, hardware normalization, Ryder dead code removal, case-insensitive tag matching)
- **22 test cases** with 71 assertions covering every bug fix and edge case
- **`CODEFIXES.md`** — Deployment-ready implementation checklist with before/after code for each fix

### Documentation Created (V4)

- Architecture Overview with accurate 27-step flow diagram
- Step 5 and Step 13 annotated code walkthroughs
- SKU Reference Guide covering all 5,555 SKUs with format breakdowns per category
- Asana Field Mapping for all 4 pipelines (Steps 21, 23, 25, 27) from manual transcriptions
- Bug Tracker with full root cause analysis and fix code
- Online vs POS Product Architecture explaining the dual-product-set system

---

## Key Resources & Files

### Code Files (Current — Live in Zapier)

| File | Description | Location |
| --- | --- | --- |
| `step-5.js` | Order Parser — currently running in Zapier Step 5 | Zapier Pipeline > Resources |
| `step-13.js` | Line Item Processor — currently running in Zapier Step 13 | Zapier Pipeline > Resources |
| `zap-export.json` | Full Zapier zap export (27 steps, v42) | Zapier Pipeline > Resources |

### Code Files (V4 Fixed — Ready to Deploy)

| File | Description | Location |
| --- | --- | --- |
| `step-5-v4.js` | Fixed Order Parser (3 bug fixes) | Project files |
| `step-13-v4.js` | Fixed Line Item Processor (8 bug fixes) | Project files |
| `CODEFIXES.md` | Deployment checklist with before/after code and test plan | Project files |
| `v4-bug-report.md` | Detailed bug analysis with affected SKU counts | Project files |
| `v4-test-cases.md` | 22 test cases for verification | Project files |

### Field Mapping Data

| File | Description |
| --- | --- |
| `step-21-pant-path-field-mapping.md` | Pant Pipeline — 49 Asana field mappings |
| `step-23-belt-path-field-mapping.md` | Belt Pipeline — 17 Asana field mappings |
| `step-25-shoe-path-field-mapping.md` | Shoe Pipeline — 15 Asana field mappings |
| `step-27-video-card-path-field-mapping.md` | Video Card Pipeline — 7 Asana field mappings |