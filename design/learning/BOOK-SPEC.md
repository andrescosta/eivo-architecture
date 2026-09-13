# Book Reader — Spec

Status: covers the reading experience — TOC, on-demand page loading, and
its integration with the highlight layer (SPEC.md). Prototype route:
`/book`. Not yet integrated into Eivo.

## Problem

Render an Eivolet as a book: a table of contents on the left, a
continuous scrollable content panel on the right. Rendering every
template up front is too expensive; pages must load on demand as the
reader approaches them — without the loading mechanism being visible in
the reading experience (no stalls, no scroll jumps, no dead navigation
targets).

## Architecture: the skeleton owns the geometry

The Eivolet's structure — sections, nesting, page order — is known before
any content is. The content panel therefore renders the **complete book
skeleton on first paint**: real section headings (from the tree) 
interleaved with one placeholder per page. Content fills into
placeholders; the document's shape never changes as it loads.

Consequences, each load-bearing:

- **Every navigation target exists from the first frame.** A TOC click is
  `scrollIntoView` on an element that is already there, loaded or not.
- **The scrollbar is honest.** Placeholders carry
  `contain-intrinsic-size` estimates; measured height takes over after a
  page renders.
- **Nothing is inserted above or below the viewport as a *document*
  operation** — pages fill *in place*, so there is no prepend scroll-jump
  and no append sentinel to stall.
- **Pages never unmount.** `content-visibility: auto` makes off-screen
  pages cost no layout or paint work, so keeping the whole book in the
  DOM is viable — and it is what the highlight layer requires: each
  page's projection and resolved Ranges live for the whole session.

## Data flow

```
tree (structure only, serializable) ──────► client, at page load
                                            │
                                            ▼
                              skeleton: headings + placeholders
                                            │  proximity observer fires
                                            ▼
                              BookPageStore.ensure(name)
                                            │  batched
                                            ▼
                    fetchPages(names) — server action ('use server')
                    renders each template server-side (renderTemplate)
                    returns { name, content: RSC payload, excerpts }[]
                                            │
                                            ▼
                              placeholder renders content inside
                              EivoTemplateHighlighter → HighlightLayer
                              mounts → projects → resolves excerpts
```

Templates render **server-side, always** — on-demand delivery is RSC
payload across the action boundary, never MDX source; the client bundles
no compiler and never sees a body. `{ content, excerpts }` travel
together, so highlight resolution has its data the moment content mounts
— no ordering race.

## `BookPageStore` (`lib/book/book-page-store.ts`)

Single owner of page content on the client. Placeholders subscribe per
page (`useSyncExternalStore` contract) and render whatever their entry
says.

- **Statuses**: `idle → loading → present | error`. Failure is recorded
  as `error` and is retryable (`ensure()` again); it is never recorded as
  presence. A page missing from a batch response errors individually.
- **Batching**: `ensure()` calls within one short window (30 ms) collapse
  into a single backend call — proximity observers fire in bursts.
- **In-flight dedupe**: the pending request counts as fetched; a page is
  never requested twice, whatever the interleaving of observer fires and
  retries.
- Content is fetched at most once per session per page (immutable
  templates); the store never invalidates.

## Observers — two jobs, two configurations

- **Proximity loading** (per placeholder, `PagePlaceholder`): root is the
  scroll container, `rootMargin: '150% 0%'` — fetch fires one-to-two
  viewports ahead of the reading position, in both directions (upward
  scrolling and downward are the same case). Fires once and disconnects;
  an `error` entry re-arms it, and the retry button calls `ensure`
  directly.
- **Reading-line detection** (one observer, `BookContent`): a narrow band
  in the upper third of the viewport (`rootMargin: '-25% 0% -65% 0%'`)
  reports which page is being read. More than one page can intersect the
  band at once (short pages, unloaded placeholders), so intersection
  state is tracked per page and the **topmost intersecting page in
  reading order** is the active one. Drives the TOC's active styling and
  auto-expands the active page's ancestor sections. The loading state carries a
  min-height matching the `contain-intrinsic-size` estimate so pre-load
  geometry behaves like post-load geometry. It is deliberately
  structure-agnostic — a template's content can be anything, so the
  indicator suggests nothing about its shape.

