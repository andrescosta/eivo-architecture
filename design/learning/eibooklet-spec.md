# Eibooklets — Spec

The **eibooklet** is a discovery mechanism — the means by which ADL objects that
declare themselves discoverable get surfaced in the reading flow of an eibook. It
is not a content type, not a node kind, not an authoring artifact. Objects exist
independently in the Eivolet tree; the eibooklet is what makes them findable and
openable from within the eibook.

This document describes the mechanism, the `discoverable` declaration that objects
use to opt in, and the concrete ADL model (`companion`) that exists purely to be
surfaced through it. It is the companion to the reader capability spec
(`learning-reader-spec`), which covers how eibooklets integrate into the reading
surface.

*Status: the mechanism and the companion (static authored) case are defined and
integrated into the reader. The `discoverable` attribute and exposure of existing
Eivolet objects (spaceenvironment, llmmaterial) is design intent, not yet built.*

---

## 1. The discoverable declaration

Any ADL object can opt in to the eibooklet mechanism by declaring a `discoverable`
attribute. The first implementation uses a summary — a small MDX body rendered
in the reading stream that contains `<AdlObjectLink/>`, the component the
eibooklet capability provides and resolves:

```yaml
discoverable:
  eibooklet:
    summary: |
      A React todo app showing component composition and state lifting.
      <AdlObjectLink/>
```

`<AdlObjectLink/>` is not tied to the eibooklet mechanism specifically — it is
the component any ADL object uses in a `discoverable` summary to reference itself.
The eibooklet capability resolves it at render time, binding it to the object's
own identity and wiring up the open action. The object does not need to know its
own id or how it will be opened.

The `discoverable` node is designed for flexibility:

```yaml
# Shared summary across mechanisms
discoverable:
  summary: ...
  eibooklet:
    enabled: true
  gym:
    enabled: true

# Per-mechanism summary when presence differs
discoverable:
  eibooklet:
    summary: ...
  gym:
    summary: ...
```

As more discovery mechanisms emerge, each gets its own key under `discoverable`.
The structure accommodates shared or specific summaries without changing the
contract.

---

## 2. How the mechanism works

When an eibook is built, it walks the Eivolet tree, finds objects with
`discoverable.eibooklet` set, and surfaces them in the reading flow at positions
derived from their location in the tree. Position gives scope: an object under a
page relates to that page, under a section to the section, at the root to the
whole eibook.

The first implementation uses the summary pattern — the object's summary renders
in the reading stream through the ordinary page path, and `<AdlObjectLink/>` opens
the full surface on click. How the author controls integration — summary link,
inline embed, sidebar entry — is future design; the first implementation is the
summary.

The same object may be accessible through other paths — Gym, indexes, dedicated
features — independently. The eibooklet adds reading-flow discovery without
replacing or duplicating those access paths.

---

## 3. How it integrates (summary)

Full detail in `learning-reader-spec` §10. Integration costs the reader almost
nothing because the eibooklet mechanism reuses machinery that already exists:

- **In the reading flow** — the summary renders through the same `getContent` as
  any page. `<AdlObjectLink/>` in that summary is resolved by the eibooklet
  capability through the component map — switching to the full-surface view on
  click. The reader never special-cases it.
- **On its own surface** — the full content renders through the `renderEibooklet`
  RSA (a single fetch-and-render, not the skeleton-first stream, since one
  eibooklet shows at a time).
- **In navigation** — the **Eibooklets** surface (a sidebar tab and the view's
  landing) lists all exposed objects in the Contents-TOC visual language,
  positioned in the tree, opening rather than scrolling.

So: summary via the untouched page path, content via one RSA, listed by a TOC
that reuses the Contents styling. No change to the reader's architecture.

---

## 4. The primary use case

The dominant use of the eibooklet mechanism is **full, runnable code examples**
attached to a position in the eibook — a React todo app, a multi-file Rust
example — used to explain or demonstrate complex concepts in depth. A
`spaceenvironment` node already exists in the Eivolet, already configured and
runnable. The author declares `discoverable.eibooklet` on it, and the eibook
surfaces it inline in the reading flow at the moment it becomes relevant, without
the learner leaving the book.

