---
id: GR-ARC-007
title: Prism 2.0 — Component Availability
artifact_type: platform-obligations
status: draft
content_state: derived
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 CHANGELOG.md §3.0.0"
  - "SRC-PRISM-UI-001 ai-resources/design.md §Components"
  - "SRC-PRISM-UI-001 docs/reference/README.md §What is still to come"
  - "SRC-CLAUDE-PRISM-001 §Figma-only"
  - "SRC-CLAUDE-PRISM-001 §Templates"
last_updated: 2026-11-19
review_due: TBC
---


# Prism 2.0 — Component Availability

**Derived, draft.** Which Prism components a Cash Intelligence screen can be built from today, and which it cannot.
The full list of the 95 shipped components is the `Components` table in `prism-design-rules.md`; it is not
repeated here. State as of Prism release 3.0.0 (2026-11-13) and the Claude Design kit version 44 (2026-10-22).

## Shipped and matching the Figma contract (9)

Badge, Button, Close Button, Divider, Helper text, Icon Button, Kbd, Label, Tooltip
(SRC-PRISM-UI-001 CHANGELOG.md §3.0.0). The component table in `prism-design-rules.md` still lists Helper text
and Label as "update in an open merge request"; the release notes are the later statement and win until the
table is corrected and recopied (DEC-018).

## Shipped with a Figma-contract update not yet released (1)

Dialog: update in merge request !33, not part of 3.0.0. Status to be confirmed with the product design owner.

## Shipped as legacy (85)

Every other shipped component, including the ones Cash Intelligence screens lean on most: Data table, Sidebar,
Topbar, Page heading, Select, Date input, Date picker, Field, Input, Number input, Card, Tabs, Pagination,
Breadcrumb, Empty state, Skeleton, Spinner, Progress, Progress ring, Switch, Checkbox, Radio Group, Menu, Drawer,
Toast. A legacy component keeps its original API until its update merges; build with the props its package
exports today (SRC-PRISM-UI-001 ai-resources/AGENTS.md §Which components match Figma).

## In the Figma-matched reference build only, not published (18)

Color swatch, Command item, Customize sidebar, Field group, Level header, Login, Multi-factor challenge, Nav group
header, Nav item, Password reset, Rail tile, Reorder row, Sticky action bar, Table bar, Table cell, Table footer,
Table header cell, Table row (SRC-PRISM-UI-001 docs/reference/README.md §What is still to come). They arrive as
new components following the repository's conventions; no date is stated.

## In Figma only, no code anywhere (20)

Quick action, Preview header, Detail line, Description item, Choice tile, Choice card, Promo card; the seven
Workflow organisms (Workflow node, Node icon, Connector, Step panel, Workflow canvas, Canvas controls, Panel
section); the six AI organisms (AI summary, Prompt input, Tool call card, Confirmation card, Question card, AI
edge) (`SRC-CLAUDE-PRISM-001 §Figma-only`).

## Page templates (11, compositions, not code)

The reference build documents eleven page templates as compositions to copy, not exports: App shell, Dashboard,
Operations console, Workspace home, Settings, Catalog, Activity log, Object detail, Workflow builder, Login
(signed-out), and the empty and loading variants that go with them (`SRC-CLAUDE-PRISM-001 §Templates`). Several
compose components from the two lists above; a template is only buildable once its components ship.

## What this means for a Cash Intelligence spec

- Name the component from the shipped list and state its status; a spec that names a reference-only or
  Figma-only component records it as a gap (OQ-019, OQ-020) and the composition chosen instead.
- Do not imitate a missing component with custom styling, and do not copy a Figma property onto a legacy
  component as if it were a prop (`prism-design-rules.md §Working from a product Figma file`).
- Specify against Prism 3.0.0's components; the build uses the template's vendored Prism copy until the DEC-023
  switch, then the first qualifying release. Which screens wait for component wave 1 is open (OQ-019).
