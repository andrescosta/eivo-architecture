# EiBot — Technical Architecture

## Overview

EiBot is the Core SDK component of the Eivolet Authoring feature. The feature spans the full stack — Chat (Service), EiBot (Core SDK), Foundry and Crafter (Libraries), Library (Service) — with EiBot at its center as a general conversational agent that can be instantiated with different roles depending on context. This document describes the technical architecture of EiBot as deployed in the **Eivolet Authoring Surface** — a single, unified screen where EiBot drives the entire authoring session through conversation and triggers node generation progressively as each node is ready.

The authoring session is one continuous conversation. EiBot collects invariants, works through structure and content types, and generates nodes — all in the same chat, without transitions or handoffs. There is no intermediate Eivolet Definition artifact. The Eivolet is an interactive document format based on ADL — the ADL specification defines its serialized structure, and an Eivolet instance is the content itself: aggregates, spaces, interactive content, and runtime-generated content definitions. EiBot produces the instance progressively as the authoring conversation unfolds and publishes it when the author is satisfied. The conversation history and the live Eivolet instance in Redis are the sources of truth.

EiBot uses **HTTP with streaming** as the transport and **Redis** for session state. Each turn is a POST request — the client sends the user's message, the server runs EiBot, streams the response back, and the connection closes. Vercel's streaming API handles response setup and delivery.

---

## Requirements

**Agent-driven flow** — the flow is not a static form or a predefined sequence of screens. EiBot decides what to ask next, evaluates responses, and determines when it has enough signal to move forward or trigger generation. The UI receives and renders dynamic requests from the agent at runtime.

**Turn-based interaction** — the author drives the direction of the session. EiBot pauses for user input between conversational turns. Within a turn, EiBot executes tools autonomously — it can call, loop, and resolve tool results without waiting for the user. The one exception is publishing: it requires explicit author intent and cannot be triggered by EiBot on its own.

**Session continuity** — the author may take time between responses. The session must be resumable at any point without loss of context. The conversation history is the only state that needs to persist.

**Streaming** — responses stream to the client as they are produced. The author sees EiBot's reply in real time without waiting for the full response.

**Extensibility** — EiBot's role vocabulary will grow. New questions, new collection sequences, and new role configurations must be addable without changes to the transport layer or the UI. The UI renders primitives; EiBot decides everything else.

**Role generality** — EiBot is not specific to authoring. The same architecture must support other roles — learner assistant, challenge evaluator, domain tutor — without structural changes. Role behavior is determined entirely by the crafts and goals configured for that instantiation.

---

# Platform

## Platform Topology

**Eivolet Authoring — Two Modes**

Eivolets can be authored in two ways:

- **Scripted** — via Anvil, the CLI tool. Anvil is the conduit for scripted generation — the same way Chat is the conduit for interactive. It receives the Eivolet Definition, calls Forge, and outputs the result. Used for batch production, CI pipelines, and content automation.
- **Interactive** — via EiBot, the conversational agent. The author and EiBot build an Eivolet instance progressively through dialogue. The instance is the direct output of the conversation — no definition required.

**`@eivo/content` — ADL Content Package**

The runtime package for ADL content objects. Libraries are pure computation — no centralized persistence, no service calls. They take inputs, produce outputs, and return control to the caller.

- **Foundry** — the higher-level generation library. Assembles instructions, resolves crafts and tools, and orchestrates LLM execution via Crafter. Pure generation — no service calls. Returns the result and token usage to the caller. Memory is passed in by the caller for Foundry's own inner working: history recovery, tool injection, and onFinish handler creation. What happens with the output is entirely the caller's responsibility.
- **Storage** — ADL object storage. Persists and retrieves Eivolet instances and other ADL objects. Two backends: Redis for the live instance during authoring (ephemeral, no search), and the filesystem for published Eivolets (permanent, indexed and searchable via Library).

**`@eivo/crafter` — LLM Execution Package**

- **Crafter** — a wrapper around the Vercel AI SDK. Owns LLM execution. Pure execution — no knowledge of sessions, roles, business logic, or storage. Deserves its own package because the underlying implementation is swappable — today Vercel AI SDK, tomorrow something else.

**Backend Services**

Services own centralized persistence and expose it to the rest of the platform. This is what distinguishes them from libraries — they are not just computation, they own data.

