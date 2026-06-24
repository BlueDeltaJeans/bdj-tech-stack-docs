# Shadow DOM Components with Tailwind CSS and Vite

This document explains how to create isolated page components using Shadow DOM, Tailwind CSS for styling, and Vite for compilation. This setup allows you to build components that don't inherit the theme's stylesheet, providing complete style isolation.

## Overview

Our setup combines:
- **Shadow DOM**: For style and DOM isolation
- **Tailwind CSS v4**: For utility-first styling
- **Vite**: For fast CSS compilation and hot reloading
- **Custom Elements**: For reusable component architecture

## Architecture

### 1. Build System (Vite + Tailwind)

The build system is configured in `vite.config.js`:

```javascript
import { defineConfig } from "vite";
import tailwindCss from "@tailwindcss/vite";
import path from "path";

export default defineConfig({
  plugins: [
    tailwindCss(), // Official Tailwind v4 plugin for Vite
  ],
  build: {
    rollupOptions: {
      input: {
        tailwind: path.resolve(__dirname, "tailwind-input.css"),
      },
      output: {
        assetFileNames: "[name][extname]", // CSS → tailwind.css
      },
    },
    outDir: "assets", // Output directly to assets folder
    emptyOutDir: false, // Preserve other files
  },
});
```

**Key Features:**
- Compiles `tailwind-input.css` → `assets/tailwind.css`
- Uses Tailwind v4 with custom theme configuration
- Preserves existing assets during builds

### 2. Tailwind Configuration

The `tailwind-input.css` file contains:

```css
@import "tailwindcss";

@theme {
  --color-gray-50: #F6F6F6;
  --color-gray-200: #F4F4F4;
  --color-gray-300: #e6e6e6;
  --color-gray-400: #D0D0D0;
  --color-gray-500: #cccccc;
  --color-gray-550: #6A6A6A;
  --color-gray-700: #444444;
  --color-red-700: #af1f31;
  --color-navy: #103E53;
  --color-blue-500: #70909E;
  --color-blue-700: #00465e;
  --color-blue-800: #023C50;
  --color-blue-850: #0C455E;
  --color-green-500: #46833A;
  --color-green-600: #008A00;
  
  --shadow-box: 0px 2px 2px 0px #0000001A;
  --container-container: 1352px;
  --container-container-secondary: 1252px;
  --font-inter: "Inter", sans-serif;
  --font-albiona: 'albiona',sans-serif;
}
```

**Custom Theme Features:**
- Brand-specific color palette
- Custom container sizes
- Typography configuration
- Component-specific utilities

## Component Structure

### 1. Shadow DOM Component Pattern

Here's the complete pattern used in `snippets/navigation.liquid`:

```html
<!-- 1. Template Definition -->
<template id="nav-component-template">
  {{ 'tailwind.css' | asset_url | stylesheet_tag }}
  
  <div class="relative z-[1000]">
    <!-- Component HTML with Tailwind classes -->
    <nav class="bg-gray-300 w-full fixed top-0 z-50">
      <!-- Navigation content -->
    </nav>
  </div>
</template>

<!-- 2. Custom Element Usage -->
<nav-component></nav-component>

<!-- 3. JavaScript Implementation -->
<script>
class NavComponent extends HTMLElement {
  constructor() {
    super();
    
    // Create shadow root
    const shadow = this.attachShadow({ mode: 'open' });
    
    // Clone template content
    const tpl = document.getElementById('nav-component-template');
    shadow.appendChild(tpl.content.cloneNode(true));
    
    // Wire up event listeners
    this.setupEventListeners(shadow);
  }
  
  setupEventListeners(shadow) {
    // Use shadow.querySelector() for internal elements
    const btn = shadow.querySelector('.mobile-menu-button');
    const menu = shadow.querySelector('.mobile-menu');
    
    btn.addEventListener('click', (e) => {
      // Component logic here
    });
  }
}

// Register the custom element
customElements.define('nav-component', NavComponent);
</script>
```

### 2. Key Implementation Details

#### Shadow DOM Creation
```javascript
const shadow = this.attachShadow({ mode: 'open' });
```
- `mode: 'open'` allows external JavaScript to access the shadow root
- Creates complete style isolation from parent document

