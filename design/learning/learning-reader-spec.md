# The Book Reader — Capability Spec

The learning capability of the Eivo platform: how a learner reads and studies an
Eivolet. Written to orient a future work session — adding features, or updating
the presentation and architecture docs. Read top-to-bottom for the whole
picture; the deep mechanisms live in linked per-component docs.

---

## 1. The capability

An **Eivolet** is a unit of learning content — a short book of pages, authored on
the platform. In learning, the learner reads it as an **eibook**: the eivolet in
its read form. The **book reader** is where they read and study it.

Reading is the centre. Around it, the learner builds a personal layer over the
material — without the reading surface ever feeling like a content system:

- **Read** the Eivolet as a book: its pages in order, with interactive elements
  (exercises, diagrams) embedded in them.
- **Mark** passages — highlights, and notes on those highlights.
- **Discuss** any part with **EiBot**, a conversational learning assistant, from
  a starting point they choose (the whole Eivolet, a section, a passage, a
  diagram, or an exercise).
- **Keep eibits** — small, keep-worthy learning units (an exercise, a diagram, an
  explanation) that EiBot produces during a discussion. Kept eibits become a
  second thing to read, alongside the book.

All of this accumulates in the learner's **workbook**: their kept eibits, their
notes and highlights, and their past discussions — a personal layer bound to this
Eivolet, reachable from the reader but never intruding on the reading.

*The material is the book; everything the learner adds is a quiet layer on top of
it.* That is the guiding principle the whole reader is built around.

---

## 2. The reading surface

### One surface, two views

The main surface is a single vertical stream, showing one of two things, switched
by two subtle **view-marks** on the left edge:

- **Material view** — the Eivolet's pages, in order. The default: "the book".
- **Eibit view** — the stream of kept eibits for this Eivolet, read the same way.

The two are deliberately symmetric — both are streams of content units rendered
by the same mechanism (§6), so switching a view changes *what* fills the stream,
not *how* it behaves. Navigation, scroll, and lazy loading are identical in both.
The view-marks are understated (active shown by width, not colour): switching is a
quiet affordance, not a mode to think about.

### One sidebar, two tabs

The sidebar separates *the book's structure* from *the learner's layer*:

- **Contents** — the Eivolet's table of contents. Jumping to any page works
  whether or not it has loaded yet.
- **Workbook** — everything the learner has accumulated (§4).

This split is the core information-architecture decision: the material is fixed
and authored, the workbook is personal and grows, and the reader never conflates
"what this book is" with "what I did with it". A third navigation surface — the
**Objects TOC** — is in design (§10): an index of addressable objects (examples,
and later exercises/diagrams) attached to the material.

---

## 3. What a learner does — the features

### Highlights and notes

The learner selects a passage and marks it. A highlight is a passage to find
again; a **note** is a highlight with the learner's own text attached — a thought
to return to. Both live in the workbook, both navigate back into the book.
(Internally these are *excerpts*; the UI calls them bookmarks.)

### Discussions with EiBot

From any starting point — the Eivolet, a section, a marked passage, a diagram, or
an exercise — the learner opens a **Discussion**: a conversational session where
EiBot helps them understand *that* thing, grounded in the material. EiBot drives;
the learner responds. The starting point (`kind`) selects how the session opens
and what it anchors to. Discussions are one feature inside the reader, not the
frame of the capability.

Each discussion is remembered in the workbook and can be reopened.

> Deep dive: the EiBot instruction set and per-kind flows —
> `assistant-instructions`.

### Eibits

During a discussion, EiBot can produce an **eibit**: a tiny, keep-worthy learning
unit — a short exercise, a diagram, or an explanation the learner will want to
return to. Eibits are a teaching instrument: a way to leave durable learning
behind, beyond the passing conversation.

An eibit is **lightweight by design** — a small piece EiBot produces, not an
entity it tracks or manages. Value is conferred only when the learner **keeps**
it. This single decision keeps everything downstream simple: no update logic, no
ownership to interpret, every generation a fresh piece.

