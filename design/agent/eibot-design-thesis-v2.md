# Eivolet Authoring — Design Specification

## Purpose

This document describes the design of the **Eivolet Authoring Surface** — a single, unified interface for creating Eivolets through conversation with EiBot.

**EiBot is the whole solution.** It is not a component inside a product — it spans UI/UX, SDK, and backend. The authoring surface, the agent runtime, and the generation pipeline are all expressions of the same system. The authoring surface described here is one face of EiBot; other faces — a learner assistant, a challenge evaluator, a domain tutor — are equally valid instantiations of the same agent, each driven by a different set of crafts and goals. The architecture must support this generality.

EiBot drives generation directly from the conversation. There is no intermediate Eivolet Definition artifact — the conversation history and the live Eivolet in Redis are the only sources of truth during authoring. As the author and EiBot align on each node, generation fires immediately. The author sees results progressively without waiting for the full Eivolet to be defined.

The live Eivolet is held in Redis during the authoring session. When the author is satisfied, EiBot publishes it — calling the Library service, which takes the draft, persists it to disk, and indexes it for platform-wide discovery. The Library does not own the draft; it only takes it at publish time.

---

## Overview

The authoring surface is a single screen. There are no steps, no transitions, no handoffs to other components. EiBot drives the entire session — collecting invariants, defining structure, configuring content types, and driving generation — all through conversation. The author responds freely; EiBot steers.

The screen has two persistent panels and one optional panel:

- **Chat** — the conversation surface. Always visible. EiBot and the author work here for the entire session.
- **Structure panel** — the shared blueprint of the session. Always visible. EiBot populates it as decisions are made; the author can reference it in conversation.
- **Preview panel** — dismissible, floats over the structure panel and part of the chat when opened. Can be pinned as a fixed third column, maximized, or closed. Points to a URL backed by the live Eivolet in Redis.

---

## The Chat

The Chat is the primary surface. EiBot owns it entirely — it decides what to ask, in what order, and when it has enough to move forward. The conversation is continuous and uninterrupted from the first message to a fully generated Eivolet.

### Session Opening

The session does not wait for the author to speak first. When the authoring surface loads, the Chat service injects a synthetic first message into the conversation carrying a single piece of author context — whether the author is new or returning — and EiBot responds immediately. That response is the first thing the author sees.

For a new author, EiBot introduces what an Eivolet is, how it is organized, and what the authoring session will involve — then offers a choice to be walked through it or dive straight in. For a returning author, EiBot opens the session ready to work.

The synthetic message is never shown to the author. For now the only context it carries is new vs. returning — more can be added in the future without any change to the architecture. The craft defines the policy; EiBot executes it conversationally. No new UI surface is needed — the existing decision pair ("Walk me through it" / "Let's dive in") is sufficient when EiBot chooses to offer it.

### Invariants Card

A collapsible card appears inline at the top of the chat area once the first invariant is collected. In its collapsed state it shows a compact prose summary of what has been gathered. Expanded, it shows each invariant in full — accommodating long values like the objective without truncation. The invariants are never editable once set.

As each invariant is confirmed, EiBot writes it directly to the Eivolet body in Redis. The invariants are part of the Eivolet — not a separate artifact. When the Eivolet is published, the invariants travel with it.

### What EiBot Collects

**The invariants:**

- **Objective** — a concrete description of the subject matter, scope, and depth. This is the primary input EiBot uses to make all structural and content decisions.
- **Audience** — the background, level, and age range of the learners.
- **Cultures** — the languages this content will support. At least one must always be present. English (en-US) is always included and cannot be removed.
- **Domain** — language learning, coding, mathematics, science, or other. Some domains require one additional clarification: coding requires knowing whether the target is a programming language or a platform, and which specific one; language learning requires knowing which language is being taught.

**The structure** is a description of the Eivolet's hierarchy — its units, topics, depth, and ordering. EiBot gathers this through open conversation, asking clarifying questions and raising objections when something conflicts with the invariants. The structure is expressed using four hierarchical levels: the **root** (the Eivolet itself), **sections** (top-level groupings for broad or deep Eivolets), **units** (groupings of related lessons), and **lessons** (individual learning steps where content lives). Not every level is required.

**Content type configuration** — EiBot decides, through conversation, what kind of content belongs at each structural level: instructional material, practice activities, assessments, or a combination. There is no separate UI for this. EiBot handles it as part of the same conversation, applying domain-appropriate defaults and adjusting based on what the author describes. The goal is to guide toward one primary evaluation level — evaluation at every level simultaneously creates a poor learner experience.

> **Future improvement:** EiBot will attempt to infer domain, audience, and cultures directly from the objective — presenting pre-filled suggestions the author confirms or adjusts, rather than asking each question explicitly.

### Interaction Patterns

Three interaction patterns are used consistently throughout the chat. No other input styles are introduced.

