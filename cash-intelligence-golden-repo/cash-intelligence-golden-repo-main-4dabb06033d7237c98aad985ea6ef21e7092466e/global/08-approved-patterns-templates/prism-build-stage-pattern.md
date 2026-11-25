---
id: GR-PAT-002
title: Prism Build-Stage Pattern for Code Repositories
artifact_type: content-pattern
status: draft
content_state: proposed
owner_role: Cash Intelligence architecture owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 ai-resources/AGENTS.md"
  - "SRC-PRISM-UI-001 docs/USING-PRISM.md"
  - "SRC-PRISM-UI-001 ai-resources/design.md §What is checked, and what is not"
  - "SRC-PRISM-UI-001 packages/prism-mcp/README.md"
  - "SRC-MTG-PRISM-001"
last_updated: 2026-11-19
review_due: TBC
---


# Prism Build-Stage Pattern for Code Repositories

**Proposed, draft.** A recommendation for how a Cash Intelligence code repository wires Prism in so that a coding
agent builds on it by default. It needs the architecture owner's and the product design owner's approval
(DEC-018, DEC-020). The Prism facts it relies on are source-backed and cited; the wiring choices are this
repository's.

## What the pattern provides

| Element | Where it lives | Source of the rule |
|---|---|---|
| Package access: an `.npmrc` that routes the `@kinective` scope to JFrog Artifactory (`kux-npm-dev-local`) with the token from `JFROG_NPM_TOKEN`; every `@kinective` package pinned to the same version, 3.0.0. *For Cash Intelligence (DEC-023): until the switch the frontend uses the micro-frontend template's vendored Prism copy and installs no `@kinective/prism-*` package; from the switch the pin is the first published release that meets DEC-023's conditions, not 3.0.0* | Code repository root, CI secret variable | SRC-PRISM-UI-001 docs/USING-PRISM.md |
| Agent instructions: a Prism section in the repository's `CLAUDE.md` (and `AGENTS.md` for other agents) that points at the Prism files in this Golden Repo, states the two hard rules, and tells the agent what to read before writing UI | Code repository root | SRC-PRISM-UI-001 ai-resources/AGENTS.md, `design.md §The two hard rules` |
| Token layer loaded once at the app root, six files in order with `prismFigma.css` last; per-component stylesheet imports; font files loaded by the app | App entry | SRC-PRISM-UI-001 ai-resources/AGENTS.md §What ships |
| `@kinective/prism-mcp` registered in `.mcp.json`, from a checkout or with `npx`, so the agent can look up tokens, component contracts, parts per state and accessibility facts instead of relying on memory | Code repository | SRC-PRISM-UI-001 packages/prism-mcp/README.md |
| An **advisory** adherence check (`prism-audit`) that reports on every agent edit and every merge request and blocks nothing: unknown or primitive token, hardcoded colour or length, a hand-built control where a Prism component exists, colours that ignore dark mode, sizes that ignore density. Prism documents these failures as silent and unchecked (`design.md §What is checked, and what is not`). It stays advisory until its rules are rebuilt from SRC-PRISM-UI-001; three rules of the current build (load order, theme and density placement, component props) were written against the Claude Design kit and flag correct code against the published packages (DEC-020) | Code repository hook and CI, reporting only | Proposed: built by the Cash Intelligence architecture owner; 13 of its rules reviewed as sound by the product design owner |
| Fixing by role, never by value: a finding is resolved by choosing the semantic token or the Prism component, not by matching the pixel or adding a suppression | Agent instructions | SRC-PRISM-UI-001 ai-resources/design.md §Working from a product Figma file |
| Human review limited to what the check cannot judge: composition and content against the product Figma frame, template choice, copy | Merge request review | `SRC-MTG-PRISM-001` (adherence validated with a designer) |

## Agent instruction block (proposed wording)

```markdown
# UI layer: Prism 2.0 (release 3.0.0; for Cash Intelligence, the template's vendored copy until the DEC-023 switch)
All UI uses Prism 2.0 components and `var(--prism-…)` tokens. Never hardcode colours, lengths, radii,
shadows, fonts or motion.

Before writing any UI, read, from the Golden Repo:
1. global/03-engineering-architecture/prism-design-rules.md (the rules and the vocabulary)
2. the screen's spec, section "UI layer" (template and components per region); if absent, stop and ask
3. global/03-engineering-architecture/prism-availability.md (what can be built today)
Look up a component's props, parts and accessibility facts with the prism MCP tools.

Rules:
- Bind the semantic layer, never a primitive. Never invent a token or a component; stop and ask.
- Mount the real component from @kinective/prism-react; never a styled <button>, <input>, <select>,
  <textarea> or <table>; never write data-scope or data-part attributes by hand.
- A legacy component is built with the props its package exports today.
- Theme and density are data-theme and data-density attributes; nearest ancestor wins; light and
  desktop are the defaults.
- One primary action per view; labels are verb phrases; icon-only controls have an accessible name.
- prism-audit is advisory. Fix what it reports by choosing the correct token role or component,
  never by matching the value. If a finding contradicts prism-design-rules.md or the props the
  package exports, follow those, leave the code as it is, and note the finding in the handover.
- Sample data is fictional; never real member names, account numbers or other end-customer data.
```

## What is not decided

- When the adherence check becomes a merge gate: after its rules are rebuilt from SRC-PRISM-UI-001 and reviewed
  by the product design owner, who also sets severities (DEC-020).
- The home of the adherence checker (a package in the Prism repository, or a Cash Intelligence tool).
- Which screens wait for component wave 1 (OQ-019).
- The chart library and theme (OQ-018).
