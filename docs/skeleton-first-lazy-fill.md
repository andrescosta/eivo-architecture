# Skeleton-First Lazy Fill

A pattern for rendering a long document whose **structure is known cheaply**
and whose **content is expensive to fetch**.

---

## The problem

You have an ordered collection of content units — pages, chapters, records,
sections. Two facts about them:

1. **The structure is available up front and cheap**: how many units there are,
   their order, their identities, their titles. Usually one small request, or
   already in hand.
2. **The content is expensive and unbounded**: each unit's body must be fetched
   and rendered; fetching all of them is slow, wasteful, or impossible.

The naive approaches both fail:

- **Fetch everything, then render** — the reader waits for content they will
  never scroll to; time-to-first-paint scales with the whole document.
- **Fetch as you go, render nothing until then** — the document has no geometry.
  The scrollbar lies, navigation targets do not exist, and jumping to unit 40
  is impossible because nothing between here and there has been created.

The second failure is the interesting one, and it is what the pattern exists to
solve.

---

## The mechanism

**Render the entire structure immediately. Fill each unit's content on demand.**

### 1. The skeleton

On first paint, render *every* unit as a placeholder: a real DOM element with
its identity, its title or heading, and an estimated intrinsic size. Nothing is
fetched.

The skeleton is derived purely from the structure — which is why the pattern
requires structure to be cheap. Every unit exists in the document from the first
frame onward, whether or not its content ever arrives.

### 2. The proximity trigger

Each placeholder observes its own distance from the viewport, using a generous
margin — content is requested *before* the reader arrives, not when they get
there. In the browser this is an `IntersectionObserver` with a large
`rootMargin` (a viewport or more of lead), scoped to the scrolling container.

The observer fires once and disconnects: a unit is fetched at most once. Error
states re-arm it so a retry is possible.

### 3. The store

Content lives in an external store keyed by unit identity, exposing:

- `getEntry(key)` — the current state: absent, loading, error, or present
- `subscribe(key, listener)` — notification when *that key* changes
- `ensure(key)` — request the content if not already present or in flight

Per-key subscription is the important part. Each placeholder subscribes only to
its own entry, so a unit filling in re-renders **that unit alone** — the parent
does not re-render, and neighbouring units are untouched. Without this, every
arrival re-renders the whole document, and the pattern's performance advantage
disappears.

In React this is `useSyncExternalStore` per placeholder; the equivalent in any
framework is a fine-grained subscription rather than a single document-level
state object.

### 4. The render

Each placeholder renders from its entry's status:

| status | renders |
|--------|---------|
| absent | placeholder shell (heading + reserved space) |
| loading | shell + progress indicator |
| error | shell + message + retry, re-arming the observer |
| present | the content |

The shell persists across all four — the unit's element identity never changes,
only what it contains.

---

## Invariants

These are what the pattern buys, and what any implementation must preserve.

**I1. Every unit exists in the DOM from first paint.**
Navigation to any unit works immediately, loaded or not. A table of contents can
scroll to unit 40 without unit 40's content — or units 2 through 39 — having
been fetched.

**I2. Scroll geometry is stable.**
Because placeholders reserve their space, the scrollbar is honest from the start
and does not jump as content arrives. Accuracy depends on how well the estimated
size matches the real one; a bad estimate causes visible reflow when content
replaces the shell.

**I3. Work is bounded by attention, not by document size.**
A 500-unit document costs one skeleton render plus the content of what the
reader approaches. Cost scales with reading, not with the collection.

**I4. Fetching is idempotent per unit.**
The store deduplicates: concurrent triggers for the same key produce one
request. Units are fetched once and cached; re-entering a unit's proximity does
not refetch.

**I5. Content arrival is locally scoped.**
A unit filling in affects only that unit's subtree. This is what makes the
pattern viable at scale, and it constrains the design: shared mutable state
across units, or side effects that reach outside a unit's subtree, break it.

---

## What this is not

**Not virtualization.** Virtual scrolling unmounts off-screen items to bound DOM
size, and computes scroll geometry from estimated row heights. This pattern does
the opposite: it keeps every placeholder mounted precisely so navigation targets
and geometry are real rather than simulated.

The tradeoff is explicit. Virtualization handles millions of rows but makes
"scroll to item 40,000" a computation; skeleton-first handles hundreds of units
and makes it a DOM operation. For documents — where the unit count is bounded by
what a human will read and each unit is substantial — the DOM cost of empty
placeholders is negligible and the guarantees are worth more.

**Not progressive enhancement.** The content is not a nicer version of what is
already there; the placeholder is genuinely empty.

**Not infinite scroll.** The collection is finite and fully known at the start.
Infinite scroll has the opposite shape: unknown extent, structure discovered by
fetching.

---

## Preconditions

The pattern applies when all of these hold:

1. **Structure is cheap** — obtainable in one small request, or already loaded.
2. **The collection is bounded** — hundreds, not millions. Empty placeholders
   are cheap but not free.
3. **Units are independent** — a unit's content can be fetched and rendered
   without any other unit's content.
4. **Order is stable** — units do not reorder during a session, so scroll
   positions and geometry remain meaningful.

If structure is expensive, you need pagination or infinite scroll instead. If
the collection is unbounded, you need virtualization. If units are
interdependent, you need a different fetching strategy.

---

## Extensions

**Explicit ensure.** Navigation can call `ensure(key)` directly rather than
waiting for the proximity observer — jumping to a distant unit should start its
fetch immediately rather than after the scroll lands.

**Scroll behaviour by distance.** Smooth-scrolling across many units drags every
intervening placeholder through the proximity band, triggering a cascade of
fetches. Long jumps should be instant; only nearby targets should animate.

**Batched fetching.** The store can coalesce concurrent `ensure` calls into one
request, which matters when a fast scroll crosses several units at once.

**Post-render hydration.** Units whose content needs work after it lands
(diagram rendering, syntax highlighting, measurement) hydrate scoped to their
own subtree, so a unit arriving late never touches the rest of the document.

**Deferred targets.** When navigation must reach something *inside* a unit's
content (a highlight, an anchor), the target may not exist when the scroll
completes. Ensure the unit, scroll to its placeholder, then retry the inner
focus until the content resolves or a timeout expires.

---

## Failure modes

**Estimated sizes far from reality.** Placeholders reserve the wrong space,
producing visible jumps as content replaces shells and undermining I2. Estimates
should be derived from the structure where possible (unit type, expected
length), not a global constant.

**Coarse subscription.** Subscribing to the whole store rather than per key
turns every arrival into a full re-render, breaking I5 and making the pattern
slower than fetching everything.

**Proximity margin too small.** Content requested only as the unit enters view
arrives after the reader does, so the shell is visible on every unit. The margin
should cover the distance a reader can scroll in the time a fetch takes.

**Proximity margin too large.** With a very generous margin, most of the
document is in range at once and the pattern degenerates toward fetching
everything, breaking I3.

**Observers on a changing collection.** If units can be added or removed, the
observer set and the store's keys must be reconciled, or a removed unit's
pending fetch resolves into nothing and a new unit is never observed.
