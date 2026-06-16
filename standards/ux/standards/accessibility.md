---
identifier: "UX-STD-001"
name: "Accessibility and Assistive Technology"
version: "1.0.0"
status: "MANDATORY"

domain: "UX"
documentType: "standard"
category: "accessibility"
appliesTo: ["web", "ios", "android"]

lastUpdated: "2026-06-16"
owner: "UX Design Authority"

standardsCompliance:
  iso: ["ISO-9241-171:2008", "ISO/IEC-40500:2012", "ISO/IEC-13066-1:2011"]
  rfc: []
  w3c: ["WCAG-2.2-AA", "WAI-ARIA-1.2", "ATAG-2.0", "UAAG-2.0"]
  other: ["Section-508-Rehabilitation-Act", "EN-301-549-V3.2.1", "ADA-Title-III"]

taxonomy:
  capability: "accessibility"
  subCapability: "assistive-technology"
  layer: "visual"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Validate all text elements pass WCAG 2.2 AA contrast ratios (4.5:1 / 3:1)"
    - "Ensure complete keyboard operation without traps or visual focus loss"
    - "Confirm focus order follows intuitive content relationships and reading layout"
    - "Verify semantic HTML landmarks and structural elements exist and match the UI structure"
    - "Validate custom elements use correct WAI-ARIA states, roles, and properties"
    - "Verify input controls are programmatically linked to their corresponding label elements"

dependsOn: []
supersedes: ""
---

# Accessibility and Assistive Technology

## Purpose

This standard defines **MANDATORY** accessibility and assistive technology requirements across all user interfaces. It guarantees that applications remain fully usable by individuals with visual, auditory, motor, cognitive, or speech impairments. Adherence is required to achieve compliance with global accessibility regulations including EN 301 549, Section 508, and the Americans with Disabilities Act (ADA).

> *Normative language (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

## Rules

### R-1: WCAG 2.2 Level AA Compliance

All digital products **MUST** achieve and maintain full compliance with W3C Web Content Accessibility Guidelines (WCAG) 2.2 Level AA criteria.

- Crucial user-flow pathways (e.g., login, payment, account recovery) **SHOULD** conform to WCAG 2.2 Level AAA requirements where feasible

### R-2: Keyboard Accessibility and Focus Control

All interactive components **MUST** be usable without a pointing device (mouse or touch screen).

- Users **MUST** be able to navigate to and activate all interactive controls using only standard keyboard inputs (e.g., Tab, Shift+Tab, Enter, Spacebar, Arrow keys)
- Keyboard focus **MUST NOT** become trapped in any UI component or overlay; the user **MUST** be able to exit all elements using keyboard navigation
- Focus indicators **MUST** be highly visible and have a contrast ratio of at least 3:1 against their background
- Focus order **MUST** follow a logical visual and structural progression
- Dialog boxes and modal overlays **MUST** trap focus inside the container while active, and **MUST** restore focus to the triggering element when dismissed

### R-3: Sensory and Color Adjustments

Color **MUST NOT** be used as the single method to convey status, select options, or prompt actions.

- Text contrast ratios **MUST** meet a minimum of 4.5:1 against the background (3:1 for large text at least 18pt or 14pt bold)
- Non-text elements (icons, borders, visual state boundaries) **MUST** maintain a minimum contrast ratio of 3:1 against adjacent colors
- Audio and video media **MUST** provide synchronous text alternatives (closed captions, transcripts)

### R-4: Document structure and Semantics

Content **MUST** be formatted using standard, semantic structural elements to facilitate assistive technology interpretation:

- Pages **MUST** contain exactly one `<h1>` element representing the primary page heading, followed by a logical hierarchy of `<h2>` through `<h6>` headings
- Visual landmarks (headers, navigation, main sections, footers) **MUST** utilize native semantic tags (e.g., `<header>`, `<nav>`, `<main>`, `<footer>`) or appropriate ARIA roles
- All form inputs **MUST** be programmatically coupled with their descriptive text labels via the HTML `for` and `id` attributes
- Custom UI components (e.g., accordions, tabs, switch controls) **MUST** implement WAI-ARIA states, roles, and properties (e.g., `aria-expanded`, `aria-controls`, `role="tablist"`) following W3C design patterns

## Examples

### Programmatic Form Label Association

```html
<!-- Valid: Explicit pairing using 'for' matching input 'id' -->
<label for="user-email-input">Email Address</label>
<input type="email" id="user-email-input" name="email" required />

<!-- Invalid: Text element adjacent to input without programmatic binding -->
<span>Email Address</span>
<input type="email" name="email" required />
```

### Landmarking and Layout Hierarchy

```html
<!-- Valid: Structural layout utilizing native landmark tags -->
<header>
  <h1>User Dashboard</h1>
</header>
<nav aria-label="Main Navigation">
  <ul>
    <li><a href="/home">Home</a></li>
  </ul>
</nav>
<main>
  <h2>Recent Activity</h2>
  <!-- Content here -->
</main>
```

## References

- [W3C Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/)
- [W3C WAI-ARIA 1.2 Specification](https://www.w3.org/TR/wai-aria-1.2/)
- [W3C Authoring Tool Accessibility Guidelines (ATAG) 2.0](https://www.w3.org/TR/ATAG20/)
- [W3C User Agent Accessibility Guidelines (UAAG) 2.0](https://www.w3.org/TR/UAAG20/)
- [ISO 9241-171:2008 — Guidance on software accessibility](https://www.iso.org/standard/39080.html)
- [ISO/IEC 40500:2012 — Information technology — W3C WCAG 2.0](https://www.iso.org/standard/58625.html)
- [ISO/IEC 13066-1:2011 — Interoperability with assistive technology](https://www.iso.org/standard/53610.html)
- [European Standard EN 301 549 — Accessibility requirements for ICT products](https://www.etsi.org/deliver/etsi_en/301500_301599/301549/03.02.01_60/en_301549v030201p.pdf)
- [US Section 508 Standards](https://www.section508.gov/)

## Rationale

**Why align with ISO-9241-171 and EN 301 549?** Designing digital tools requires meeting global legislative baselines. EN 301 549 forms the legal foundation for the European Accessibility Act (EAA), while Section 508 governs public sector software in the United States. Following these standards ensures products can be procured globally without compliance re-engineering.

**Why prohibit single-mode sensory cues?** Users with color vision deficiencies or visual impairments cannot perceive color shifts (e.g., green borders for success, red borders for error). Supplementary cues (icons, inline messages, semantic markup) ensure that the message is transmitted regardless of sensory capability.

## Version History

| Version | Date       | Change                            |
| ------- | ---------- | --------------------------------- |
| 1.0.0   | 2026-06-16 | Initial definition; gold standard |

