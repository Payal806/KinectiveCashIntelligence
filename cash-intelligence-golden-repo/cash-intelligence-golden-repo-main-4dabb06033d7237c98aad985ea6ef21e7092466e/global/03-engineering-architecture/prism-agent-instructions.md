---
id: GR-ARC-003
title: Prism 2.0 — Instructions for Coding Agents (verbatim AGENTS.md)
artifact_type: platform-obligations
status: draft
content_state: source-backed
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 ai-resources/AGENTS.md"
last_updated: 2026-11-19
review_due: TBC
---


> **Verbatim copy, draft.** Everything below this line is ai-resources/AGENTS.md from the Prism 2.0 repository
> (`SRC-PRISM-UI-001`, branch `monorepo`, commit `8c4c80bb`, release 3.0.0, blob `7aaad9b5`), unchanged. It is refreshed
> from the Prism repository on each Prism release and never edited here (DEC-018). Relative links inside it point at
> the Prism repository, not at this one. Where it and the Prism repository differ, the repository wins.

# Prism: instructions for coding agents

You are building product UI with Prism 2.0, Kinective's design system, from the packages this
repository publishes. This file says what to install, what to import, and which file to read next.

<!-- prism:stamp -->
> **Stamped 2026-11-06.** Describes `@kinective/prism-css` 3.0.0, `@kinective/prism-react` 3.0.0, `@kinective/prism-vue` 3.0.0, `@kinective/prism-contract` 3.0.0, `@kinective/prism-mcp` 3.0.0.
> Ark UI as declared: `@ark-ui/react` `5.40.0`, `@ark-ui/vue` `5.40.0`.
> Registry: `https://kinective.jfrog.io/artifactory/api/npm/kux-npm-dev-local/` (JFrog Artifactory, project `kux`; needs a token). Figma contract 1.0.0, file `Dbh3fLPGHxMLttXZVbpIMs`.
> Generated blocks come from `node ai-resources/build-agent-docs.mjs`; re-run it on every publish.
<!-- /prism:stamp -->

## What ships

| Package | What it is |
|---|---|
| `@kinective/prism-css` | The token layer (`tokens/*.css`) and one stylesheet per component (`components/*.css`). Both bindings render the same markup against these stylesheets. |
| `@kinective/prism-react` | React 18 components. Named exports, one per component, plus a stylesheet per component. |
| `@kinective/prism-vue` | Vue 3.5 components. Named exports, plus a `createPrism()` plugin and a stylesheet per component. |
| `@kinective/prism-contract` | `prism.contract.json`: the Figma contract (tokens, component API, a11y facts, contrast results), machine-readable. |
| `@kinective/prism-mcp` | An MCP server over the contract: `get_tokens`, `get_component_contract`, `get_component_parts`, `get_a11y_by_criterion`, `check_drift` (plus `get_component`, an alias of `get_component_contract` kept for older clients). |

Packages install from Kinective's JFrog Artifactory (the address is in the stamp above). It
needs an access token, so a project installs them only after its `.npmrc` routes the
`@kinective` scope there with a token; the repository README shows the two lines.

```ts
// React
import { Button } from '@kinective/prism-react'
import '@kinective/prism-react/Button.css' // or '@kinective/prism-react/index.css' for every component
```

```ts
// Vue
import { Button } from '@kinective/prism-vue'
import '@kinective/prism-vue/styles/button.css' // or '@kinective/prism-vue/styles' for every component
```

Load the token layer once, at or above the app root, in this order:

```css
@import '@kinective/prism-css/tokens/prismPrimitive.css';
@import '@kinective/prism-css/tokens/prismTheme.css';
@import '@kinective/prism-css/tokens/prismToken.css';
@import '@kinective/prism-css/tokens/prismSize.css';
@import '@kinective/prism-css/tokens/prismTypography.css';
@import '@kinective/prism-css/tokens/prismFigma.css'; /* Figma code-syntax names; load last */
```

`prismFigma.css` defines the token names Figma uses (`--prism-bg-primary-default`), which is the
vocabulary in [`design.md`](design.md), including the effect composites (`--prism-effect-*`) and
the alpha tokens. Load it last: it is also where the Figma values win over older ones in the files
above it, so without it `--prism-font-family-sans` resolves to Haffer instead of Inter.

Theme and density are attributes on any element; the nearest ancestor wins:

- `data-theme="light"` or `"dark"`. Light is the default; there is no automatic
  `prefers-color-scheme` switch, so set `dark` explicitly. Put a page-level theme on `<html>`.
- `data-density="touch"`, `"kiosk"` or `"mobile"`. Desktop is the default and needs no attribute.
  Density is a device class, not a breakpoint.

## Read next

[`design.md`](design.md) is the file to load first: the judgment for composing a screen, the full
token vocabulary, and the component list with each component's status. Everything else is a
lookup:

| File | When |
|---|---|
| [`components.md`](components.md) | A component's Figma properties, variant axes and usage |
| [`accessibility.md`](accessibility.md) | The accessibility rules you build to |
| [`raw/a11y.psv`](raw/a11y.psv) | One component's accessibility facts, row by row |
| [`raw/ark-mapping.psv`](raw/ark-mapping.psv) | Which Ark UI machine a component is built on |
| [`casing.md`](casing.md) | You are writing UI copy |

**Do not take token names from `raw/`.** The files there are the Figma export of 2026-10-01 that
the prism-css tests and generators are pinned to. The colour, dimension, typography and
code-syntax files predate the token renames (`bg/primary`, not `bg/primary/default`) and the
Inter/Haffer split, and some component facts predate later Figma changes. The vocabulary in
`design.md` and the property lists in `components.md` are current; where `raw/` disagrees, they
win.

`packages/prism-css/ai/` holds the earlier foundation-layer docs (`llms.txt`, `design-policy.md`,
`accessibility-policy.md`). They predate the components; where they disagree with this folder,
this folder wins.

## Which components match Figma

The Figma file is the source of truth, and this repo is being brought up to it one component at a
time. A component marked **legacy** in `design.md` still has its original API and look: build with
its current props, and do not copy an API from a Figma frame or from another codebase onto it. A
component marked **Figma contract** matches the Figma component's properties and visuals.

If you find the Figma file, the tokens, a component and these docs disagreeing anywhere else, that
is drift, not a choice: build from the token and component the package ships, and report the
disagreement in your handover. Keeping them in agreement is described in the repository
[`README.md`](../README.md#keeping-figma-tokens-components-and-claude-design-in-agreement).

## The two rules that matter most

1. **Bind to the semantic layer, never to a primitive.** `bg/primary/default`, not `color/blue/600`.
   `prismPrimitive.css` is published only so the semantic tokens resolve.
2. **Never invent a token or a component.** If what you need is not in `design.md`, stop and ask. A
   missing token is a design-system decision, not an implementation detail.

## Before you hand work over

There is no automated output checker in this repo yet. Check by hand, against `design.md`, that:
- every `var(--prism-…)` you wrote is in the vocabulary and not in its "Not yet defined" list;
- you used no hex, rgb, px or millisecond literal where a token exists;
- you gave no `var()` a literal fallback.

`get_tokens` and `check_drift` in `@kinective/prism-mcp` answer the same questions when you have
tool access.

## Data

Never put real member names, account numbers, SSNs or any other personally identifiable data
belonging to a financial institution's end customers into code, fixtures, stories, screenshots or
documentation. Prism sample data is fictional and stays that way. If you are handed real data, flag
it and ask for it to be removed.
