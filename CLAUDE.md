# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

`precepts-dev/standards` is the canonical source of multi-discipline standards for the Precepts platform. It contains machine-readable standards (Markdown + YAML frontmatter) consumable by both humans and AI agents.

**GitHub org:** `precepts-dev`

**License:** CC BY-SA 4.0. All content is open.

**Published as:** `@precepts/standards` on npm.

## Repository Structure

```
standards/
  integration/          Integration standards (API, events, resilience, observability)
    standards/          Mandatory, enforceable
    governance/         Governance documents
    guidelines/         Recommended, advisory
  product/              Product management standards
  ux/                   UX standards
  project-management/   Project management standards
schema/
  standards.schema.json   Required frontmatter fields
  document-standard-template.md   Template for new standards
```

## Standards Schema

Each standard has YAML frontmatter with required fields: `identifier`, `name`, `version`, `status`, `domain`, `documentType`.

- Identifier pattern: `[INTG | PRD | PRJ | UX | SEC]-[STD | GDL | GOV | BP]-NNN`
- Document types: `standard`, `guideline`, `governance`, `best-practice`
- Status values: `DRAFT`, `MANDATORY`, `RECOMMENDED`, `DEPRECATED`
- Domains: `INTEGRATION`, `PRODUCT-MANAGEMENT`, `PROJECT-MANAGEMENT`, `UX`, `SECURITY`
- Version: semver (patch = typo, minor = new fields/status change, major = breaking)

## Content Format Conventions

- Body section "## Rules" for prescriptive instructions
- RFC 2119 keywords (**MUST**, **SHOULD**, **MAY**) bolded throughout
- Rule numbering: `### R-N: Title` as H3 headings
- Examples: concept-level (pseudocode/generic), not technology-specific
- Concise: 200-300 lines per standard

## Consumers

This package is consumed by:
- `precepts-dev/platform` repo (Docusaurus site, MCP server, validator)
- `precepts-dev/engine` (commercial compliance scanner - separate private repo)
- Any third party building tooling around Precepts standards
