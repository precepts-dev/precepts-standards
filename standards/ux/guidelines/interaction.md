---
identifier: "UX-GDL-001"
name: "Mobile and Web Interaction Design"
version: "1.0.0"
status: "RECOMMENDED"

domain: "UX"
documentType: "guideline"
category: "interaction"
appliesTo: ["web", "ios", "android"]

lastUpdated: "2026-06-16"
owner: "UX Design Authority"

standardsCompliance:
  iso: ["ISO-9241-11:2018", "ISO-9241-110:2020", "ISO-9241-125:2017"]
  rfc: []
  w3c: ["WCAG-2.2-SC-2.5.2", "WCAG-2.2-SC-2.5.5", "WCAG-2.2-SC-2.5.8"]
  other: ["Apple-HIG-Layout", "Material-Design-3-Layout", "NNg-10-Usability-Heuristics"]

taxonomy:
  capability: "interaction-design"
  subCapability: "input-layout"
  layer: "interaction"

enforcement:
  method: "review-based"
  reviewChecklist:
    - "Confirm mobile touch targets are at least 44x44 CSS pixels and desktop are at least 32x32 CSS pixels"
    - "Verify spacing between targets is at least 8px to prevent accidental adjacent activation"
    - "Ensure button text or icons represent their specific actions clearly"
    - "Confirm modal overlay backdrops close on click, except during unsafe transactions"
    - "Verify form submit controls disable upon submission and display loader indicators"
    - "Ensure system status changes are visible and clear (user control state visibility)"

dependsOn: ["UX-STD-001"]
supersedes: ""
---

# Mobile and Web Interaction Design

## Purpose

This guideline outlines **RECOMMENDED** interaction standards for mobile and web systems. Standardizing interactive sizing, element spacing, transitional behaviors, and system state responses improves usability, reduces input error rates, and ensures consistent user flows in compliance with ISO 9241 usability principles and platform design patterns.

> *Normative language (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

## Rules

### R-1: Interactive Control Dimensions and Spacing

Interactive elements **MUST** offer target dimensions proportional to their activation methods (touch or mouse pointer).

- Touch targets on mobile interfaces **MUST** have a minimum dimension of 44x44 CSS pixels (or device-independent pixels)
- Mouse targets on desktop interfaces **SHOULD** have a minimum dimension of 32x32 CSS pixels
- Inline text links are exempted from minimum target dimensions but **MUST** have visible text identifiers (underlines or high-contrast color shifts)
- A minimum spacing of 8px **MUST** separate adjacent interactive components to prevent accidental activation of nearby elements

### R-2: System Status and Component States

User interfaces **MUST** visually convey status and control options clearly (corresponding to Nielsen's "Visibility of System Status").

- Interactive controls **MUST** support distinct styles for hover, focus, active (pressed), and disabled states
- State changes **MUST NOT** trigger browser recalculations (reflow) that adjust structural layout boundaries; styles **SHOULD** use features like `outline`, `box-shadow`, or opacity changes to maintain layout boundaries
- Submit buttons in forms **MUST** disable immediately after selection, transition to a loading state, and block duplicate click submissions during network processes

### R-3: Modals and Dialog Overlays

Overlays and dialog containers **MUST** allow clear dismissal strategies:

- Close buttons **MUST** be present in the upper corner of any modal or dialog container
- Clicking outside the modal container (on the backdrop overlay) **SHOULD** dismiss the modal unless it is a transactional step where closing could cause unsaved data loss
- Focus **MUST** trap inside the modal while open, and focus **MUST** return to the invoking element when the modal is closed

## Examples

### Input Touch Target Styling

```css
/* Valid: Interactive target padded appropriately */
.btn-primary {
  min-height: 44px;
  padding: 12px 24px;
  box-sizing: border-box;
  display: inline-flex;
  align-items: center;
}

/* Invalid: Target too small for mobile touch */
.btn-small {
  height: 24px;
  padding: 2px 4px;
}
```

### Safe State Indicators

```css
/* Valid: Visual changes do not alter bounding box dimensions */
.btn-action:hover {
  background-color: var(--accent-hover);
  box-shadow: inset 0 0 0 2px var(--accent-border);
}

/* Invalid: Changing border thickness changes the element's layout footprint */
.btn-action:hover {
  border: 2px solid var(--accent-border);
}
```

## References

- [ISO 9241-11:2018 — Usability: Definitions and concepts](https://www.iso.org/standard/63500.html)
- [ISO 9241-10:2020 — Ergonomics of human-system interaction — Part 110: Interaction principles](https://www.iso.org/standard/74233.html)
- [Apple Human Interface Guidelines — Layout and Platform Conventions](https://developer.apple.com/design/human-interface-guidelines)
- [Google Material Design 3 — Layout and Spacing Systems](https://m3.material.io/)
- [Nielsen Norman Group — 10 Usability Heuristics for User Interface Design](https://www.nngroup.com/articles/ten-usability-heuristics/)

## Rationale

**Why align with ISO 9241-110?** ISO 9241-110 defines critical ergonomic interaction principles, such as self-descriptiveness, suitability for the task, and error tolerance. Standardizing button states, modal behavior, and touch target bounds directly reduces cognitive friction and improves task completion efficiency.

**Why target dimensions of 44x44px?** An average human finger pad covers about 8–10mm, which corresponds to roughly 44x44 pixels on standard mobile screens. Smaller boundaries increase error rates and make tools unusable for individuals with tremors or motor limitations.

## Version History

| Version | Date       | Change                            |
| ------- | ---------- | --------------------------------- |
| 1.0.0   | 2026-06-16 | Initial definition; gold standard |
