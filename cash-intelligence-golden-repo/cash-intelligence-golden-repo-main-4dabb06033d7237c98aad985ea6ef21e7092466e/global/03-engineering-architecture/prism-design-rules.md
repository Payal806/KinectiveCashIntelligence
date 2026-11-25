---
id: GR-ARC-004
title: Prism 2.0 — Design Rules and Vocabulary (verbatim design.md)
artifact_type: platform-obligations
status: draft
content_state: source-backed
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 ai-resources/design.md"
last_updated: 2026-11-19
review_due: TBC
---


> **Verbatim copy, draft.** Everything below this line is ai-resources/design.md from the Prism 2.0 repository
> (`SRC-PRISM-UI-001`, branch `monorepo`, commit `8c4c80bb`, release 3.0.0, blob `885dabf8`), unchanged. It is refreshed
> from the Prism repository on each Prism release and never edited here (DEC-018). Relative links inside it point at
> the Prism repository, not at this one. Where it and the Prism repository differ, the repository wins.

# Prism design.md

<!-- prism:stamp -->
> **Stamped 2026-11-06.** Describes `@kinective/prism-css` 3.0.0, `@kinective/prism-react` 3.0.0, `@kinective/prism-vue` 3.0.0, `@kinective/prism-contract` 3.0.0, `@kinective/prism-mcp` 3.0.0.
> Ark UI as declared: `@ark-ui/react` `5.40.0`, `@ark-ui/vue` `5.40.0`.
> Registry: `https://kinective.jfrog.io/artifactory/api/npm/kux-npm-dev-local/` (JFrog Artifactory, project `kux`; needs a token). Figma contract 1.0.0, file `Dbh3fLPGHxMLttXZVbpIMs`.
> Generated blocks come from `node ai-resources/build-agent-docs.mjs`; re-run it on every publish.
<!-- /prism:stamp -->

You are building product UI with Prism, Kinective's design system. This file carries the judgment
to compose a screen, the closed vocabulary to build it from, and the list of components with each
one's status in this repository. Install and import instructions are in [`AGENTS.md`](AGENTS.md).

**Values are deliberately absent.** You do not need to know what `bg/primary/default` resolves to:
emit `var(--prism-bg-primary-default)` and the stylesheet resolves it at render time, in whichever
theme and density the page is running. If you need a resolved value for a contrast judgement, ask
`get_tokens` in `@kinective/prism-mcp`; do not inline it.

## The two hard rules