The lifecycle:

1. **Produce → session only.** EiBot authors the eibit; it renders in the chat.
   Even one the learner asked for starts here — it is not kept.
2. **Keep → workbook.** Only on the learner's explicit request does it become
   durable and appear in the eibit view. Never on EiBot's initiative.
3. **Remove → workbook.** The learner can take a kept eibit back out.

> Deep dives: eibit authoring — `eibit-producer`; the store-time MDX pipeline —
> `mdx-namer/README`.

---

## 4. The workbook

The learner's layer over the Eivolet, shown in the Workbook tab as four roots,
ordered by how the learner relates to each:

**Eibits · Notes · Highlights · Discussions** — what they kept, what they wrote,
what they marked, what they discussed.

Each root is an index over a different source, and each item navigates to its own
destination: an eibit switches to the eibit view, a note or highlight scrolls
into the book, a discussion reopens its session. The panel is a **router over the
learner's layer**, not a store of its own — it reflects the live state of the
sources that own each concern.

Sources:

- **Eibits + bookmarks** — the workbook aggregate (server-side). Eibits list
  flat (they belong to the Eivolet, not a page); notes/highlights group under
  their page.
- **Discussions** — *not* stored in the workbook. Found by a Redis prefix scan
  (`assistant:userId:eivoletId:*`, delimiter-anchored so ids can't collide by
  prefix), self-cleaning on TTL. Each session stores its own title at creation.

**Titles are denormalized** — computed once when the creating context has what's
needed (a discussion's title at session creation, a bookmark's at highlight
creation), not reconstructed on read. Frozen templates ⇒ no drift. A discussion's
title says *what* it's about (component title / passage / section / "{Kind} in
{template title}" fallback); its meta line says *kind + where*.

---

## 5. How the layer stays out of the way

A load-bearing property: **changes to the learner's layer never disturb
reading.** Scroll position lives in the DOM, the active page comes from an
observer, loaded pages live in their own store — none of it derives from the
workbook lists. So keeping an eibit, adding a discussion, or refreshing a note
re-renders the sidebar and nothing else.

The one exception is bookmarks, which *do* reach into rendered pages (the
highlight layers) — and that path already exists (`onBookmarksChanged` →
refresh), so it is handled, not new.

The reader **opens with its whole layer in hand** — eibits, bookmarks, and
discussions seed state at mount (server-provided; empty in preview). No mount-time
refetch. After seeding, each concern refreshes itself on the action that changes
it. The props seed; the callbacks maintain.

---

## 6. Skeleton-first lazy fill (the rendering pattern)

Both streams — pages and eibits — render the same way, because structure is cheap
and known while content is expensive:

1. **Skeleton** — render every unit as a placeholder immediately (real id, title,
   reserved space). The document has honest geometry and every navigation target
   exists from first paint, loaded or not.
2. **Proximity fill** — each placeholder fetches its content when it nears the
   viewport (an `IntersectionObserver` with a generous margin), before the reader
   arrives.
3. **Per-key store** — content lives in an external store keyed by unit id, with
   **per-key subscriptions**, so a unit filling in re-renders only itself.

Placeholders stay mounted (the opposite of virtualization), which is what makes
navigation a DOM operation and the scrollbar honest. Eibit names and page names
share one element registry, so one `scrollToElement` serves both streams.

> Deep dive: the pattern in full — `skeleton-first-lazy-fill`.

---

## 7. Keeping and removing — how it actually works

Keeping/removing an eibit can happen two ways, converging on one backend action
and one UI update:

- **The keep control** (a bookmark icon on the inline eibit) calls the **backend
  directly** with the eibit id. Deterministic, and **locale-independent** — it
  generates no message. This matters: the chat culture is whatever language the
  learner chose, unknown at build time, so a control that had to *compose* a
  keep/remove message couldn't be localized.
