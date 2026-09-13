# Eivolet Authoring — Design Specification

## Purpose

This document describes the design of three composable components for authoring Eivolets: the **EiBot Chat**, the **Eivolet Definition Editor**, and the **Eivolet Editor**. Together they form the complete authoring surface — from collecting the parameters that define an Eivolet, through configuring its content types, to browsing and refining the generated result.

An **Eivolet Definition** is the blueprint for an Eivolet. It does not contain learning content — it contains the instructions for generating it. The definition is produced through the EiBot Chat and the Eivolet Definition Editor, then consumed by the platform to generate the actual Eivolet delivered to learners.

EiBot is the AI layer embedded in the Chat. Its role is to drive the conversation, collect the Eivolet's invariants, and guide the author through defining its structure.

---

## Overview

The three components are used in sequence. Every authoring session starts in the **EiBot Chat**, where EiBot collects the Eivolet's invariants and guides the author through defining its structure. When the structure conversation is complete, the **Eivolet Definition Editor** takes over to configure the content types for each structural level and produce a complete Eivolet Definition. Generation then runs and the result opens in the **Eivolet Editor**, where the author browses, refines, and publishes the finished Eivolet.

---

## EiBot Chat

The EiBot Chat is a conversation interface where EiBot acts as an agent with a clear goal: gather everything needed to produce a complete Eivolet Definition and hand off to the next component. EiBot owns the conversation — it decides what to ask, in what order, and when it has enough to move forward. The author responds freely; EiBot steers.

### Layout

The Chat has three persistent areas:

**Invariants card** — a collapsible card that appears inline at the top of the chat area once the first invariant is collected. In its collapsed state it shows a compact prose summary of what has been gathered so far. Expanded, it shows each invariant in full — accommodating long values like the objective without truncation. The invariants are never editable once set.

**Chat area** — the main conversation surface below the invariants card. EiBot asks; the author responds. The conversation is continuous and uninterrupted — there are no steps, no phases, no transitions.

**Structure sketch panel** — a right-side panel that remains visible throughout. As EiBot gathers structural information, it populates the panel with a live sketch of the emerging structure — units, lessons, and their relationships.

### What EiBot Needs to Collect

EiBot's goal is to collect four invariants and a structural description. It decides how and when to ask for each.

The **invariants** are:

- **Objective** — a concrete description of the subject matter, scope, and depth. This is the primary input EiBot uses to make all structural and content decisions.
- **Audience** — the background, level, and age range of the learners.
- **Cultures** — the languages this content will support. At least one must always be present.
- **Domain** — language learning, coding, mathematics, science, or other. Some domains require one additional clarification: coding requires knowing whether the target is a programming language or a framework; language learning requires knowing which language is being taught.

The **structure** is a description of the Eivolet's hierarchy — its units, topics, depth, and ordering. EiBot gathers this through open conversation, asking clarifying questions and raising objections when something conflicts with the invariants. The structure is expressed using four hierarchical levels: the **root** (the Eivolet itself), **sections** (top-level groupings for broad or deep Eivolets), **units** (groupings of related lessons), and **lessons** (individual learning steps where content lives). Not every level is required.

Once EiBot has enough signal on both the invariants and the structure, it presents a decision pair asking the author to confirm before moving on:

- **Looks good, let's continue** — locks the structure and opens the Eivolet Definition Editor.
- **Not yet, keep going** — continues the conversation.

Once all four invariants are collected, EiBot presents a path decision pair before the structure conversation begins:

- **Guide me through it** — EiBot continues the conversation to gather the structure, then hands off to the Eivolet Definition Editor.
- **I'll build it myself** — skips the structure conversation entirely and opens the Eivolet Editor directly.

> **Future improvement:** EiBot will attempt to infer domain, audience, and cultures directly from the objective — presenting pre-filled suggestions the author confirms or adjusts, rather than asking each question explicitly.

### Interaction Patterns

Three interaction patterns are used consistently throughout the chat. No other input styles are introduced.

**Single-select list** — a scrollable list of options with radio indicators. Used for all closed questions requiring one answer: domain, coding type, coding target, language target.

**Multi-select list** — a scrollable list of options with checkboxes and a Confirm button. Used for cultures and any future questions requiring multiple selections. English (en-US) is always pre-selected and locked — it cannot be deselected. Additional languages are optional. The selection is committed only when the author presses Confirm; nothing is recorded until that point.

