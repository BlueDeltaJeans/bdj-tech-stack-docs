# Blue Delta Jeans - Shopify Theme

This repository contains the Shopify theme for Blue Delta Jeans' e-commerce store. The theme is named "Web Rescue | Fall 2024" (Retina 4.7.3) and includes custom product templates, virtual tailor functionality, and various third-party integrations.

> **Start here:** [THEME_DOCUMENTATION.md](THEME_DOCUMENTATION.md) is the full engineering reference — theme identity, the three generations of code (stock Retina / GemPages / custom Vite+Tailwind+Shadow DOM), the CSS build pipeline, local dev, the git ↔ Shopify bidirectional sync model, the active-vs-dead file map, the order note-attributes contract, and a **before-you-edit checklist**. Read it before changing anything.

## Project Structure

The theme follows the standard Shopify theme structure:

- **assets/** - CSS, JavaScript, images, and other static files
- **config/** - Theme settings and configuration
- **layout/** - Theme layout templates
- **locales/** - Internationalization files
- **sections/** - Reusable content sections
- **snippets/** - Reusable code snippets
- **templates/** - Page templates

### Key Features

- **Custom Product Templates** - Specialized templates for different product types (jeans, hats, gift cards)
- **Virtual Tailor System** - Custom fitting functionality with related JS and templates
- **GemPages Integration** - Page builder templates and sections
- **Internationalization** - Support for multiple languages (English, Spanish, French, German, Portuguese, Chinese)
- **Third-party Integrations** - Bold, Klaviyo, SMS Bump, and other services

## Development

### Prerequisites

- [Shopify CLI](https://shopify.dev/docs/api/shopify-cli#installation) installed
- Access to the Blue Delta Jeans Shopify store

### Available Commands

```bash
# Start local development server
npm run local:start
```

The `local:start` command will:
1. Check if a theme exists on the remote Shopify environment with the name `blue-delta-jeans/{current-branch-name}`
2. If the theme doesn't exist, it will create a new unpublished theme with that name and upload it to Shopify
3. Start the local development server connected to that theme
4. Note that your changes are being synced live to the newely created theme on the Shopify remote environment

### Development Workflow

1. Create a new branch for your changes from the `main` branch
2. Run `npm run local:start` to create a new theme (if needed) and start development
3. Make your changes locally
4. Test your changes in the local preview environment
5. Push your changes to GitHub when ready
6. When you are ready to push to production merge your branch into the `main` branch

## Theme Customization

### Product Templates

The theme includes multiple specialized product templates:

- Standard product template
- Hat template
- Gift card templates
- Learfield product templates
- And many more

### Online vs POS products

This theme renders **website (online) products only**. Blue Delta keeps **two separate sets of products** in Shopify — online and POS — because Shopify caps every product at **100 variants** and a fully-personalized garment has far more combinations than that (a men's pant is 8 fabrics × 4 styles × 19 thread colors = 608). Online products stay under the cap because **SC Product Options** (`{% render 'sc-includes' %}` in `layout/theme.liquid`) moves thread color and monogram out of the variant matrix and into **line-item properties** captured at checkout. POS products can't use that app, so they encode those choices in the SKU instead — and they do **not** appear on the storefront this theme renders. For the full architecture (SKU formats, the 100-variant math, and how each channel captures customization) see the **Online vs POS Product Architecture** doc: [Website vs POS Products](../../04-website-vs-pos-products.md).

### Virtual Tailor

The Virtual Tailor system delivers a dynamic, multi-step measurement form by combining a Liquid snippet with dedicated JavaScript logic.

> **Note:** Bold Metrics has been **fully removed from the live storefront**. The **main VT path** call was removed via **PR #83** (merged 2026-06-19), and per the owner the GemPages **quick-tailor** page also **no longer fires** Bold Metrics on the live site (the hardcoded key was removed from the live page). The form now only **collects the customer's fit inputs** and attaches them to the order (via the `BDJ User Data` cart note attribute); body **measurements are computed post-order by the backend** ("Skynet") from those inputs.
>
> **Honest repo nuance:** a legacy GemPages quick-tailor snippet export committed on `main` still holds the old Bold Metrics endpoint and the hard-coded `user_key`. GemPages sections are app-managed, so these committed exports **lag** the live page (LIVE = removed; the committed exports are stale). They remain a **cleanup/rotation target** — do not treat the repo as fully key-free, and do not assume the live site still fires.
>
> **Remaining owner tasks** (handled by the BDJ owner directly): (1) **rotate** the Bold Metrics `user_key` vendor-side (not yet done); (2) **deactivate** the now-unused GemPages quick-tailor page; (3) **purge** the stale committed key copies in the legacy GemPages quick-tailor snippet exports.
>
> See [VIRTUAL_TAILOR_BOLDMETRICS.md](VIRTUAL_TAILOR_BOLDMETRICS.md) for the full data flow, file map, and removal details.

- **Snippet**: `snippets/virtual-tailor-3.liquid`
  - Renders a container `<div id="tailor-steps">` with sequential `<section data-step="1">` through `<section data-step="12">` blocks.
  - Defines size arrays at the top (`shoe_sizes`, `waist_sizes`, `bra_band_sizes`, `inseam_sizes`, `bra_cup_sizes`) and loops to generate radio buttons for each option.
  - Each step is a `<form class="tailor-step-form" data-validate="..."><fieldset>...` that ties into validation logic.

- **JavaScript**: `assets/VirtualTailor.js`
  - Contains a `VirtualTailor` class, instantiated on `DOMContentLoaded`.
  - **Event Setup** (`setupEventListeners()`): listens for form submits, radio/number inputs, custom height/shoe inputs, and back-button clicks.
  - **Validation**:
    - `validateForm()` uses HTML5 validity for form validation.
    - `validateInput()` handles per-field errors using built-in constraint validation.
    - `showError(fieldName, message)` centralizes error UI by targeting `div.tailor-error-message[data-error-for="fieldName"]` and setting `aria-invalid` on inputs.
  - **Navigation**:
    - `goToNextStep()` / `goToPreviousStep()` hide/show steps by `data-step`, update the progress bar (`.tailor-progress-bar`), and toggle the back button.
  - **Data Persistence**:
    - `saveFormData()` collects form values into `this.formData` and stores them under `localStorage.bdjFormData`.
    - `formComplete()` marks completion (`localStorage.tailor2Complete = "Yes"`), writes a human-readable inputs block to `localStorage.bdjUserData`, and submits a copy to Klaviyo. _(It no longer builds a Bold Metrics URL.)_

- **Cart hand-off**: on the cart page, `assets/bdj_vtailor2_boldmetrics-postAPI.js` copies the saved inputs/flags from `localStorage` into the hidden cart note-attribute fields in `sections/cart-new-template.liquid` (notably `BDJ User Data`, `Virtual Tailor`, `Jean Fit`, `Shoe Type`). These serialize onto the order's note attributes at checkout, where Zapier forwards them to Asana for the backend to process.

- **Usage**:
  - The modal is rendered site-wide from `layout/theme.liquid` via `{% render 'virtual-tailor-3' %}`.
  - Ensure the CSS (`assets/virtual-tailor-styles.css`) and JS (`assets/VirtualTailor.js`) are loaded (the snippet loads both).
  - To customize the steps or fields, edit `snippets/virtual-tailor-3.liquid`; the input model and Klaviyo/localStorage logic live in `assets/VirtualTailor.js`.

### GemPages Integration

The theme includes numerous GemPages templates and sections for enhanced page building capabilities.

## Creating new templates and components using Tailwind

Read the [SHADOW_DOM_COMPONENTS.md](SHADOW_DOM_COMPONENTS.md) document for information on how to create new components and templates using Tailwind and Vite, and attaching them using the shadow DOM so that they do not inherit any of the existing theme styles.

## Localization

The theme supports multiple languages:

- English (default)
- Spanish
- French
- German
- Portuguese (Brazil)
- Portuguese (Portugal)
- Chinese

## License

ISC 