The two are deliberately separate: loading wants a wide margin,
position detection a tight band; one observer cannot serve both.

## Components

- **`BookReader`** — takes the material's `materialId: string[]`,
  threaded down (BookContent → PagePlaceholder → EivoTemplateHighlighter
  → StoreClient) so bookmark persistence carries the material identity,
  matching Eivo's `addBookmark(materialId, name, …)` contract. Layout
  shell and navigation state: active page, open
  sections, sidebar visibility, element registry (name → element, for TOC
  scrolls), discuss panel. Owns the `BookPageStore` instance. The sidebar
  collapses via a panel icon in its tab row; while hidden, the same icon
  floats at the top-left of the reading area to reopen it.
- **`BookToc`** — pure view over the tree plus that state. Sections
  toggle; pages navigate. Active page and its ancestor path are styled.
- **`BookContent`** — renders the skeleton from the tree and owns the
  reading-line observer.
- **`PagePlaceholder`** — one page: registers its element, runs its
  proximity observer, renders by status (structure-agnostic loading
  indicator / error with retry / content), and wraps present content in
  `EivoTemplateHighlighter` — which is where the highlight layer's
  contract (mount → project → resolve, per instance) plugs in unchanged.

Navigation: TOC clicks scroll smoothly to nearby targets and jump
instantly to far ones — smooth-scrolling across many pages would drag
their proximity observers through the loading band and fetch everything
in between.

## Highlight popups under containment

`content-visibility: auto` on `.book-page` implies paint containment:
the page clips its descendants' painting and becomes the containing
block for absolutely-positioned ones. The highlight layer's popups are
therefore fixed-position and portaled to `document.body` with viewport
coordinates (see SPEC.md, "Popup positioning") — rendered in place they
would be mispositioned and clipped, which presents as selection doing
nothing. A pending popup does not follow content while the book scrolls;
it closes on the next interaction.

## Eivolet and section discussions

A floating action button (Remix `chat-ai-2-line`, inlined — Eivo would
route it through the Iconify platform abstraction) at the bottom-right
of the reading area opens the EiBot panel with the **Eivolet itself as
the subject**. `EivoDiscussPanel` takes a `DiscussSubject` — a specific
excerpt highlight (with its delete affordance) or the whole Eivolet —
so excerpt, section, and material-level discussions share one panel and
one open/close state; opening any replaces the current one. The button
hides while the panel is open.

The panel header follows the EiBot preview-panel pattern (uppercase
title, `eb-panel-btn`-style icon buttons): **maximize/restore** toggles
a fixed full-viewport state (`--maximized`, matching
`eb-preview-panel--maximized`), and **close** dismisses. The panel is **resizable** by dragging its left edge (clamped between a
minimum and 70% of the viewport; the handle is absent while maximized).
Maximization and width are panel-local state — they reset when the panel
opens for a new subject.

**Every subject carries the full hierarchical `id: string[]`** the
discussion RSA needs: TOC subjects take it from the node in hand;
excerpt subjects resolve it through the reader's `name → EiNode` index
(the placeholder tags each activation with its page name); the
**eivolet's id is the id of the first node passed to the reader**.

Every **TOC row** carries the same icon at the mockup's smallest icon
size (`--width/height-icon-xxs`), revealed on row hover, opening the
panel with that section as the subject; the click neither toggles nor
navigates. The discussion entry points are exactly three — **excerpt,
section (aggregate), eivolet** — there is no page/template entry point:
templates are content, not navigable structure.

## Preview mode

The reader also serves content preview. `BookReader` takes `isPreview`
(default `false`) — the session's own flag, kept as-is rather than
narrowed to a marking switch, so future preview-only behavior hangs off
the same prop. Currently, preview disables marking: pages render their
content **bare** — the highlight layer never mounts, so there is no
selection capture, no popups, no excerpt resolution, and no highlighter
handles (`PagePlaceholder` receives this as `bookmarksSupport`, the
content-level capability it maps to); the Bookmarks tab is absent and
the bookmark list is never fetched. **Every EiBot activation point is
absent too**: the FAB doesn't render and the TOC receives no
`onDiscussNode` handler — the TOC's affordance contract is
handler-presence, so it stays mode-ignorant. Prototype route:
`/book/preview`.

## Column width under multiple panels

