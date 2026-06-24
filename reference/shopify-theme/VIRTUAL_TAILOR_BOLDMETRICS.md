# Virtual Tailor & Bold Metrics — Cart-Note Flow & Removal Map

> **Scope:** The active Virtual Tailor (VT) → cart → order-note chain, and exactly what to change to
> remove the Bold Metrics API call from the storefront (branch `bold-metrics-api-removal`). Companion
> to [THEME_DOCUMENTATION.md](THEME_DOCUMENTATION.md).
>
> **Status:** ✅ **Implemented** on branch `bold-metrics-api-removal` (this PR). The "edit lists" below
> describe the changes that were made. Line numbers are **pre-edit** references — use the grep anchors
> (function names, `attributes[...]` strings) as the durable handles, since numbers shift after editing.
>
> **Verified:** `grep "api.boldmetrics.io" | user_key | 'setItem("boldPostUrl"'` across `assets/`
> returns **0**; all 11 edited JS files pass `node --check`; the `BDJ User Data` input path is preserved.
>
> **Migration in one sentence:** Stop the cart from calling Bold Metrics and writing the 6 computed
> measurements into the order note; keep writing the raw VT **inputs** (`BDJ User Data`) so Zapier →
> Asana → Skynet can compute the measurements post-order.

---

## 0. The whole migration is 3 files

Everything else is either unchanged or **dead code your predecessor left behind** (§5). The only
ACTIVE files you edit:

| # | File | Change | Why |
|---|---|---|---|
| 1 | `sections/cart-new-template.liquid` | 🔴 Delete the 6 measurement `<textarea>` note-attribute fields (`:361–388`) | Removes the Bold Metrics outputs from `note_attributes` |
| 2 | `assets/bdj_vtailor2_boldmetrics-postAPI.js` | 🔴 Remove the Bold Metrics `$.ajax` + 6 measurement writes; ⚠️ **re-wire so the kept fields still populate** | Kills the cart-time API call |
| 3 | `assets/VirtualTailor.js` | 🔴 Delete `generateBoldMetricsUrl()` + its call + the `boldPostUrl` validity check | Stops building the BM URL & removes a hard-coded API key |

**Do NOT touch** (pure input-collection / transport, zero Bold Metrics surface):
`snippets/virtual-tailor-3.liquid`, `assets/virtual-tailor-styles.css`, `assets/pdp2024_sizing.js`,
`assets/app.js.liquid`. See §6.

---

## 1. Current vs. target flow

```
CURRENT (Bold Metrics fires at cart):
  VT modal (virtual-tailor-3 → VirtualTailor.js)
    └─ writes localStorage: bdjFormData, bdjUserData, tailor2Complete=Yes,  boldPostUrl ← BM URL+key
                                                                            ▲ 🔴 remove
  Cart (cart.liquid → cart-new-template.liquid)
    └─ bdj_vtailor2_boldmetrics-postAPI.js:  $.ajax(boldPostUrl) → BM API   🔴 remove
         └─ writes 6 measurements + inputs into hidden attributes[…] textareas
  Order placed → note_attributes = { Virtual Tailor, BDJ User Data, Jean Fit, Shoe Type,
                                      Hip Circum, Jean Inseam, Knee Circum, Thigh Circum,
                                      U Crotch, Waist Average }   ← 6 measurements 🔴

TARGET (inputs only; Skynet computes measurements post-order):
  VT modal → same inputs in localStorage (bdjUserData)            ✅ unchanged
  Cart     → poster writes ONLY inputs/flags into attributes[…]   ✅ no BM call
  Order placed → note_attributes = { Virtual Tailor, BDJ User Data, Jean Fit, Shoe Type,
                                      Pattern on File, White Glove, Pocket Initials }
  → Zapier Step 5 parses BDJ User Data → Asana VT* fields → Skynet fires Bold Metrics
```

### note_attributes contract (before → after)