**Decision pair** — two equal-sized cards presented side by side. Used exclusively for directional choices: path selection and the structure exit. Each card has a short label and a brief description. Both cards are always the same width and height.

No icons are used in any of the three patterns.

### EiBot in the Chat

EiBot drives the entire Chat. It owns the conversation, decides when each phase is complete, and produces the structural description that feeds into the Eivolet Definition Editor.

EiBot's behavior is shaped by the domain. For well-defined domains like language learning and coding, it is opinionated — it knows the typical structures, makes concrete suggestions, and can propose complete configurations. For the catch-all domain, it stays neutral and works only with what is universally applicable.

Every user message is evaluated before a generation call is made. If the message has no plausible connection to the current phase, EiBot redirects without calling the model — keeping the interaction focused and economical.

---

## Eivolet Definition Editor

The Eivolet Definition Editor is a visual configurator for selecting the content types that make up each level of the Eivolet's structure. It is the bridge between the structural description produced by the EiBot Chat and the generation step that produces the actual Eivolet.

The editor presents one column per structural level present in the hierarchy — levels of the same type are condensed into a single column. A shallow structure shows one column; a deep structure shows up to four.

Each column asks the same question for its level: what kind of content should go here? The author decides whether that level should have instructional material, practice activities, assessments, or a combination. The choices available depend on the domain selected in the Chat — a language learning course sees different options than a coding course.

Sensible defaults are pre-selected based on the depth of the structure. The goal is to guide the author toward one primary evaluation level — evaluation at every level simultaneously creates a poor learner experience:

| Depth | Evaluation defaults |
|---|---|
| 1 | root: challenge on · championship off |
| 2 | root: challenge on · championship off · lesson: both off |
| 3 | root: off · unit: challenge on · championship off · lesson: off |
| 4+ | root: off · section: off · unit: challenge on · championship off · lesson: off |

The author can override any default. They are a guide, not a constraint.

An **Invariants panel** is accessible via a button in the top corner and slides in from the right. It shows the four invariants collected in the Chat — read-only reference while configuring content types.

Once the author confirms, the platform generates the complete Eivolet. The result opens in the Eivolet Editor.

---

## The Eivolet Editor

The Editor is a visual component for browsing, refining, and progressively building an Eivolet. It presents the Eivolet as an interactive tree — the author navigates the structure on the left and inspects each node on the right. Every node in the tree is backed by a definition that was used to generate it. The definition is the lever for changing content — the author modifies it and regenerates the node.

The Editor can be reached in two ways: as the destination after the Eivolet Definition Editor completes generation, or directly via the self-build path from the Chat.

### The Interface

The Editor is a two-panel layout. The **left panel** is the tree — it shows the full structure of the Eivolet at a glance and is the primary navigation surface. The **right panel** is the detail view — it shows the metadata and a preview of whichever node is currently selected.

The tree is always visible and always reflects the current state of the Eivolet. Each entry shows the node's type, its label, and a status indicator — whether it is pending, being worked on, AI-generated, or complete. The author navigates by clicking. Hovering an entry reveals inline actions. Right-clicking opens a context menu.

The detail panel has two tabs: **Metadata** — the label fields for the selected node (title, overview, tooltip) per configured culture — and **Preview** — a live view of the generated content for that node, what the learner will see.

> **Future:** The detail panel will also expose the prompt used to generate the node, allowing the author to inspect it, modify it, and trigger regeneration directly from the panel.

### Ownership Model

The Editor opens in one of two modes: **read-only** or **editable**.

Which mode depends on how the Eivolet was created. An Eivolet produced via the AI-guided path opens in read-only mode — the tree is visible but nothing can be changed. An Eivolet started via the self-build path opens in editable mode.

A read-only Eivolet can be taken over at any time using the **Edit** action. This switches the Editor to editable mode permanently and irreversibly. The author takes full ownership of the Eivolet from that point.

One element is always locked regardless of mode: the root. It holds the invariants — objective, audience, cultures, domain — that the entire Eivolet was built from. Changing them would invalidate the entire structure.

### What Is Being Edited

The tree represents the **Eivolet** — its structure and generated content. Every node in the tree is backed by a definition that was used to generate it. The author does not edit the Eivolet content directly — they work through the backing definition to drive regeneration.

