# Design System Document: The Ethereal Editorial

## 1. Overview & Creative North Star: "Luminous Depth"
This design system is built to transcend the rigid, boxy constraints of traditional SaaS interfaces. Our Creative North Star is **Luminous Depth**—an approach that treats the screen not as a flat canvas, but as a layered, ambient environment. 

We break the "template" look by favoring intentional white space, soft-focus glassmorphism, and a typographic hierarchy that feels more like a high-end fashion editorial than a utility app. By utilizing asymmetrical layouts and overlapping elements, we invite the user into an experience that feels fluid, organic, and premium.

---

## 2. Colors & Atmospheric Gradients
The palette is rooted in a refreshing base of `surface` (#e2ffff) and `primary` teal (#006b64). It is designed to feel "breathable" and expensive.

### The "No-Line" Rule
**Strict Mandate:** Designers are prohibited from using 1px solid borders to define sections. Layout boundaries must be established exclusively through background color shifts. Use `surface-container-low` (#d4fbfb) for secondary sections sitting on a `surface` background. The contrast is felt, not seen.

### Surface Hierarchy & Nesting
Treat the UI as a series of stacked, semi-transparent sheets. 
*   **Base:** `surface` (#e2ffff)
*   **Secondary Sections:** `surface-container-low` (#d4fbfb)
*   **Floating Cards:** `surface-container-lowest` (#ffffff)
*   **Overlays:** Use `surface_variant` (#b0eeee) at 60% opacity with a `24px` backdrop blur to create a "frosted glass" effect.

### The "Glass & Gradient" Rule
Flat colors are for utilities; gradients are for "soul." 
*   **Hero/Primary Gradient:** `primary` (#006b64) to `secondary` (#006595).
*   **Aesthetic Accents:** 
    *   *Mint to Azure:* `primary_fixed` (#73f1e4) → `secondary_fixed_dim` (#aed9ff)
    *   *Deep Teal to Emerald:* `primary_dim` (#005e57) → `on_primary_container` (#005852)
    *   *Blush to Lavender:* (Custom Accent) Use `tertiary_fixed` (#c9aaff) transitioning into a soft peach.

---

## 3. Typography: Editorial Authority
We use a dual-font strategy to balance character with legibility.

*   **Display & Headlines (Manrope):** The "voice" of the brand. We utilize `display-lg` (3.5rem) and `headline-lg` (2rem) with tight letter-spacing (-0.02em) to create a bold, authoritative presence. Large headers should often be center-aligned or placed asymmetrically to break the grid.
*   **Body & Labels (Inter / Manrope):** `body-lg` (Manrope, 1rem) provides a modern, clean reading experience. For utility-heavy areas, `label-md` (Inter, 0.75rem) offers the precision required for high-density data without sacrificing the "high-end" feel.

---

## 4. Elevation & Depth: Tonal Layering
Traditional drop shadows are too "heavy" for this system. We convey hierarchy through light and tone.

*   **The Layering Principle:** To lift an element, move it to a lighter surface tier. A `surface-container-lowest` (#ffffff) card sitting on a `surface-container` (#c8f7f7) creates a natural, soft lift.
*   **Ambient Shadows:** For floating modals or menus, use an ultra-diffused shadow: `box-shadow: 0 20px 40px rgba(0, 57, 58, 0.06)`. Note the use of `on_surface` (#00393a) as the shadow tint rather than pure black; this mimics natural, cool-toned light.
*   **The "Ghost Border":** If a border is required for accessibility, use `outline_variant` (#80bcbd) at **15% opacity**. It should be a suggestion of a line, not a boundary.

---

## 5. Components

### Buttons
*   **Primary:** A vibrant gradient from `primary` (#006b64) to `primary_dim` (#005e57). Use `xl` (1.5rem) or `full` (9999px) roundedness to emphasize the "soft" modern aesthetic.
*   **Tertiary/Ghost:** No container. Use `on_surface` text with a subtle `primary_container` glow on hover.

### Cards
*   **Styling:** Forbidden use of divider lines. Separate content using `40px` (or `xl` spacing) vertical gaps.
*   **Composition:** Images within cards should use `xl` (1.5rem) corner radius. Elements should slightly overlap the card boundaries in "Editorial Mode" to create visual interest.

### Input Fields
*   **Style:** Minimalist. Use `surface_container_highest` (#b0eeee) as a subtle background fill with a `none` border. On focus, transition the background to `surface_container_lowest` (#ffffff) with a 1px `primary` ghost border.

### Interactive Glass Chips
*   **Selection:** Use `secondary_container` (#cbe6ff) with 40% transparency and a backdrop-blur. This allows the sophisticated background gradients to peek through the UI controls.

---

## 6. Do's and Don'ts

### Do
*   **Do** use asymmetrical margins (e.g., 80px left, 120px right) for hero sections to create a "designed" feel.
*   **Do** use `on_surface_variant` (#2a6869) for secondary text to maintain a soft, low-contrast elegance.
*   **Do** leverage the "Blush to Lavender" (`tertiary` tokens) for high-intent notifications or "soft" alerts.

### Don't
*   **Don't** use 100% black (#000000) for text. Use `on_surface` (#00393a).
*   **Don't** use "Drop Shadows" from a standard library. Every shadow must be tinted and highly diffused.
*   **Don't** use "Dividers" (`<hr>`). Use a `24px` height jump or a subtle shift from `surface` to `surface-container-low`.
*   **Don't** use sharp corners. Our world is rounded (`md` 0.75rem minimum for small elements, `xl` 1.5rem for containers).