| Order note attribute | Source | Migration |
|---|---|---|
| `Virtual Tailor` (`= "Yes"`) | `tailor2Complete` flag | ✅ keep |
| `BDJ User Data` (newline block of inputs) | `localStorage.bdjUserData` | ✅ **keep — this is the whole point** |
| `Jean Fit`, `Shoe Type` | VT inputs | ✅ keep |
| `Pattern on File`, `White Glove`, `Pocket Initials` | PDP flags | ✅ keep |
| `Hip Circum`, `Jean Inseam`, `Knee Circum`, `Thigh Circum`, `U Crotch`, `Waist Average` | **Bold Metrics output** | 🔴 **remove** |

The `BDJ User Data` block already contains every input the 5 new Asana `VT *` fields need
(`Height`, `Shoe Size`, `Waist`, `Inseam`, `Bra Size`) plus `Gender`/`Age`/`Weight`. The Shopify
side requires **no new fields** — only the removal of the 6 outputs.

---

## 2. Active end-to-end chain (proven)

1. **Modal rendered site-wide** — `layout/theme.liquid:363` `{% render 'virtual-tailor-3' %}` (also via the GemPages layouts).
2. **Modal loads engine + styles** — `snippets/virtual-tailor-3.liquid:2` loads `virtual-tailor-styles.css`; `:3` loads `VirtualTailor.js` (defer); `:41` loads Klaviyo (`company_id=SkKEvJ`).
3. **Form → localStorage** — `assets/VirtualTailor.js`:
   - every change → `localStorage.bdjFormData` (`:1634`, `:2052`)
   - on complete `formComplete()` (`:2560`): `tailor2Complete="Yes"` (`:2583`) · `bdjUserData` = `formatUserData()` block (`:2598–2599`) · Klaviyo POST (`:2594`→`:2749`) · 🔴 `generateBoldMetricsUrl()` (`:2602`) → `boldPostUrl` (`:2706`).
   - **VirtualTailor.js never calls Bold Metrics and never writes the cart** — it only builds the URL string. The cart fires it.
4. **PDP flags (Gem PDPs only)** — `assets/pdp2024_sizing.js` sets `patternOnFile`/`whiteGlove`/`tailor2Complete` from the measurement-option radios. No measurements, no `boldPostUrl`.
5. **Cart** — `templates/cart.liquid:5` → `sections/cart-new-template.liquid`:
   - hidden `<textarea name="attributes[…]">` block at `:341–412`
   - 🔴 poster enqueued at `:418` `bdj_vtailor2_boldmetrics-postAPI.js` (guarded `cart.item_count > 0`, `:417`)
6. **Poster fires BM** — `assets/bdj_vtailor2_boldmetrics-postAPI.js`:
   - `checkCartAndRun()` (`:171`) re-checks `/cart.js` is non-empty → `runBoldMetricsCartFlow()`
   - `retrievePostURL()` (`:18`) reads `bdjUserData`/`tailor2Complete`/`boldPostUrl`/`patternOnFile`/`whiteGlove`
   - if `tailor2Complete === "Yes"` (`:53`) → 🔴 `setAndSendData()` (`:82`) → `$.ajax(boldPostUrl)` (`:85–86`) → on success computes 6 values (`:93–99`) → `retrieveLocalData()` (`:103`) → `addDataToFields()` (`:125`) writes all hidden textareas via `$(id).val()`.
7. **Order placed** — Shopify serializes the `attributes[…]` textareas into `note_attributes` (native form POST; the visible cart note `#cart-note name="note"` at `:296` is the separate `cart.note`, unrelated).

---

## 3. ⚠️ THE #1 GOTCHA — read before editing the poster

`addDataToFields()` writes **both** the 6 measurements **and** the kept input fields
(`#bdj_user_data`, `#jean_fit`, `#shoe_type`, `#pocket_initials`, `#tailor2_complete`). But it is
reachable **only** through the Bold Metrics ajax success callback:

```
"Yes" branch (:54) → setAndSendData() (:82) → $.ajax success (:89) → retrieveLocalData() (:103) → addDataToFields() (:125)
```