**Single-select list** — a scrollable list of options with radio indicators. Used for all closed questions requiring one answer: domain, coding type, coding target, language target.

**Multi-select list** — a scrollable list of options with checkboxes and a Confirm button. Used for cultures and any future questions requiring multiple selections. English (en-US) is always pre-selected and locked. The selection is committed only when the author presses Confirm; nothing is recorded until that point.

**Decision pair** — two equal-sized cards presented side by side. Used exclusively for directional choices. Each card has a short label and a brief description. Both cards are always the same width and height. The one decision pair in the current flow is the generation confirmation — presented once, after EiBot has enough signal on invariants, structure, and content types:

- **Looks good, let's build it** — EiBot begins generating nodes progressively.
- **Not yet, keep going** — continues the conversation.

No icons are used in any of the three patterns.

### Coding Domain Flow

When the domain is programming, EiBot collects two things in sequence:

1. **Language or platform?** — two options only. Runtime is not a standalone choice; it exists only as part of a platform pair.
2. **Which language?** — always asked first, regardless of whether the target is a language or a platform.
3. **Which platform?** — only if platform was chosen. Options are self-descriptive pairs (e.g. React · Node.js, Spring Boot · Tomcat) that encode both framework and runtime in a single selection. EiBot does not ask for the runtime separately.

### EiBot's Behavior

EiBot drives the entire session. It owns the conversation, decides when each phase is complete, and generates content progressively behind the scenes without surfacing that complexity to the author.

Once the invariants are established, EiBot's mode is intent-driven. When the author arrives with clear direction — a defined structure, specific content types, a particular sequencing — EiBot follows and executes. When the author is uncertain or open, EiBot takes the lead: it proposes structure, suggests content types, and guides the conversation toward a complete Eivolet. It reads intent from each message and adapts without requiring the author to declare which mode they want.

EiBot's behavior is also shaped by the domain. For well-defined domains like language learning and coding, it is opinionated — it knows the typical structures, makes concrete suggestions, and can propose complete configurations. For the catch-all domain, it stays neutral and works only with what is universally applicable.

The structure panel is EiBot's working reference throughout the session. As decisions are made in the conversation, EiBot updates the panel. Both parties use it to stay oriented — the author can point to what's there, and EiBot uses it to track what remains.

Every user message is evaluated before a generation call is made. If the message has no plausible connection to the current phase, EiBot redirects without calling the model — keeping the interaction focused and economical.

During node-by-node generation, EiBot distinguishes between two kinds of turns:

**Exploration** — the author is thinking, asking questions, or reconsidering. EiBot responds conversationally and nothing is generated.

**Generation** — the author has provided enough signal for a node. EiBot triggers generation directly. Each generation turn is scoped to one node — the node is the natural unit of work.

EiBot's context deepens as the session progresses. It always carries the invariants and the full structural context. When working on a specific node, the node's type and its position in the tree are part of the active context.

To revisit a generated node, the author describes what they want changed in the chat. EiBot regenerates it. The rest of the Eivolet is unaffected.

---

## Structure Panel

The structure panel is the shared blueprint of the authoring session. It is always visible alongside the chat. EiBot populates it as decisions are made and uses it as the working reference for the session — both EiBot and the author orient to it as the Eivolet takes shape.

The panel cannot be edited directly. All changes go through the chat. The author can reference nodes by name or position in conversation — "rename that unit", "add a lesson after the second one", "remove the championship from the root" — and EiBot acts on the panel accordingly.

The structure panel inherits the visual language of the Eivolet Editor directly — the same design tokens, node icons, and colors. No new visual design is introduced. Node types and their Lucide icons: root (`layers`), section (`folder`), unit (`folder-open`), lesson (`book-open`), template (`file-text`), exercise (`zap`), flashcards (`square-stack`), playroom (`monitor`), challenge (`trophy`), championship (`medal`), space (`globe`).

Publishing is initiated through the chat — the author asks EiBot to publish. A floating action button (circular, semi-transparent) appears at the bottom-right of the structure panel once the Eivolet is ready; pressing it sends the publish request to EiBot as a chat message.

---

## Preview Panel

The preview panel shows the full current state of the generated Eivolet — everything EiBot has produced so far, as the learner would experience it. It is not a per-node inspector.

The preview points to a URL backed by the live Eivolet in Redis, served by an RSC endpoint. The browser fetches directly from this URL — no polling, no intermediate layer. As EiBot generates each node, the preview reflects the updated state.

The panel has four states:

**Floating** — the default when opened. The panel slides in from the right, overlaying the structure panel and part of the chat. The two main panels do not reflow — their layout is unaffected.

**Pinned** — the panel becomes a fixed third column. The chat and structure panel reflow to share the remaining space alongside it.

**Maximized** — the panel expands to occupy the full screen, hiding the other panels temporarily.

