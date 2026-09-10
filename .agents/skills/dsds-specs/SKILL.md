---
name: dsds-specs
description: Everything about Design System Doc Spec (DSDS) — entry kinds, sections, schema structure, and how it fits into the ecosystem. Use when authoring, reviewing, or reasoning about DSDS specs and `*.dsds.yaml` files.
metadata:
  version: 0.20.1
---

# Design System Doc Spec (DSDS)

[DSDS](https://designsystemdocspec.org/) is a machine-readable YAML format for documenting design systems. DSDS specs are the **single source of truth** — everything else (React components, Figma, docs, AI catalogs) derives from them.

DSDS documents a graph of **entries** (a system, a component, a token, a theme, or the generic `entry` kind for anything else), each carrying typed **sections** (`definitions`, `guidelines`, `steps`, or the generic `section`). It never duplicates data a better source of truth already owns — a component's `sourceFiles` points at the real code instead of hand-typing its props; a token's `source` points at the real DTCG value instead of restating it.

## Schema Sources

When you need precise field-level details beyond this skill, consult these in order:

1. **Bundled schema**: `https://designsystemdocspec.org/v0.20.1/dsds.bundled.schema.json` (or `node_modules/design-system-documentation-schema/schema/dsds.bundled.schema.json` if installed as a dependency)
2. **Schema architecture reference**: https://designsystemdocspec.org/schema#how-the-schema-is-organized (and [Conformance](https://designsystemdocspec.org/conformance) for conformance classes and the full rule catalog)
3. **Quick start with examples**: https://designsystemdocspec.org/quickstart
   — and [STYLE_GUIDE.md](https://github.com/somerandomdude/design-system-documentation-schema/blob/main/STYLE_GUIDE.md)
   for the field order every example follows. The schema permits any order;
   the style guide picks one. It is a convention, not a constraint — the
   `DSDS-17`–`DSDS-23` rules that report deviations run only in the spec
   repo itself.
4. **GitHub source** (split schema + examples): https://github.com/somerandomdude/design-system-documentation-schema/tree/main/schema

The site is four pages: `index.html`, `quickstart.html`, `extending.html`, `schema.html`. Every schema definition lives on `schema.html` under an anchor named `<directory>-<filename>` — `schema.html#entries-component`, `schema.html#sections-guidelines`, `schema.html#common-ref`. There is **no** per-definition page; a URL like `/entries-component` or `/sections-guidelines` is dead.

## Entry Kinds

| Kind | Purpose | Suggested directory |
| --- | --- | --- |
| `system` | The design system as a whole — version, organization, url, license, platforms. One per project, usually the root `index.dsds.yaml`. | (root) |
| `component` | A reusable UI element — API (via `sourceFiles`), variants/states (via `traits`), accessibility, usage. | `components/` |
| `token` | A single design token. Points at its real value via `source`; never carries the value itself. | `tokens/` |
| `theme` | A named set of token overrides (dark mode, brand variant). Points at its DTCG source file. | `themes/` |
| `entry` (generic, or a namespaced custom kind like `acme.icon-library`) | Anything else — a foundation, a pattern, a guide. Organize by folder for clarity even though the schema `kind` is uniform. | `foundations/`, `patterns/`, `guides/`, etc. |

There is no `token-group` kind: a group of related tokens is a `metadata.group` fact on the tokens in it, not a separate artifact.

## The Entry Envelope

Every entry kind shares one open base — learn it once and it generalizes:

```
id, kind, name, description, purpose, metadata, related, extends, refs, sections, $extensions
```

Only the kind-specific fields beyond that envelope differ: a token's `tokenType`/`source`, a component's `sourceFiles`/`specs`/`imports`/`traits`/`combos`, a theme's `colorScheme`. `sections/section.schema.yaml` plays the same role one level down: every section kind shares `kind`, `for`, `title`, `description`, `context`, `items`, `freeform`, `metadata`, `$extensions`.

## Document Structure

A **standalone entry** file (most components, tokens, themes) has no wrapper — the entry's own fields sit at the file's top level:

```yaml
kind: component
id: checkbox
name: Checkbox
description: A styled checkbox input for boolean or indeterminate selection.
```

A **base document** (the root `index.dsds.yaml`, or any file meant to hold more than one entry) requires `schemaVersion`, `name`, and a non-empty `entries` array. System-wide facts live on that list's own `kind: system` entry:

```yaml
schemaVersion: "0.20.1"
name: Acme Design System

entries:
  - kind: system
    id: acme-design-system
    name: Acme Design System
    description: Acme's cross-platform design system.
    metadata:
      version: 1.4.0
      platforms: [react, web-component]

refs:
  - href: ./components/checkbox.dsds.yaml
    rel: file
    role: Checkbox component
```

Splitting a system across many files uses `refs` (`rel: file`) pointing at sibling documents — not a `$ref`/JSON-Pointer include. There's also `scripts/tools/compose.js` upstream, for concatenating many hand-authored fragment files into one document before validation.

## Sections

Every entry's structured docs live in one `sections` array. Each section has a `kind` and a `for` (`human`, `agent`, or `all`, naming its audience):

- **`definitions`** — term/definition pairs. Use for anatomy, naming conventions, or a prop/event list when there's no real source file to extract from.
- **`guidelines`** — a `statement` paired with a `level` (`must`/`should`/`should-not`/`must-not`/`may`). Carries `framing: when-to-use` (a fit judgment) or `how-to-use` (the default, an implementation rule).
- **`steps`** — an ordered procedure or unordered checklist.
- **`section`** (generic) — for anything else, or purely `freeform` narrative prose.

Every section kind can also carry `freeform`: headed, nestable prose alongside its own structured `items`.

### `context` — what job a section is doing

Any section kind can set `context` to make its job machine-readable, instead of leaving it to a human-written `title` nothing validates: `anatomy`, `terms`, `keyboard`, `events`, or a namespaced custom value (`acme.slots`). Prefer this over titling a `definitions` section "Anatomy" and hoping a renderer matches on the heading.

```yaml
sections:
  - kind: definitions
    for: all
    context: keyboard
    title: Keyboard interactions
    items:
      - term: Space
        definition: Activates the button.
```

Don't confuse it with `guidelines`' own `framing` field (`when-to-use`/`how-to-use`), which is a different idea on one specific kind.

### Guidelines items

A guidelines item is `{level, statement}` plus optional `id`, `example`, `alternatives`, `evidence`, `related`, `checks`, `checkedBy`, and `$extensions`. `checkedBy` is `automated`/`assisted`/`manual`; `checkedBy: automated` **requires** a `checks` (or `refs`) pointer with `rel: test` or `rel: lint-rule` — that's `DSDS-03`.

An item can borrow its `statement` from a shared item instead of restating it: `refs: [{to: "shared-a11y#touch-target", rel: same-as}]`. It still declares its own `level`, and that `level` **must match the target's** (`DSDS-10`).

## A Component's Own Fields

Not sections — facts about the component as a build artifact:

- **`sourceFiles`** — one entry per platform, pointing a tool at the real file to extract the API **from** (`./src/Button.tsx`). Prefer this over hand-typing props in a `definitions` section. At most one entry per platform (`DSDS-01`).
- **`specs`** — a list of refs (`rel: contract`) to an already-generated API contract **document**: a W3C Custom Elements Manifest, a DS Contracts file, or similar. One step later in the pipeline than `sourceFiles`. DSDS never parses the target, so any standard format works.
- **`imports`** — one entry per platform: install package + import statement.
- **`traits`** — every variant (`kind: enum`) and state (`kind: boolean`) the component can be in.
- **`combos`** — pairing rules between traits, tokens, or entries (e.g. "loading and disabled must not both be set"). A combo's `subject`/`items` must resolve against real traits, tokens, or entries (`DSDS-09`).

### `traits[].setBy`

`kind` (`boolean`/`enum`) doesn't say who sets a trait — a boolean can be either. `setBy` does, and it's the field that matters most for codegen:

- **`consumer`** — the caller chooses it (`size`, `variant`, `disabled`). Becomes a prop.
- **`component`** — the component sets it on its own (`hover`, `loading`); the caller only observes it. **Never** becomes a prop.

```yaml
traits:
  - kind: enum
    id: variant
    description: Which visual style to render.
    setBy: consumer
    values:
      - id: primary
        description: The default, high-emphasis style for the main action.
      - id: secondary
        description: A lower-emphasis alternative.
  - kind: boolean
    id: hover
    description: The pointer is over the control.
    setBy: component
```

Every trait needs its own `description`, and so does every enum `value`.

## Agent-Only Sections

Mark a section `for: agent` for firm, ready-to-act notes a person wouldn't need — hard MUST/MUST NOT rules, notes that keep an agent from confusing this entry with a similar one, checks an agent can run against its own output. Tools never surface these to people. It must extend the human-facing sections on the same entry, never contradict or repeat them.

## Going Beyond the Shipped Fields

Three mechanisms, in order of reach — see https://designsystemdocspec.org/extending.html:

- **`$extensions`** — namespaced tool data (`com.figma:`), valid on the document, an entry, a section, **and on individual `definitions`/`guidelines`/`steps` items and `freeform` entries**. Use when you need to attach data to one specific rule.
- **A custom kind** — a namespaced `kind` (`acme.icon-library` on an entry, `acme.slots` on a section) when the document wants its own recognizable name. Validated against the open base.
- **A profile** — a file at `profiles/entries/<kind>.schema.yaml` or `profiles/sections/<kind>.schema.yaml` that **narrows** a built-in kind (makes its optional fields required) without forking it. A profile may narrow; it must not extend. Picked up automatically by the validator, never bundled into the published schema.

Before reaching for any of them, check the "Things people often want to add" list on that page — a prop table, an anatomy diagram, a `token-group` kind, and typed accessibility fields are deliberate absences with an intended answer, not gaps.

## Schema Validation

The bundled schema is published at `https://designsystemdocspec.org/v0.20.1/dsds.bundled.schema.json`, using JSON Schema draft 2020-12. Validate with:

```bash
npx dsds-validate <files-or-globs>
```

See the `dsds-validate` skill for the full rule catalog (`DSDS-01`–`DSDS-23`) and how to interpret failures.

## Deep-Dive References

Every definition is on one page. Fetch the anchor:

| Topic | Reference |
| --- | --- |
| Component (sourceFiles, specs, imports, traits, combos) | https://designsystemdocspec.org/schema.html#entries-component |
| Token | https://designsystemdocspec.org/schema.html#entries-token |
| Theme | https://designsystemdocspec.org/schema.html#entries-theme |
| System | https://designsystemdocspec.org/schema.html#entries-system |
| Generic entry | https://designsystemdocspec.org/schema.html#entries-entry |
| Section base (`for`, `context`, `freeform`) | https://designsystemdocspec.org/schema.html#sections-section |
| Definitions section | https://designsystemdocspec.org/schema.html#sections-definitions |
| Guidelines section | https://designsystemdocspec.org/schema.html#sections-guidelines |
| Steps section | https://designsystemdocspec.org/schema.html#sections-steps |
| The one pointer type | https://designsystemdocspec.org/schema.html#common-ref |
| Combos | https://designsystemdocspec.org/schema.html#common-combo |
| Metadata | https://designsystemdocspec.org/schema.html#metadata-entry-metadata |
| How the schema is organized | https://designsystemdocspec.org/schema.html#how-the-schema-is-organized |

## Gotchas

- A standalone entry file has no `entity`/`entityGroups` wrapper — `id`/`kind`/`name`/`description` sit at the top level directly. A base document requires `schemaVersion`, `name`, and a non-empty `entries` array.
- `id` must match the filename without `.dsds.yaml` (e.g. `checkbox` → `checkbox.dsds.yaml`).
- Requirement levels: `must`, `should`, `should-not`, `must-not`, `may` (lowercase, hyphenated — RFC 2119).
- `metadata.status` is always an object: `{status: "stable"}`, optionally scoped with `platform`, `since`, `deprecationNotice`, `note`. There's no bare-string shorthand.
- All pointers — dependencies, composition, citations, external links — use one shape: `common/ref` (`to` for this document's own graph, `href` for outside it, plus a `rel`). There's no separate "relationship" or "link" type.
- **`to:` is an id, not a display name.** It has a structural `pattern`: an id, optionally `#itemId`. `to: Button` or anything containing a space fails schema validation outright, with no rule id — it's a plain `pattern` failure.
- **`metadata.tags[0]` is the entry's category** by convention; the rest are free-form tags.
- **`metadata.group` groups any kind**, not just tokens — `color.action` on a set of tokens, `button` on a `button`/`icon-button` family. It's what replaced the deleted `token-group` kind.
- A guideline with **no `statement`** is valid when it carries a `rel: same-as` or `rel: external-link` ref instead. The `external-link` form is deliberately unreadable without following the link; a renderer should show the level and the link, not treat it as empty.
- Content **always** lives in a section's `items`, never in a field named after the kind (no `steps.steps`, no `definitions.definitions`).
- Every section item is addressable even without an explicit `id`: a conforming consumer derives one from the item's own text (lowercase, non-alphanumeric runs collapsed to a dash).
- **Don't put prose in a YAML flow mapping.** `{id: primary, description: The default, high-emphasis style}` parses as *three* keys — a flow mapping splits on every comma, including one inside a sentence. It surfaces as a baffling `must NOT have unevaluated properties` error. Use block style for anything with a `description`.