1. **Bind to the semantic layer, never to a primitive.** The names in [the vocabulary](#the-vocabulary)
   are the whole public API. Primitives (`color/blue/600` and friends) are private.
2. **Never invent a name.** If what you need is not below, stop and ask. A missing token is a
   design-system decision, not an implementation detail.

## Priority order

When these conflict, the higher one wins. Most bad UI comes from resolving a conflict at the wrong
level rather than from not knowing a rule.

1. **Correctness of the data.** Never round, truncate or reformat a number, name or status to make
   a layout work. Change the layout.
2. **Accessibility.** A keyboard path, an accessible name, and a visible focus ring are not
   negotiable against visual preference.
3. **The token and component contract.** Use what exists. A missing token is a design decision,
   not an implementation detail: stop and ask.
4. **Consistency with the rest of the product.** A screen that is locally elegant and globally
   foreign is a defect.
5. **Refinement.** Spacing, alignment, and polish, once the four above hold.

If a requirement is ambiguous and the answer changes the structure of the screen, ask. If it is
ambiguous and only changes a detail, choose the more conservative option and label the assumption.

## Work in four passes

Do not start by placing elements. Four passes, in order.

**1. Frame the task.** Privately establish who uses this screen, what they are trying to
accomplish, what they need to see to decide, and what they must not be able to do by accident.
Destructive actions, irreversible submissions and money movement change the composition; find them
now, not after the layout exists.

**2. Choose the composition.** Pick the shape from the work, not from a template. A list of things
to triage is not the same screen as a single record to edit, which is not the same as a form to
complete. Reject your first instinct if it is a grid of equal cards: that shape is almost always
the absence of a decision about what matters most.

**3. Build with the vocabulary.** Compose from existing components and semantic tokens. If you are
reaching past the vocabulary, the answer is to ask, not to improvise.

**4. Inspect privately, then deliver.** Run the checklist below. Fix the systemic problems, not the
cosmetic ones. Do not narrate the inspection or hand over a critique; deliver the corrected work.

## Working from a product Figma file

Sometimes a prototype comes with its own Figma file, sometimes only with a brief. When there is a
file, it becomes the source of truth for **that screen**, but not for Prism. The split:

| The product file owns | Prism owns |
|---|---|
| What is on the screen, in what order, and what it says | Every token: colour, space, size, radius, type, motion, effect |
| The flow between screens | Every component, its variants and its behaviour |
| Copy, content and sample data | Accessibility: roles, names, keys, focus, announcements |

So when the frame and Prism disagree, **use the Prism version and say so**. Build the layout the
frame shows, out of Prism components and tokens, and list every divergence in your handover ("the
frame uses a detached button with a literal blue fill; built as `Button` variant primary"). Do not
quietly pick one.

- **Read the bindings, do not guess them.** If the frame binds a Prism token, use that token. If it
  shows a literal colour or a raw number, that is a divergence to report: find the semantic token
  by role, never by matching the value.
- **A detached or local copy of a Prism component is still that component.** Build the real one.
- **Ignore project-local collections.** Anything marked `(Local)`, or prefixed with the project's
  name, belongs to that product. Never alias a Prism token into it. If the product needs its own
  tokens, give them the product's prefix and declare them on `:root, [data-theme], [data-density]`.
- **Dev Mode code syntax is a hint, not the contract.** The names in [the vocabulary](#the-vocabulary)
  are what exist.
- **A Figma property is not a code prop until the component is on the Figma contract.** Check the
  component's status in [the component list](#components). For a legacy component, use the props its
  package exports today.

## Composition

- **One primary action per view.** If two actions both look primary, neither is. Everything else
  steps down through Secondary and Tertiary emphasis.
- **Group by decision, not by data type.** Fields that are read together belong together, whatever
  their types.
- **Give the eye one column of prose.** Long-form reading text does not span a wide container; put
  it in a bounded measure and let tables and charts take the full width.
- **Tables are full-bleed within their container, never truncated.** If a table does not fit, remove
  columns deliberately or let it scroll in its own region. Never clip a value or use an ellipsis
  where a number belongs.
- **Density is a mode, not a design.** Do not hand-tighten spacing to fit more in. Set
  `data-density` and let the token layer re-resolve.
- **Empty, loading and error states are part of the composition.** A screen designed only in its
  happy state is unfinished.

## Depth and surface

Prism has three planes. The app frame never casts a shadow, the page sheet casts exactly one, and
everything else that does sits on the page.

- **The app frame**: the sidebar. No shadow, ever. It sits straight on the
  `bg/neutral/surface-sunken` canvas, and that ground is what holds it back.
- **The page sheet**: the content surface, `bg/neutral/surface`. It casts `Elevation/02 Card`
  and never takes a heavier step. The topbar is the sheet's first row, not a strip on the canvas
  above it, so it casts nothing of its own; the page content sits below it inside the same sheet.
- **Things on the page**: cards, inputs, raised controls, and above them overlays. Each takes the
  smallest step on the ramp that clears the panel: a card is `Elevation/02 Card`, an input or
  raised control `Elevation/Control Raised`, and popovers, menus and dialogs climb to
  `Elevation/04 Overlay` through `Elevation/06 Modal`.

In CSS each effect style is one token, `--prism-effect-*` (for example
`--prism-effect-elevation-card`), declared so a nested theme or density re-resolves it. Never
compose a shadow from the elevation dimension tokens yourself, never author a bespoke shadow, and
never stack two effect styles on one element to get both a shadow and a blur: nest an element.

## Emphasis, not hue

`Button` and `Icon Button` take a `variant` of `primary`, `secondary`, `tertiary`,
`secondary-neutral`, `tertiary-neutral`, `danger` or `success`. Primary, Secondary and Tertiary are
*emphasis levels* on the **primary blue** ramp, not the teal and navy colour roles. Never build
button hierarchy out of `bg/secondary/default` or `bg/tertiary/default`; those are for accents,
tags, charts and marketing surfaces.

The two neutral variants are the Secondary and Tertiary shapes on the neutral ramp, for toolbars and
dense rows where only the one Primary should carry the brand colour. Danger and Success carry a
status meaning: Danger for a destructive action, Success for an affirmative decision that commits
an outcome (Approve), never for an ordinary primary action.

Tone tokens (`bg/danger/default`, `bg/warning/default`, `bg/success/default`, `bg/info/default`)
encode **state**, never rank. A button is not red because it is important; it is red because the
action destroys something.

## Choosing between components

| Use | Not | Because |
|---|---|---|
| `Field` | bare `Input` | Field wires label, control and helper so validation state cannot disagree with itself |
| `Switch` | `Checkbox` | Switch takes effect immediately. Checkbox is saved on submit |
| `Badge` | `Tag` | Badge is system status, read-only. Tag is user data, removable |
| `Popover` / `Hover Card` | `Tooltip` | Nothing focusable may live in a Tooltip: it vanishes on blur |
| `Progress` / `Skeleton` | `Spinner` | Past roughly ten seconds a spinner tells the user nothing |
| `Skeleton` | `Spinner` | First load. When refreshing data already on screen, keep the content and use a Spinner |
| `Alert` | `Progress` in a danger tone | A failed upload is an Alert, not a red progress bar |
| `Table of Contents` | `Tabs` | A ToC reflects reading position; it is not navigation |
| `Surface` | a hand-picked elevation | If you are choosing a shadow by hand you have skipped Surface |

## State and interaction

- Use the component's own states. Do not hand-tint a hover or a pressed state.
- **Focus is additive.** The focus ring (`Focus/Ring`, `--prism-effect-focus-ring`) layers on top of
  whatever state the control is already in. There is deliberately no focus variant.
- `Read-only` and `Disabled` are not interchangeable. Read-only keeps the value selectable and
  copyable; Disabled removes interaction entirely.
- Status is never colour alone. Every tone ships a second signal: an icon, a glyph, a border-weight
  change, or a check.
- Overlays trap focus, return it to the trigger on close, and close on Escape. Do not dismiss on
  scrim click when there is unsaved input.

## Reject generated-UI reflexes

These are the defaults that make an interface look generated. None of them is forbidden by the
token layer, which is exactly why they need saying.

- A centred hero followed by a grid of three equal cards.
- A dashboard built from a row of identical metric boxes, none more important than the others.
- Cards nested inside cards. One border, one surface, one level.
- A shadow on the app frame, or anything heavier than `Elevation/02 Card` on the page sheet.
- Decorative gradients.
- Icon tiles: a large rounded square containing a glyph, used as ornament.
- ALL-CAPS eyebrow labels above headings. Prism is sentence case throughout.
- Tiny prose. If text is smaller than `Body/S`, it is either not important or not readable.
- A tooltip carrying information available nowhere else.
- Truncated table cells hiding the value the table exists to show.
- A summary section that restates the section above it in different words.
- Emoji as status indicators.
- Charts styled for decoration: gradient fills, drop shadows, 3D, non-zero baselines.
- Placeholder text used as a label.
- A spinner for an operation with a knowable duration.
- Narrating your own process in the delivered UI or its copy.

## Copy

Sentence case everywhere except proper nouns and component names. No trailing colons on labels.
Buttons say what they do (`Save changes`, not `Submit` or `OK`). Error messages state what happened
and what to do next, never just "Invalid input." Details in [`casing.md`](casing.md). Never put real
member names, account numbers or any other personally identifiable data belonging to a financial
institution's end customers into copy, fixtures, stories or screenshots; all Prism sample data is
fictional.

## Two typefaces

In the Figma contract, Inter is the UI typeface and Haffer the display typeface. `font/family/sans`
(Inter) is for everything: body, labels, controls, captions and the smaller headings.
`font/family/display` (Haffer) is for page and display titles only: the Display L, M and S and
Heading XL and L text styles. `font/family/mono` is for code and identifiers.

In this repo, prefer the text-style tokens (`--prism-text-{style}-font-family`, `-font-size`,
`-font-weight`, `-line-height`, `-letter-spacing`) over the raw family and weight tokens: they
carry the right pairing. `--prism-font-family-sans` resolves to Inter and `--prism-font-weight-*`
to numbers only when `prismFigma.css` is loaded last, as [`AGENTS.md`](AGENTS.md) shows. Without
it, the older token files make sans Haffer and the weights face names ("Regular").

**Weights do not cross families.** `font/weight/*` pairs with `font/family/sans` and `mono`;
`font/weight/display/*` pairs with `font/family/display`.

## What is checked, and what is not

For the components that have a parity test (`packages/prism-css/tests/*.parity.test.ts`), the
test rejects a stylesheet that uses a literal colour, length or typography value, or binds a token
the foundation does not define. **Nothing checks your application code.** These are the failures that are silent there, so check them yourself:

| Rule | Why it is silent |
|---|---|
| Unknown token | CSS does not error on an undefined custom property. It resolves to nothing and the element renders unstyled |
| Hardcoded colour | A hex, `rgb()`, `hsl()` or named colour where a semantic token belongs. It looks right in one theme only |
| Raw length | A `px`, `rem` or `em` number where a `space`, `size`, `radius` or `border-width` token belongs. It looks right in one density only |
| Token redefinition | Re-declaring a real Prism token desynchronises theme and density |
| Literal fallback | `var(--prism-…, <value>)` renders a stale literal after a rename instead of failing visibly |
| Root alias | Your own custom property built from Prism tokens and declared on `:root` only resolves once and freezes the root theme and density into every nested region |
| Raw motion | A literal duration or `cubic-bezier()` where a `duration/*` or `easing/*` token belongs |

Prism has no layer (`z-index`) tokens yet. If you set one, say which value you chose and why.

## Inspect before you deliver

Answer each of these privately. Fix what fails, then hand over the work, not the review.

1. **First read.** Is the most important thing on the screen also the most visually prominent?
2. **One primary action.** Exactly one, and it is the action the user came to perform.
3. **Vocabulary.** Every colour, space, radius and duration is a semantic token in the list below,
   and not in its "Not yet defined" list. No hex, no raw px, no primitive, no invented name.
4. **Depth.** Nothing in the app frame casts a shadow, and the page sheet casts only
   `Elevation/02 Card`. Cards, inputs and overlays use their own step on the ramp, and no heavier.
5. **Keyboard.** Every interactive element is reachable, in a sensible order, with a visible focus
   ring, and no trap you cannot leave.
6. **Names.** Every icon-only control has an accessible name. Every input's label is associated.
7. **States.** Empty, loading, error and disabled all exist and all read correctly.
8. **Both themes, all four densities.** Nothing overflows, clips, or loses contrast.
9. **Colour alone.** Every status carries a second signal.
10. **Restraint.** Remove one decorative thing. If the screen is not worse, leave it removed.

## Known gaps: do not paper over them

- **`border/neutral/control` fails contrast** in both themes on both surfaces: four measured
  pairings, 1.87:1 to 2.14:1 against a 3:1 requirement (the contract's `contrast` record). Do not
  describe Prism as having a clean contrast record, and do not substitute a different border token
  to make an audit pass.
- **Most components are still legacy.** See [the component list](#components). A legacy
  component's props and look differ from its Figma component; that is expected until its update
  merges.
- **Gradients do not re-theme.** Figma cannot bind a variable to a gradient stop, so gradients in
  `Marquee`, `Carousel`, `Color picker` and `Image cropper` hold resolved values.
- **`QR Code` ships a placeholder pattern.** Replace it with real encoded output before anything is
  meant to be scanned.
- **Motion tokens are authored in Figma only.** Use `duration/*` and `easing/*` like any other
  token; never hardcode `150ms` or a raw cubic-bezier, and never add a parallel duration token.
  Easing tokens are named for their use, not their curve: `easing/enter` decelerates because
  entrances do.

## The target

A screen that a person can act on confidently, at speed, without being taught. Everything above is
in service of it.

## Components

<!-- prism:components -->
95 components ship in both bindings. **7 match the Figma contract**; 3 have an update in an open merge request; the rest are legacy (built before the Figma contract and still on their original API).

Every one is a named export of `@kinective/prism-react` and `@kinective/prism-vue`, with the
same export name in both.

| Component (Figma name) | Export | Behaviour | Status |
|---|---|---|---|
| Accordion | `Accordion` | Ark `accordion` | legacy |
| Alert | `Alert` | tier 3 | legacy |
| Angle Slider | `AngleSlider` | Ark `angle-slider` | legacy |
| Avatar | `Avatar` | Ark `avatar` | legacy |
| Avatar Group | `AvatarGroup` | Ark `avatar` | legacy |
| Badge | `Badge` | tier 3 | Figma contract |
| Breadcrumb | `Breadcrumb` | tier 3 | legacy |
| Button | `Button` | tier 3 | Figma contract |
| Card | `Card` | tier 3 | legacy |
| Carousel | `Carousel` | Ark `carousel` | legacy |
| Checkbox | `Checkbox` | Ark `checkbox` | legacy |
| Checkbox Group | `CheckboxGroup` | Ark `checkbox` | legacy |
| Clipboard | `Clipboard` | Ark `clipboard` | legacy |
| Close Button | `CloseButton` | tier 3 | Figma contract |
| Collapsible | `Collapsible` | Ark `collapsible` | legacy |
| Color picker | `ColorPicker` | Ark `color-picker` | legacy |
| Combobox | `Combobox` | Ark `combobox` | legacy |
| Command palette | `CommandPalette` | Ark `dialog+combobox` | legacy |
| Crumb | `Crumb` | tier 3 | legacy |
| Data table | `DataTable` | tier 3 | legacy |
| Date input | `DateInput` | Ark `date-input` | legacy |
| Date picker | `DatePicker` | Ark `date-picker` | legacy |
| Dialog | `Dialog` | Ark `dialog` | legacy; update in !33 |
| Divider | `Divider` | tier 3 | Figma contract |
| Divider with label | `DividerWithLabel` | tier 3 | legacy |
| Download Trigger | `DownloadTrigger` | Ark `download-trigger` | legacy |
| Drawer | `Drawer` | Ark `drawer` | legacy |
| Editable | `Editable` | Ark `editable` | legacy |
| Empty state | `EmptyState` | tier 3 | legacy |
| Field | `Field` | Ark `field` | legacy |
| Fieldset | `Fieldset` | Ark `fieldset` | legacy |
| File upload | `FileUpload` | Ark `file-upload` | legacy |
| Floating Panel | `FloatingPanel` | Ark `floating-panel` | legacy |
| Format | `Format` | Ark `format` | legacy |
| Helper text | `HelperText` | Ark `field` | legacy; update in !24 |
| Highlight | `Highlight` | Ark `highlight` | legacy |
| Hover Card | `HoverCard` | Ark `hover-card` | legacy |
| Icon Button | `IconButton` | tier 3 | Figma contract |
| Image cropper | `ImageCropper` | Ark `image-cropper` | legacy |
| Input | `Input` | tier 3 | legacy |
| JSON Tree View | `JSONTreeView` | Ark `json-tree-view` | legacy |
| Kbd | `Kbd` | tier 3 | Figma contract |
| Label | `Label` | Ark `field` | legacy; update in !27 |
| Link | `Link` | tier 3 | legacy |
| Listbox | `Listbox` | Ark `listbox` | legacy |
| Marquee | `Marquee` | Ark `marquee` | legacy |
| Menu | `Menu` | Ark `menu` | legacy |
| Menu item | `MenuItem` | Ark `menu` | legacy |
| Menu section header | `MenuSectionHeader` | Ark `menu` | legacy |
| Number input | `NumberInput` | Ark `number-input` | legacy |
| Option item | `OptionItem` | Ark `listbox` | legacy |
| Page heading | `PageHeading` | tier 3 | legacy |
| Page item | `PageItem` | Ark `pagination` | legacy |
| Pagination | `Pagination` | Ark `pagination` | legacy |
| Password input | `PasswordInput` | Ark `password-input` | legacy |
| Pin input | `PinInput` | Ark `pin-input` | legacy |
| Popover | `Popover` | Ark `popover` | legacy |
| Progress | `Progress` | Ark `progress` | legacy |
| Progress ring | `ProgressRing` | Ark `progress` | legacy |
| QR Code | `QRCode` | Ark `qr-code` | legacy |
| Radio | `Radio` | Ark `radio-group` | legacy |
| Radio Group | `RadioGroup` | Ark `radio-group` | legacy |
| Rating Group | `RatingGroup` | Ark `rating-group` | legacy |
| Scrim | `Scrim` | Ark `dialog` | legacy |
| Scroll Area | `ScrollArea` | Ark `scroll-area` | legacy |
| Segment Group | `SegmentGroup` | Ark `segment-group` | legacy |
| Select | `Select` | Ark `select` | legacy |
| Sidebar | `Sidebar` | tier 3 | legacy |
| Signature pad | `SignaturePad` | Ark `signature-pad` | legacy |
| Skeleton | `Skeleton` | tier 3 | legacy |
| Slider | `Slider` | Ark `slider` | legacy |
| Spinner | `Spinner` | tier 3 | legacy |
| Splitter | `Splitter` | Ark `splitter` | legacy |
| Step marker | `StepMarker` | Ark `steps` | legacy |
| Steps | `Steps` | Ark `steps` | legacy |
| Surface | `Surface` | tier 3 | legacy |
| Swap | `Swap` | Ark `swap` | legacy |
| Switch | `Switch` | Ark `switch` | legacy |
| Tab item | `TabItem` | Ark `tabs` | legacy |
| Table of Contents | `TableOfContents` | Ark `toc` | legacy |
| Tabs | `Tabs` | Ark `tabs` | legacy |
| Tag | `Tag` | tier 3 | legacy |
| Tags input | `TagsInput` | Ark `tags-input` | legacy |
| Textarea | `Textarea` | tier 3 | legacy |
| Timer | `Timer` | Ark `timer` | legacy |
| Toast | `Toast` | Ark `toast` | legacy |
| Toast region | `ToastRegion` | Ark `toast` | legacy |
| ToC item | `ToCItem` | Ark `toc` | legacy |
| Toggle | `Toggle` | Ark `toggle` | legacy |
| Toggle Group | `ToggleGroup` | Ark `toggle-group` | legacy |
| Toolbar | `Toolbar` | tier 3 | legacy |
| Tooltip | `Tooltip` | Ark `tooltip` | Figma contract |
| Topbar | `Topbar` | tier 3 | legacy |
| Tour | `Tour` | Ark `tour` | legacy |
| Tree View | `TreeView` | Ark `tree-view` | legacy |

Open merge requests that are not about a single component:

- **!32**: CI runs on merge requests, plus a typecheck job
- **!34**: Design decisions record (D1-D37) and the current Figma exports under docs/reference/
<!-- /prism:components -->

## The vocabulary

<!-- prism:vocabulary -->
Every semantic token in the Figma contract, by Figma name. The CSS custom property is the name
with `/` turned into `-` and a `--prism-` prefix: `bg/primary/default` is
`var(--prism-bg-primary-default)`. Bind these and nothing else.

### `ai/*`: 20

`ai/edge-wash` · `ai/fill-end` · `ai/fill-mid` · `ai/fill-start` · `ai/label-from` · `ai/label-to` · `ai/orb-deep` · `ai/orb-deep-glow` · `ai/orb-light` · `ai/orb-mid` · `ai/spectrum-1` · `ai/spectrum-10` · `ai/spectrum-2` · `ai/spectrum-3` · `ai/spectrum-4` · `ai/spectrum-5` · `ai/spectrum-6` · `ai/spectrum-7` · `ai/spectrum-8` · `ai/spectrum-9`

### `alpha/*`: 16

`alpha/ai-glow-deep` · `alpha/ai-wash-rest` · `alpha/badge-border` · `alpha/badge-dot` · `alpha/highlight` · `alpha/invisible` · `alpha/ring` · `alpha/scrim` · `alpha/shadow-default` · `alpha/shadow-emphasis` · `alpha/shadow-medium` · `alpha/shadow-soft` · `alpha/shadow-strong` · `alpha/sidebar-edge` · `alpha/sidebar-mid` · `alpha/translucent`

### `badge/*`: 23

`badge/danger/bg` · `badge/danger/border` · `badge/danger/fg` · `badge/info/bg` · `badge/info/bg-solid` · `badge/info/border` · `badge/info/fg` · `badge/info/fg-solid` · `badge/neutral/border` · `badge/primary/bg` · `badge/primary/border` · `badge/primary/fg` · `badge/success/bg` · `badge/success/border` · `badge/success/fg` · `badge/violet/bg` · `badge/violet/bg-solid` · `badge/violet/border` · `badge/violet/fg` · `badge/violet/fg-solid` · `badge/warning/bg` · `badge/warning/border` · `badge/warning/fg`

### `bg/*`: 50

`bg/danger/active` · `bg/danger/default` · `bg/danger/hover` · `bg/danger/subtle` · `bg/danger/subtle-active` · `bg/danger/subtle-hover` · `bg/info/default` · `bg/info/subtle` · `bg/neutral/active` · `bg/neutral/canvas` · `bg/neutral/canvas-invisible` · `bg/neutral/canvas-translucent` · `bg/neutral/disabled` · `bg/neutral/hover` · `bg/neutral/inverse` · `bg/neutral/muted` · `bg/neutral/overlay` · `bg/neutral/selected` · `bg/neutral/skeleton` · `bg/neutral/subtle` · `bg/neutral/surface` · `bg/neutral/surface-raised` · `bg/neutral/surface-raised-translucent` · `bg/neutral/surface-sunken` · `bg/neutral/track` · `bg/primary/accent` · `bg/primary/accent-subtle` · `bg/primary/active` · `bg/primary/default` · `bg/primary/hover` · `bg/primary/subtle` · `bg/primary/subtle-active` · `bg/primary/subtle-hover` · `bg/secondary/accent` · `bg/secondary/active` · `bg/secondary/default` · `bg/secondary/hover` · `bg/secondary/subtle` · `bg/success/active` · `bg/success/default` · `bg/success/hover` · `bg/success/subtle` · `bg/success/subtle-active` · `bg/success/subtle-hover` · `bg/tertiary/active` · `bg/tertiary/default` · `bg/tertiary/hover` · `bg/tertiary/subtle` · `bg/warning/default` · `bg/warning/subtle`

### `border/*`: 24

`border/accent/subtle` · `border/danger/default` · `border/danger/subtle` · `border/info/subtle` · `border/neutral/control` · `border/neutral/decorative` · `border/neutral/default` · `border/neutral/disabled` · `border/neutral/hover` · `border/neutral/interactive` · `border/neutral/inverse` · `border/neutral/strong` · `border/neutral/subtle` · `border/primary/default` · `border/primary/focus` · `border/primary/subtle` · `border/secondary/default` · `border/secondary/subtle` · `border/success/default` · `border/success/subtle` · `border/tertiary/default` · `border/tertiary/subtle` · `border/warning/default` · `border/warning/subtle`

### `border-width/*`: 4

`border-width/focus` · `border-width/medium` · `border-width/thick` · `border-width/thin`

### `chart/*`: 8

`chart/1` · `chart/2` · `chart/3` · `chart/4` · `chart/5` · `chart/6` · `chart/7` · `chart/8`

### `cursor/*`: 2

`cursor/body` · `cursor/outline`

### `duration/*`: 13

`duration/alert` · `duration/context-enter` · `duration/dwell` · `duration/enter` · `duration/exit` · `duration/expand` · `duration/hover-intent` · `duration/instant` · `duration/large` · `duration/loop-fast` · `duration/loop-slow` · `duration/micro` · `duration/zoom-enter`

### `easing/*`: 6

`easing/attention` · `easing/enter` · `easing/exit` · `easing/fade` · `easing/move` · `easing/overshoot`

### `effect/*`: 17

`effect/ai/glow` · `effect/ai/hover-glow` · `effect/blur/backdrop` · `effect/elevation/card` · `effect/elevation/card-hover` · `effect/elevation/control-raised` · `effect/elevation/control-raised-with-depth` · `effect/elevation/hairline` · `effect/elevation/modal` · `effect/elevation/overlay` · `effect/elevation/popover` · `effect/elevation/raised` · `effect/focus/ring` · `effect/focus/ring-caution` · `effect/focus/ring-danger` · `effect/focus/ring-success` · `effect/inner/subtle`

### `elevation/*`: 53

`elevation/backdrop/blur` · `elevation/card-hover/1/blur` · `elevation/card-hover/1/offset-y` · `elevation/card-hover/1/spread` · `elevation/card-hover/2/blur` · `elevation/card-hover/2/offset-y` · `elevation/card-hover/2/spread` · `elevation/card/1/blur` · `elevation/card/1/offset-y` · `elevation/card/1/spread` · `elevation/card/2/blur` · `elevation/card/2/offset-y` · `elevation/card/2/spread` · `elevation/control/blur` · `elevation/control/inner-blur` · `elevation/control/inner-offset-y` · `elevation/control/inner-spread` · `elevation/control/offset-y` · `elevation/control/spread` · `elevation/focus/blur` · `elevation/focus/offset-y` · `elevation/focus/spread` · `elevation/glass/blur` · `elevation/hairline/blur` · `elevation/hairline/offset-y` · `elevation/hairline/spread` · `elevation/inner/blur` · `elevation/inner/offset-y` · `elevation/inner/spread` · `elevation/modal/1/blur` · `elevation/modal/1/offset-y` · `elevation/modal/1/spread` · `elevation/modal/2/blur` · `elevation/modal/2/offset-y` · `elevation/modal/2/spread` · `elevation/overlay/1/blur` · `elevation/overlay/1/offset-y` · `elevation/overlay/1/spread` · `elevation/overlay/2/blur` · `elevation/overlay/2/offset-y` · `elevation/overlay/2/spread` · `elevation/popover/1/blur` · `elevation/popover/1/offset-y` · `elevation/popover/1/spread` · `elevation/popover/2/blur` · `elevation/popover/2/offset-y` · `elevation/popover/2/spread` · `elevation/raised/1/blur` · `elevation/raised/1/offset-y` · `elevation/raised/1/spread` · `elevation/raised/2/blur` · `elevation/raised/2/offset-y` · `elevation/raised/2/spread`

### `fg/*`: 26

`fg/accent/on-accent` · `fg/danger/default` · `fg/danger/on-danger` · `fg/info/default` · `fg/info/on-info` · `fg/neutral/default` · `fg/neutral/disabled` · `fg/neutral/faint` · `fg/neutral/inverse` · `fg/neutral/muted` · `fg/neutral/placeholder` · `fg/neutral/subtle` · `fg/primary/default` · `fg/primary/hover` · `fg/primary/link` · `fg/primary/link-hover` · `fg/primary/link-visited` · `fg/primary/on-primary` · `fg/secondary/default` · `fg/secondary/on-secondary` · `fg/success/default` · `fg/success/on-success` · `fg/tertiary/default` · `fg/tertiary/on-tertiary` · `fg/warning/default` · `fg/warning/on-warning`

### `font/*`: 27

`font/family/display` · `font/family/mono` · `font/family/sans` · `font/size/2xl` · `font/size/2xs` · `font/size/3xl` · `font/size/4xl` · `font/size/5xl` · `font/size/6xl` · `font/size/7xl` · `font/size/control/lg` · `font/size/control/md` · `font/size/control/sm` · `font/size/control/xs` · `font/size/lg` · `font/size/md` · `font/size/sm` · `font/size/xl` · `font/size/xs` · `font/weight/bold` · `font/weight/display/bold` · `font/weight/display/medium` · `font/weight/display/regular` · `font/weight/display/semibold` · `font/weight/medium` · `font/weight/regular` · `font/weight/semibold`

### `illustration/*`: 1

`illustration/accent-wash`

### `letter-spacing/*`: 10

`letter-spacing/body` · `letter-spacing/display` · `letter-spacing/heading` · `letter-spacing/heading-lg` · `letter-spacing/heading-sm` · `letter-spacing/heading-xl` · `letter-spacing/label` · `letter-spacing/none` · `letter-spacing/wide` · `letter-spacing/wider`

### `line-height/*`: 20

`line-height/2xl` · `line-height/2xs` · `line-height/3xl` · `line-height/4xl` · `line-height/5xl` · `line-height/6xl` · `line-height/7xl` · `line-height/control/lg` · `line-height/control/md` · `line-height/control/sm` · `line-height/control/xs` · `line-height/lg` · `line-height/md` · `line-height/sm` · `line-height/tight-lg` · `line-height/tight-md` · `line-height/tight-sm` · `line-height/tight-xl` · `line-height/xl` · `line-height/xs`

### `logo/*`: 11

`logo/accent` · `logo/gradient-1` · `logo/gradient-2` · `logo/gradient-3` · `logo/gradient-4` · `logo/ink` · `logo/ink-fixed` · `logo/ink-secondary` · `logo/plate` · `logo/plate-edge` · `logo/shade`

### `radius/*`: 11

`radius/2xl` · `radius/2xs` · `radius/3xl` · `radius/3xs` · `radius/full` · `radius/lg` · `radius/md` · `radius/none` · `radius/sm` · `radius/xl` · `radius/xs`

### `scale/*`: 1

`scale/zoom-from`

### `shadow/*`: 10

`shadow/default` · `shadow/emphasis` · `shadow/inner-highlight` · `shadow/medium` · `shadow/ring` · `shadow/ring-caution` · `shadow/ring-danger` · `shadow/ring-success` · `shadow/soft` · `shadow/strong`

### `sidebar/*`: 3

`sidebar/bg/edge` · `sidebar/bg/mid` · `sidebar/bg/start`

### `size/*`: 149

`size/animation/backdrop` · `size/arrow/md` · `size/avatar/2xl` · `size/avatar/lg` · `size/avatar/md` · `size/avatar/sm` · `size/avatar/xl` · `size/avatar/xs` · `size/badge/icon-slot/md` · `size/badge/icon-slot/sm` · `size/badge/icon/md` · `size/badge/icon/sm` · `size/badge/md` · `size/badge/sm` · `size/calendar/cell` · `size/calendar/header` · `size/caret/height` · `size/caret/width` · `size/checkbox/md` · `size/checkbox/sm` · `size/color-picker/hue` · `size/color-picker/saturation` · `size/connector/thickness` · `size/container/2xl` · `size/container/lg` · `size/container/md` · `size/container/sm` · `size/container/xl` · `size/container/xs` · `size/content-max` · `size/control/2xl` · `size/control/2xs` · `size/control/3xl` · `size/control/icon/2xs` · `size/control/icon/lg` · `size/control/icon/md` · `size/control/icon/sm` · `size/control/icon/xs` · `size/control/lg` · `size/control/max-width/lg` · `size/control/max-width/md` · `size/control/max-width/sm` · `size/control/max-width/xs` · `size/control/md` · `size/control/min-width/lg` · `size/control/min-width/md` · `size/control/min-width/sm` · `size/control/min-width/xs` · `size/control/sm` · `size/control/textarea/lg` · `size/control/textarea/md` · `size/control/textarea/sm` · `size/control/textarea/xs` · `size/control/xl` · `size/control/xs` · `size/divider` · `size/dot/2xs` · `size/dot/lg` · `size/dot/md` · `size/dot/sm` · `size/dot/xs` · `size/empty-icon` · `size/field/width` · `size/form/measure` · `size/form/nav` · `size/handle/md` · `size/icon/2xl` · `size/icon/2xs` · `size/icon/3xl` · `size/icon/lg` · `size/icon/md` · `size/icon/sm` · `size/icon/xl` · `size/icon/xs` · `size/illustration/lg` · `size/illustration/md` · `size/illustration/sm` · `size/logo/height` · `size/logo/lockup/height` · `size/logo/lockup/width` · `size/logo/md` · `size/logo/sm` · `size/logo/width` · `size/menu/width` · `size/panel/2xl` · `size/panel/2xs` · `size/panel/lg` · `size/panel/md` · `size/panel/sm` · `size/panel/xl` · `size/panel/xs` · `size/pin/cell/lg/height` · `size/pin/cell/lg/width` · `size/pin/cell/md/height` · `size/pin/cell/md/width` · `size/pin/cell/sm/height` · `size/pin/cell/sm/width` · `size/pin/cell/xs/height` · `size/pin/cell/xs/width` · `size/pin/slot/lg/height` · `size/pin/slot/lg/width` · `size/pin/slot/md/height` · `size/pin/slot/md/width` · `size/pin/slot/sm/height` · `size/pin/slot/sm/width` · `size/pin/slot/xs/height` · `size/pin/slot/xs/width` · `size/progress/ring` · `size/progress/track` · `size/radio/md` · `size/radio/sm` · `size/rail` · `size/sidebar/collapsed` · `size/sidebar/expanded` · `size/skeleton/line` · `size/spinner/lg` · `size/spinner/md` · `size/spinner/sm` · `size/splitter/handle-length` · `size/splitter/handle-thickness` · `size/splitter/hit-area` · `size/stepper/lg/height` · `size/stepper/lg/width` · `size/stepper/md/height` · `size/stepper/md/width` · `size/stepper/sm/height` · `size/stepper/sm/width` · `size/stepper/xs/height` · `size/stepper/xs/width` · `size/swatch/md` · `size/switch/height/md` · `size/switch/height/sm` · `size/switch/thumb/md` · `size/switch/thumb/sm` · `size/switch/width/md` · `size/switch/width/sm` · `size/table/header/comfortable` · `size/table/header/compact` · `size/table/header/spacious` · `size/table/row/comfortable` · `size/table/row/compact` · `size/table/row/spacious` · `size/tag/icon/md` · `size/tag/icon/sm` · `size/tag/md` · `size/tag/sm` · `size/timer/unit` · `size/toast` · `size/touch-target`

### `space/*`: 58

`space/0` · `space/2xl` · `space/2xs` · `space/3xl` · `space/3xs` · `space/4xl` · `space/5xl` · `space/6xl` · `space/7xl` · `space/8xl` · `space/9xl` · `space/badge/padding/md` · `space/badge/padding/sm` · `space/container/gap/lg` · `space/container/gap/md` · `space/container/gap/sm` · `space/container/gap/xl` · `space/container/gap/xs` · `space/container/padding/lg` · `space/container/padding/md` · `space/container/padding/sm` · `space/container/padding/xl` · `space/container/padding/xs` · `space/control/gap/2xl` · `space/control/gap/lg` · `space/control/gap/md` · `space/control/gap/sm` · `space/control/gap/xl` · `space/control/gap/xs` · `space/control/inset/2xs` · `space/control/inset/lg` · `space/control/inset/md` · `space/control/inset/sm` · `space/control/inset/xs` · `space/control/label-padding/lg` · `space/control/label-padding/md` · `space/control/label-padding/sm` · `space/control/label-padding/xs` · `space/control/nest/lg` · `space/control/nest/md` · `space/control/nest/sm` · `space/control/nest/xs` · `space/control/padding/lg` · `space/control/padding/md` · `space/control/padding/sm` · `space/control/padding/xs` · `space/lg` · `space/md` · `space/px` · `space/sidebar/logo-inset` · `space/sm` · `space/switch/inset` · `space/tag/padding/md` · `space/tag/padding/sm` · `space/wordmark/padding/bottom` · `space/wordmark/padding/top` · `space/xl` · `space/xs`

### Text styles: 30

`Body/L` · `Body/L Strong` · `Body/M` · `Body/M Strong` · `Body/S` · `Body/S Strong` · `Body/XS` · `Caption` · `Code/M` · `Code/S` · `Control/L` · `Control/M` · `Control/S` · `Control/XS` · `Display/L` · `Display/M` · `Display/S` · `Heading/L` · `Heading/M` · `Heading/S` · `Heading/XL` · `Heading/XS` · `Label/2XL` · `Label/3XL` · `Label/L` · `Label/M` · `Label/S` · `Label/XL` · `Label/XS` · `Overline`

### Effect styles: 17

`AI/Glow` · `AI/Hover Glow` · `Blur/Backdrop` · `Elevation/01 Hairline` · `Elevation/02 Card` · `Elevation/03 Raised` · `Elevation/04 Overlay` · `Elevation/05 Popover` · `Elevation/06 Modal` · `Elevation/Card Hover` · `Elevation/Control Raised` · `Elevation/Control Raised with depth` · `Focus/Ring` · `Focus/Ring Caution` · `Focus/Ring Danger` · `Focus/Ring Success` · `Inner/Subtle`

### Not yet defined by prism-css

None: every name above resolves in prism-css.
<!-- /prism:vocabulary -->