The learner encounters a summary link while reading, opens it, and gets the full
workbench surface. The object doesn't change; the exposure does.

---

## 5. What can be exposed

Any Eivolet object can participate by declaring `discoverable.eibooklet`. Natural
candidates:

- **`spaceenvironment`** — a configured coding environment, the primary use case.
  Full workbench surface when opened.
- **`llmmaterial`** (practice/challenge) — a generator exposed as a standing,
  returnable practice point at a position in the book.
- **`companion`** — a simple authored content object with no other access path,
  existing purely to be surfaced here (see §6).

---

## 6. The companion model

`model: companion` is the concrete ADL model for objects that exist solely to be
surfaced through the eibooklet mechanism — authored off-path content with no
generative capability and no other access path. It is the simplest implementor of
the eibooklet contract, and its `discoverable.eibooklet` is always set.

It carries:

- **`discoverable.eibooklet.summary`** — the in-flow presence, containing
  `<AdlObjectLink/>`.
- **`def.content`** — the full content, rendered on its own surface when opened.
- **`tags`** — informational labels used for navigation and grouping, not for
  selecting how it renders.

Because a companion node is not a leaf template, its content is stored via
**contentgroup** — the same technology as space environments. This is a
persistence detail, invisible above the storage layer.

There are **two tiers** for the programming example case, rendered by different
components so they can evolve independently:

- **`<ProgrammingExampleEditor>`** — a short example embedded inline in a
  `model: template` page. Compact.
- **`<ProgrammingExampleWorkbench>`** — the full example in a companion's
  `def.content`. Full-surface.

Both are named components, stamped uniquely at store time.

---

## 7. Tags as a launchpad

Tags are informational — they organize the Eibooklets index and let the learner
filter, but do not select the renderer. Once multiple object types are exposed
through the mechanism, tags become how the Eibooklets surface groups everything:
*example*, *practice*, *challenge*, *session*. The index can grow from a flat
list into a categorized launchpad over all objects attached to the eibook.

---

## 8. Keeping roles crisp

Eibooklets share primitives (llmmaterial, exercise rendering) with other things
in the platform; the distinction is one of **role**:

- an **eibit** is a small piece EiBot makes inside a discussion, kept by the
  learner (ephemeral → kept);
- an **exercise in a page** is authored inline on the reading path;
- an **eibooklet-exposed object** is a standing, positioned access point the
  learner returns to (durable location, discoverable from the reading flow).

Same primitives, different roles. If those roles stay clear, learners always know
where a given kind of practice lives.

---

## 9. Design principles

- **Discovery mechanism, not a model.** Eibooklet is the integration contract any
  ADL object can satisfy. It is not a node kind or an authoring artifact.
- **Exposure is separate from existence.** An object exists in the Eivolet for
  its own reasons. The `discoverable` declaration makes it available; the eibook
  decides where to surface it.
- **Position carries meaning.** An object's place in the Eivolet tree is its
  scope — no separate scoping mechanism.
- **Reuse the reading machinery.** Summary via the untouched page path,
  `<AdlObjectLink/>` via the component map, action via context. Nothing new in
  the reader's core.
- **One discovery pattern.** All positioned, openable objects in an eibook are
  discovered the same way — through the eibooklet mechanism.
- **`discoverable` is extensible.** New mechanisms add a key; the structure
  accommodates shared or specific summaries without changing the contract.

---

## 10. Open threads

- Define where `discoverable` lives in the ADL structure (`metadata` or top-level).
- Define the eibook build process — how it walks the Eivolet tree, reads
  `discoverable.eibooklet`, and materializes eibooklet nodes at positions.
- Finalise `EiNode` to reflect the exposure model: `pageSequence` and
  `ancestorIndex` helpers, `renderEibooklet` RSA return shape.
- Rename `<EibookletLink/>` to `<AdlObjectLink/>` throughout.
- Add `ProgrammingExampleEditor` and `ProgrammingExampleWorkbench` to the stamped
  component tags.
- Design `spaceenvironment` exposure end-to-end as the primary eibooklet case.
- Design `llmmaterial` exposure (practice/challenge) end-to-end.
- Evolve the Eibooklets index toward a tag-categorized launchpad once more than
  one object type is exposed.
- Non-programming companion domains.