`retrieveLocalData()` has exactly one caller (`:103`); `addDataToFields()` has exactly one caller
(`:121`). **So if you simply delete `setAndSendData()`, you also stop `#bdj_user_data` from being
populated — and the VT inputs never reach the order, defeating the migration.**

(This same coupling is the current data-loss bug the Skynet doc cites: when the BM ajax **errors**
(`:105`), `retrieveLocalData()` never runs, so the raw inputs are dropped from that order too.)

**Correct fix:** in the `tailor2Complete === "Yes"` branch, call `retrieveLocalData()` **directly**
(no ajax), and strip the 6 measurement keys out of `fieldMappings`. The inputs then populate
unconditionally — which also *fixes* the data-loss bug as a side effect.

---

## 4. The 3 surgical files — precise edit lists

### 4.1 `sections/cart-new-template.liquid` (ACTIVE cart)

The note attributes are hidden `<textarea name="attributes[X]">` inside `<form action="{{ routes.cart_url }}" method="post" id="cart_form">` (`:27`). Shopify serializes `name="attributes[X]"` → order note attribute `X` on submit. **Delete the element → the attribute never serializes.**

🔴 **DELETE — the 6 measurement fields (each in its own `<p class="cart-attribute__field">`), contiguous span `:361–388`:**

| line | attribute |
|---|---|
| `:363` | `attributes[Hip Circum]` |
| `:368` | `attributes[Jean Inseam]` |
| `:373` | `attributes[Knee Circum]` |
| `:378` | `attributes[Thigh Circum]` |
| `:382` | `attributes[U Crotch]` |
| `:387` | `attributes[Waist Average]` |

After deletion, `White Glove` (`:356–359`) is immediately followed by `Jean Fit` (`:389–391`). No kept field is interleaved.

✅ **KEEP (do not touch):**
- `:346` `attributes[Virtual Tailor]`, `:352` `attributes[Pattern on File]`, `:358` `attributes[White Glove]`
- `:391` `attributes[Jean Fit]`, `:395` `attributes[Shoe Type]`, `:403` `attributes[BDJ User Data]`, `:410` `attributes[Pocket Initials]`
- `:296` `#cart-note name="note"` (standard cart note), `:417–419` poster enqueue (**keep — it still populates the kept fields; the BM removal is inside the JS**), `:27`/`:318–319` form + submit.

### 4.2 `assets/bdj_vtailor2_boldmetrics-postAPI.js` (THE removal target)

🔴 **REMOVE:**
- `setAndSendData()` entirely (`:82–109`) — the `$.ajax({ url: boldPostUrl })` GET, the 6 dimension reads/derivation (`:93–99`), the `localStorage.vTailorDimensions` cache (`:102`).
- The `boldPostUrl` read (`:22`) and the 6 measurement entries in `fieldMappings` (`:135–140`) + their `let` decls (`:12–13`).

⚠️ **RE-WIRE (the §3 gotcha):** in `retrievePostURL()`, change the `"Yes"` branch (`:53–54`) from
`setAndSendData();` to `retrieveLocalData();` so the kept fields (`#bdj_user_data`, `#jean_fit`,
`#shoe_type`, `#pocket_initials`, `#tailor2_complete`) still populate without any API call. Keep
`retrieveLocalData()`/`addDataToFields()` minus the 6 measurement keys.

✅ **KEEP:** the cart-empty guard (`checkCartAndRun()` `:171`), the localStorage reads for
`bdjUserData`/`tailor2Complete`/`patternOnFile`/`whiteGlove`, and all non-measurement
`fieldMappings` (`#tailor2_complete`, `#bdj_user_data`, `#pocket_initials`, `#pattern_on_file`,
`#jean_fit`, `#shoe_type`).

