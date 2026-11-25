---
id: GR-ARC-006
title: Prism 2.0 — Naming and Casing (verbatim casing.md)
artifact_type: platform-obligations
status: draft
content_state: source-backed
owner_role: Product design owner (TBC)
applies_to:
  - PRJ-KIN-BRAU-CI
source_of_truth: golden-repo
source_refs:
  - "SRC-PRISM-UI-001 ai-resources/casing.md"
last_updated: 2026-11-19
review_due: TBC
---


> **Verbatim copy, draft.** Everything below this line is ai-resources/casing.md from the Prism 2.0 repository
> (`SRC-PRISM-UI-001`, branch `monorepo`, commit `8c4c80bb`, release 3.0.0, blob `a32b2a15`), unchanged. It is refreshed
> from the Prism repository on each Prism release and never edited here (DEC-018). Relative links inside it point at
> the Prism repository, not at this one. Where it and the Prism repository differ, the repository wins.

# Prism naming and casing

Version 1.0.0 · 2026-08-20.

One rule underneath all of it, borrowed from Google's Material guidance:

> **Names are Title Case. Everything a person reads as ordinary language is sentence case.**

A "name" means a component, a page, a style, a defined term — something you could point at in a library. A
label, a heading, a button, a message, a tooltip, a description is ordinary language, so only the first word
and any proper nouns are capitalised.

## Identifiers

| Thing | Pattern | Example |
|---|---|---|
| Primitive variable | `color/{hue}/{step}` · `dimension/{px}` · `font-size/{px}` | `color/blue/600` |
| Semantic colour | `{bg\|fg\|border}/{family}/{role}[-{state}]` — `chart`, `shadow`, `cursor` stay flat as `{group}/{role}` | `bg/primary/hover`, `bg/primary/subtle-hover`, `fg/neutral/default` |
| Dimension | `{space\|radius\|size\|border-width\|container}/{t-shirt}` | `space/2xl` |
| Typography | `font/{family\|weight\|size}/{t-shirt}` | `font/size/md` |
| Component | Title Case, singular | `Button`, `Icon Button`, `Field` |
| Variant property | `Property=Value` | `Variant=Primary`, `Size=Md`, `State=Hover` |
| Icon | `Icon / {lucide-name}` | `Icon / chevron-down` |
| Text style | `{Group}/{Step}` | `Body/S Strong`, `Heading/XL` |
| Effect style | `{Group}/{Number} {Name}` | `Elevation/02 Card` |
| Page | `{Level} · {Group}` | `Molecules · Fields` |
| Doc section frame | `section/{Component}` | `section/Table of Contents` |

### Token names are lowercase, always

`bg/primary/hover`, never `bg/Primary-Hover`. Slashes make the hierarchy, hyphens join words inside one
segment. Figma rejects `.` and `-` inside a *numeric* segment, which is why negative tracking values use safe
slugs: `tracking/n1-5` means −1.5, `tracking/p0-4` means +0.4.

### `primary`, not `brand`

The primary colour role is called **`primary`** throughout. There is no `brand` token. If you find `brand`
anywhere — a description, a comment, a code reference — it is a leftover and should be corrected to `primary`.

### Emphasis words are not colour words

`Variant=Secondary` and `Variant=Tertiary` on Button and Icon Button describe **emphasis level** and are
drawn from the primary blue ramp. `Secondary neutral` and `Tertiary neutral` are the same two emphasis levels on
the neutral ramp. None of them uses the secondary (teal) or tertiary (navy) *colour* roles, which live in
`bg/secondary/default` and `bg/tertiary/default` and are for accents, tags, charts and marketing surfaces. A "secondary button"
is not teal. This trips people up more than anything else in the system.

## Sentence case in practice

| Context | Correct | Wrong |
|---|---|---|
| Button label | `Save changes` | `Save Changes` |
| Form label | `Account number` | `Account Number` |
| Section heading | `Rules of the road` | `Rules Of The Road` |
| Menu item | `Download statement` | `Download Statement` |
| Badge | `Pending review` | `Pending Review` |
| Empty state | `No transfers yet` | `No Transfers Yet` |
| Tab | `Overview` | (single word — same either way) |
| Overline / eyebrow | `Foundations` | `FOUNDATIONS` |

The Overline text style is small and letter-spaced but **not** forced to uppercase. Set the words in sentence
case and let the style do the work — `textCase` is left `ORIGINAL` on purpose, because Figma syncs text case
across every same-named layer in a component set, so a forced-caps style cannot be varied per variant.

## Wording

- **No trailing colons** on form labels.
- **No terminal period** on labels, buttons, badges or single-phrase helper text. Full sentences in
  descriptions and error messages do take one.
- **Errors say how to fix it.** "Enter a date in DD / MM / YYYY format", not "Invalid date".
- **Switch labels describe the ON state.** "Email notifications", not "Disable email notifications".
- **Sentence case includes proper nouns**, so product and institution names keep their capitals: "Export to
  Meridian Credit Union".
- **Numerals** for everything countable, including under ten: "3 accounts", not "three accounts".

## Sample data

Everything illustrative in the library is fictional — "Meridian Credit Union", "Cascade Trust", reference
numbers like 8842. Keep it that way. **Never put real member names, account numbers, or any other
personally-identifiable data belonging to a financial institution's end customers into the design file, a token
name, a component description, or these documents.**

## Layer names inside components

- The layer name is the **property anchor**. Renaming `Label` to something else breaks the text property bound
  to it.
- Set `autoRename = false` on every text node you create. `figma.createText()` returns a node already named
  `"Text"`, so assigning `t.name = 'Text'` is a no-op that leaves `autoRename` on — and the next `characters`
  assignment silently renames the layer to its own content, breaking the property link.
- Structural layers get structural names: `Rail`, `Cell`, `Items`, `Spacer`, `Content`, `Header`. Not
  `Frame 427`.
