We should select one language to create the eivolet other than English and use it as the main language guide. The others will be generated as batch as was decribed.

# Session Language & Cultures Split — Spec

## Problem

The current cultures question asks the author to select all languages the Eivolet will support in a single step. English (en-US) is locked. All content is generated in en-US only, with other cultures deferred to batch rendering.

This conflates two distinct concerns:
- The language the author works in during the session (authoring / generation language)
- The languages the Eivolet should eventually support (rendering targets)

## Proposal

Split the single cultures question into two sequential questions in Objective 1, and store the working language on the Eivolet's `metadata.extra` — not as an invariant field.

### Why `metadata.extra`

`metadata.extra` is already the ADL extension point for arbitrary Eivolet-level properties (`extra.eivoletDefName` is already there). Storing `sessionCulture` here:

- Requires no invariants schema change
- Requires no `save-eivolet-instance` schema change
- Persists with the Eivolet in both Redis and on disk
- Is available to any future agent or feature that loads the Eivolet — the Eivolet carries its own working context

This makes `sessionCulture` future-proof beyond EiBot authoring. When an edit flow loads an existing Eivolet from disk into Redis, it reads `metadata.extra.sessionCulture` to know what language to operate in. No separate session state needed.

```yaml
model: eivolet
metadata:
  name: ...
  namespace: ...
  extra:
    sessionCulture: fr        # working language — set at session start, persists with the Eivolet
```

### Question 1 — Session Language (new)

Single select. The author picks the language they will work in.

```
model: agQuestion
def:
  question: "What language will you work in during this session?"
  multi: false
  options: <from get-cultures-question>
```

Result stored as `metadata.extra.sessionCulture` via `save-eivolet-instance`.

### Question 2 — Additional Cultures (updated)

Multi select. Same as current cultures question but with the session language shown as locked.

```
model: agQuestion
def:
  question: "Which other languages should this Eivolet eventually support?"
  multi: true
  options: <from get-cultures-question, session language locked>
```

Result merged with session language → stored as `def.invariants.cultures`.

## Data Model

```yaml
model: eivolet
metadata:
  name: ...
  namespace: ...
  extra:
    sessionCulture: fr          # working / generation language
def:
  invariants:
    objective: ...
    audience: ...
    cultures:
      - fr                      # session language always first
      - en-US
      - es
    domain: ...
```

## Craft Changes

### Objective 1 — Invariants Collection

Replace the single cultures step with two steps:

1. Ask session language (single select, `get-cultures-question`)
2. Ask additional cultures (multi select, `get-cultures-question`, session language locked)
3. Save `metadata.extra.sessionCulture` and `def.invariants.cultures` via `save-eivolet-instance`

### Hard Constraints — content generation language

Replace:
> All content is generated in en-US only during the authoring session.

With:
> All content (template body, llmmaterial prompts, labels) is generated in `metadata.extra.sessionCulture`. Never generate content in any other language during the session.

## Future: Edit Flow

When an existing Eivolet is loaded from disk into Redis for editing:

1. Load the Eivolet — `metadata.extra.sessionCulture` is already set
2. Any agent or tool operating on the Eivolet reads this field to determine the working language
3. No session configuration needed — the Eivolet is self-describing

This makes `sessionCulture` a general-purpose working language marker, not just an EiBot authoring concern. It applies to any future feature that mutates Eivolet content: edit flows, bulk update agents, translation agents operating on a specific culture.

## Tool Changes

None — `save-eivolet-instance` already accepts the full Eivolet object including `metadata.extra`.

## Schema Changes

`metadata.extra` is already `additionalProperties: true` — no schema change needed. `sessionCulture` is a convention, not a schema-enforced field.

## Out of Scope

- Batch rendering for additional cultures — unchanged, deferred as before
- `get-cultures-question` tool — unchanged
- Generation pipeline changes — `sessionCulture` is an authoring/editing concern only