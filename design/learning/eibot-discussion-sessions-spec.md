# EiBot Discussion Sessions — Spec

Status: design for the "Discuss with EiBot" capability in the learning
experience (the book reader). Covers entry points, session identity and
lifecycle, context assembly, the tool surface, and inline exercise
generation. The reader-side entry points and subject identities exist in
the book prototype; the panel is chrome around the standard EiBot chat
component.

## Entry points

Discussions open from exactly three entry points:

- **excerpt** — a highlighted passage (the discuss mark)
- **aggregate** — a section row in the TOC (leaf sections stand for
  their single template — the authoring invariant)
- **eivolet** — the whole material (the reader's floating action button)

There is no template entry point: templates are content, not navigable
structure. `exercise` is a declared future entry point (component-owned,
inside the exercise's boundary, carrying attempt state).

The client sends **identity only**: `kind` plus the subject anchor id,
named by kind — `eivoletId`, `aggregateId`, or `templateId` (an excerpt
anchors to its containing template). `kind` is the platform-wide
discriminant name (`EiNode`, subjects, and assistance data all use it).
Ids are **full arrays** — the complete `[namespace, eivolet, …, node]`
path from the namespace down to the node — matching what the data-access
routines expect, so nothing reconstructs or normalizes a partial id. The
eivolet id is the `[namespace, eivolet]` prefix; an aggregate/template
anchor is the full path to that node. The server resolves everything
else.

## Session identity

- **`externalId`** — the caller-supplied, EiBot-opaque id segment. An
  **id factory** — one function, the single derivation seam — builds it
  from the kind and its anchor id; composition rules per kind live
  nowhere else.
  Both the open and close paths use it, so agreement is structural.
- **`sessionId = role + user + externalId`** — composed, not mapped. No
  pointer store, no minted ids: the session exists at its composed key
  or it doesn't, which makes open a get-or-create and close an exact
  deletion by derivation.
- **No client-side session state.** The client holds the sessionId only
  as a runtime handle born from open and dying with the panel; nothing
  is persisted (the discuss mark stores no session id in `data` — the
  deterministic derivation makes reopening land in the same
  conversation from any device, by construction).
- **Close** — `closeDiscussion({ kind, ...anchorId })`, same shape as open,
  fired **fire-and-forget** from the discuss-mark delete path: the
  discussion's lifetime is the mark's lifetime. No-op if the session
  never existed.

## EiBotAssistant

The domain-blind core owning the mechanics of every EiBot feature: loop
activation, message recovery, context assembly, persistence. A session
is fully determined by **role + session**: the Role (YAML) fixes what
the agent is — instructions assembly from crafts, the tool surface —
and the session carries what it is about. Features never extend the
core; they differ only in what they hand in.

The learning feature's server-side footprint is therefore: the id
factory, an opener RSA that resolves the kind and anchor id into an
`AssistanceData` object and hands it to the core with role `assistant`,
and the two YAML artifacts. Everything conversational — storage,
recovery, the loop, the protocol — is existing EiBot infrastructure,
unaware a new feature exists.

### AssistanceData

The opener's output and the core's input — resolved server-side, typed,
discriminated on `kind` (the client never supplies content, only ids):

```ts
export type AssistanceData =
  | { kind: 'eivolet';   materialId: string[] }
  | { kind: 'aggregate'; materialId: string[] }
  | { kind: 'template';  materialId: string[]; templateName: string[] }
  | { kind: 'excerpt';   materialId: string[]; templateName: string[]; excerpt: string }
  | { kind: 'diagram';   materialId: string[]; templateName: string[]; diagramName: string; diagramType: string }
  | { kind: 'exercise';  materialId: string[]; templateName: string[]; name: string };
```

`template`, `diagram`, and `exercise` are core input kinds without
reader entry points today (or component-owned, in the case of `diagram`
and `exercise`) — entry points and context kinds are different sets.

For the excerpt kind, the opener resolves the discuss mark's **record by
id** (the sequencing guarantee makes this safe: highlight creation is
awaited before `activateOnCreate` fires, so the record exists before
any open call) to obtain the excerpt text. Only `discuss` marks open
sessions — `note` marks are a separate type with their own dialog and
no path to EiBot — so an excerpt session is always a discuss mark, and
the mark type carries no information. Excerpt text never participates
in identity.

## Context

Principle: **skeleton always, bodies by kind.**

- **The skeleton** — the Eivolet's structure, rooted at the eivolet node
  (aggregates and templates: titles, kinds, each node's **full id** — the
  complete `[namespace, eivolet, …, node]` array the tools consume as-is,
  with ancestry also visible in the nesting — and a one-line authored
  `overview` per node where present; no full content, no parent
  back-references) — is assembled **once at session creation** and
  stored as a **session property** alongside the messages. Templates
  are immutable, so it is session-constant: every turn and every
  recovery reads it from the session, never re-walking the material.
  Every id in it is directly usable as a tool argument.
- **The content layer** varies by kind: `excerpt` → the excerpt text plus the containing template's body (`templateBody`);
  `template` → the body; `aggregate` and `eivolet` → nothing beyond the
  skeleton (plus material metadata for `eivolet`). Breadcrumbs are
  gone — derivable from the skeleton.
- Cross-cutting, all kinds: the **Eivolet objective** (the declared
  learning objective from the aggregate's values — sets level, framing,
  and scope; grounds recommendations and exercises), learner
  culture/language, learning-cap role context. All session-constant,
  assembled once.

Excerpts are located semantically: the quoted excerpt plus the full
template body is sufficient LLM context. No server-side offset
resolution against MDX — that would reintroduce the two-representations
problem the highlight architecture eliminated.

The session record thus splits into **constant properties** (skeleton,
content layer, role, culture) written once at creation, and **the
growing message log**; each turn's injected context is a pure read over
both.

## Client lifecycle

A session is **not conversable until its messages are recovered** — a
state, not a loading nicety, and a pattern shared by every EiBot
feature:

```
opening → recovering → ready | failed
```

- `openDiscussion({ kind, ...anchorId })` → session data (sessionId, chat
  endpoint, token, schema — the schema is session data, delivered here).
  The panel renders immediately; input is disabled.
- A **common, feature-agnostic recovery RSA** fetches messages **by
  sessionId** from session storage (bounded: most recent page;
  scroll-up pagination is a later additive). Recovery is idempotent —
  replace, never append (double-fired mount effects).
- `failed` is first-class and retryable: recovery failure retries with
  the held sessionId; open failure retries from the top.

The pattern lives in one shared client piece (hook/controller): a
feature supplies its open call and a message renderer; lifecycle,
`send` gating, idempotent recovery, and retry come from the shared
layer. The discuss panel (`EivoDiscussPanel`) is chrome — header,
maximize/restore, resize, close, excerpt delete footer — wrapping the
standard EiBot chat component as `children`; the chat mounts unchanged.

## Tools

Launch surface — scoped to the session's eivolet:

- **`get-template(id)`** — the body of a template, by its **full id**
  from the skeleton (the complete `[namespace, eivolet, …, node]` array,
  used as-is). Solution and answer fields of any exercises are stripped.
  Never called for the session's own template — `templateBody` is in
  context.
- **`load-craft(name)`** — loads a knowledge craft into the session; see
  **Craft loading** below. Load `eibit-producer` before authoring eibits.
- **`store-eibit-session(typedMaterialId, mdx)`** — EiBot authors the
  eibit (MDX) and calls this to **store it in the session**; it returns
  an **id**. EiBot returns the id as the `agEibit` item; the client
  renders it by id (RSA, below). Same shape as eivolet authoring: the
  work-in-progress lives in the session and is previewed from there.
  This is the **only** landing place at first — every eibit is
  session-only until the learner keeps it.
- **`store-eibit-workbook(id)`** — **promotes** a session eibit into the
  learner's workbook (durable), **only on the learner's explicit keep**
  — the keep link on the rendered eibit, or a spoken "keep that". The
  agent never promotes on its own initiative, and a *generation* request
  is never a keep (asking for an eibit is not asking to store it).
  Analogous to publish in authoring.

Rendering is a client **RSA** (`get-eibit-content(id)`), not an agent
tool: the client resolves the session-stored eibit by id and renders it
through the existing pipeline — the book's placeholder fetch-by-id
pattern, and the same resolve-from-session preview used in authoring.
`useObject` only ever sees the id.

The chat surface is exactly these two stores plus `load-craft`. Viewing
the collection and removing eibits are workbook/TOC concerns, outside the
chat; no read-back or remove tool exists in the conversation.

(`get-tree` was removed: the skeleton is always present and
session-constant — there is nothing left for the tool to fetch.)

Deferred, each with its trigger:

- **`get-page-summary(id)`** — added when transcripts show the agent
  fetching many bodies in one turn (wide questions on wide subjects).
  Lazy generation via `PromptFoundry` on a cheap `ModelConfig`, cached
  permanently by template id (immutability makes the cache equivalent
  to publish-time precomputation; Anvil can warm it at publish).
  Session open never pays summary cost.
- **`get-learner-marks(templateId)`**, **`get-progress(materialId)`** — as
  their entry points mature. (Exercise attempts are not a tool: they ride
  the `exercise`-kind context directly, as the learner's work is that
  session's subject.)
- A UI-facing navigation reference: EiBot emits a node id (+ optional
  excerpt) the panel renders as a jump chip, riding the existing agent
  protocol message family.

## Solution custody

One principle, enforced at both boundaries: **exercise solutions never
cross the client boundary, in either direction.**

- Inbound: `get-template` strips solution/values fields from
  exercise-bearing templates. The **`exercise` kind is the one
  exception**: its context carries the full authored body *with*
  `<Solutions>`, because helping a learner with an exercise they are
  working on needs the answers and tips. This does not breach the
  principle — the strip protects the *client* boundary; the exercise
  context is agent-only server-side context and never crosses to the
  client. Whether the learner sees an answer is governed by the
  hint-ladder pedagogy, not by withholding it from the agent.
- Outbound: eibit content travels as a **tool-call argument**
  (`store-eibit-session` — server-only), and the agent's visible message
  carries only an id — custody by mechanism, not prompt discipline.
  What the client fetches to render is the stripped projection, the
  same strip the page pipeline applies. Nothing content-bearing ever
  rides the message stream, by construction.

Stated limit: the agent itself knows the answers (it authored them),
and a determined learner can always cheat (ask another LLM). The tool
boundary eliminates *accidental* exposure; making honest use the best
path is craft-level pedagogy (hint ladders, probing misconceptions,
revealing when revealing teaches). Two layers, two jobs; neither is
anti-cheating enforcement.

## Craft loading

**`load-craft(name)`** is EiBot infrastructure, not a discussion-only
feature: it loads a knowledge craft into the session on demand, keeping
base instructions lean by deferring heavyweight knowledge (eibit
authoring vocabulary, exercise element shapes, future capability packs)
until a capability is actually used. Any role can adopt it — the editor's
authoring instructions are a natural next customer.

**Progressive disclosure — the skills pattern.** The design is the
agent-skill model: a lightweight **index** is always present, the heavy
**bodies** load on demand.

- **The role's `library`** lists the loadable crafts by reference:

  ```yaml
  model: agenticrole
  metadata: { name: assistant }
  def:
    crafts:   [ { name: assistant-instructions } ]   # always assembled
    tools:    [ { name: get-template } ]              # static tool surface
    library:  [ { name: eibit-producer } ]            # loadable on demand
  ```

  `crafts` assemble into the instructions always; `tools` is the static
  surface; **`library`** is the shelf `load-craft` fetches from.

- **Each craft self-describes** through its `labels` (title + overview),
  the same labeling mechanism every other labeled object uses. The
  overview does double duty: human description, and the model's
  load-trigger cue.

- **Assembly renders the index** — at final-instruction assembly, each
  `library` reference resolves to its craft, its `labels` are pulled, and
  a **craft-library table** (name / title / overview) is emitted at the
  **end** of the instructions. This index is the always-present part: it
  is cheap (a line per craft), and its only job is *recognition* — it
  lets the agent know a capability exists and when to reach for it. The
  bodies (`def.prompt`) never enter context until loaded, so adding
  crafts keeps context flat.

Decisions:

- **Index always present, bodies on demand** — deferring the index too
  would leave the agent unable to know a craft exists to load it (blind
  name-guessing → hallucination). The index stays lean (name + title +
  overview only, never the body) precisely so the bodies can be lazy.
- **No enum — the list is knowledge, not schema.** `load-craft`'s `name`
  parameter is a plain `string`; the valid names live in the rendered
  library table (injected knowledge), which satisfies the Craft Contract
  lesson (the model won't hallucinate names when the valid list is in
  context) without an enum and its provider quirks. The executor
  validates the name against the role's `library` and returns a
  correctable error for an unknown name.
- **`library` is role-scoped** — a tutor role's library differs from an
  authoring role's; the loadable set is correctly tied to the role by
  construction. One source (the role's `library` + each craft's
  `labels`), no duplication, no drift; adding a loadable craft is a
  one-line `library` entry the assembly picks up automatically.
- **Static tools, lazy knowledge** — the tool surface is fixed at session
  creation; only knowledge loads dynamically. Dynamic tool-availability
  tied to loaded crafts was considered and rejected: mutating the tool
  schema mid-session against the loop's session-constant assembly is real
  machinery for a benefit lazy knowledge already delivers.
- **Self-enforcing sequencing** — a tool whose use requires loaded
  knowledge (`store-eibit-session` requires `eibit-producer`) states it,
  and its executor returns a correctable "load `eibit-producer` first"
  error if called without it — sequencing by mechanism, prompt discipline
  as backup.
- **Loaded once per session for free** — the tool result lands in the
  message log, persisting across turns and recovery without re-loading.

**Concept always, mechanics on demand.** The index tells the agent a
craft *exists to load*, but not what the concept *is*. Where a capability
is core to the role (eibits are core to tutoring — see below), the base
instructions carry a short **primer** so the agent understands the
concept and recognizes when it applies; only the *authoring mechanics*
defer to the loaded craft. Without the primer, the index row refers to a
concept the agent doesn't hold; with it, the agent recognizes the moment
and then loads the producer craft to build.

## Eibits

An **eibit** is a tiny eivolet — a small, self-contained learning unit
(an exercise, a diagram, a worked explanation) that EiBot generates in a
discussion. Structurally it is a template body (`def: TemplateBody`) like
any authored content, rendered through the existing pipeline; what makes
it an eibit is its **lifecycle**, which is deliberately **user-driven**
and legible to the learner:

> **EiBot makes eibits in your chat; store the ones you want to keep,
> and they collect in your Eibits tab.** Generation is EiBot's; keeping
> is yours.

**Eibits are EiBot's teaching instrument, not just a learner feature.**
EiBot reaches for an eibit deliberately: an explanation substantial
enough to *deserve permanence* (not a throwaway clarification — a
clarifying, reusable account the learner will want to return to), or an
exercise that consolidates a concept. Default is still words; the eibit
is for content that earns keeping, a higher bar than merely being
helpful in the moment.

**Two acts, cleanly separated:**

- **Store to session** (any trigger) — EiBot authors the eibit and calls
  `store-eibit-session`; it renders inline via the id. This is
  **ephemeral** — it lives in the TTL'd session and goes when the
  conversation goes. **Every eibit starts here**, without exception.
- **Store to workbook** (explicit keep only) — the learner keeps it via
  the keep link on the rendered eibit or a spoken "keep that", which
  drives `store-eibit-workbook`. This is the *only* path to durability;
  the two acts never chain automatically — a request to *generate* is not
  a request to *keep*, even when the learner asked for the eibit.

The keep link **emits a chat message** (a learner turn, like authoring's
preview actions) rather than firing a silent side-effect — so the keep is
a visible turn in the transcript and the agent sees it in its own history
and can acknowledge it naturally ("kept — it's in your eibits now").

**Triggers EiBot recognizes** (all produce an eibit directly — generation
is cheap, the keep is the learner's gate, so no pre-asking):

- **Its own judgment** — content that deserves permanence, or a check
  that consolidates.
- **A direct request** — "make me an exercise on X", "write an
  explanation I can keep".
- **A nudge** — softer signals an eibit fits: "I want to practice this",
  "I wish I had this written down". A literal reading would answer with
  prose; the craft teaches EiBot to recognize the eibit-shaped need.
- **"Make an eibit of this"** — the learner pointing at material (a
  concept, passage, the page). Grounds in the touched content, anchors
  via the session's `typedMaterialId`.

This gives the feature a spine the "auto-persist" model lacked: the
durable store only ever holds what the learner deliberately kept,
"remove" has an obvious meaning, and the collection is curated by
construction. **All of the above — the pedagogy, the triggers, the
session-only invariant, the keep affordance, and the authoring mechanics —
lives in the loadable `eibit-producer` craft** (see Craft loading); the
base instructions carry only a short eibit *primer* so EiBot holds the
concept and recognizes when to load the producer.

**The mechanism** (mirrors eivolet authoring — stage in session, preview
from session, promote to durable):

1. `store-eibit-session(typedMaterialId, mdx)` — EiBot writes a full MDX
   fragment in the authoring vocabulary (prose, ContentKit components,
   `<FillBlankMultiExercise>`, `<ProgrammingExercise>`, `<Diagram>`, …).
   Being a template body, an eibit naturally carries explanation *around*
   an exercise — a micro-lesson, not a bare widget.
2. The executor stores the fragment in the **session** and returns an id;
   the agent's message carries an **`agEibit` item holding just that
   id**. Nothing content-bearing rides the stream (relay risk nil — a
   mangled id fails to resolve; `useObject` streams plain JSON).
3. The client resolves the id through the **`get-eibit-content` RSA** and
   renders it through the existing MDX pipeline (extraction, strip
   projection, component DTOs) — the book placeholder fetch-by-id
   pattern, and the same resolve-from-session preview used in authoring.
4. On **keep** (`store-eibit-workbook`), the session eibit is promoted to
   the workbook as an `Eibit` ADL object (`def: TemplateBody`, `type` and
   `typedMaterialId` in `metadata.extra`, a single `labels[0].title` as
   its displayable name) hanging from the **owner** derived from the
   `typedMaterialId`.

**The owner and `typedMaterialId`.** The context exposes the session's
anchor as **`typedMaterialId: { type: 'eivolet' | 'aggregate' |
'template' | …, materialId }`** — a reference whose `type` tells
consumers how to interpret the id. The agent copies it into
`store-eibit-session` verbatim. On store, the executor interprets it by the
**container vs asset** split: container types (`eivolet`, `aggregate`)
can own children, so the eibit hangs directly from them; asset types
(`template`) are content and cannot own, so it resolves to the asset's
containing aggregate. The eibit lives where the discussion's scope
lives. No new identity scheme — the data-access routines, validation
paths, and `get-content` resolution all work unchanged.

**Exercises inside an eibit** register exactly as rendering a book page
does: validation resolves `(template, exercise name)`, Judge0 for code,
comparison for fill-blank, attempts recorded in the `exerciseAttempts`
shape. A generated exercise **is** an authored exercise, delivered
through a different surface.

**Interaction model** (per the UX exploration): while in the
conversation, an exercise eibit renders **in-stream, live until
resolved** — a client component whose interactivity comes from its
session data, not its stream position. Attempts are **private by
default** (Validate is an RSA against the definition; no chat turn);
**Share is by choice** (submitting through the chat, EiBot's reply
appended like any turn). The learner decides whether to engage; nothing
nags.

**Reading surface** (this eivolet's saved eibits): a saved eibit is read
in its own surface, not merged into the authored page. The reader offers
a **view switch** — *Contents* (the eivolet) and *Eibits* (the saved
collection rendered as one continuous stream, each separated by its
`title`). The material is never altered because eibits were never in it;
they live in a parallel surface the learner switches to. Each stream
entry stays interactive (exercises validate, etc.). Ordering follows the
eibits' anchors in the material by default; drag-to-reorder is a future
addition. Action surfaces come from **capability presence** — components
render their action row from what the mdx-services context provides
(the `isPreview` design generalized), single-source, no variants.

**Remove**: removal is a **workbook management action, not a chat
concept** — the learner removes an eibit from the **Eibits TOC tab**,
where the whole collection is visible and curatable. EiBot generates and
(on request) stores; it has no removal capability. Attempts pose no
integrity constraint — informational records on a TTL'd session, nothing
attached; they age out or are removed with the eibit without
consequence.

**Growth path**: anything the ContentKit vocabulary learns to express
becomes an eibit with zero new plumbing — challenges (`cap: challenge`
already exists in the Descriptor), interactive visualizations,
explorable components. And **promotion** of a broadly useful eibit into
the published eivolet is an insert, not a transform, since eibits are
ADL-shaped by construction.

## Reader tabs and the workbook

The reader's TOC area carries **three tabs**, each a lens on the same
eivolet and each able to drive the main surface:

- **Contents** — the eivolet TOC (exists today).
- **Excerpts** — the learner's marks (exists today).
- **Eibits** — the learner's saved eibits for this eivolet (new). The
  list is read from the durable workbook (no TTL validation needed);
  each entry shows the eibit's `title` and a kind icon
  (exercise / diagram / explanation). Clicking an entry switches the
  main surface to the **Eibits** view and jumps to that eibit in the
  stream.

The main reading surface switches between **Contents** (the eivolet) and
**Eibits** (the saved collection as one continuous stream). Excerpts and
eibits are both workbook entries (`WorkBookBody = Excerpt | Eibit`), so
these tabs are the reader's window onto the **Dossier**'s workbook
branch for this `(user, eivolet)`.

Future, on the same "list + pointer" shape:

- **Chats** — the learner's started discussions for this eivolet.
  Sessions enumerate by a prefix scan of the session store
  (`role + user + eivolet…`), so expiry self-cleans; each entry
  reconstructs its subject from the `{kind, id}` behind the externalId
  and reopens via `openDiscussion` (the deterministic id lands in the
  existing conversation). Earns its keep for section/eivolet/exercise
  discussions, which otherwise have no persistent UI pointer (excerpt
  sessions reopen via their mark).
- **Reorder** — drag-to-arrange the Eibits stream (in the stream or the
  Eibits tab list).

## Craft structure

One shared tutor Role (`assistant`); the instructions craft keys its
flows on the `kind` in `<context:assistance>` — **five flows**:
`eivolet`, `aggregate`, `excerpt`, `diagram`, `exercise`.
Skeleton-based orientation (the map, routing, recommendations by title)
is global behavior, not flow-specific. The eibit rules govern
`store-eibit-session` use: **default is words** — generate an eibit only for
content that requires machinery (an interactive component, a diagram
that genuinely reads better drawn), never for prose, never to restate
existing material (route by title instead), at most one per turn, framed
by narrative. Exercise eibits additionally follow the grounded/gated/
small rules: derived only from content the session has touched,
generated when a check serves the learner, probes not assignments. And
the lifecycle rule: **never store an eibit unless the learner asks** —
generation is EiBot's, keeping is the learner's.

Session identity, recovery, and open/close plumbing are deliberately
absent from the instructions: the agent never sees a session id — that
architecture lives in this spec only.