- **Forge** — reusable content generation. Serves runtime LLM materials (exercises, flashcards, challenges) with a persistent content cache on disk. Also executes Eivolet Definitions for scripted generation (Definition → Modeler → Prompt). The cache is a cost-saving layer: deletable at any time without data loss, but valuable over time by eliminating redundant generation costs.
- **Library** — storage and retrieval for published Eivolets. Receives publish requests from EiBot, persists to disk, indexes for discovery, and exposes APIs for all platform consumers. The publish API is exposed as an MCP tool, allowing EiBot to call it directly within the conversation.

---

## Layer Responsibilities

The EiBot execution chain has four distinct layers. Each owns a clear concern and nothing else.

### Chat

A pure conduit. Receives the request, retrieves the EiBot session, calls `startSession`, and pipes the stream back. It has no knowledge of the EiBot type, role, crafts, or any session internals. A single generic endpoint serves all roles and all experiences.

`startSession` is an abstract method implemented by each typed EiBot — `EditorEiBot`, `AssistantEiBot`, etc. Each implementation brings its own Foundry configuration, crafts, and role-specific context. The polymorphism lives in the EiBot hierarchy, not in the Chat layer.

The URL structure is `/chat/{namespace}/eibot/{session_id}` where `{namespace}` is the experience's own namespace. The typed EiBot is created by the Experience's React handler, which knows which concrete type to instantiate. Retrieval by session ID is generic — Chat does not need to know the type.

### EiBot

The core SDK layer. Responsibilities:

- Load the current session (role ID, domain)
- Create and scope the short-term `Memory` instance for the session
- Pass role, domain, message, and `Memory` to Foundry

EiBot has no knowledge of crafts, tools, messages, `onFinish` handlers, or LLM primitives. It does not load conversation history — Foundry recovers it from Memory.

### Foundry

Owns assembly and orchestration. Responsibilities:

- Load the role definition
- Recover conversation history from Memory
- Resolve and assemble crafts in declaration order, filtering optional crafts by domain
- Resolve tools from the registry, attach handlers, inject `Memory` where needed
- Create `onFinish` handlers internally: history persistence back to Memory, token usage emission to EiQueue
- Pass assembled instructions, resolved tools, messages, and `onFinish` handlers to Crafter

Foundry never streams. It assembles and delegates.

### Crafter

Owns LLM execution. Responsibilities:

- Call `streamText` via ToolLoopAgent with instructions, tools, and messages
- Execute the tool loop via `stopWhen`
- Invoke all `onFinish` handlers on stream completion
- Pipe the response stream back to the caller

Crafter has no knowledge of sessions, roles, crafts, or business logic. The tool loop and step management are internal to the SDK — Crafter does not manage the loop manually.

---

## Execution Flow

Chat receives the POST request and calls EiBot. EiBot loads the session (role, domain), creates a Memory instance, and passes everything to Foundry. Foundry recovers conversation history from Memory, loads the role definition, resolves and assembles crafts in declaration order (filtering optional crafts by domain), resolves tools from the registry, creates `onFinish` handlers for history persistence and token accounting, then calls Crafter. Crafter calls `streamText` via ToolLoopAgent, executes the tool loop as needed, and fires `onFinish` handlers on completion. Chat passes the result to Vercel's streaming API. The UI renders `AgentMessage` spec objects progressively as they arrive.

---

# Agent

## Introduction

EiBot is the Core SDK component of the Eivolet Authoring feature — a general conversational agent that can be instantiated with different roles depending on context. It runs server-side and is the single source of truth for the conversation — it knows the current context, the current goal, and what to do next. The UI never makes flow decisions.

EiBot is stateless between turns. It is reconstituted from the persisted conversation history on every turn. The conversation history is the state. EiBot's own responsibility is minimal by design — it creates and scopes the Memory instance for the session and passes everything else to Foundry for assembly and execution.

A role is a named instantiation of EiBot with a specific goal, a specific set of crafts, and a specific set of tools. The agent infrastructure is the same across all roles — only the crafts, tools, and goal differ. When this document refers to EiBot's behavior, it refers to an instance running with the authoring role.

---

## System Prompt

### Role

A role declares the goal and the ordered lists of crafts and tools available in the session. Each entry is either required or optional. Optional crafts are included only when the session domain matches. Required crafts are always included. The declaration order is preserved — instructions are assembled in the order the crafts appear in the role definition.

### ObjectRef

`ObjectRef` is the standard reference type for named platform objects within a role. It carries a `name` and an `optional` marker. Used for both craft and tool references.

