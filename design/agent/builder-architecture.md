# Eivolet Definition Builder — Technical Architecture

## Purpose

This document describes the technical architecture of the components that make up the Eivolet authoring flow: the **EiBot Chat** and the **Eivolet Definition Editor**. It is a technical derivative of the Eivolet Authoring Design Specification — it takes the components that document defines and describes the architectural and LLM decisions required to implement them: agent design, transport, session state, turn cycle, prompt construction, and data models.

The Eivolet Editor is covered in a separate architecture document.

---

## Eivolet Authoring Flow

The author starts a conversation with EiBot. EiBot drives the conversation toward a single goal — a complete Eivolet Definition — asking questions and emitting commands to the chat as it goes. When the definition is ready, the session transitions to the Eivolet Definition Editor, where the author reviews and adjusts the Eivolet structure. When satisfied, the author triggers generation. Editorial Services generates the Eivolet from the definition and, when complete, the session transitions to the Eivolet Editor where the author previews and refines the result.

---

## EiBot Chat

### Overview

The EiBot Chat is backed by a server-side agent with a single goal: produce a complete Eivolet Definition. EiBot owns the conversation entirely — it decides what to ask, when it has enough information, and when to hand off to the Eivolet Definition Editor. The conversation is the means; the definition is the end.

The UI is a chat component that supports three interaction patterns — free text, single-select list, multi-select list, and decision pair. The agent decides which pattern to use for each turn. The chat renders it.

---

### Agent Design

#### Goal

The agent has one goal: produce a complete Eivolet Definition. To reach it, the agent must gather the four invariants — objective, audience, cultures, and domain — and a structural description of the Eivolet. The agent decides how to gather these, in what order, and when it has enough to move forward. There are no fixed phases and no hardcoded sequence.

#### System Prompt

The system prompt is assembled from a set of skills managed by the Eivo platform's Promptbook. There is no base prompt — every aspect of the agent's behavior is defined by skills: its goal, the invariants it must collect, the structural description it must produce, the interaction patterns it can use, and its guardrails.

The skill set is rebuilt on every turn based on context accumulated so far. As the conversation progresses and more is known — particularly once the domain is confirmed — additional skills are loaded to provide domain-specific structural knowledge such as typical hierarchies, common patterns, and known pitfalls. The agent can also request additional skills via tool call mid-session when the conversation reveals the need for context not yet loaded.

#### Agent Commands

The agent emits a stream of commands that the client executes. All commands travel on the same channel — conversation messages, interaction pattern requests, invariants card updates, and structure sketch updates are all part of the same flow. The client obeys; it does not infer anything from the conversation itself.

**Text message** — a plain conversation message from EiBot. Used for open questions, clarifications, suggestions, and objections.

**Single-select list** — a closed single-answer question; the agent provides the option set.

**Multi-select list** — a question requiring multiple selections; the agent provides the option set and specifies any locked pre-selections.

**Decision pair** — a directional choice; the agent provides two options.

**Invariants card update** — emitted whenever the agent has gathered or refined an invariant. The agent decides what the card shows and when it appears.

**Structure sketch update** — emitted as structural information accumulates. The agent decides when the sketch is ready to update and what it contains.

#### Guardrails

Every user message is classified for intent before any model call is made. If the message has no plausible connection to the agent's current goal, the agent redirects without consuming a model call.

The agent raises objections when something the author proposes conflicts with already-collected information — particularly the objective and domain — before accepting it.

#### Handoff

When the agent judges it has a complete Eivolet Definition, it presents a decision pair asking the author to confirm. On confirmation it closes the WebSocket and hands off to the Eivolet Definition Editor. If the author chooses to build the structure themselves instead, the agent closes the WebSocket and the client navigates directly to the Eivolet Editor — bypassing the Definition Editor entirely.

---

### Agent Infrastructure

#### Transport — WebSocket

The Chat uses a persistent WebSocket connection for the duration of the session. The WebSocket is the delivery pipe between the client and the server-side agent coordinator — it carries agent messages and interaction pattern requests from server to client, and user responses from client to server.

