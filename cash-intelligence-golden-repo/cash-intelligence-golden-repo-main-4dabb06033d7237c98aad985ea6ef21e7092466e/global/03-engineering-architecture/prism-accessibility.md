---
id: GR-ARC-005
title: Prism 2.0 — Accessibility (verbatim accessibility.md)
artifact_type: platform-obligations
status: draft
content_state: source-backed
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 ai-resources/accessibility.md"
last_updated: 2026-11-19
review_due: TBC
---


> **Verbatim copy, draft.** Everything below this line is ai-resources/accessibility.md from the Prism 2.0 repository
> (`SRC-PRISM-UI-001`, branch `monorepo`, commit `8c4c80bb`, release 3.0.0, blob `26eb2d4a`), unchanged. It is refreshed
> from the Prism repository on each Prism release and never edited here (DEC-018). Relative links inside it point at
> the Prism repository, not at this one. Where it and the Prism repository differ, the repository wins.

# Prism accessibility

<!-- prism:stamp -->
> **Stamped 2026-11-06.** Describes `@kinective/prism-css` 3.0.0, `@kinective/prism-react` 3.0.0, `@kinective/prism-vue` 3.0.0, `@kinective/prism-contract` 3.0.0, `@kinective/prism-mcp` 3.0.0.
> Ark UI as declared: `@ark-ui/react` `5.40.0`, `@ark-ui/vue` `5.40.0`.
> Registry: `https://kinective.jfrog.io/artifactory/api/npm/kux-npm-dev-local/` (JFrog Artifactory, project `kux`; needs a token). Figma contract 1.0.0, file `Dbh3fLPGHxMLttXZVbpIMs`.
> Generated blocks come from `node ai-resources/build-agent-docs.mjs`; re-run it on every publish.
<!-- /prism:stamp -->

> **An authoring target, not a conformance claim.** Prism builds to WCAG 2.2 Level AA. Nothing in
> this file, in the packages, or in the contract certifies that Prism or a product built with it
> conforms to WCAG or to any law or regulation. Conformance can only be established by testing the
> assembled product with assistive technology and by expert review. **Kinective Legal must review
> any accessibility or conformance statement before it is relied on or made to a customer.**
> Where Prism's records cite a Level AAA criterion (2.3.3 Animation from Interactions appears in the
> motion facts), it is an extra goal and never inflates an AA claim.

These are the rules you build to. The per-component facts (role, name, keys, focus, announcements)
are in [`raw/a11y.psv`](raw/a11y.psv), one row per fact, and in the contract's `a11y` key. Both
are the 2026-10-01 export: where a component's states have changed since, [`components.md`](components.md)
is newer. `get_a11y_by_criterion` in `@kinective/prism-mcp` lists every component that owes a
given success criterion.

## Contrast

The contract carries a pair-by-pair contrast record measured on the Figma token values
(`get_tokens` / `check_drift`, or `contrast` in `prism.contract.json`). It is not a clean record:

- **`border/neutral/control` fails 1.4.11 non-text contrast** in both themes on both surfaces
  (1.87:1 to 2.14:1 against 3:1). It is the default boundary of an input, checkbox or radio. The fix
  is a design decision that is still open. Do not substitute another border token to make an audit
  pass, and do not describe Prism as having no contrast failures.
- **Deliberate exemptions.** `fg/neutral/disabled` is intentionally low: disabled controls are
  exempt from 1.4.3, and it must never be used for merely de-emphasised text (that is
  `fg/neutral/subtle`). `border/neutral/subtle` and `border/neutral/default` are decorative: they may
  separate content but must never be the only signal of a boundary, a control or an error. Anything
  load-bearing uses `border/neutral/strong`, `border/neutral/interactive` or a tone border.

## Never colour alone (1.4.1)

Every status carries a second signal:

| Component | Second signal beyond colour |
|---|---|
| Helper text (error) | Warning glyph plus the message text |
| Alert, Toast | Leading tone icon |
| Input, Textarea (focus, error) | Border weight change |
| Option item (selected) | Trailing check, not just a tint |
| Listbox (multiple) | A real Checkbox per row |
| Steps | Check glyph on completed steps |
| Badge | The label itself states the status |
| Link (inline) | Underline at rest |

## Focus (2.4.7, 1.4.11)

Focus is **additive**, never a variant. The focus ring (`Focus/Ring`, `--prism-effect-focus-ring`)
layers on top of whatever state the control is already in; `Focus/Ring Danger`,
`Focus/Ring Caution` and `Focus/Ring Success` are the same geometry in the tone colours.

- Never remove the ring without replacing it with something at least as visible.
- The ring must clear 3:1 against the adjacent background. Its dark-theme colour is deliberately
  bright because a blue ring disappears on a dark canvas.
- In menus and option lists, keyboard focus and hover share one visual state: there is only ever
  one active row.

## Touch targets (2.5.8)

Several controls render smaller than the 44px `size/touch-target` by design: small Buttons and Icon
Buttons, Close Button, Switch, Checkbox and Radio, and a Tag's remove control. Close the gap with
padding or a wrapping hit area (for Checkbox, Radio and Switch the label is part of the target),
never by enlarging the glyph. A Tag's remove control also needs its own accessible name.

## Structure and naming, per component

- **Field**: associate the label (`for`/`id`), put helper text in `aria-describedby`, set
  `aria-invalid` on error. The red border is never the only signal.
- **Fieldset**: a real `<fieldset>` with a `<legend>`. Radio Group and Checkbox Group always need one.
- **Label**: associate it, or clicking it will not focus the field. Sentence case, no trailing colon.
- **Icon Button**: always an `aria-label`; there is no visible text.
- **Tooltip**: never the only source of information. It is unavailable on touch and to many
  screen-reader configurations, and nothing focusable may live inside it.
- **Menu section header**: an ARIA group label, not a list item, or keyboard users land on it.
- **Option item (disabled)**: stays in the DOM and stays announced; skipped by arrow keys.
- **Spinner**: announces nothing on its own. Pair it with a text status ("Saving…").
- **Avatar**: the person's name is the alternative text. An avatar is never decorative.
- **Scrim, Dialog, Drawer**: trap focus above the scrim, return it to the trigger on close, close
  on Escape. Do not dismiss on scrim click when there is unsaved input.
- **Password input**: the reveal toggle's name changes between "Show password" and "Hide
  password". Never block paste; password managers depend on it.

## Motion (2.3.3, 2.2.2)

- Honour `prefers-reduced-motion`: drop the Marquee scroll, Carousel auto-advance and Skeleton pulse
  to a static state, and collapse exit transitions to none. Keep state changes (a tone change on
  hover) because they are feedback, not decoration.
- Skeleton is a slow opacity pulse, never a fast shimmer.

## Dark mode

The semantic layer carries both themes, so theming is one attribute. Two things do not translate
automatically and are handled in the tokens: shadows stop separating surfaces on a dark canvas, so
raised surfaces also carry a border; and the scrim is deeper in dark (`bg/neutral/overlay`).