**Closed** — the panel is dismissed. The layout returns to the two fixed panels.

From the floating state the author can pin it, maximize it, or close it. From the pinned state they can maximize it or unpin it back to floating. From maximized they can return to the previous state.

The preview is read-only. If the author asks EiBot to regenerate a node, the preview reflects the updated content once generation is complete.

---

## Generation Model

Generation is progressive and node-by-node. As EiBot finalizes each node through conversation, generation fires immediately. The author does not need to wait for the full Eivolet to be defined before seeing results.

There is no intermediate Eivolet Definition artifact. EiBot drives generation directly from the conversation — the conversation history and the live Eivolet state in Redis are the sources of truth. A definition artifact was considered and eliminated: it becomes stale the moment the Eivolet is edited directly, creating a sync problem with no compensating benefit.

The author decides when to publish. There is no forced completion state — nodes can always be revisited through the chat after publishing.

---

## Library Service

The Library service owns published Eivolets. It does not own the draft.

The draft lives in Redis for the duration of the authoring session. It is owned by the Chat/EiBot session. When the author publishes, EiBot calls the Library API with the current Eivolet from Redis. The Library takes the draft, persists it to disk, indexes it, and makes it available for platform-wide discovery. The draft lifecycle is entirely outside the Library's concern.

**Library responsibilities:**
- Accept publish requests from EiBot
- Persist the Eivolet to disk
- Index it for search and discovery
- Expose APIs for searching, browsing, and fetching Eivolets by ID or metadata

Any platform consumer — Lingv, Coursework, or any future experience — queries the Library API to discover and load Eivolets. The Library is the single source of truth for published content.

---

## Key Architectural Decisions

**Authoring Surface**

| Decision | Choice | Rationale |
|---|---|---|
| Component model | Single unified surface | Eliminates handoffs and transitions; one continuous authoring session |
| Layout | Two persistent panels + one optional | Chat and structure always present; preview on demand to avoid clutter |
| Structure panel | Shared blueprint, not an editing surface | EiBot writes to it; author references it in chat; all changes via conversation |
| Publish trigger | FAB in structure panel sends chat message to EiBot | All actions route through EiBot; FAB is a convenience shortcut, not a direct action |
| Preview panel | Floating overlay, pinnable, maximizable; URL-backed by Redis | No reflow on open; pin makes it a fixed third column; browser fetches live Eivolet directly |

**EiBot**

| Decision | Choice | Rationale |
|---|---|---|
| Agent model | General conversational agent, role-instantiated | EiBot is not authoring-specific; same infrastructure serves any role via different crafts and goals |
| Session opening | Synthetic first message injected by Chat service; EiBot responds; author sees only the response | No new UI surface; craft owns orientation policy; everything inside the EiBot loop |
| New author orientation | Offered by EiBot when user context is new; craft owns the policy | New authors get platform introduction and opt-in to walkthrough; returning authors go straight to work; more context can be added to the synthetic message in future |
| Post-invariants mode | Adaptive — follows when intent is clear, guides when it isn't | No mode declaration required; EiBot reads intent from each message |
| Content type configuration | Conversational, no separate UI | Fewer surfaces; EiBot applies domain-appropriate defaults and adjusts through dialogue |
| Generation trigger | Direct, per node; no intermediate definition artifact | Fires as each node is ready; conversation + Redis state are the sources of truth |
| Node revision | Via chat | Author describes the change; EiBot regenerates that node; rest of Eivolet unaffected |
| Domain behavior | Opinionated for known domains, neutral for catch-all | Known domains get concrete proposals; catch-all stays universal |
| Coding domain flow | Language or platform (two options); language always asked; platform uses self-descriptive pairs | Runtime is not standalone; pair labels encode framework + runtime in one selection |
| Exploration vs generation | Two distinct turn types | Exploration is conversational and cheap; generation scoped to one node |

**Data Model**

| Decision | Choice | Rationale |
|---|---|---|
| Eivolet Definition artifact | Eliminated | Becomes stale on direct Eivolet edit; no reuse benefit in the agent flow; conversation + Redis are sufficient |
| Draft Eivolet | Redis, owned by Chat/EiBot session | Live during authoring; accessible via preview URL; not the Library's concern |
| Published Eivolet | Library service — persisted to disk, indexed | Single source of truth for published content; queried by all platform experiences |
| Preview URL | RSC endpoint reading from Redis by session ID | Browser fetches directly; live Eivolet state always reflected |
| Publish | EiBot calls Library API; Library takes draft and stores it | Draft lifecycle stays in session; Library only acts at publish time |
| Root invariants | Written to Eivolet body in Redis as collected; locked once set | Part of the Eivolet, not a side artifact; anchor the entire generated tree; travel with the Eivolet on publish |
| Cultures | Configured once, propagated to every node | Configure once; surfaces everywhere automatically |
