---
name: dsds-add
description: Author a new Design System Doc Spec (DSDS) spec from component implementation, Figma design, or written requirements. Triggers on "add spec", "create spec", "new spec", "author spec", "spec from component", "spec from Figma".
metadata:
  version: 0.20.1
---

# Add a DSDS Spec

Create a new standalone `.dsds.yaml` entry file in your project's spec directory (see File Placement below for where a given kind lives).

## Procedure

1. Determine entry kind: `component`, `token`, `theme`, or the generic `entry` kind (for a foundation, pattern, guide, or anything else — use a namespaced custom kind like `acme.icon-library` instead if the document wants its own recognizable name).
2. Gather inputs — read the source (component source code, Figma frame, requirements doc).
3. Create `{directory}/{id}.dsds.yaml` using the template below.
4. Add a `refs` entry (`rel: file`) in `index.dsds.yaml` pointing at the new file.
5. Run `npx dsds-validate {directory}/{id}.dsds.yaml` — fix errors until it passes.
6. Order the entry's fields per [STYLE_GUIDE.md](https://github.com/somerandomdude/design-system-documentation-schema/blob/main/STYLE_GUIDE.md): identity (`id`, `kind`, `name`, `description`, `purpose`, plus a token's `tokenType`/`source`), then `metadata`, then a component's `sourceFiles`, then `sections`, then structured facts (`specs`, `imports`, `traits`, `combos`), then `related`/`extends`/`refs`, with `$extensions` always last. Order never affects validity, and nothing in your project checks it: `npx dsds-validate` won't mention it, and the `DSDS-17`–`DSDS-23` advisory rules that report it live in the spec repo's own tooling, which the published package doesn't ship. The template below already follows the order — keep it and you're done.
7. If your project generates its own index or catalog from spec files, regenerate it now.

## File Placement

| Kind | Directory |
| --- | --- |
| `component` | `components/` |
| `token` | `tokens/` |
| `theme` | `themes/` |
| `entry` (foundation) | `foundations/` |
| `entry` (pattern) | `patterns/` |
| `entry` (guide) | `guides/` |

## Template (Component)

```yaml
kind: component
id: <filename-without-extension>
name: <PascalCase>
description: <one-sentence summary>

metadata:
  tags: [<action|feedback|form|disclosure|overlay|navigation|layout>]
  since: <version>
  status: {status: draft}

sourceFiles:
  - platform: <react|web-component|...>
    file: <path to the real source file>

# Optional: an already-generated API contract (Custom Elements Manifest,
# DS Contracts, ...) - one step past sourceFiles, not a replacement for it.
specs:
  - rel: contract
    href: <./contracts/<id>.contract.json>
    role: <format name>

imports:
  - platform: <react|web-component|...>
    code: <import statement, written out>
    package: <package name>

traits:
  - kind: enum
    id: <variant>
    description: <what this dimension of variation controls>
    setBy: consumer
    values:
      - id: <value>
        description: <what it's for>
  - kind: boolean
    id: <loading>
    description: <what this state means>
    setBy: component

sections:
  - kind: guidelines
    for: all
    framing: how-to-use
    items:
      - level: must
        statement: <one concrete rule>
```

## Sections to Include (Components)

Include at minimum: a `guidelines` section (`framing: how-to-use`) covering usage rules and accessibility requirements. Add `traits` (top-level, not a section) for variants/states, a `guidelines` section with `framing: when-to-use` for fit judgments, and a `definitions` section for props/anatomy only when there's no real source file to point `sourceFiles` at instead. Add a `for: agent` section for firm rules an agent needs but a person wouldn't.

## Extraction Guidelines

- **From code**: Point `sourceFiles` at the real file instead of hand-typing props — that's the whole point of the field. Map variant/state props → `traits` (`kind: enum` or `kind: boolean`), and set `setBy: consumer` for anything the caller passes in, `setBy: component` for anything the component sets on its own (`hover`, `loading`). Map CSS parts or named sub-elements → a `definitions` section with `context: anatomy`.
- **From Figma**: Map component properties → `traits`, layer structure → a `definitions` section with `context: anatomy`, variable bindings → token `refs`.
- **From requirements**: Map acceptance criteria → `guidelines` items (`level` from RFC 2119: `must`/`should`/`should-not`/`must-not`/`may`), interaction requirements → a `definitions` section with `context: keyboard` (term = key, definition = action).

Set `context` rather than relying on a `title` alone — `anatomy`, `terms`, `keyboard`, `events` are the well-known values, and they're what makes a section's job machine-readable. A `title` is still fine alongside it, for people.

## Writing Guidelines That Are Actually Checkable

- Give a rule `checkedBy` (`automated`/`assisted`/`manual`) when you know how it's verified.
- `checkedBy: automated` **requires** a `checks` pointer at the thing that runs it, or validation fails (`DSDS-03`):

  ```yaml
  - level: must
    statement: The control meets a 44×44px minimum touch target.
    checkedBy: automated
    checks:
      - href: ./tests/button.a11y.test.ts
        rel: test
  ```

- To reuse a rule declared once on a shared entry, borrow it instead of restating it — and keep `level` identical to the target's, or `DSDS-10` fails:

  ```yaml
  - level: must
    refs: [{to: "shared-a11y#touch-target", rel: same-as}]
  ```

Before finishing, read `examples/anti-patterns/` upstream — three documents that validate cleanly and are still bad. The failure modes it catalogs (a definition that only restates its own term, a `checkedBy: manual` claim too vague to check, prose naming a concept the spec doesn't have) are the ones validation cannot catch for you.

## Schema References

When unsure about field shapes or required properties, consult:

- **Bundled schema**: `https://designsystemdocspec.org/v0.20.1/dsds.bundled.schema.json` (or `node_modules/design-system-documentation-schema/schema/dsds.bundled.schema.json` if DSDS is installed as a dependency)
- **Entry docs**: `https://designsystemdocspec.org/entries-{kind}` (e.g. `/entries-component`)
- **Section docs**: `https://designsystemdocspec.org/sections-{kind}` (e.g. `/sections-guidelines`)
- **Quick start examples**: https://designsystemdocspec.org/quickstart

## Gotchas

- `id` must match the filename (e.g. `checkbox` → `checkbox.dsds.yaml`).
- A component's `sourceFiles`, `specs`, `imports`, `traits`, and `combos` are top-level fields on the entry, never inside a section.
- Use RFC 2119 levels in guidelines: `must`, `should`, `should-not`, `must-not`, `may`.
- `metadata.status` is always an object (`{status: "draft"}`), never a bare string.
- A guidelines section's field is **`framing`** (`when-to-use`/`how-to-use`). `context` is a different, section-wide field (`anatomy`/`terms`/`keyboard`/`events`).
- `to:` takes an id, optionally `#itemId` — never a display name. `to: Button` fails schema validation on a `pattern`.
- At most one `sourceFiles` entry per platform (`DSDS-01`), and every `platform` value must appear in the system entry's `metadata.platforms` once that list exists (`DSDS-02`).
- A component with no `sourceFiles` is valid — documenting something designed but not yet built is supported, not a gap.