- **Conversationally** ("keep that", "remove that") goes through **EiBot tools**.
  The language is the learner's own, understood in context — never synthesized.

**Titles are spoken, ids are executed.** Messages reference an eibit by title;
tools use the id. EiBot recovers the id from the transcript — `store` returns a
message pairing title + id, so the id lives in history and EiBot quotes it
verbatim later rather than memorizing it.

Because keeping is asynchronous (it may route through EiBot), the inline keep
control has three states — `unkept`, `keeping`, `kept` — driven by a client-side
**broadcast bus**: the client publishes `{ name, state }` once, every mounted
eibit reacts only to its own name. Both paths end at the same publish, so the UI
is correct regardless of origin. A workbook change also emits an `agEvent` that
refreshes the reader's eibit list.

---

## 8. Presentation details

### Inline eibit (in chat)

Renders on a **lighter surface** than the chat panel — a distinct artifact
without a frame. Captioned at the bottom like a magazine figure: **Eibit** —
{title}, with a hairline above. The keep control is a single bookmark icon
top-right, revealed on hover when unkept, filled and persistent when kept — and
it belongs to the **wrapper**, not the eibit template (the template is EiBot's
authored content, identical wherever it renders).

### Programming-exercise component: size presets

The interactive coding component renders in three contexts along **two orthogonal
axes that cohabit**:

- **structure** (`isMaximized`) — resizable panels (modal) vs fixed divs
  (learning, chat).
- **size** (`large | compact`) — a CSS-variable **preset** set once on the group
  root and cascaded to leaves; learning is large, chat is compact.

Everything size-related routes through `--prog-*` variables defined per preset,
so the leaf CSS is size-agnostic and structure stays independent of size.

### Vocabulary boundary

The ADL model keeps its terms (`Excerpt`, `template`, `aggregate`); the reader
speaks the learner's (a template is a **page**, an aggregate a **section**;
"Eivolet" is fine to say). UI objects are `Bookmark*`; an excerpt's reference is a
`TemplateRef { name, culture }` (culture required — it's what makes the reference
meaningful). `TypedMaterialId` was eliminated once eibits anchored to the Eivolet:
its only job was a container-vs-asset branch that then vanished.

---

## 9. Design principles (that recur across the feature)

- **The material is the book; the learner's work is a quiet layer.** Reading is
  never disturbed by the layer; structure and personal accumulation stay
  separate.