```yaml
crafts:
  - name: eivo-editor
  - name: eivo-exercise
  - name: eivo-eibolet
    optional: true
  - name: eivo-space-prog
    optional: true
```

### Crafts

A craft is a named, domain-specific capability available within a session. Crafts are the units of EiBot's generation capability — they represent the concrete generators EiBot can invoke when producing node content.

Crafts are assembled by Foundry from the role's craft list before each turn. Required crafts are always included. Optional crafts are included only when the session domain matches. The assembled craft list is exhaustive and authoritative for the session — **EiBot must never reference a craft not present in the assembled list.** Hallucinating craft names outside this list is a hard constraint violation.

There is no base prompt — instructions are assembled entirely from the role goal and the resolved craft chain, in declaration order.

---

## Tools

Tools are agent-callable functions registered in the platform tool registry. Each tool is identified by a registry key. A role declares which tools are available via its `tools` field, using the same required/optional semantics as crafts.

Foundry resolves tool references at assembly time: it looks up each key in the tool registry, retrieves the handler, and attaches it to produce a resolved tool ready for the LLM call. EiBot never accesses the tool registry directly. Tools that require session-scoped storage receive a `Memory` instance injected by Foundry at assembly time.

### Eivolet Tool

A local tool — not MCP — that stores and retrieves the live Eivolet instance from Redis during the authoring session. EiBot uses it throughout the conversation to build the instance progressively: writing invariants as they are collected, adding and updating nodes as they are generated, and reading the current state when needed. The Eivolet instance in Redis is the working artifact for the entire session.

### `model: values` — Agent Option Lists

`model: values` is an ADL artifact type for defining option lists. Each artifact defines a named set of entries — each entry carries a `labels` array and a `value` (either a string or a record). Values artifacts are the platform's answer to hardcoded option lists in crafts or client code.

```yaml
model: values
metadata:
  name: get-cultures-values
labels:
  - culture: en-US
    overview: Returns the list of available cultures.
def:
  items:
    - labels:
        - culture: en-US
          title: English (US)
      value: en-US
    - labels:
        - culture: en-US
          title: Spanish
      value: es
```

Values artifacts placed in the `agentvalues/` directory are **auto-exposed as tools** at startup — no `agenttool` YAML needed. The framework scans the directory and creates one tool per artifact:

- `metadata.name` → tool name
- `labels[en-US].overview` → tool description passed to the LLM
- `def.items` → what the tool returns when called

The execute handler is always the same — return `def.items`. No custom handler, no registry wiring. The crafts instruct EiBot which values tool to call for which invariant. EiBot never hardcodes option lists — it calls the appropriate tool and uses what it gets back.

Each item's `labels[en-US].title` is the display label shown to the user in the UI. The `value` is what gets written into the invariant.

Record values are used for compound options — a single selection that encodes multiple properties at once:

```yaml
model: values
metadata:
  name: get-programming-platforms-values
labels:
  - culture: en-US
    overview: Returns the available programming platform targets.
def:
  items:
    - labels:
        - culture: en-US
          title: React · Node.js
      value:
        progLang: typescript
        platform: react
        runtime: node
    - labels:
        - culture: en-US
          title: Spring Boot · Tomcat
      value:
        progLang: java
        platform: spring
        runtime: tomcat
```

### `model: agenttool` — Custom Agent Tools

`model: agenttool` defines a custom agent tool. The artifact declares the tool's identity, type, schemas, and optionally scopes it to a domain.

```yaml
model: agenttool
metadata:
  name: get-eivolet-instance
labels:
  - culture: en-US
    overview: Retrieves the current state of the Eivolet instance from the session.
def:
  type:
    protocol: function
```

**`def` fields:**

- `domain` *(optional)* — scopes the tool to a specific domain; same resolution as optional crafts
- `inputSchema` *(optional)* — Zod-compatible schema the LLM must conform to when calling the tool
- `outputSchema` *(optional)* — schema of the value returned by the tool
- `type` — discriminated by `protocol`:
  - `protocol: function` — a local function tool; handler registered in the tool registry and resolved by Foundry at assembly time
  - `protocol: mcpHttp` *(planned — no current implementations)* — an MCP HTTP tool for genuine LLM-to-LLM tool exposure; carries `server`, `tool` (tool name on the server), and `variant` (`sse` or `stream`). The `server` field is a structured object supporting three shapes:
    - Full URL from environment variable: `server: { env: API_MCP }`
    - Environment variable base + path: `server: { env: API, url: /mcp }`
    - Literal URL: `server: { url: 'http://localhost:8080/mcp' }`