Up to three columns can be visible: TOC, book, discuss panel
(resizable, hosting the standard EiBot chat as children). The layout is
governed by a five-rule contract (stated in globals.css at
`.book-layout`): a viewport-high row; exactly **one elastic column**
(the book scroll, `flex: 1 1 0` + `min-width: 0`) with every other
column `flex: none` at explicit width; per-column scrolling; overlays
as `position: fixed` outside the flow; and embedded `width: 100%`
components resolving against the panel body's definite box. No
per-state style classes exist anywhere. The book column is centered
with gutters in **container query units** (`clamp(1.5rem, 6cqi, 4rem)`
against the scroll container) — one declaration that scales
continuously with the column's actual width, whatever combination of
panels produced it.

## Bookmarks

The sidebar has two tabs: **Contents** (the TOC) and **Bookmarks** — every
bookmark in the book, grouped by page in reading order, each showing its
type color, an excerpt preview, and the note text where present. The
list loads on mount (`fetchBookmarks` server action) and refreshes when
a bookmark is created, deleted, or its note saved
(`EivoTemplateHighlighter`'s `onBookmarksChanged`, threaded up through
the placeholder).

**Navigating to a bookmark** works whether or not its page is loaded:
`handleNavigateToBookmark` requests the page from the store, scrolls to
its element (placeholder or content — it always exists), and then calls
the page's `HighlightLayerHandle.focusHighlight(id)` (see SPEC.md,
"Focus"), retrying on an interval until the highlight layer has resolved
the excerpt — content must arrive, mount, project, and resolve first —
and giving up silently after a timeout, since an excerpt may be
unresolved by design (the silent-drop policy). A successful focus scrolls
the highlight to center and flashes it.

Per-page `HighlightLayerHandle`s reach the reader through a registration
callback (`registerHighlighter`), mirroring the element registry — the
reader holds `name → handle` and never reaches into a page's internals.

## Wire types (`lib/book/book-types.ts`)

**Invariant: leaf sections contain exactly one template.** The TOC
relies on it — it projects **structure only** (sections; template nodes
never render as rows), with each leaf section standing for its single
template: its row navigates to the section heading, its discuss
affordance opens a `section` subject, and the active row is the parent
section of the template under the reading line (`parentSectionIndex`).
The content skeleton still projects the full tree (headings +
per-template placeholders) — one tree, two projections. If authoring
ever relaxes the invariant, this section must be revisited.

`EiNode` — the same tree shape Eivo's learning session produces
(`{ id, kind, name, title, status, children?, parent? }`), consumed
unchanged. Two contracts on top of it: `parent` must be undefined on the
wire (a back-reference is circular; ancestry is derived by traversal,
`ancestorIndex`), and the node's `status` is the structural field from
the model — the runtime load status of a page's content lives in the
store (`PageEntry`), which remains its single owner. `PagePayload` —
`{ name, content, excerpts }`. `PageEntry` — status plus optional
content/excerpts; immutable, replaced on change.

## Testing

`scripts/test-page-store.ts` (part of `npm test`) covers the store:
burst batching into a single fetch, in-flight and present dedupe,
error/retry transitions, per-page errors on partial responses, and
subscription notifications. The observer/scroll behavior and
`content-visibility` interaction require a real browser.

## Prototype emulation notes

The backend is emulated: a mock tree and MDX bodies
(`lib/book/book-data.ts`), a server action with simulated latency
(`app/book/actions.ts`), and the in-memory excerpt store shared with the
`/` route (page name serves as `templateId`). The architecture — skeleton,
store, observers, RSC-across-action delivery — is the integration target;
the emulated parts are the data source and persistence only.

## Known limitations

- **`contain-intrinsic-size` is a fixed estimate** (30rem per page). Far
  scroll positions drift as unrendered pages above correct to measured
  height. A per-page size hint in the tree (word count or stored measured
  height) is the refinement if drift bothers in practice.
- **No reading-position persistence** — reopening the book starts at the
  top.
- **Discuss panel is the prototype stub** shared with the `/` route.
- **Bookmark list refresh is fetch-based** — created/deleted bookmarks
  refetch the full list rather than patching it locally; fine at book
  scale, and the seam (`onBookmarksChanged`) is where a real integration
  would swap in smarter cache handling.