- **Reliability via data-in-context, not model memory.** Ids EiBot must reproduce
  live in the transcript; it copies rather than invents. (Same principle as
  store-time naming: don't trust the LLM to carry identity.)
- **Store-time over render-time.** Corrections (normalize, stamp) happen once
  before storage; stored content is valid and needs no runtime repair.
- **Denormalize titles** where the creating context has what's needed; frozen
  sources ⇒ no drift.
- **Reuse existing mechanisms, never add an outsider path** (highlighter handles,
  `onBookmarksChanged`, the element registry, the external-store pattern). Two
  paths that must converge is a smell; when unavoidable, make them end at one
  action + one event.
- **Keep concerns where they're needed.** The keep control is the wrapper's, not
  the template's. Size is the caller's choice, the component's definition.
- **Orthogonal axes stay orthogonal** — structure vs size, never collapsed.
- **Lightweight where possible.** An eibit has no identity to track; value comes
  from the learner keeping it.

---

## 10. Eibooklets and Examples (in design)

*This section describes work still being designed. The programming example is
defined; other domains and some integration details are open.*

> The eibooklet **concept** — the general model, why it is deliberately open,
> and the access-point direction it unlocks — has its own document,
> `eibooklet-spec`. This section covers only how eibooklets integrate into the
> reader.

### Eibooklets — a third node kind

The reading tree today has two learner-facing node kinds — `section` and `page`.
A third is being added: **`eibooklet`**. So the tree the reader flattens is
`section | page | eibooklet`.

An **eibooklet** is a named, addressable, tree-positioned booklet attached to the
eibook that is **not part of the reading path's content** — you jump to it, you
don't read through it. Its meaning comes from *where it sits* in the tree: an
object under a page relates to that page, under a section to the section, under
the Eivolet to the whole material (scope-by-position). Because an eibooklet node is
not a leaf template, its content is stored via **contentgroup** — the same
technology as space environments (the "outside a leaf → contentgroup" rule). Like
every persistence detail, the contentgroup is invisible above the storage layer.

An eibooklet carries `metadata.tags` — **optional, informational** labels
(`[example, programming, runnable]`) for navigation and filtering (the Objects
TOC *could* group on `example`). Tags carry no behavioural contract and do **not**
select the renderer — they are metadata, not a discriminator. (This also keeps
`kind`, the node's structural kind, distinct from freeform `tags`.)

**Rendering is content-driven.** The object's payload is authored MDX (see
below), so the *components in it* determine what renders — `<ProgrammingExampleWorkbench>`
resolves to its renderer through the normal component map, exactly like
`<Diagram>` or `<FillBlankMultiExercise>` in a page. The reader renders the MDX;
it does not dispatch on tags. This keeps a single source of truth (the content)
and removes any coherence contract between metadata and the payload.

### How an eibooklet joins the reading flow

An eibooklet node renders in the reading stream **like a page**, because it
carries its own small summary body:

```yaml
model: eibooklet
metadata: { name: react-hooks-examples, namespace: eivo, tags: [example, programming, runnable] }
labels: [ ... ]            # multi-culture title + overview
def:
  format: markdown
  summary:                 # the in-flow presence — MDX with <ObjectLink/>
    - culture: en-US
      body: >-
        Explore the hooks used in this section hands-on. <ObjectLink/>
  content:                 # the full booklet — MDX with the payload components
    - culture: en-US
      body: |-
        <ProgrammingExampleWorkbench lang="typescript" platform="react" name="abc123">
          <ExampleFile path="src/App.tsx" role="scaffold" runnable>{`...`}</ExampleFile>
          <ExampleFile path="src/Counter.tsx" role="subject" title="Counter">{`...`}</ExampleFile>
        </ProgrammingExampleWorkbench>
```

Both halves are **MDX bodies**, culture-arrayed, rendered through the same
pipeline as pages: `def.summary` is the reading-flow teaser, `def.content` is the
full booklet. The payload's structure (files, roles, runnable, domain) lives as
**component attributes** (`<ProgrammingExampleWorkbench lang platform>`, `<ExampleFile path role
runnable>`), read by the renderer — not as separate YAML fields.

The summary body is ordinary MDX containing **`<ObjectLink/>`** — a real
component, not a placeholder. There is **no `{link}` substitution, no stamp
step**: at render, the eibooklet's summary is rendered in its own scope,
and the component map is built with `ObjectLink` **bound to the eibooklet's full
id** (a small reintroduction of the `buildBoundedComponents(materialId)` binding
pattern, scoped to this one component). The bound `ObjectLink` receives the id by
binding and, on click, calls the exposed **`renderEibooklet(id)`** action with it.
That action switches to the eibooklet's full-surface view (the same swap the eibit
view uses — the runnable example needs the room) and renders its `content` there. The authored body is never mutated;
the identity arrives the same way exercises and diagrams used to receive their
material id.

The payload's `<ProgrammingExampleWorkbench name="...">` is a **named component**, so
`def.object` goes through the same store-time **normalize → stamp** pipeline as a
template body (`ProgrammingExampleWorkbench` added to the stamped tags; the author's
`name` is replaced by a system-assigned one). `<ObjectLink/>` in `def.summary`
needs no stamp — it is a self-reference resolved by binding (above).

So integrating eibooklets into reading costs the reader almost nothing, and it
does so **without changing the reader's architecture** — two render paths, split
by body:

- **`summary`** renders through the **existing `getContent`**, unchanged. To the
  reading stream an eibooklet's summary is just page content, so the skeleton-
  first flow, scroll, and navigation handle it like any page.
- **`content`** (the full booklet) renders through a **separate action
  (`renderEibooklet`) and a `DynamicContentObjectRender`**, invoked by
  `ObjectLink`. The
  payload component tree renders there, in the eibooklet's full-surface view. This
  keeps the heavy render out of the reading path entirely — `getContent` never
  learns about eibooklets.

### The Eibooklets index

A navigation surface distinct from Contents. Where **Contents** is the *reading
path* (sections and pages, in order) and **Workbook** is the *learner's layer*,
the **Eibooklets index** lists the eibooklets attached to the eibook — things you
go to rather than read through. It routes by node kind: Contents lists `section`
+ `page` and excludes `eibooklet`; the Eibooklets index claims `eibooklet`.

Two groupings, reflecting the two ways an object is located:

- **Positional eibooklets** — `eibooklet` nodes in the tree, organised by their tree
  position, overview-rich. Standalone examples live here.
- **In-template objects** *(future)* — exercises and diagrams are already
  addressable (named, addressable, discussable); they simply live embedded in page
  bodies rather than as nodes. Listed by containing page, they'd make the index a complete catalogue of every addressable thing in the material. Not built yet;
  the eibooklet model is kept general so this fits later without rework.

### Two tiers of example

`example` (a kind) appears at two scales, distinguished by *location*, not by
being different things:

- **Summary example** — a standalone `eibooklet` node (multi-file, runnable,
  domain-scoped: the model above). A **positional** eibooklet. Reached from the
  reading flow via its link-template and from the Objects TOC.
- **Short inline example** — a small illustrative snippet embedded in a page
  body, like a diagram or exercise. An **in-template** object of kind `example`,
  sitting beside exercises and diagrams.

The two tiers use **different components** — `<ProgrammingExampleEditor>` inline
in a template, `<ProgrammingExampleWorkbench>` in an eibooklet — so the compact
inline editor and the full-surface workbench can evolve independently. Both are
named components and both are stamped at store time.

A `<ProgrammingExampleEditor>` embedded inline in an ordinary `model: template` — just
another component in the page body, rendered on the reading path like a diagram
or exercise (no eibooklet node, no separate view):

```yaml
model: template
metadata:
  name: variables-and-mutability
  namespace: eivo
labels:
  - culture: en-US
    title: Variables and Mutability
def:
  format: markdown
  content:
    - culture: en-US
      body: |-
        # Variables and Mutability

        In React, state is what makes a component re-render. `useState` gives a
        component a value and a setter; calling the setter schedules a re-render
        with the new value.

        Try it — increment the counter and watch it update:

        <ProgrammingExampleEditor lang="typescript" platform="react" name="abc123">
          <ExampleFile path="src/App.tsx" role="scaffold" runnable readonly>
            {`import { Counter } from './Counter';
        export default function App() { return <Counter />; }`}
          </ExampleFile>
          <ExampleFile path="src/Counter.tsx" role="subject" title="Counter">
            {`import { useState } from 'react';
        export function Counter() {
          const [count, setCount] = useState(0);
          return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
        }`}
          </ExampleFile>
        </ProgrammingExampleEditor>

        Notice the setter takes a function — `setCount(c => c + 1)` — so each
        click reads the latest value rather than a stale one.
```

The template renders through the ordinary page path (`getContent`); the embedded
`<ProgrammingExampleEditor>` resolves through the component map like any other embedded
component, and its `name` is stamped at store time. No eibooklet, no view switch
— this example lives *in* the reading flow. The standalone eibooklet tier (below)
is for larger, multi-file examples that warrant their own surface.

### The programming example (first concrete kind)

An `example`-tagged eibooklet in the programming domain. Its `def.content` is
authored MDX built around a `<ProgrammingExampleWorkbench>` component whose children are
role-tagged `<ExampleFile>`s. The component is **only the editor** (files, tabs,
run) — any framing prose is ordinary body MDX around it, styled like all other
content, not a slot inside the component. Roles:

- **subject** — the code under study; the focus.
- **context** — supporting code needed to run/understand, not the focus.
- **scaffold** — a runnable entry point (`runnable`) mounting a subject.

Each `<ExampleFile>` carries its `path`, `role`, `runnable`, `readonly` (which files
the learner can't edit — independent of role; absent means editable), and its own
title/description. `<ProgrammingExampleWorkbench>` carries the domain (`lang`, optional
`platform`) and a `name` — stamped uniquely at store time like any named
component. Rendering reuses the **read/run mode** of the programming-exercise
component (multi-file + run, no solve, no grade), with the role tags driving
which file is foregrounded, composing with the size presets (large in a page,
compact in chat). Discussing an example is a natural EiBot starting point: the
files-with-roles become context, and the roles tell EiBot what is focal
(subject) vs incidental (context).

A full eibooklet — `summary` (the in-flow presence) and `content` (the booklet
itself), both culture-arrayed MDX:

```yaml
model: eibooklet
metadata:
  name: react-hooks-examples
  namespace: eivo
  tags: [example, programming, runnable]
labels:
  - culture: en-US
    title: React Hooks Examples
    overview: Demonstrates common React hooks through isolated, runnable components.
  - culture: fr
    title: Exemples React Hooks
    overview: Démontre les hooks React courants à travers des composants isolés et exécutables.
def:
  format: markdown
  summary:                       # renders in the reading stream (page path)
    - culture: en-US
      body: >-
        Explore the hooks used in this section hands-on. <ObjectLink/>
    - culture: fr
      body: >-
        Explorez les hooks utilisés dans cette section. <ObjectLink/>
  content:                       # the full booklet (renderEibooklet RSA)
    - culture: en-US
      body: |-
        This example shows how `useState` drives re-renders. Change the subject
        and run it to see the effect.

        <ProgrammingExampleWorkbench lang="typescript" platform="react" name="abc123">
          <ExampleFile path="src/App.tsx" role="scaffold" runnable readonly>
            {`import { Counter } from './Counter';
        export default function App() { return <Counter />; }`}
          </ExampleFile>
          <ExampleFile
            path="src/Counter.tsx"
            role="subject"
            title="Counter Component"
            description="A counter using useState — state updates trigger a re-render."
          >
            {`import { useState } from 'react';
        export function Counter() {
          const [count, setCount] = useState(0);
          return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
        }`}
          </ExampleFile>
          <ExampleFile
            path="src/Toggle.tsx"
            role="context"
            readonly
            title="Toggle Component"
            description="A toggle switch built with useState."
          >
            {`import { useState } from 'react';
        export function Toggle() {
          const [on, setOn] = useState(false);
          return <button onClick={() => setOn(v => !v)}>{on ? 'On' : 'Off'}</button>;
        }`}
          </ExampleFile>
        </ProgrammingExampleWorkbench>
```

`summary` renders in the reading stream through the same `getContent` as a page
(its `<ObjectLink/>` calls `show-eibooklet` to switch views). `content` renders
in the full-surface eibooklet view through the `renderEibooklet` RSA. The
`<ProgrammingExampleWorkbench name="abc123">` goes through normalize → stamp at store time
(the author's `name` is replaced with a system-assigned one); `<ObjectLink/>`
needs no stamp — it is a self-reference bound to the eibooklet's own id.

Other domains (non-programming examples) are not yet designed; the `eibooklet`
model is kept general so new content types slot in with their own payload
components and (optionally) their own tags.

---

## 11. Component map

| Concern | Artifact |
|---|---|
| EiBot instructions & per-kind flows | `assistant-instructions` |
| Eibit authoring craft | `eibit-producer` |
| Store-time MDX normalize + name | `mdx-namer/` (`normalize.ts`, `stamp-names.ts`, `README`) |
| Rendering pattern | `skeleton-first-lazy-fill` |
| Workbook panel, eibit stream, view-marks | `workbook/` (panel, models, `eibits-stream`, `book-view-marks`) |
| Inline eibit + keep bus | `eibit-chat`, `eibit-keep-bus` |
| Coding component size presets | `prog-exercise.css` |

---

## 12. Pending activities

Work not yet complete, each with what it is and why. Grouped by area; roughly
ordered by how much they block other work.

### Eibooklets

- **Finish the `eibooklet` node kind end-to-end.** `EiNode.kind` must include
  `'eibooklet'`, and the tree-flattening helpers (`pageSequence`,
  `ancestorIndex`) must include eibooklet nodes — otherwise an eibooklet's
  summary renders in the stream but is missing from the reading order (bookmarks
  and eibits near it sort wrong) and scrolling onto it won't open its section in
  the Contents TOC. The reader view, view-mark, `show-eibooklet` action, the
  `EibookletView` (via the `renderEibooklet` RSA), and the `EibookletToc` (sidebar
  tab + in-view landing) are done. *Blocking: eibooklets don't order/highlight
  correctly until the helpers include them.*
- **Stamp the example components.** Add `ProgrammingExampleEditor` and
  `ProgrammingExampleWorkbench` to the stamped component tags, so their `name`s
  are system-assigned at store time like exercises and diagrams. Without this the
  author's placeholder `name` survives and `(template, name)` resolution is
  unreliable. *Blocking: example discussion/validation resolution.*
- **CSS for the eibooklet view and TOC.** The `book-eibooklet*` classes
  (`EibookletView` states, the `book-eibooklet-toc` behaviour overrides) need
  styling; the TOC reuses `book-toc-*` and only marks its open-not-scroll
  difference.
- **First generative eibooklet kind (design).** The eibooklet is the access-point
  pattern for positioned capabilities (see `eibooklet-spec` §5): the first
  generative kind — *practice*, an llmmaterial-backed component that generates
  exercises scoped to the eibooklet's tree position, repeatably — needs its
  content component, its llmmaterial binding, and a rule for how repeated
  generation is scoped. *Not started; unlocks the broader "access point to
  everything" direction.*
- **Eibooklets index → launchpad.** Once more than one kind exists, evolve the
  Eibooklets surface from a flat positional list into a tag-categorized launchpad
  (example / practice / challenge / session). Tags already exist for this; the
  surface doesn't group on them yet.
- **Non-programming example domains.** The `example` shape is defined only for
  programming (workbench + role-tagged files). Other domains are unexplored; the
  model is general enough to admit them as new content components.

### Store-time pipeline

- **Migration pass for existing templates.** Templates stored before
  normalize/stamp need a one-time pass to normalize and name their components.
  Until then the render-time transformers stay as idempotent no-ops so old and
  new content both render. *Blocking: retiring the render-time transformers.*
- **MDX pipeline package move.** Move normalize/stamp/render into the Content
  package, with the render provider parameterized by the components + extractor
  plugins supplied from Facets (resolves the Facets↔Content cycle via constructor
  injection). Architectural cleanup, not learner-facing.

### Reader & workbook

- **Excerpt→session link for deletion.** Deleting a highlight should also remove
  its discussion, but the excerpt doesn't store the session id. Plan:
  reconstruct the session key on remove; if the reconstruction is ever wrong the
  session GCs on its TTL, and the row is dropped from the local discussions list
  regardless so the UI stays correct. *Low-risk by design; not yet wired.*
- **Tune the compact preset.** The programming-exercise component's `compact`
  size preset (used in chat) has conservative starting values; tune them against
  real chat renders.

### Discussions

- **`example` discussion kind.** Opening a discussion from an example
  (component-owned entry point, files-and-roles as context) is declared and the
  `'example'` kind is added on the client; confirm the opener/context assembly
  end-to-end. See `eibot-discussion-sessions-spec`.