> After this, consider renaming the file (it's no longer a "boldmetrics-postAPI") — but that means
> updating the enqueue at `cart-new-template.liquid:418`. Optional; not required for correctness.

### 4.3 `assets/VirtualTailor.js` (ACTIVE VT engine)

🔴 **REMOVE:**
1. `generateBoldMetricsUrl()` entirely — this holds the endpoint and a **hard-coded `user_key` secret** and writes `boldPostUrl`.
2. Its call in `formComplete()` (`:2602`).
3. The `boldPostUrl` `removeItem` in the gender-change reset (`:1554`) once the key is gone.

⚠️ **ALSO FIX — `checkMeasurementValidity()` (`:2799`)** reads `boldPostUrl` via `hasBoldUrl`
(`:2802–2804`) and shows "invalid measurements" when it's absent (`:2814–2821`, consumed at `:2901`).
Once you stop writing `boldPostUrl`, this would **falsely report invalid** and break the modal's
"measurements ready" UX. Remove the `hasBoldUrl` check so validity rests on `isComplete + hasUserData`.

✅ **KEEP (untouched):** `saveFormData()`/`formData` (`:1988–2052`), `formatUserData()` (`:2609`),
`getBraSize()` (`:2646`), the `bdjFormData`/`bdjUserData`/`tailor2Complete` writes, and
`submitToKlaviyo()` (`:2712`, list `VDqK3F`) — Klaviyo is independent of Bold Metrics.

---

## 5. Active-vs-OLD map (your predecessor left many backups — these are TRAPS)

🟡 = looks relevant, is **dead** — do not waste time editing, can be deleted in a separate cleanup.

| file | role | status | proof |
|---|---|---|---|
| `sections/cart-new-template.liquid` | **ACTIVE cart** | ✅ EDIT | `cart.liquid:5` renders it |
| `sections/cart-template.liquid` | old cart dup | 🟡 DEAD | not referenced by any template; **mirrors the active cart incl. the poster enqueue (`:307`) — the most dangerous trap** |
| `sections/cart-template-backup-apr-2024.liquid` | dated cart backup | 🟡 DEAD | unreferenced; enqueues old `boldmetrics-post.js` |
| `snippets/virtual-tailor-3.liquid` | **ACTIVE modal** | ✅ KEEP | `theme.liquid:363` |
| `snippets/virtual-tailor-2.liquid` | old modal | 🟡 DEAD | zero `render` callers; loads `bdj_vtailor2.js` + `bdj_vtailor2_styles.css` |
| `assets/VirtualTailor.js` | **ACTIVE engine** | ✅ EDIT | loaded by `virtual-tailor-3.liquid:3` |
| `assets/bdj_vtailor2_boldmetrics-postAPI.js` | **ACTIVE poster** | ✅ EDIT | enqueued by `cart-new-template.liquid:418` |
| `assets/bdj_vtailor2.js` | old engine | 🟡 DEAD | only via orphan `virtual-tailor-2.liquid`; writes `boldPostUrl:864` |
| `assets/boldmetrics-post.js` | old poster | 🟡 DEAD | only via dead `cart-template-backup-apr-2024.liquid` |
| `assets/v-tailor-setandsend-prodpage.js` | old PDP poster | 🟡 ORPHAN | no `.liquid` ref; fires ajax `:99` |
| `assets/v-tailor-store.js` / `…-klaviyo.js` / `…-archived-ga-tracking..js` | old store-page engines | 🟡 DEAD | only on `page.v-tailor-*` / `page.event-tailor` utility pages |
| `assets/vtailor-clean.js` | old | 🟡 DEAD | only `page.vtailor-clean.liquid` |
| `assets/quick-tailor.js` / `quick-tailor-steel-tmp.js` | niche co-brand PDP tailor | 🟡 OUT OF SCOPE | Learfield / `product.steel-quick` pages — separate flow, build `boldPostUrl` of their own |
| `assets/gcf-sizr-app.js` / `backend-app.js` | sizr / staff backend | 🟡 DEAD | only `page.vtailor.liquid` / `page.tailor-backend.liquid` |
| `assets/pdp2024_sizing.js` | **ACTIVE Gem-PDP flags** | ✅ KEEP | via `snippets/pdp-sizing.liquid` + `gp-section-*` |
| `assets/v-tailor-product-detector.js` | Gem PDP detector | ✅ KEEP (niche) | `product.gem-*-template.liquid`; ⚠️ verify it posts no BM |
| `assets/virtual-tailor-styles.css` | **ACTIVE modal CSS** | ✅ KEEP | `virtual-tailor-3.liquid:2` |
| `assets/bdj_vtailor2_styles.css` | old modal CSS | 🟡 DEAD | only orphan `virtual-tailor-2.liquid` |

> ⚠️ **Not Bold Metrics:** `snippets/bold-common.liquid`, `snippets/bold-loyalties-widget.liquid`,
> `assets/bold-options.css` are the **Bold Commerce** product-options/loyalty app — a *different*
> vendor. Leave them alone.
>
> 🔴 **Hard-coded secret:** a Bold Metrics `user_key` is duplicated across the (then-active) VT
> engine and several dead engine files. The key should be rotated on Bold Metrics' side regardless
> of this removal (already flagged as a background task). Deleting the dead engines is the easiest
> way to purge the redundant copies.

---

## 6. Files that are NOT migration touchpoints (proving the negative)

- **`snippets/virtual-tailor-3.liquid`** — pure form UI. `grep boldmetrics|boldPostUrl|hip_circum|…` → no matches. No hidden `attributes[…]` fields (those live only in the cart section). ✅ no edits.
- **`assets/virtual-tailor-styles.css`** — cosmetic only; one `font-weight:bold` is the only "bold". ✅ no edits.
- **`assets/pdp2024_sizing.js`** — reads `patternOnFile`/`tailor2Complete`/`bdjUserData`/`whiteGlove`; writes only those flags; keys VT-completion off **`bdjUserData`** (a kept input), so unaffected. No measurements, no `boldPostUrl`. ✅ no edits.
- **`assets/app.js.liquid`** (3085 lines) — the vendored jQuery theme bundle. `grep boldmetric|boldPostUrl|VirtualTailor|attributes[` → none. It IS the generic transport: `ajaxSubmitCart()` (`:2231`) does `$.ajax('/cart/update.js', data:$cart.serialize())`, which sweeps the `attributes[…]` textareas — but it's content-agnostic. Once the 6 textareas are gone, it simply has nothing Bold-Metrics to serialize. ✅ no edits.

---

## 7. localStorage contract

| key | written by (active) | read by (active) | migration |
|---|---|---|---|
| `bdjFormData` | `VirtualTailor.js:1634,2052` | `VirtualTailor.js`; poster `:114` | ✅ keep (inputs) |
| `bdjUserData` | `VirtualTailor.js:2599` | poster `:20,115,130` → `#bdj_user_data` | ✅ keep — becomes `BDJ User Data` |
| `tailor2Complete` | `VirtualTailor.js:2583`; `pdp2024_sizing.js:22` | poster `:21,53` (gate) | ✅ keep |
| `patternOnFile` / `whiteGlove` | `pdp2024_sizing.js` | poster → `#pattern_on_file`/`#white_glove` | ✅ keep |
| `pocketInitials` | PDP initials UI | poster `:117` → `#pocket_initials` | ✅ keep |
| 🔴 `boldPostUrl` | `VirtualTailor.js:2706` (URL + key) | poster `:22,86` (`$.ajax`) | 🔴 remove writer + reader |
| 🔴 `vTailorDimensions` | poster `:102` (BM response) | poster `:116` | 🔴 remove (BM-only) |
| `jeanStyle` | only OLD engines | only OLD engines | 🟡 not on active path |

---

## 8. ⚠️ Cross-team findings (hand these to the Skynet / Zapier side)

These come straight from the active cart code and resolve open questions in the Skynet migration doc:

1. **`Waist Average` is NOT `waist_circum_preferred`.** The cart computes it as a **3-way mean**,
   then rounds to 2 decimals (`bdj_vtailor2_boldmetrics-postAPI.js:99`):
   ```js
   waist_average = round( (waist_circum_natural + waist_circum_preferred + waist_circum_stomach) / 3 , 2 )
   ```
   Skynet must replicate this exact mean when it recomputes, or historical and new orders won't match.
   *(This corrects Skynet doc §9 / open question #1, which assumed `waist_circum_preferred`.)*
2. **`Thigh Circum` = `thigh_circum_proximal`** (`:96`) — confirmed (Skynet doc guessed this correctly).
3. The other four are direct: `Hip Circum`←`hip_circum`, `Jean Inseam`←`jean_inseam`,
   `Knee Circum`←`knee_circum`, `U Crotch`←`u_crotch` (`:93–97`).
4. **All 5 new Asana `VT *` inputs are already present** in the `BDJ User Data` block emitted by
   `VirtualTailor.formatUserData()` (`:2609–2644`): `Height` (total inches, `:2625`),
   `Shoe Size` (`:2627`), `Waist` (men, `:2631`), `Inseam` (men, `:2632`), `Bra Size`
   (women, band+cup, `:2636`). Gender split is enforced in the form: **men never emit Bra Size;
   women's block omits Waist/Inseam/Common Shoe.** So the only remaining upstream task is the
   **Zapier mapping** of those `BDJ User Data` keys into the `VT *` Asana fields.
5. **Current data-loss bug** (relevant to "why we're migrating"): the inputs (`#bdj_user_data`) are
   written **only on a successful Bold Metrics ajax** (§3). A failed/timed-out BM call drops the raw
   inputs from that order entirely. The re-wire in §4.2 fixes this — inputs will persist
   unconditionally after the migration.

---

## 9. Removal checklist (ordered)

1. `cart-new-template.liquid` — delete the 6 measurement `<textarea>` blocks (`:361–388`).
2. `bdj_vtailor2_boldmetrics-postAPI.js` — delete `setAndSendData()`; change the `"Yes"` branch
   (`:54`) to call `retrieveLocalData()`; drop the 6 measurement `fieldMappings` keys + `boldPostUrl`
   read. **(Verify `#bdj_user_data` still populates — §3.)**
3. `VirtualTailor.js` — delete `generateBoldMetricsUrl()` (`:2659–2707`) + its call (`:2602`); remove
   the `hasBoldUrl` check in `checkMeasurementValidity()` (`:2802–2821`).
4. (Optional cleanup, separate PR) delete the 🟡 dead files in §5 — this also purges the redundant
   copies of the hard-coded `user_key` left in the dead engines.
5. Rotate the Bold Metrics `user_key` on the vendor side (background task already filed).

## 10. Verification / test plan

- **Place a test order with a completed VT** (men + women): confirm `note_attributes` contains
  `BDJ User Data`, `Virtual Tailor: Yes`, `Jean Fit`, `Shoe Type` and **no** `Hip Circum` /
  `Jean Inseam` / `Knee Circum` / `Thigh Circum` / `U Crotch` / `Waist Average`.
- **DevTools → Network**: confirm **no request to `api.boldmetrics.io`** on cart load.
- **DevTools → Application → Local Storage**: `boldPostUrl` should no longer be written on form
  complete; `bdjUserData` should still be present.
- **Grep the shipped assets** for `api.boldmetrics.io` / `boldPostUrl` and confirm only dead files
  remain (then delete those).
- **Zapier Step 5 dry-run**: confirm `BDJUserData{}` still parses (Gender/Age/Height/Weight/Shoe
  Size/Waist|Bra Size/Inseam/Jean Fit/Common Shoe/Style) and `Extras{}` no longer carries the 6
  measurements.
- **Modal UX**: complete the form and confirm the "measurements ready" state still shows (proves the
  `checkMeasurementValidity()` fix landed).

---

*Verified against the working tree on branch `bold-metrics-api-removal`, 2026-06-18, via a 6-agent
forensic pass plus direct grep confirmation of every line reference in §4 and §8. External contract
sourced from the Zapier "Orders – PRODUCTS" (Zap 281794942) Step-5 docs and the Skynet
`bold-metrics-skynet-migration-context.md`.*