There are two categories of nodes: **container nodes**, which define the structure and hold other nodes, and **content nodes**, which are the actual learning objects that sit inside lessons.

**Container nodes** — four types, each representing a level of depth:

- **Root** — the Eivolet itself. The top of the tree. Every Eivolet has exactly one, and it is always locked.
- **Section** — a top-level grouping used in deep structures (four levels or more).
- **Unit** — a grouping that holds lessons.
- **Lesson** — an individual learning step. This is where content nodes live.

**Content nodes** — the learning objects inside lessons:

- **Template** — the core instructional content: explanations, examples, annotated code, cultural notes.
- **Fill-in-the-blank** — generates gap-fill exercises at runtime for each learner session.
- **Multiple Choice** — generates comprehension exercises at runtime.
- **Flashcards** — generates flashcard sets at runtime.
- **Playroom** — an open practice environment within a lesson.
- **Challenge** — a focused assessment. Can appear at any structural level, not just lessons.
- **Championship** — a broader, higher-stakes assessment. Can appear at any structural level.
- **Space** — a course-level free practice environment. One per Eivolet, at the root only.

The tree can be at most 8 levels deep. The Section level is optional and only relevant at depth 4 and above.

### Content Model

Content nodes are organized into four functional groups. Each lesson can hold at most one node of each type.

**Learning** — static content authored once: the Template node.

**Gym** — practice activities generated at runtime. The author configures the generator once; each learner session produces fresh exercises. Includes Fill-in-the-blank, Multiple Choice, Flashcards, and Playroom. The available types depend on the domain — a language learning course has different options than a coding course.

**Evaluation** — assessment activities: Challenge and Championship. Unlike Learning and Gym nodes, these can be placed at any container level, not just lessons.

**Space** — the course-level free practice environment. One per Eivolet, at the root only.

### Operations

All operations in editable mode target individual nodes. There is no bulk editing.

**Regenerate** — available on any node in editable mode. EiBot works on the node's backing definition and Editorial Services regenerates that node. The rest of the Eivolet is unaffected.

**Add** — triggered from the inline `+` on any node. A short wizard asks what type to add and what to name it. EiBot builds the definition for the new node and Editorial Services generates it immediately. The available types are constrained by the selected node — only valid children are offered.

**Delete** — available inline or from the detail panel. Requires confirmation when the node has children.

**Duplicate** — available from the context menu. The copy and all its children receive new identifiers and are regenerated.

### The Root Node

The root node is the Eivolet itself — not just the top of the tree but the configuration that governs generation across the entire Eivolet. It is always locked. When selected, it shows a dedicated panel with five areas:

**Identity** — the name and namespace that identify this Eivolet within the platform.

**Objective** — the subject matter of this Eivolet: its topic, scope, and depth. Combined with the domain, this drives content generation throughout the tree.

**Cultures** — the languages this Eivolet supports. Configured once here and propagated automatically to every node. At least one culture must always be present.

**Models** — the AI models used for content and image generation.

**Prompts and Contexts** — the prompt and context configurations that guide generation across the entire Eivolet.

### Internationalization

Every node has a set of label fields — one per configured culture — covering its title, a short overview, and a tooltip. Cultures are defined once at the root and propagated automatically to every node in the tree. Adding a new culture to the root immediately surfaces its label fields everywhere. EiBot can generate labels for all configured cultures in a single pass.

### Visual Design

The Editor uses the Eivo design system. The palette centers on a warm orangey-salmon accent with neutral grays. Typography is DM Sans with DM Serif Display for display text. The theme is light, consistent with the learner-facing product.

Each node type has a distinct icon and color so the tree is readable at a glance. Container nodes: root (orange), section (folder), unit (open folder), lesson (book). Content nodes: template (document), fill-in-the-blank (zap), multiple-choice (checkbox), flashcard (card stack), playroom (screen), challenge (trophy), championship (medal), space (globe). A small status dot on each node indicates its state: gray (pending), orange (active), blue (AI-generated), green (complete).

### Generation Readiness

The Editor tracks the completion status of every node. When all nodes have been generated — each one with valid content conforming to its type — the Eivolet is ready to publish. The author decides when to publish. There is no forced completion state; nodes can always be regenerated after publishing.

### EiBot in the Editor

In the Editor, EiBot's role is to build and refine the backing definition for each node in order to drive generation. Each node has its own conversation thread where the author can describe what they want — EiBot produces the corresponding definition and Editorial Services generates the node content from it.