The WebSocket is opened when the Chat mounts and closed when the session exits.

#### Session State — Redis

The conversation history is persisted in Redis between turns, keyed by session ID. The agent is stateless — it is reconstituted from the full conversation history on every turn. Redis holds nothing beyond the conversation history; all session context is derived from the history by the agent on reconstitution.

#### The Assistant Pattern

The agent is implemented as an instance of the **Assistant pattern** — the platform's standard architecture for agent-backed interactions. It separates LLM interaction from session coordination into two layers:

**Streaming REST endpoint** — a stateless `POST /assistant/stream` endpoint. Accepts the conversation history and system prompt, runs `streamText` via the Vercel AI SDK, and streams the result back — either a tool call or a text response. Has no knowledge of session logic or WebSocket.

**WebSocket coordinator** — owns the session lifecycle. On each incoming user response it: appends to the conversation history in Redis, builds the system prompt for this turn, calls the streaming REST endpoint, and pipes the result to the client over WebSocket.

The coordinator never touches the LLM directly. The streaming endpoint never touches session state.

#### Turn Cycle

```
1. Client sends user response over WebSocket
2. Coordinator appends response to conversation history in Redis
3. Coordinator builds system prompt
4. Coordinator POSTs to streaming REST endpoint with history + prompt
5. SDK calls streamText; model evaluates context against its goal
6. Model emits a tool call (interaction pattern request) or a text response
7. Coordinator forwards result to client over WebSocket
8. Client renders the response; author responds
9. Cycle repeats from step 1
```

---

### Key Architectural Decisions — EiBot Chat

| Decision | Choice | Rationale |
|---|---|---|
| Agent model | Goal-driven; no fixed sequence | The agent decides how to reach the goal; the conversation is not a prescribed flow |
| Agent location | Server-side | Conversation history managed server-side; consistent with platform architecture |
| Transport | WebSocket | Persistent connection; no per-turn overhead |
| Session state | Redis — conversation history only | Agent is stateless; history is the only state needed |
| Agent statefulness | Stateless between turns, reconstituted from history | Enables session resume; no long-lived LLM context kept alive |
| LLM integration | Vercel AI SDK — `streamText` | Already in use across Eivo; handles tool calls, streaming, and turn structure natively |
| Turn model | One `streamText` call per user interaction | SDK transaction ends on tool call; user wait time happens in the WebSocket layer |
| System prompt | Assembled from skills via Promptbook | No base prompt; every aspect of agent behavior is defined by skills |
| Skills | Composed per turn based on accumulated context | Goal, invariants, guardrails, and domain knowledge all expressed as skills; agent can request additional skills mid-session |
| Agent commands | Single unified stream of tool calls; client obeys and does not infer | Conversation, input patterns, invariants card, and sketch are all commands on the same channel |
| Handoff | Agent presents confirmation decision pair when goal is reached | Agent decides when it has enough; author confirms before proceeding |

---

## Eivolet Definition Editor

### Overview

The Eivolet Definition Editor receives a fully formed Eivolet Definition produced by EiBot — including invariants, prompts, structural description, and activity configuration. The definition is the source of truth. The editor interprets and presents what is in it; it does not maintain its own registry or derive configuration from external sources.

How optional activities are expressed within the definition — so that the editor can present them as enabled or disabled — is to be determined.

The Definition Editor is not agent-driven. It requires no WebSocket connection and makes no LLM calls.

---

### Key Architectural Decisions — Eivolet Definition Editor

| Decision | Choice | Rationale |
|---|---|---|
| Agent involvement | None | Content configuration is produced by EiBot; the editor interprets the definition |
| Transport | None | No real-time interaction needed |
| Source of truth | The Eivolet Definition | The editor interprets what EiBot produced; no external registry or derived configuration |
| Optional activities | To be determined | How disabled activities are expressed in the definition is not yet defined |
| Generation trigger | Definition submitted as-is to Editorial Services | The definition is a complete, self-contained artifact |
| Post-generation | Navigate to Eivolet Editor | Natural continuation of the authoring flow |
