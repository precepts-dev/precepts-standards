---
identifier: "UX-BP-001"
name: "Dark Mode and Color Palette Design"
version: "1.0.0"
status: "RECOMMENDED"

domain: "UX"
documentType: "best-practice"
category: "design-system"
appliesTo: ["web", "ios", "android", "design-tokens"]

lastUpdated: "2026-06-16"
owner: "UX Design Authority"

standardsCompliance:
  iso: ["ISO-9241-307:2008", "ISO-9241-391:2016"]
  rfc: []
  w3c: ["WCAG-2.2-SC-1.4.3", "WCAG-2.2-SC-1.4.11", "WCAG-3.0-APCA-Draft"]
  other: ["Material-Design-3-Color-System", "Apple-HIG-Color", "W3C-Design-Tokens-Format-2023"]

taxonomy:
  capability: "design-system"
  subCapability: "color-tokens"
  layer: "visual"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Verify dark mode is implemented using responsive prefers-color-scheme media queries or data attributes"
    - "Ensure dark mode backgrounds use off-blacks (e.g. HSL neutral-dark shades) rather than pure #000000"
    - "Confirm accent colors are adjusted (slightly desaturated/lightened) to remain legible in dark mode"
    - "Verify color values are defined using semantic variables (design tokens) rather than hardcoded hex values"
    - "Check contrast ratios in both light and dark mode using WCAG 2.2 / APCA targets"

dependsOn: ["UX-STD-001"]
supersedes: ""
---

# Dark Mode and Color Palette Design

## Purpose

This best practice document outlines **RECOMMENDED** practices for designing dark mode themes and constructing tokenized color systems. Standardizing contrast, visual hierarchy, and color adjustments across display modes minimizes optical fatigue, preserves accessibility conformance, and ensures consistent rendering on modern electronic displays.

> *Normative language (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

## Rules

### R-1: Theme Implementation and OS Preference

Applications **SHOULD** support both light and dark visual themes.

- Visual theme switching **SHOULD** automatically respond to the host operating system's configuration via CSS `@media (prefers-color-scheme)`
- When a manual override toggle is provided, the selection **MUST** persist locally (e.g., via browser `localStorage` or server profile preference) to avoid resetting on page refresh

### R-2: Dark Mode Palette Construction

Dark themes **SHOULD NOT** use pure black (`#000000`) for primary, high-area background zones.

- Primary dark mode background colors **SHOULD** utilize desaturated off-blacks or charcoal tones (e.g., HSL neutral-dark values, `#121212`, `#1e1e1e`) to reduce screen glare and eye strain
- Text in dark mode **SHOULD NOT** use pure white (`#FFFFFF`) — use high-contrast off-whites or light grays instead to prevent visual vibration
- Accent and brand colors **SHOULD** be adjusted (lightened or slightly desaturated) when transitioning to dark mode to ensure they maintain sufficient contrast without causing optical bleeding/halo effects
- UI elevation (stacking hierarchy) in dark mode **SHOULD** be indicated by applying lighter color overlays or background values on top-level elements (the closer the element is to the user, the lighter its color)

### R-3: Design Token Abstraction

Color variables **MUST** be declared as semantic design tokens (variables) rather than static, hardcoded hex values.

- Token definitions **SHOULD** represent element roles (e.g., `--color-bg-primary`, `--color-text-secondary`, `--color-accent`) rather than literal color definitions (e.g., `--color-gray-100`, `--color-blue-500`)

## Examples

### HSL Theme Token Configuration

```css
/* Valid: Color configuration using semantic CSS Custom Properties */
:root {
  --color-bg-primary: hsl(220, 15%, 98%);
  --color-text-primary: hsl(220, 20%, 12%);
  --color-accent: hsl(210, 100%, 50%);
  --color-elevation-1: hsl(220, 15%, 95%);
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-bg-primary: hsl(220, 15%, 8%);      /* Off-black */
    --color-text-primary: hsl(220, 10%, 90%);    /* Off-white */
    --color-accent: hsl(210, 100%, 65%);         /* Slightly desaturated accent */
    --color-elevation-1: hsl(220, 15%, 12%);     /* Lighter background for elevation */
  }
}
```

```css
/* Invalid: Static hex colors hardcoded in components */
.card-component {
  background-color: #ffffff;
  color: #333333;
}
```

## References

- [W3C WCAG Contrast Minimum (SC 1.4.3)](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)
- [W3C Advanced Perceptual Contrast Algorithm (APCA) Draft Spec](https://github.com/Myndex/SupaDupaContrast)
- [W3C Design Tokens Format Specification (Community Draft)](https://tr.designtokens.org/format/)
- [ISO 9241-307:2008 — Analysis and compliance test methods for electronic visual displays](https://www.iso.org/standard/38952.html)
- [ISO 9241-391:2016 — Requirements, analysis and compliance methods for the reduction of photosensitive seizures](https://www.iso.org/standard/65942.html)
- [Material Design 3 Color System](https://m3.material.io/styles/color/system/overview)
- [Apple Human Interface Guidelines — Color](https://developer.apple.com/design/human-interface-guidelines/color)

## Rationale

**Why avoid pure black (#000000) backgrounds?** Sighted users with astigmatism (roughly 30-40% of the population) experience "haloing" or "bleeding" when reading high-contrast white text on a pure black background. Softening the contrast to charcoal tones minimizes iris dilation and eye fatigue.

**Why adjust accent colors for dark mode?** Accent colors designed for light mode (e.g., dark blue) have poor contrast and readability against dark backgrounds. Conversely, highly saturated colors designed for light mode (e.g., hot pink) bleed visually in dark environments. Standardizing lightened or desaturated variants for dark themes guarantees visual clarity.

## Version History

| Version | Date       | Change                            |
| ------- | ---------- | --------------------------------- |
| 1.0.0   | 2026-06-16 | Initial definition; gold standard |