#### Template Cloning
```javascript
const tpl = document.getElementById('nav-component-template');
shadow.appendChild(tpl.content.cloneNode(true));
```
- Templates are defined in HTML for better maintainability
- Content is cloned to avoid moving the original template

#### Event Handling
```javascript
// Use shadow.querySelector() for internal elements
const btn = shadow.querySelector('.mobile-menu-button');

// Handle events that need to interact with main document
window.addEventListener('click', (e) => {
  const path = e.composedPath(); // Important for shadow DOM
  const clickedInside = path.includes(menu) || path.includes(btn);
});
```

## Development Workflow

### 1. Build Commands

```bash
# Build CSS once
npm run build:css

# Watch for changes and rebuild
npm run watch:css
```

### 2. Component Development Process

1. **Create Template**: Define HTML structure with Tailwind classes
2. **Include Stylesheet**: Add `{{ 'tailwind.css' | asset_url | stylesheet_tag }}` to template
3. **Implement JavaScript**: Create custom element class with shadow DOM
4. **Register Component**: Use `customElements.define()`
5. **Test Isolation**: Verify styles don't leak in/out

### 3. Style Isolation Benefits

**Before (Regular DOM):**
```html
<!-- Theme styles affect component -->
<div class="my-component">
  <button class="btn">Click me</button> <!-- Inherits theme button styles -->
</div>
```

**After (Shadow DOM):**
```html
<!-- Complete isolation -->
<my-component>
  #shadow-root
    <style>/* Only Tailwind styles here */</style>
    <button class="bg-blue-700 text-white px-4 py-2">Click me</button>
</my-component>
```

## Template-Level Usage

For full pages (like `templates/customers/account.liquid`), you can use Tailwind directly without shadow DOM:

```liquid
{%- comment -%} Include required assets {%- endcomment -%}
{{ 'tailwind.css' | asset_url | stylesheet_tag }}

<section class="max-w-container-secondary mx-auto px-4 mb-20">
  <h2 class="lg:pt-36 md:pt-16 pt-9 lg:pb-10 pb-6 font-medium md:text-3xl text-2xl">
    {{ 'customer.account.title' | t }}
  </h2>
  
  <div class="w-full max-w-[1440px] h-auto relative flex lg:flex-row flex-col">
    <!-- Page content with Tailwind classes -->
  </div>
</section>
```

## Best Practices

### 1. Component Design

- **Single Responsibility**: Each component should have one clear purpose
- **Reusability**: Design components to work across different contexts
- **Accessibility**: Include proper ARIA attributes and keyboard navigation

### 2. Style Management

- **Use Tailwind Classes**: Prefer utility classes over custom CSS
- **Custom Properties**: Define brand colors in `tailwind-input.css`
- **Responsive Design**: Use Tailwind's responsive prefixes (`sm:`, `md:`, `lg:`)

### 3. Event Handling

- **Shadow DOM Events**: Use `shadow.querySelector()` for internal elements
- **Global Events**: Use `window` or `document` for events that need to interact with main document
- **Event Delegation**: Consider using event delegation for dynamic content

### 4. Performance

- **Lazy Loading**: Load components only when needed
- **Event Cleanup**: Remove event listeners when components are destroyed
- **Memory Management**: Avoid memory leaks in long-running applications

## Troubleshooting

### 1. Styles Not Applying
- Ensure `tailwind.css` is included in the template
- Check that Vite build completed successfully
- Verify Tailwind classes are correct

### 2. Events Not Working
- Use `shadow.querySelector()` for elements inside shadow DOM
- Check `e.composedPath()` for cross-boundary event handling
- Ensure event listeners are attached after DOM is ready

### 3. Build Issues
- Run `npm run build:css` to compile Tailwind
- Check `vite.config.js` for correct input/output paths
- Verify all dependencies are installed

## Examples in Codebase

### Navigation Component (`snippets/navigation.liquid`)
- Complete shadow DOM implementation
- Mobile menu with accordion functionality
- Integration with existing theme systems

### Account Page (`templates/customers/account.liquid`)
- Template-level Tailwind usage
- Responsive design patterns
- Complex layout with sidebar and content areas