**`labels[en-US].overview`** is the tool description — used both for the tool directory and as the description passed to the LLM at runtime. Consistent with all other ADL objects.

Tools placed in the `agenttools/` directory are auto-exposed at startup — `metadata.name` as tool name, `labels[en-US].overview` for discovery. For `function` protocol tools, Foundry resolves the execute handler from the tool registry by name and injects `Memory` where needed. For `mcpHttp` tools, Foundry constructs the MCP client from the server and tool fields. EiBot never accesses the tool registry directly.

### Memory

`Memory` is a session-scoped key-value store. It is the primitive for all session state beyond conversation history. EiBot creates the `Memory` instance scoped to the current session and passes it to Foundry.

**Future — Profile**

A dedicated service — **Profile** — will remember user-specific data across sessions: past dialogs, preferences, accumulated token usage.

### Tool Result Envelope

All internal function tools return a consistent envelope that the LLM can reason on unambiguously. Infrastructure errors (network failures, Redis timeouts) are thrown and handled by the SDK — they never reach the LLM. The envelope carries only application-level signals:

```typescript
interface ToolResult<T> {
  data?: T;
  error?: string;
}
```

- `data` present, no `error` → success
- No `data`, no `error` → success, nothing found yet (e.g. session not yet initialized)
- `error` present → application-level failure the LLM can act on

### History Management

Foundry owns the full history lifecycle. It recovers messages from Memory before the call and creates an `onFinish` handler to persist the updated message array back to Memory after the call. The `onFinish` list supports multiple handlers — each fires independently on stream completion:

- **History handler** — persists the full message array including tool calls and results back to Memory
- **Token accounting handler** — emits token usage to EiQueue

EiBot has no visibility into or responsibility for either handler.

### AgentMessage — The Response Format

`AgentMessage` (model identifier `agMessage`) is the structured output format EiBot produces on every turn. It is defined as a `model: schema` artifact and passed to the LLM via Crafter as the response schema — the LLM is constrained to produce output conforming to this shape. The schema is defined once and shared across all EiBot roles.

An `AgentMessage` is an object with a fixed `agMessage` identifier and a `def` array. Each element in the array is one typed item — a turn's response can contain one item or several in sequence. The UI renders each item in order as it arrives in the stream.

**The four item types:**

**`agNarrative`** — prose. EiBot's conversational text: explanations, confirmations, orientation, any free-form response.

```yaml
model: agMessage
def:
  - model: agNarrative
    def:
      text: >
        Got it — a French A1 course for adult beginners. Let me confirm the
        structure before we start generating.
```

**`agQuestion`** — a structured input request. Carries the question text, the options for the author to choose from, and an optional `payload` for richer renders.

```yaml
model: agMessage
def:
  - model: agQuestion
    def:
      question: What domain best describes this content?
      multi: false
      options:
        - value: language-learning
          description: Language learning
        - value: coding
          description: Coding
        - value: mathematics
          description: Mathematics
        - value: science
          description: Science
        - value: catch-all
          description: Other
```

For decision pairs, options carry the same shape — `value` is the selection id, `description` is what the card shows:

```yaml
model: agMessage
def:
  - model: agQuestion
    def:
      question: Ready to start building?
      multi: false
      options:
        - value: build
          description: "Looks good, let's build it"
        - value: continue
          description: Not yet, keep going
```

For multiselect (cultures, multiple selections), `multi: true`:

```yaml
model: agMessage
def:
  - model: agQuestion
    def:
      question: Which languages will this content support?
      multi: true
      options:
        - value: en-US
          description: English (US)
        - value: fr
          description: French
        - value: es
          description: Spanish
        - value: de
          description: German
```

**`payload`** *(optional on agQuestion)* — a data structure displayed alongside the question and options. Read-only — it is what the author is being asked about, not part of the input itself. `type` tells the UI which renderer to use for the data; `data` carries the content to display.

Common pattern: EiBot shows a table or structured summary, asks "Are these values correct?", and presents confirmation options. The author sees the data and the options together and responds.

