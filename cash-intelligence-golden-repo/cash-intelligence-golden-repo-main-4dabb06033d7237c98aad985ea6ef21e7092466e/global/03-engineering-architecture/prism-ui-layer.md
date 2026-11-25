---
id: GR-ARC-002
title: Prism 2.0 — The UI Layer for Cash Intelligence
artifact_type: platform-obligations
status: draft
content_state: derived
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 README.md §Keeping Figma, tokens, components and Claude Design in agreement"
  - "SRC-PRISM-UI-001 CHANGELOG.md §3.0.0"
  - "SRC-PRISM-UI-001 docs/USING-PRISM.md"
  - "SRC-PRISM-UI-001 ai-resources/AGENTS.md"
  - "SRC-PRISM-UI-001 ai-resources/design.md §Components"
  - "SRC-PRISM-UI-001 docs/reference/README.md"
  - "SRC-XR-TDD-001"
  - "SRC-MTG-PRISM-001"
last_updated: 2026-11-19
review_due: TBC
---


# Prism 2.0 — The UI Layer for Cash Intelligence

**Derived, draft.** Reorganised from the registered Prism sources; it adds no rules of its own. Where this
summary and `prism-design-rules.md` or `prism-agent-instructions.md` differ, those two (verbatim copies of
the Prism repository's own agent documents) win, and the Prism repository wins over them.

## Why Prism is the UI layer

The Crossroads Foundation Technical Design Document requires every ABS frontend to be React with the Prism 2.0
React library, `@kinective/prism-react`, shipped as child micro-frontends of the KBOP shell (`SRC-XR-TDD-001`
TDD:378, carried in `crossroads-abs-obligations.md`). The 2026-10-29 Prism meeting agreed that the Golden Repo
carries a few curated Markdown files rather than the Figma file (`SRC-MTG-PRISM-001`). Every screen specified or
built for Cash Intelligence therefore uses Prism components and `--prism-*` tokens; styling outside Prism is a
defect to report, not a choice.

## Prism exists in four places

| Place | What it holds | Readable by Astra |
|---|---|---|
| Figma file `Dbh3fLPGHxMLttXZVbpIMs` (`SRC-FIGMA-PRISM-001`) | The source of truth: every token, component, variant, state and accessibility annotation | No |
| Tokens: packages/prism-css/tokens/ and packages/prism-contract/prism.contract.json | The Figma variables as CSS custom properties, and the machine-readable contract | Through this repository's copies |
| Components: packages/prism-css/components/, `prism-react`, `prism-vue` | The same component API and markup in both bindings, styled only through the tokens | No (code) |
| Claude Design kit *Prism Design System* (`SRC-CLAUDE-PRISM-001`) | A rendering of the Figma-matched reference build, for prototyping | No (artifact) |

Change order is Figma, then tokens, then components, then the Claude Design kit, then the docs. When two of them
disagree, that is drift: Figma wins and the other is fixed (SRC-PRISM-UI-001 README.md).

## What ships: release 3.0.0

Prism 3.0.0 was released on 2026-11-13 (SRC-PRISM-UI-001 CHANGELOG.md §3.0.0). All five packages share one
version and are released together:

| Package | What it is | Version |
|---|---|---|
| `@kinective/prism-css` | The token layer (`tokens/*.css`) and one stylesheet per component | 3.0.0 |
| `@kinective/prism-react` | React 18 components, named exports, one stylesheet per component | 3.0.0 |
| `@kinective/prism-vue` | Vue 3.5 components, plus a `createPrism()` plugin | 3.0.0 |
| `@kinective/prism-contract` | `prism.contract.json`: tokens, component API, accessibility facts, contrast results | 3.0.0 |
| `@kinective/prism-mcp` | A read-only MCP server over the contract: `get_tokens`, `get_component_contract`, `get_component_parts`, `get_a11y_by_criterion`, `check_drift` | 3.0.0 |

Ark UI is pinned to 5.40.0 in both bindings.

**Where the packages come from.** Kinective's JFrog Artifactory, project `kux`, repository `kux-npm-dev-local`;
anonymous access is off. A project reaches them with an `.npmrc` that routes only the `@kinective` scope there and
takes the access token from the `JFROG_NPM_TOKEN` environment variable, never from a committed file. The setup
steps, from getting a token to loading the tokens, are in SRC-PRISM-UI-001 docs/USING-PRISM.md. Keep every
`@kinective` package on the same version and upgrade them together: `prism-react` and `prism-vue` 3.0.0 depend on
exactly `prism-css` 3.0.0.

## Component state at 3.0.0

- 95 components ship in both bindings.
- 9 match the Figma contract: Badge, Button, Close Button, Divider, Helper text, Icon Button, Kbd, Label,
  Tooltip (SRC-PRISM-UI-001 CHANGELOG.md §3.0.0). The component table inside `prism-design-rules.md` still shows
  Helper text and Label with an update in an open merge request; it is corrected at the next refresh (DEC-018).
- Dialog has a Figma-contract update in merge request !33, not part of 3.0.0; its status is to be confirmed.
- 85 are **legacy**: built before the Figma contract and still on their original API and look. A legacy component
  is built with the props its package exports today, never with properties read from its Figma frame.
- 18 components exist only in the Figma-matched reference build and are not published; 20 exist in Figma only.
  Both lists are in `prism-availability.md`.
- The repository is brought up to Figma one component per merge request, tracked in Jira project PDS: wave 1 is
  the 39 components Signing Room needs, up to Sidebar, Topbar and Data table; wave 2 is everything else.

## How a product loads Prism

From SRC-PRISM-UI-001 ai-resources/AGENTS.md and docs/USING-PRISM.md:

- Install the binding and `@kinective/prism-css` at the same version, e.g. `@kinective/prism-react@3.0.0` and
  `@kinective/prism-css@3.0.0`.
- Import a component and its stylesheet: `import { Button } from '@kinective/prism-react'` and
  `import '@kinective/prism-react/Button.css'` (or `index.css` for every component).
- Load the token layer once, at or above the app root, in this order: `prismPrimitive.css`, `prismTheme.css`,
  `prismToken.css`, `prismSize.css`, `prismTypography.css`, then `prismFigma.css` **last**. `prismFigma.css`
  defines the Figma code-syntax names (`--prism-bg-primary-default`) that the vocabulary uses; without it the UI
  typeface resolves to Haffer instead of Inter.
- Theme and density are attributes on any element, nearest ancestor wins: `data-theme="light"` or `"dark"`
  (light is the default; there is no automatic `prefers-color-scheme` switch), and `data-density="touch"`,
  `"kiosk"` or `"mobile"` (desktop is the default). Density is a device class, not a breakpoint. Both bindings
  export theme helpers (`applyPrismTheme`, `usePrismTheme`) that set these attributes.
- The packages do not ship font files: the app loads Inter and Haffer itself.
- The MCP server runs from a checkout (`node packages/prism-mcp/src/server.mjs`) or with
  `npx -y @kinective/prism-mcp`, which needs the two `.npmrc` lines in the user's own `~/.npmrc`.

## Rules that govern every screen

The rules themselves are in `prism-design-rules.md` (verbatim `design.md`): the two hard rules (bind the semantic
layer, never a primitive; never invent a name), the priority order (correctness of the data, accessibility, the
token and component contract, consistency, refinement), the four passes, the split between a product Figma file
and Prism, composition, depth, emphasis, state, the generated-UI reflexes to reject, copy, typefaces, the silent
failures nobody checks, the inspection checklist, and the known gaps. `prism-accessibility.md` holds the
accessibility rules and `prism-naming-and-casing.md` the naming and casing rules. None of them is restated here.

## Where Prism lives in this repository

| File | Tier | Content state | Use it when |
|---|---|---|---|
| `global/03-engineering-architecture/prism-ui-layer.md` (this file) | global | derived | Orientation |
| `global/03-engineering-architecture/prism-agent-instructions.md` | global | source-backed, verbatim ai-resources/AGENTS.md | Installing, importing, what to read next |
| `global/03-engineering-architecture/prism-design-rules.md` | global | source-backed, verbatim ai-resources/design.md | Composing or reviewing any screen; the token vocabulary and the component status list |
| `global/03-engineering-architecture/prism-accessibility.md` | global | source-backed, verbatim ai-resources/accessibility.md | Accessibility rules; Legal review before any conformance statement |
| `global/03-engineering-architecture/prism-naming-and-casing.md` | global | source-backed, verbatim ai-resources/casing.md | Writing UI copy, naming |
| `global/03-engineering-architecture/prism-availability.md` | global | derived | Which components can be built today, which cannot |
| `global/08-approved-patterns-templates/prism-build-stage-pattern.md` | global | proposed | Wiring Prism, its MCP server and an advisory adherence check into a code repository |
| `shared/templates/spec-ui-layer-template.md` | shared | proposed | The UI-layer section a screen spec carries |
| `projects/branch-automation-cash-intelligence/03-design-architecture/prism-ui-layer-map.md` | project | proposed | Cash Intelligence screens mapped to Prism templates and components |

The verbatim files carry a short repository preface and front matter and nothing else of ours. They are refreshed
from the Prism repository on each Prism release and never edited here (DEC-018). The Prism repository's
ai-resources/components.md (per-component Figma properties and usage) and `raw/` exports are not copied: look
them up in the repository or through `@kinective/prism-mcp`. docs/USING-PRISM.md is cited, not copied: it is
setup for a code repository, not context for a spec.

## What is excluded (DEC-019)

- The legacy Prism v0.1 steering file held in the ESP repository: it predates the Prism 2.0 contract and calls
  itself a single source of truth with older token names.
- The styling of the Cash Intelligence prototype `cash-analytics` (DEC-002): it uses unprefixed tokens and
  resolved colour values.

If either reaches an Astra selection, remove it: two Prisms in context produce conflicting UI.

## Legal review

Prism builds to WCAG 2.2 Level AA as an authoring target, not a conformance claim. Kinective Legal must review
any accessibility or conformance statement before it is relied on or made to a customer
(`prism-accessibility.md`).