EiBot never starts a blank conversation — every thread opens with a message tailored to the node type.

EiBot distinguishes between two kinds of turns. During **exploration** the author is thinking, asking questions, or restructuring — EiBot responds conversationally and nothing is generated. During **generation** the author has provided enough signal — EiBot produces the node's definition and triggers generation. Each generation turn is scoped to one node — the node is the natural unit of work.

EiBot's context deepens as the author works through the tree. It always knows the domain and the Eivolet's objective. When a node is selected, the node's type and its position in the tree become part of the active context. Its opening message for each node type reflects this:

- **Section** — asks about the section's theme and what it groups together
- **Unit** — asks about the unit's focus and learning outcomes
- **Lesson** — asks about the topic and suggests appropriate content types
- **Template** — asks about the content structure and what the lesson should explain
- **Fill-in-the-blank / Multiple Choice / Flashcard** — asks about difficulty, kind, and type
- **Challenge / Championship** — asks about the evaluation criteria and expected output
- **Space** — asks about the environment type and available tools
- **Playroom** — asks about the practice context and available activities

---

## Key Architectural Decisions

**EiBot Chat**

| Decision | Choice | Rationale |
|---|---|---|
| Component model | Single chat window, agent-driven | Invariants and structure are one continuous conversation — no steps, no transitions |
| Invariants card | Inline collapsible card at top of chat, appears after first invariant is collected | Part of the same surface as the conversation; handles multiline values naturally |
| Invariants | Objective, audience, cultures, domain | Collected by EiBot in whatever order makes sense; anchor all downstream decisions |
| Path selection | Decision pair immediately after all invariants are collected | Presented once — before the structure conversation begins; determines what follows |
| Structure conversation | Free conversation, EiBot decides when complete | Author describes freely; EiBot stays anchored to the invariants |
| Structure sketch | Live right panel, visible throughout | Gives the author a running picture of what is being decided |
| Interaction patterns | Single-select list · Multi-select list · Decision pair | Three patterns only, used consistently, no icons |
| Course format | Not asked — inferred from conversation | Less friction; emerges naturally from the structure discussion |

**Eivolet Definition Editor**

| Decision | Choice | Rationale |
|---|---|---|
| Entry point | After structure conversation or directly from self-build path | Two valid entry points; invariants are always present |
| Layout | One column per structural level | Derived from the structure conversation; domain-filtered |
| Defaults | Depth-aware evaluation defaults | Guide toward one primary evaluation level |
| Invariants access | Slide-in panel via button | Available for reference without occupying permanent space |
| Revision | Not available — return to Chat to start over | The definition is produced as a whole |

**Eivolet Editor**

| Decision | Choice | Rationale |
|---|---|---|
| Primary surface | The Eivolet, not the definition | Authors work with generated content; the definition is a backing artifact |
| Editing model | Modify backing definition → regenerate | Direct content editing not in scope; definition is the lever for change |
| Detail panel | Metadata + preview | Labels per culture and generated content preview; prompt access is a future addition |
| Ownership model | AI-guided (read-only) or self-build (editable) | Determined by the path chosen in the Chat |
| Ownership transfer | Read-only → editable only | One-way and irreversible |
| Root node | Always locked | Holds the invariants everything else was built from — changing them invalidates the tree |
| Root node panel | Dedicated Eivolet configuration panel | The root is the Eivolet itself, not just another node |
| Cultures | Configured once at root, propagated to every node | Configure once; surfaces everywhere automatically |
| Data structure | Directed tree | Foundation for future conditional paths and reusable nodes |
| Generation readiness | Node completion tracking; publish when all nodes generated | Author decides when to publish — no forced completion state |

**EiBot**

| Decision | Choice | Rationale |
|---|---|---|
| Role in Chat | Drives the conversation across both phases | EiBot owns the flow; author has freedom of expression, not direction |
| Role in Editor | Assists at node level | Each node has its own conversation thread |
| Domain behavior | Opinionated for known domains, neutral for catch-all | Known domains get full structure proposals; catch-all stays universal |
| Guardrails | Conversational · Intent classification · Schema validation | Layered — keeps EiBot focused without being obstructive |
| Generation vs exploration | Two distinct turn types in the Editor | Exploration is conversational and cheap; generation is scoped to one node |