```yaml
model: agMessage
def:
  - model: agQuestion
    def:
      question: Are these values correct?
      multi: false
      options:
        - label: Looks good
          description: Save and continue.
        - label: No, try again
          description: Let me adjust.
      payload:
        type: eivolet-structure
        data:
          nodes:
            - id: "1"
              type: unit
              label: Greetings
              children: []
            - id: "2"
              type: unit
              label: Daily Routines
              children: []
```

`payload.type` is the contract between EiBot and the UI — adding a new display type requires only a new `type` value and a corresponding UI renderer. Nothing changes in the `agQuestion` schema.

**`agResultUpdated`** — a signal that EiBot has written something to the live Eivolet instance in Redis. The UI uses this to trigger a structure panel or preview refresh without EiBot describing what changed.

```yaml
model: agMessage
def:
  - model: agNarrative
    def:
      text: Invariants saved. Now let's define the structure.
  - model: agResultUpdated
    def:
      key: invariants
```

**`agArtifact`** — an inline rich render for any content EiBot wants to surface visually inside the chat beyond plain prose. `type` identifies the renderer; `data` carries the content.

```yaml
model: agMessage
def:
  - model: agArtifact
    def:
      type: eivolet-node-preview
      data:
        nodeId: lesson-1
        title: Introduction to Greetings
        content: "..."
```

---

## Model

`ModelConfig` adds an `agent` field for the model used in agentic interactions, separate from generation models:

```yaml
model: modelconfig
def:
  models:
    object:
      model:
        id: cerebras:llama-3.3-70b
    agent:
      model:
        id: anthropic:claude-sonnet-4-20250514
```

The `when` condition syntax applies to agent model selection identically to generation models, enabling different agent models per domain, user tier, or any other resolution axis:

```yaml
def:
  rules:
    - when:
        equal: [domain.subject, programming]
      agent:
        model:
          id: anthropic:claude-opus-4-20250514
    - agent:
        model:
          id: anthropic:claude-sonnet-4-20250514
```

Foundry resolves the agent model via `ModelResolver` before each agentic turn, using the same evaluation mechanism as generation model selection.

---

# Authoring

## Actors

### UI

The UI is a renderer. It maintains three input primitives and renders whichever one the current response payload describes. It has no registry of named actions, no hardcoded flow logic, and no knowledge of what question is being asked. When a response arrives with no input payload, the UI renders a free-text input. The author's selection or text is returned as the next message.

Three primitives are supported:

- **`select`** — single-select list. EiBot provides the options.
- **`multiselect`** — multi-select list with a Confirm button. EiBot provides the options and any locked pre-selections.
- **`decision`** — two equal-sized cards for directional choices. EiBot provides the labels and descriptions.

The option content, order, and locked state are entirely EiBot's responsibility — nothing is hardcoded in the client. Adding a new question requires no UI change. Changing the collection order is a crafts change, not a code change. The same UI works for any EiBot role without modification.

### Chat

In the authoring context, Chat is responsible for reading the author's context at session open and injecting the synthetic first message. It owns the transport boundary between the author and EiBot but makes no authoring decisions.

### EiBot

