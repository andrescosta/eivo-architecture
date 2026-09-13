
# Tags & Views (Catalog Discovery Design)

**Status: design sketch, not implemented.** To be implemented and finalized before content generation, or as part of it — not before that phase (see roadmap). This spec records the design as agreed so far, so it can be picked up without re-deriving it.

## Motivation

Today, discoverability is modeled as a static `category` tree (`eivo/configs/categories/*.yaml`): each eivolet is tagged with a `category` array that is a path through a fixed parent/child tree (e.g. `['programming', 'full-stack', 'typescript']`). Every time the tree is restructured, existing eivolets risk needing re-categorization.

The fix is to stop storing a tree path on the eivolet at all, and separate two independent concerns:

- **Tags** — a large, fine-grained vocabulary assigned to an eivolet once, at publish time. Additive only: new tags get added over time, existing tags are never renamed or restructured.
- **Views** — named, reusable definitions of "which tags matter and how they combine," used purely for navigation/browsing. A view is a query over the tag vocabulary, not data stored on the eivolet.

Reclassifying content under a new or changed view becomes free — it's a new query, not a migration.

## What stays out of scope

`progLang` and `platform` (and other execution-relevant fields) are **not** part of this tag vocabulary and are never folded into it. They are per-element, execution-time fields that drive how a challenge/example actually runs, with a completely different lifecycle (can't be casually restructured — doing so breaks runtime behavior). Tags and views are strictly a discovery/navigation concern, independent of execution.

## Core model

### Tags

- A large, broad vocabulary, covering "almost any aspect" of an eivolet — not limited to language/platform-shaped facets.
- Assigned to the eivolet at publish time (by Eibot inferring from context, or fixed by a scripted-generation context variable — same resolution pattern as today's `category` field, always human-overridable).
- The only permanent, stable layer in this design. Tags are added to over time; they are not renamed, removed, or restructured once real content depends on them.

### Views

A **view** is a named definition of a query over tags, used to power one navigable concept in the catalog UI — e.g. "Categories," "Career Paths," "Realms." Multiple views can and do coexist in the same UI at once; they are not mutually exclusive presentation modes.

Each view defines:
- **Its member tags** (or tag expression).
- **Its own combining logic** — AND or OR (or a more general boolean expression) is a property of the individual view definition, not a global system-wide rule. Different views may reasonably need different logic (e.g. a "category" might be an OR across related tags; a "career path" might be an AND across prerequisite tags).
- **A UI-facing label**, shown or not depending on the experience. "Categories" is one possible label for one possible view — it is not the name of the underlying concept.

### Naming

The underlying data-model concept (a saved query/definition over tags) should **not** be internally called "category" — that term is too narrow, since the same mechanism also covers career paths, realms, and whatever comes next. No specific internal term has been chosen yet; pick one deliberately when this is actually designed rather than defaulting to "category."

**"Tags" is also likely the wrong word for the per-eivolet primitive (raised 2026-09-05, not resolved).** "Tag" ordinarily implies something real-world-short and often user-facing — a hashtag, a quick label someone types. What's being described here is different: a large, curated, fixed vocabulary assigned deliberately at publish time, used purely as an internal query substrate for views — not necessarily short, not necessarily user-visible, not casually free-form. Using "tags" risks importing the wrong mental model (freeform, lightweight, user-authored) onto something that's meant to be closer to a controlled taxonomy of atomic facts. No replacement term chosen yet — flag this alongside the "category" naming question above when this is actually designed; both the container concept (view) and the primitive (tag) need names that don't borrow baggage from narrower, more familiar things.

**Possible two-tier refinement, same day, explicitly unresolved ("this needs more work"):** tags and the still-unnamed larger primitive vocabulary might not be the same thing at all — tags could be a smaller, real, user-facing *subset* of the larger primitive set, both still assigned at pre-publish time, but kept as independent concepts rather than one being just a synonym for the other. One is larger than the other (which one wasn't settled). Don't collapse these into a single layer when this gets designed — check whether the two-tier shape (small user-facing tags + large internal primitive vocabulary, both pre-publish, related but distinct) is still the intended direction, since it wasn't fully worked out here.

## Open questions for the implementation discussion

- Exact tag vocabulary and its initial seed list.
- Exact shape of a view's tag-expression (flat list with one combinator, vs. a fuller boolean expression tree).
- Migration path from today's `configs/categories/*.yaml` tree into tags + a "Categories" view.
- Where views are authored/stored, and whether a customer can define their own views (the original motivating idea — a customer-defined taxonomy that never requires re-tagging existing eivolets).