In the authoring context, EiBot drives the entire session — collecting invariants, defining structure, configuring content types, and generating nodes — all through conversation. It builds the Eivolet progressively in Redis and calls the Library API when the author publishes. See the [Agent](#agent) section for EiBot's general architecture.

---

## Authoring Flow

### Session Opening

Every authoring session begins with the creation of an EiBot session. The session ID is returned to the client immediately and becomes the root of all session-scoped URIs:

- `/editor/{session_id}/chat` — the streaming chat endpoint for the authoring conversation
- `/editor/{session_id}/assets` — rendering endpoint for the live Eivolet, templates, and other session assets

Once the session ID is established, Chat injects a synthetic first message — either "The author is new to the platform." or "The author is a returning author." — and EiBot responds. That response is what the author sees first. The synthetic message is never surfaced.

The craft defines the policy: new authors are offered an orientation covering what an Eivolet is, how it is organized, and what the authoring session will involve — before any invariant is collected. Returning authors get a direct opening. EiBot decides the exact phrasing and whether to present a choice. A decision pair ("Walk me through it" / "Let's dive in") is sufficient if EiBot chooses to ask.

For now the only context injected is new vs. returning. Additional author context can be added in the future without any architectural change.

### Tutorial

New authors can opt into an EiBot-driven tutorial before starting their first Eivolet. The tutorial is a distinct authoring experience — not a static walkthrough or a set of tooltips, but a live conversation where EiBot guides the author through building a real Eivolet step by step, explaining decisions as they are made.

The exact format — whether it runs in the same surface, uses a pre-seeded Eivolet, offers a guided vs. free-form mode, or something else — is not yet defined. What is fixed is the delivery mechanism: the tutorial is entirely EiBot-driven, runs inside the same conversation loop, and requires no separate UI surface or tutorial engine. It is a role and craft configuration, not a product feature built on top.

---

### Invariants Collection

EiBot collects the four invariants through conversation. The order, phrasing, and input primitives used are EiBot's decisions — nothing is prescribed by the UI or Chat.

As each invariant is confirmed, EiBot writes it directly to the Eivolet instance in Redis. The invariants are part of the instance — not a separate artifact. When the Eivolet is published, the invariants travel with it. Any platform consumer that loads the instance has the invariants in place.

The option content for structured inputs (domain types, language lists, coding targets) lives entirely in values artifacts (`agentvalues/`). The crafts instruct EiBot which values tool to call for each invariant. The client has no hardcoded lists.

**Invariant shape:**

```typescript
interface EivoletInvariants {
  objective: string;
  cultures: string[];
  audience: string;
  domain: EivoletDomain;
}
interface EivoletDomain {
  subject: string;
  properties?: Record<string, string>;
}
```

Domain properties are typed per subject. For `computer-science`, `properties` carries `progLang`, `platform`, and `runtime`. For `language`, it carries `language`. All other subjects carry no properties.

```typescript
interface ProgrammingSubjectProperties {
  progLang: string;
  platform: string;
  runtime: string;
}
interface LanguageSubjectProperties {
  language: string;
}
```

### Structure, Content Types, and Node Generation

After invariants are collected the conversation opens up. EiBot's mode becomes intent-driven — it follows the author when intent is clear, guides when it isn't.

When EiBot has enough signal on structure and content types, it presents the generation confirmation decision pair. On confirmation, EiBot generates nodes one by one. For each node:

1. EiBot finalizes the node through conversation
2. EiBot emits a generation tool call scoped to that node
3. The platform generates the node content; it is written to the live Eivolet in Redis
4. The structure panel updates; the preview panel reflects the new state if open

Each generation turn is scoped to one node. The rest of the Eivolet is unaffected. The author can revisit any generated node by describing the change in chat — EiBot regenerates that node directly.

EiBot distinguishes between two kinds of turns throughout this phase:

**Exploration** — the author is thinking, asking questions, or reconsidering. EiBot responds conversationally and nothing is generated.

**Generation** — the author has provided enough signal for a node. EiBot triggers generation directly.

### Publish

When the author is satisfied, they publish via the FAB or by asking EiBot in chat. EiBot calls the Library API with the current Eivolet from Redis. The Library takes the draft, persists it to disk, and indexes it. The draft lifecycle stays entirely within the session — the Library does not own it and is not involved until publish time.

---

# Reference

## Key Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Agent scope | Core SDK component | The Eivolet Authoring feature spans the stack: Chat (Service), EiBot (Core SDK), Foundry + Crafter (Libraries), Library (Service) |
| Agent location | Server-side | Consistent with platform architecture; conversation history managed server-side |
| Transport | HTTP with streaming | Request/response model; streaming keeps the interaction live; no persistent connection overhead |
| Eivolet | Interactive document format based on ADL | The ADL specification defines the serialized format; an Eivolet instance is the content itself — aggregates, spaces, interactive content, and runtime-generated content definitions; can be rendered in different ways depending on context and experience |
| Session URI | `/editor/{session_id}/chat` + `/editor/{session_id}/assets` | Session ID created first; scopes all chat and asset rendering under a single addressable root; preview URL backed by live Eivolet in Redis |
| Tutorial | EiBot-driven; runs inside the conversation loop; no separate UI surface | Format TBD; delivery mechanism is fixed — role and craft configuration, not a separate product feature |
| Session opening | Synthetic first message injected by Chat; EiBot responds; user sees only the response | No client branching; no new UI surface; craft owns orientation policy; everything inside the EiBot loop |
| User context | New vs. returning author injected at session start | Craft decides behavior; minimal for now; more context can be added without architectural change |
| Agent state | Stateless between turns, reconstituted from history | Standard agentic pattern; enables session resume; no long-lived LLM context |
| SDK | Vercel AI SDK — `streamText` via ToolLoopAgent | Already in use across Eivo; handles tool calls, streaming, and turn structure natively over HTTP |
| Tool loop | ToolLoopAgent factory + `stopWhen` | Configured once, called per turn; loop internals owned by SDK |
| Instructions | Assembled from crafts via Promptbook | No base prompt; role goal + resolved craft chain composed before each turn |
| Craft filtering | Required always; optional filtered by domain | Role declares ordered list; Foundry filters at assembly time |
| Craft ordering | Declaration order preserved | Instructions assembled in role's craft sequence; role author controls composition priority |
| Tools | Declared on role `tools` field; resolved by Foundry | EiBot never touches tool registry; Foundry owns resolution and handler attachment |
| Memory | Session-scoped Redis primitive | Foundry creates typed wrappers; expires with the session; Profile service will provide user-scoped persistence in future |
| onFinish handlers | List, Foundry-created | History persistence + token accounting; EiBot has no visibility |
| History ownership | Foundry | Recovers from Memory before call; persists back via onFinish handler |
| Token accounting | Foundry onFinish handler → EiQueue | Foundry sits at execution boundary; natural place for cross-cutting metering |
| EiBot responsibility | Create and pass short-term Memory | EiBot owns nothing else — no history loading, no onFinish, no tools |
| Agent model | Separate `agent` field on ModelConfig | Distinct from generation models; same `when` condition resolution |
| Role model | Goal + crafts + tools, same infrastructure | Different roles share the same agent infrastructure; behavior determined entirely by configuration |
| `AgentMessage` | Structured response format (`agMessage`) with typed item array | Four item types: `agNarrative` (prose), `agQuestion` (structured input with options and optional DTO payload), `agResultUpdated` (Redis write signal), `agArtifact` (rich inline render); UI renders each item in stream order |
| Input model | Primitive payload in response | EiBot sends `select`, `multiselect`, or `decision` with options; UI renders without knowing the question |
| `model: agenttool` | ADL artifact for custom agent tools with schema and handler | Declares tool name, description, and input schema; handler registered in tool registry; auto-exposed from `agenttools/` directory; Foundry resolves and injects Memory at assembly time |
| `model: values` | ADL artifact for option lists; auto-exposed as tools from `agentvalues/` | No hardcoded lists anywhere — crafts, client, or tools; `metadata.name` is the tool name, `labels[en-US].overview` is the description, `def.items` is what the tool returns; each item carries `labels[en-US].title` for display and `value` for the invariant |
| Option content | Server-side, in EiBot's crafts and values artifacts | No hardcoded lists in the client; EiBot calls values tools at runtime |
| UI flow logic | None | The UI has no flow state, no action registry, no step tracking — it renders what arrives |
| Node generation | Progressive, per node, direct from conversation | No intermediate definition artifact; fires as each node is ready; live Eivolet in Redis is the state |
| Invariant storage | Written to Eivolet body in Redis as collected | Invariants are part of the Eivolet; published Eivolet carries them; any consumer gets them directly |
| Draft ownership | Chat/EiBot session via Redis | Draft lifecycle is entirely within the session; Library is not involved until publish |
| Publish | EiBot calls Library API; Library takes draft and stores it | Library owns published Eivolets only; draft stays in session until publish |
| Authoring modes | Scripted (Anvil) vs Interactive (EiBot) | Scripted: CLI-driven, definition-based, batch production via Forge; Interactive: conversation-driven, EiBot produces an Eivolet instance progressively through dialogue |
| Anvil | CLI tool for scripted content generation | Executes Eivolet Definitions through Forge from the command line; used for batch production, CI pipelines, and content automation; interactive authoring moved to EiBot |
| Editor service | Deprecated for Eivolet authoring | Authoring is now owned by EiBot with the Editor role; Editor service labor absorbed into the EiBot conversation flow |
| Forge service | Runtime content generator with persistent cache; also executes Eivolet Definition pipeline | Existing service, now formally part of the architecture; two paths: cacheable LLM materials for learners, and scripted Eivolet generation via Definition → Modeler → Prompt for batch content production |
| Library service | Accepts draft at publish; persists to disk; indexes; exposes discovery API | Single source of truth for published content; queried by all platform experiences |
| Eivolet tool | Local tool; stores and retrieves live Eivolet from Redis | EiBot's primary write surface during authoring; invariants, nodes, and structure all written through this tool; read when current state is needed |
| Library publish API | Exposed as MCP tool | EiBot calls publish directly as a tool within the conversation; no custom integration layer needed |
