# System Prompt Refactoring: Roles and Crafts

## Overview

This document specifies the replacement of the partial prompts system with a two-axis model for assembling system prompts. The center of this model is the **Promptbook** — the subsystem that owns all prompt and craft artifacts, manages their storage and indexing, and serves them to the Foundry for execution.

The two-axis model introduces two complementary artifact types:

- **Role** — defines who the LLM is acting as. Always provided by the caller (Foundry / Editorial API).
- **Crafts** — composable prompt units that define a specific generation concern. Declared by the generation object (LLMMaterial, Prompt, or Modeler prompt). Resolved and assembled by the Promptbook.

The assembled system prompt is always additive: Role prompt + Crafts resolved from the declared craft IDs, in dependency order.

This replaces `model: promptpartial` and removes `prompts.system.partials` and `prompts.system.units` from the Eivolet Definition.

### Motivation

The partial prompts system assembled system prompts from a fixed set of named sections defined in large static documents. As the platform grew, these documents became harder to reason about, difficult to test in isolation, and risky to modify without unintended side effects.

The Promptbook + Role + Crafts model addresses this with four key advantages:

**Composability** — each craft is a small, focused, single-responsibility prompt block. A generation object only needs to declare its top-level crafts; the `require` chain pulls in everything else automatically. The system prompt for any given generation is exactly as large as it needs to be and no larger.

**Lean runtime execution** — at runtime, the assembled system prompt contains only the crafts relevant to that generation object. No unrelated context is included, keeping LLM calls focused and cost-efficient.

**Maintainability** — each craft lives in its own file and does one thing. Updating a craft affects a small, well-defined unit. Because the dependency graph is explicit, it is always clear which crafts — and therefore which generation objects — are affected by any change.

**Agentic capability discovery** — the Promptbook exposes a directory and tool interface for agentic flows. Agents read the directory to understand what capabilities exist, then call the Promptbook tool to fetch crafts dynamically at runtime — no predefined craft lists required.

---

## Promptbook

The **Promptbook** is the data layer of the platform — the subsystem that manages all craft, prompt, modeler, and role artifacts. It is responsible for indexing, serving, resolving, assembling, and enabling discovery of everything generation-related. The Foundry never accesses artifacts directly — it always goes through the Promptbook.

### Folder Structure

Roles and crafts are stored in a `prompts/` directory tree within the Promptbook. Two scopes exist: **global** (available across all namespaces) and **namespace-scoped** (local to a specific namespace).

```
prompts/
├── crafts/              # global crafts
├── roles/               # global roles
└── [namespace]/
    ├── crafts/          # namespace-scoped crafts
    └── roles/           # namespace-scoped roles
```

### Resolution Order

Resolution always proceeds from most specific to most general: **namespace-local first, global second**. If a craft or role is found in the namespace scope, it is used and the global scope is not consulted for that ID.

This enables namespace-level overrides of global crafts and roles without modifying shared artifacts.

---

### Internal Architecture

The Promptbook is a facade over two internal subsystems:

```
Promptbook
├── Registry  — artifact indexing, lookup, trait discovery, directory export
└── Builder   — require graph traversal, system prompt assembly
    └── Cache — assembled system prompt keyed by craft ID set
```

**Registry** — crawls the `prompts/` directory at startup and builds an in-memory index of all craft, prompt, and role artifacts. Handles craft lookup by name + domain, domain-aware disambiguation, and exports the directory — a human-readable catalog of available capabilities derived from craft `overview` fields.

**Builder** — consumes the Registry to assemble system prompts. Traverses the `require` graph depth-first using name + domain resolution at each step, deduplicates globally, and concatenates craft prompt blocks in dependency order. The assembled system prompt string is the primary cache artifact — keyed by the declared craft name set and session domain, cached until a craft artifact changes.

### Platform Architecture

All generation and runtime operations flow through a single dependency chain:

```
Assistance → Foundry → Promptbook
```

- **Assistance** — the learner-facing runtime. Handles conversations, challenge evaluation, exercise help, and tutoring threads. Never executes generation directly — delegates everything to Foundry.
- **Foundry** — the orchestration layer. Executes generation workflows, validates output against schemas. Requests assembled system prompts from the Promptbook — never assembles them directly.
- **Promptbook** — the data and assembly layer. Owns the Registry and Builder. Serves assembled system prompts, resolves capabilities via traits, and exports the directory for agentic flows.

### Example Flow: Challenge Evaluation

1. Learner submits a challenge → **Assistance** receives the submission
2. Assistance asks **Foundry**: "evaluate this challenge for this Eivolet"
3. Foundry resolves `evaluation_writer` against the session domain via the Promptbook
4. Registry returns the domain-specific variant — most specific match wins
5. Foundry asks Promptbook to assemble the system prompt for that craft set
6. Builder checks cache — on miss, traverses `require` graph, assembles, caches result
7. Foundry executes LLM call with assembled prompt
8. Result flows back: Foundry → Assistance → Learner, opening the tutoring conversation thread

### Indexing

At startup, the Registry crawls the `prompts/` directory tree and indexes all artifacts:

- **Crafts** indexed by `metadata.name` + `def.domain` — multiple crafts may share the same name, disambiguated by domain. Two crafts with identical name and identical domain is a startup error.
- **Prompts** indexed by `metadata.name`
- **Roles** indexed by `metadata.name`

Namespace-local entries take precedence over global entries for the same name + domain combination, enabling namespace-level overrides without modifying shared artifacts.

### System Prompt Assembly

When the Foundry needs to execute a generation, it asks the Promptbook to assemble the system prompt. The Builder checks its cache first — on a hit, the assembled prompt is returned immediately. On a miss, assembly proceeds:

1. **Load Role**: Retrieve the role artifact from the Registry. Append its `def.prompt` as the first block.
2. **Resolve prompt-level crafts**: Collect all craft IDs declared in `def.crafts` of the active generation object (LLMMaterial, Prompt, Modeler prompt). For each craft ID in declaration order, resolve it and its dependencies depth-first. Append matched `def.prompt` blocks, skipping already-included crafts.

The final system prompt is the concatenation of all matched blocks. Assembly is always additive. Deduplication is enforced globally — a craft prompt block is included at most once regardless of how many dependency chains reference it. The result is cached by craft ID set.

#### Craft Resolution

For a given craft name, resolution proceeds as follows:

1. Look up all craft artifacts in the Registry matching `metadata.name` (namespace-local first, then global)
2. If multiple matches exist, select the one whose `def.domain` best matches the session domain — most specific match wins (subject + properties > subject only > domain-agnostic)
3. If no domain-specific match exists, fall back to the domain-agnostic variant (no `def.domain`)
4. If no match resolves — error
5. For each craft name in `require`, in declaration order:
   - If already included, skip
   - Otherwise, resolve recursively using the same name + domain mechanism
6. Append this craft's `def.prompt` block
7. Mark this craft name + domain as included

This resolution applies uniformly everywhere a craft name appears — `require` declarations, `def.crafts` on generation objects, and explicit `lookup_craft` calls. Callers always use names; domain disambiguation is transparent, handled entirely by the resolver using the session domain.

**Example** — prompt declaring `def.crafts: [fill_blank_generator]`:

```
Role prompt
→ eivo_spec_writer          (required by fill_blank_generator transitively)
→ aggregate_writer          (required by llmmaterial_generator)
→ schema_writer             (required by llmmaterial_generator)
→ llmmaterial_generator     (required by exercise_generator)
→ exercise_generator        (required by fill_blank_generator)
→ fill_blank_generator
```

### Domain-Aware Resolution

When multiple crafts share the same name, the Registry uses the session domain to select the right variant. This applies uniformly to all craft lookups — `require` declarations, generation object craft lists, and explicit calls.

```
Foundry: "resolve bundle_writer for domain: { subject: programming, properties: { platform: react, progLang: typescript } }"
Registry: name match → domain match → returns react_ts variant
```

If no domain-specific variant matches, the domain-agnostic variant (no `def.domain`) is used as fallback. If neither exists — error at resolution time.

New domain-specific craft variants are added by dropping a new craft artifact with the appropriate `def.domain` into the Promptbook. No configuration changes required.

### Execution Modes

The Promptbook supports two execution modes depending on whether the generation flow is fully defined upfront or driven by an agent at runtime.

#### Defined Flows

Everything is declared upfront — craft IDs on the prompt, modeler, or LLMMaterial. The Foundry asks the Promptbook to assemble the system prompt from those IDs, then executes. Deterministic, fast — Builder cache hits are the common case in production.

This covers all standard workflows — modeler tree execution, LLMMaterial generation, prompt execution, and capability-based flows like evaluation and bundle generation.

```
Generation object declares craft IDs
→ Foundry asks Promptbook to assemble system prompt
→ Builder checks cache → hit: return immediately / miss: traverse graph, assemble, cache
→ Foundry executes LLM call with assembled prompt
```

#### Agentic Flows

The agent does not have a predefined craft list. It uses a tool on its own side to call into the Promptbook API. The Registry exports a directory — a human-readable catalog of available capabilities derived from craft `overview` fields — that the agent reads to understand what capabilities exist and decide which ones to request.

```
Agent reads Registry directory export
→ Decides which crafts are needed for the task
→ Calls Promptbook API via its own tool
→ Builder assembles system prompt (cache hit or miss)
→ Foundry executes LLM call with assembled prompt
```

This is why the `overview` field in every craft's `labels` is not just documentation — it is the directory entry the agent reads to decide which craft to pull. Action-oriented, capability-describing overviews are essential for agentic flows to work correctly.

### Interfaces Summary

| Interface | Owned By | Used By | Purpose |
|---|---|---|---|
| System prompt assembly | Builder | Foundry | Traverse `require` graph, assemble and cache system prompt from declared craft names |
| Domain-aware resolution | Registry | Foundry, Builder | Resolve craft by name + session domain; most specific match wins |
| Directory export | Registry | Agent | Human-readable catalog of available capabilities via craft `overview` fields |
| Promptbook API | Promptbook facade | Agent tool | Single entry point for all external callers — assembly, discovery, directory |


---


## New ADL Objects

### Role

Defines the identity and behavioral context of the LLM for a generation session. Always provided by the caller.

#### Structure

```yaml
model: role
metadata:
  name: [snake_case_role_id]
labels:
  - culture: [locale]
    title: [display-title]
def:
  prompt: |
    [role instructions]
```

**Elements**

- **metadata.name**: The role ID. Snake_case. Used by the caller to reference the role. This is the single identifier for the role — no separate `def.role` field needed.
- **def.prompt**: The role instructions injected at the start of the system prompt

#### Example

```yaml
model: role
metadata:
  name: eivo_editor
labels:
  - culture: en-US
    title: Eivo Content Editor
def:
  prompt: |
    # System Prompt
    ## Role
    You are **Eivot**, an assistant capable of generating structured
    educational content for various domains and proficiency levels.
```

---

### Craft

A Craft is a composable prompt unit — a focused, single-responsibility prompt block that defines a specific generation concern. Crafts are assembled into system prompts before LLM execution.

Crafts can declare dependencies on other crafts via `require`, enabling compositional craft graphs. Each craft artifact maps to exactly one craft ID — one craft, one artifact.

#### Structure

```yaml
model: craft
metadata:
  name: [craft-name]
labels:
  - culture: [locale]
    title: [display-title]
def:
  domain:                  # optional — restricts this craft to a specific domain
    subject: [subject]
    properties:
      [key]: [value]
  require:
    - [craft-id-a]
    - [craft-id-b]
  prompt: |
    [craft instructions]
```

**Elements**

- **metadata.name**: The craft name. Multiple crafts may share the same name — `def.domain` is the disambiguator. A craft without `def.domain` is domain-agnostic and matches any domain.
- **def.domain** *(optional)*: Restricts this craft to a specific domain. When multiple crafts share the same name, the resolver uses the session domain to select the most specific match. Two crafts with identical name and identical domain is a registry startup error.
- **def.require** *(optional)*: Flat list of craft names this craft depends on. Each name is resolved using the same name + domain mechanism — the session domain is used to select the right variant. Required crafts are resolved recursively and prepended before this craft's prompt block. Already-included crafts are skipped.
- **def.prompt**: The craft instructions appended to the system prompt when this craft is included.

#### Examples

Base craft with no dependencies:

```yaml
model: craft
metadata:
  name: eivo_spec_writer
labels:
  - culture: en-US
    title: Eivo Spec Writer
def:
  prompt: |
    ## Eivo Compliance Rules
    All generated content must adhere to the following rules...
```

Craft with dependencies:

```yaml
model: craft
metadata:
  name: llmmaterial_generator
labels:
  - culture: en-US
    title: LLM Material Generator
def:
  require:
    - eivo_spec_writer
    - eivolet_spec
  prompt: |
    ## LLM Material Generation
    ...
```

Craft with transitive dependencies:

```yaml
model: craft
metadata:
  name: trial_generator
labels:
  - culture: en-US
    title: Trial Generator
def:
  require:
    - eivo_spec_writer
    - eivolet_spec
    - llmmaterial_generator
  prompt: |
    ## Trial Generation
    ...
```

Domain-specific craft — same name, different domain:

```yaml
model: craft
metadata:
  name: bundle_writer
labels:
  - culture: en-US
    title: Bundle Writer — React/TS
def:
  domain:
    subject: programming
    properties:
      platform: react
      progLang: typescript
  prompt: |
    ## Bundle Writer — React + TypeScript
    ...
```

```yaml
model: craft
metadata:
  name: bundle_writer
labels:
  - culture: en-US
    title: Bundle Writer — Spring/Java
def:
  domain:
    subject: programming
    properties:
      platform: spring
      progLang: java
  prompt: |
    ## Bundle Writer — Spring Boot + Java
    ...
```

`lookup_craft('bundle_writer', domain)` resolves the variant whose `def.domain` best matches the session domain. If no domain-specific variant matches and no domain-agnostic `bundle_writer` exists — error.


---


## Changes to Existing ADL Objects

### Eivolet Definition

Remove `prompts.system.partials`, `prompts.system.units`, and `prompts.supportObjects`. Add `def.invariants` for runtime context. Make `def.models` optional — when omitted, the platform uses infrastructure-level defaults. Crafts are no longer declared on the definition — they are declared on the individual generation objects (`prompt`, `modeler`, `llmmaterial`) that need them.

**Before:**

```yaml
model: eivoletdef
metadata:
  name: trainning_french
def:
  models:
    object:
      model:
        id: cerebras:gpt-oss-120b
    image:
      model:
        id: crafter@gemini-2.0-flash-preview-image-generation
  prompts:
    supportObjects:
      - frech-challenge-feedback
    trees:
      - syllabusFrenchTest
    images:
      - image_french_course
    system:
      partials:
        - system_base_partial
        - markdown_lang_advanced
    eivolet: eivolet_french
  contexts:
    - context-for-eivolet
```

**After:**

```yaml
model: eivoletdef
metadata:
  name: trainning_french
def:
  invariants:
    objective: |
      A comprehensive French language course covering greetings, daily routines,
      and basic conversation for adult beginners.
    cultures:                      # required — at least one
      - fr
      - en-US
    audience: |                    # optional — used as generation hint when present
      Adult professionals with no prior French knowledge targeting A1-A2 level.
    domain:                        # optional — unlocks domain-specific capabilities
      id: language
      properties:
        targetLang: french
        level: a1
  models: global_model_config        # optional — reference to a ModelConfig artifact
  prompts:
    objects:
      - some_prompt
    trees:
      - syllabusFrenchTest
    images:
      - rust_course_icon
    eivolet: eivolet_french
  contexts:
    - context-for-eivolet
```

**Changes:**

- `prompts.system.partials` removed
- `prompts.system.units` removed
- `prompts.supportObjects` removed — evaluation and runtime capability prompts are now discovered via craft traits
- `models.image` removed — image generation is no longer configured in the definition
- `def.models` is now a string reference to a `ModelConfig` artifact rather than an inline block — model selection is externalized and shared across definitions
- `def.models` is optional — when omitted, Foundry uses the namespace-level `ModelConfig` if one exists, then the platform default
- `prompts.crafts` removed — crafts are now declared on individual generation objects (`prompt`, `modeler`, `llmmaterial`)
- `prompts.objects` added — named prompt objects available during generation (replaces `prompts.supportObjects`)
- `prompts` shape is now: `objects`, `trees`, `images`, `eivolet`
- `def.invariants` added — stores objective (required), cultures (required, at least one), audience (optional generation hint), and domain (optional, unlocks domain-specific capabilities). Co-located with the Eivolet in the content hierarchy, accessible at runtime by the Assistance engine, Foundry, and any other platform service without duplication

---

### ModelConfig

`ModelConfig` is a new first-class ADL object that externalizes model selection from the Eivolet Definition. Instead of inlining model IDs in the definition, a definition references a named `ModelConfig` artifact. Foundry resolves the actual model at generation time by evaluating the config's rules against the runtime context.

This allows model selection to vary by user tier, prompt kind, or any other runtime signal — without touching the Eivolet Definition itself.

#### Structure

```yaml
model: modelconfig
metadata:
  name: [snake_case_id]
  namespace: [namespace]
def:
  default:
    object: [model-id]     # required — fallback for all object generation
    image: [model-id]      # required — fallback for all image generation
  rules:                   # optional — evaluated in declaration order, first match wins
    - when:
        userTier: [tier]         # optional condition
        promptKind: [kind]       # optional condition
      use:
        object: [model-id]       # override for object generation
        image: [model-id]        # override for image generation (optional)
```

**Elements**

- **def.default**: Required. Fallback models used when no rule matches. `object` and `image` are both required here.
- **def.rules**: Optional. Ordered list of conditional overrides. Each rule specifies a `when` block (match conditions) and a `use` block (model overrides that apply when all conditions match).
- **when.userTier**: Matches the authenticated user's tier (e.g. `premium`, `free`).
- **when.promptKind**: Matches the kind of the generation object being executed (same `kinds` field on `LLMMaterial` and `Prompt`).
- **use.object / use.image**: Model IDs to use when the rule fires. Only the fields present in `use` are overridden — omitted fields fall through to a less specific matching rule or to `default`.

**Resolution logic** — Foundry walks `rules` in declaration order at generation time. The first rule where all `when` conditions match is applied. If no rule matches, `default` is used. More specific rules (multiple conditions) should be declared before less specific ones.

#### Full example

```yaml
model: modelconfig
metadata:
  name: global_model_config
  namespace: eivo
def:
  default:
    object: gemini-2.5-flash
    image: imagen-3
  rules:
    - when:
        userTier: premium
      use:
        object: claude-sonnet-4
    - when:
        promptKind: modeler
      use:
        object: gemini-2.5-pro
    - when:
        userTier: premium
        promptKind: modeler
      use:
        object: claude-opus-4
```

#### Minimal example (no rules)

When all Eivolets in a namespace use the same models, a config with only defaults is sufficient:

```yaml
model: modelconfig
metadata:
  name: global_model_config
  namespace: eivo
def:
  default:
    object: gemini-2.5-flash
    image: imagen-3
```

#### Changes to Eivolet Definition `def.models`

The inline model configuration in `def.models` is replaced by a reference to a `ModelConfig` artifact. The definition no longer owns model selection — it delegates entirely to the config.

**Before:**

```yaml
def:
  models:
    object:
      model:
        id: cerebras:gpt-oss-120b
      overrides:
        - type: some
          kinds: [exercise, flashcard]
          model:
            id: mistralai/devstral-2512:free
```

**After:**

```yaml
def:
  models: global_model_config    # reference to a ModelConfig artifact
```

`def.models` is optional. When omitted, Foundry falls back to the namespace-level `ModelConfig` if one exists, then to the platform-level default. This means most Eivolet Definitions will omit `def.models` entirely and inherit the namespace config automatically.

---

### LLMMaterial

`def.crafts` is added as a first-class attribute. `kinds` is retained for classification, filtering, and model override resolution.

**Before:**

```yaml
model: llmmaterial
metadata:
  name: fb_exercises_greetings_a1
  kinds:
    - exercise
    - fill_blank
  namespace: lingv
def:
  type: exercise
  schema: fill_blank_schema
  prompts: [...]
```

**After:**

```yaml
model: llmmaterial
metadata:
  name: fb_exercises_greetings_a1
  kinds:
    - exercise
    - fill_blank
  namespace: lingv
def:
  crafts:
    - fill_blank_exercise_writer
  type: exercise
  schema: fill_blank_schema
  prompts: [...]
```

**Changes:**

- `metadata.kinds` retained for classification and model overrides
- `def.crafts` added as a first-class attribute — declares the prompt-level craft set used for system prompt assembly

---

### Prompt

`def.crafts` is added as a first-class attribute.

**Before:**

```yaml
model: prompt
metadata:
  name: eivolet_french
def:
  content: |
    ...
  schema: eivolet_schema
```

**After:**

```yaml
model: prompt
metadata:
  name: eivolet_french
def:
  crafts:
    - eivolet_writer
  content: |
    ...
  schema: eivolet_schema
```

**Changes:**

- `def.crafts` added as a first-class attribute — declares the prompt-level craft set used for system prompt assembly

---

### Modeler

`def.prompt.crafts` is added to each prompt within the modeler tree. Each modeler prompt declares exactly the crafts needed for its generation context — the top-level prompt may need different skills than the child prompts.

**Before:**

```yaml
model: modeler
metadata:
  name: syllabusFrenchTest
def:
  prompt:
    content: |
      ...
    schema: syllabusFrench
  children:
    - model: modeler
      metadata:
        name: lessonLevelInitiatesEivoFrench
      def:
        prompt:
          content: |
            ...
          schema: lessonFrench
```

**After:**

```yaml
model: modeler
metadata:
  name: syllabusFrenchTest
def:
  prompt:
    crafts:
      - aggregate_writer
      - essay_challenge_generator
    content: |
      ...
    schema: syllabusFrench
  children:
    - model: modeler
      metadata:
        name: lessonLevelInitiatesEivoFrench
      def:
        prompt:
          crafts:
            - aggregate_writer
            - template_writer
            - language_template_writer
            - fill_blank_generator
            - fill_blank_exercise_writer
            - multiple_choice_generator
            - multiple_choice_exercise_writer
            - mnemonics_generator
            - flashcard_writer
          content: |
            ...
          schema: lessonFrench
```

**Changes:**

- `def.prompt.crafts` added at each level of the modeler tree — declares the craft set for that generation context
- Each level declares only the crafts relevant to the objects it generates — parent and child prompts are independent
- Crafts declared at a given level are resolved through the `require` graph at generation time

---

### Image Prompt

`model: promptimage` is replaced by standard `model: prompt` objects declaring the `image_generator` craft. Image prompts are still listed under `prompts.images` in the Eivolet Definition — this array signals to Foundry which prompts require image generation, so Foundry can instruct Crafter accordingly. Crafter remains unaware of the definition structure and simply executes what Foundry tells it. The craft is discovered via traits — no explicit registration or mapping required.

#### Image Generator Craft

```yaml
model: craft
metadata:
  name: image_generator
  extra:
    traits:
      capability: generate_image
      domains:
        - icon
        - cover
        - thumbnail
labels:
  - culture: en-US
    title: Image Generator
    overview: Generates images using the Vercel AI SDK generateImage API.
def:
  require:
    - eivo_spec_writer
  prompt: |
    # Image Generation

    Generate an image using the Vercel AI SDK `generateImage` API. The prompt content defines the visual description of the image to generate. Be specific about style, composition, color palette, and intended use.

    ## Generation Principles

    1. **Clear Visual Description**: Describe the subject, style, composition, and color palette explicitly.
    2. **Minimalist Style**: Prefer clean, geometric designs suitable for interface use — avoid photorealistic or overly complex compositions.
    3. **Intended Use**: Always consider the rendering context (icon, cover, thumbnail) when describing size constraints and level of detail.
```

#### Example Prompt

```yaml
model: prompt
metadata:
  name: rust_course_icon
  extra:
    traits:
      capability: generate_image
      domains:
        - icon
def:
  crafts:
    - image_generator
  content: |
    Create a professional, minimalist icon for a Rust programming course. The
    design should feature a clean, simplified symbol such as a stylized gear
    representing Rust, integrated with a code bracket or a blinking cursor.
    Render the entire icon in a subtle gradient of cool gray shades, emphasizing
    sharp lines and negative space. Clean, geometric composition suitable for a
    website or application interface.
```

This pattern — a capability craft discovered via traits, with a companion prompt declaring that craft — is the standard model for all capability-based generation in the platform. The same pattern applies to evaluators, bundle writers, space generators, and any future capability added to the Promptbook.


---


## Artifact Types Overview (Updated)

| Artifact Type | Used For | Description |
|---|---|---|
| Eivolet Definition | Content Generation Configuration | Specifies invariants, prompt trees, contexts, and generation workflows. References a `ModelConfig` artifact for model selection. Crafts are declared on individual generation objects, not on the definition. |
| ModelConfig | Model Selection Configuration | Named artifact defining default models and runtime resolution rules. Referenced by Eivolet Definitions and resolved by Foundry at generation time based on user tier and prompt kind. |
| Domain | Domain Vocabulary | Defines one domain field and its valid values, keyed by `metadata.name`. Used by EiBot via `resolve_options` and by the platform for domain object validation. Reusable across any platform component that needs a controlled vocabulary. |
| Modeler | Hierarchical Content Generation | Defines recursive generation of nested content structures |
| Prompt | Single Object Generation | Generates individual content objects; also covers image generation via `image-generator` craft |
| Role | System Prompt — Role Axis | Defines the LLM identity for a generation session; provided by the caller |
| Craft | System Prompt — Crafts Axis | Composable prompt unit that defines a specific generation concern; supports dependency graphs via `require` |
| Schema | Output Structure Validation | Zod-compatible specifications that validate generated content |
| Context | Shared Generation Variables | Key-value pairs available across all prompts during generation |

**Removed:**

| Artifact Type | Replacement |
|---|---|
| Partial Prompt (`promptpartial`) | Role + Craft objects |
| Image Prompt (`promptimage`) | `prompt` with `image_generator` craft, listed under `prompts.images` |


---


## Runtime Assistance and Evaluation

### The Role of Eivolet Invariants

Invariants and crafts serve distinct purposes in the platform — understanding the difference is essential:

**Crafts** are composable prompt units. They are assembled into system prompts to guide LLM generation. No platform decisions are made based on them — they are purely content for the LLM context window, whether that generation happens at content creation time or runtime.

**Invariants** are the decision layer. They are the source of truth for every capability decision the platform makes — which evaluator to select, which bundle writer to use, which space generator to configure, what assistance to provide, and what capabilities are available at all. They drive decisions at both authoring time and runtime.

- **Authoring time** — EiBot uses invariants to make structural and content decisions during the Builder flow.
- **Generation time** — Foundry uses invariants to select the right prompts, crafts, and models.
- **Runtime** — The Assistance engine uses invariants to discover capabilities, seed conversation context, calibrate responses, and keep every interaction anchored to the Eivolet's purpose.

The invariants are defined once by the author in the Eivolet Definition and stored alongside the Eivolet in the content hierarchy. Since the definition and the Eivolet object are always co-located, any runtime service can load the invariants directly from the definition when needed — no duplication, no sync issues, one source of truth.

```
lingv/
└── french_essentials/
    ├── french_essentials.eivolet.yaml       # the eivolet object
    ├── trainning_french.eivoletdef.yaml     # invariants live here
    └── [aggregates, templates, materials...]
```

### Invariant Structure

```typescript
interface Invariants {
  objective: string              // required — at least something
  cultures: string[]             // required — at least one
  audience?: string              // optional — generation hint only
  domain?: Domain                // optional — unlocks domain-specific capabilities
}

interface Domain {
  subject: string
  properties?: Record<string, string>  // well-known fields per subject — defined by model: domain artifacts
}
```

**Objective** — required. The foundation for content scope, assistance calibration, and evaluator context.

**Cultures** — required, at least one. Drives language, localization, and cultural appropriateness across all platform layers.

**Audience** — optional. When present, used as a generation hint to calibrate complexity and tone. When absent, ignored — no defaults, no fallback.

**Domain** — optional. When absent, the Eivolet operates with a reduced capability surface — generic assistance and evaluation still work, but capabilities that require environment or stack knowledge (spaces, programming challenges, bundle generation) are unavailable. When present, `domain.subject` identifies the domain type and `domain.properties` provides the well-known fields the Promptbook uses to resolve the most specific capability available.

```yaml
# programming — full environment capabilities
domain:
  subject: programming
  properties:
    progLang: typescript
    runtime: nodejs
    platform: react

# language learning
domain:
  subject: language
  properties:
    targetLang: french

# no domain — reduced capabilities, no environment-dependent features
domain: ~
```

The valid values for `subject` and each property key are defined by `model: domain` ADL artifacts — one artifact per field, keyed by `metadata.name`. The Promptbook uses `domain.subject` first for capability matching, then refines against `properties` entries. The most specific match wins; the Promptbook falls back to less specific matches when no exact match exists. Properties are not open-ended — each field has a defined set of valid values queryable via `resolve_options`.

Subject values and property names are platform identifiers — always English, always lowercase, independent of the cultures configured for the Eivolet. They are never translated or localized.

A domain with `subject: other` and no properties is valid. However, domain-specific capabilities — challenges, spaces, and specialized learning components — require a known subject with resolvable properties to provision the execution environment, select evaluator crafts, and generate CRD templates. With `subject: other`, only the universal capabilities are available: lessons, templates, exercises, flashcards, and playroom.

### Domain Accessor

The `Domain` DTO is stored on the learner session. At runtime, features access domain properties through typed accessors rather than reaching into the raw properties bag:

```typescript
abstract class DomainAccessor<T> {
  constructor(protected readonly domain: Domain) {}
  abstract getProperties(): T;

  static getAccessor(domain: Domain): DomainAccessor<unknown> {
    switch (domain.subject) {
      case 'programming':            return new ProgrammingDomainAccessor(domain);
      case 'language': return new LanguageDomainAccessor(domain);
      default:                  return new GenericDomainAccessor(domain);
    }
  }
}

class ProgrammingDomainAccessor extends DomainAccessor<ProgrammingSubjectProperties> {
  getProperties(): ProgrammingSubjectProperties {
    return this.domain.properties as ProgrammingSubjectProperties;
  }
}

class LanguageDomainAccessor extends DomainAccessor<LanguageSubjectProperties> {
  getProperties(): LanguageSubjectProperties {
    return this.domain.properties as LanguageSubjectProperties;
  }
}

class GenericDomainAccessor extends DomainAccessor<AnyData> {
  getProperties(): AnyData {
    return this.domain.properties ?? {};
  }
}
```

Usage at the call site:

```typescript
// typed where subject is known
(DomainAccessor.getAccessor(session.domain) as ProgrammingDomainAccessor).getProperties().progLang

// safe fallback where subject is unknown
DomainAccessor.getAccessor(session.domain).getProperties()
```

This eliminates raw property casts (`(traits as any).progLang`) throughout the codebase. The `Domain` DTO is stored as-is; the accessor is constructed on demand via the static factory.

This is why Step 1 of the Builder is architecturally critical — the invariants determine the entire capability surface of the Eivolet. An Eivolet without a domain is perfectly valid — it just has a narrower set of available platform features.

### Assistance Engine Context

Every conversation thread the Assistance engine opens — whether for challenge evaluation, exercise help, or general tutoring — is permanently anchored to the Eivolet invariants:

- **Objective** — keeps assistance on topic and scoped to the Eivolet's subject matter
- **Domain** — when present, determines what kind of help is appropriate (language correction, code review, math explanation) and which capabilities are available; `domain.subject` drives capability selection
- **Audience** — when present, calibrates complexity, tone, and depth of explanations
- **Cultures** — ensures responses are in the right language and culturally appropriate

The learner never has to provide context — the engine already knows what they are learning, at what level, and for what audience. Every response is automatically calibrated to that context.

The invariants act as the Assistance engine's persistent memory for the entire Eivolet, making every interaction feel coherent and purposeful across all capabilities.

### Evaluation as a Craft

Challenge evaluation is a runtime capability of the Assistance engine. When a learner submits a challenge, the engine needs to:

1. Select the right evaluator for the challenge type and domain
2. Generate structured feedback anchored to the submission
3. Open a conversation thread seeded by that feedback
4. Keep follow-up responses within the scope defined by the evaluation

Evaluation is implemented as a **craft** — an artifact in the Promptbook that the engine discovers at runtime using the Eivolet invariants.

#### Evaluator Discovery via Traits

Evaluator crafts are resolved by name + domain — the same mechanism used everywhere. A craft named `evaluation_writer` with `def.domain: { subject: language }` is selected when the session domain matches. No separate traits mechanism required.

```yaml
model: craft
metadata:
  name: evaluation_writer
labels:
  - culture: en-US
    title: Evaluation Writer — Language
def:
  domain:
    subject: language
  prompt: |
    ## Language Evaluation
    ...
```

The Assistance engine resolves `evaluation_writer` against the session domain. The Promptbook returns the domain-specific variant, falling back to the domain-agnostic one if no specific match exists. No explicit mapping in the Eivolet Definition, Eivolet object, or a new ADL type.

This pattern generalizes naturally — new evaluators are added by dropping a new craft artifact with the appropriate `def.domain` into the Promptbook. No configuration changes required.

#### Evaluation as Conversation Anchor

The evaluation result is not a static one-shot response. It is the **opening message of a tutoring conversation** anchored to the specific submission. The generated `evaluation` object serves two purposes:

1. **Starting point** — the structured feedback the learner sees first (summary, strengths, suggestions)
2. **Guardrail** — the context the Assistance engine uses to keep all follow-up responses focused on the submission, the feedback areas, and the Eivolet's objective

The learner can then continue the conversation to:
- Ask why something was marked as needing improvement
- Request examples of how to fix a specific issue
- Explore the grammar rules behind a correction
- Ask for an alternative way to express an idea

The Assistance engine maintains the evaluation context throughout the thread, ensuring every response stays anchored to the original submission and the Eivolet invariants.

#### Evaluation Object

The `evaluation` ADL object is generated by an evaluator craft at runtime. It carries the structured feedback and the criteria that constrain the conversation thread:

```yaml
model: evaluation
metadata:
  name: [generated-id]
def:
  summary: |
    A brief overview of the submission's approach, content quality, and proficiency level.
  strengths: |
    - Specific things the learner did well
  suggestions: |
    - Clear, actionable improvement areas
  criteria:
    [archetype-specific validation criteria]
```

#### Domain-Specific Evaluators

Evaluator crafts follow the same `require` pattern as all other crafts, with domain-specific variants requiring a base evaluator:

```
evaluation_writer                    # base craft — defines evaluation object structure
└── language_evaluation_writer       # language writing challenges
└── code_evaluation_writer           # programming challenges
└── essay_evaluation_writer          # essay challenges
```

Each domain-specific evaluator declares its traits so the Promptbook can resolve it from the Eivolet invariants at runtime.

### Bundle Writer

When a programming challenge starts, the platform needs to bootstrap a project workspace with starter files. This is handled by the `bundle_writer` craft and its domain-specific variants.

The `bundle_writer` base craft (domain-agnostic) defines the bundle object structure, file layout conventions, formatting requirements, and generation principles. Domain-specific variants share the same name and add `def.domain` to handle tech-stack-specific file sets:

```
bundle_writer                        # domain-agnostic — bundle object structure and principles
bundle_writer (platform: react, progLang: typescript)    # React + TypeScript projects
bundle_writer (platform: react, progLang: javascript)    # React + JavaScript projects
bundle_writer (subject: programming, progLang: python)   # Python projects
```

Resolution is automatic — `lookup_craft('bundle_writer', domain)` returns the most specific match for the session domain. The domain-agnostic variant is the fallback. No traits, no explicit mapping required.

```yaml
model: craft
metadata:
  name: bundle_writer
def:
  domain:
    subject: programming
    properties:
      platform: react
      progLang: typescript
  require:
    - bundle_writer      # resolves domain-agnostic base
  prompt: |
    ## Bundle Writer — React + TypeScript
    ...
```

Bundle writers generate infrastructure only — scaffolding, boilerplate, and a minimal passing test. They never implement or hint at the challenge solution. The generated bundle must be immediately runnable before the learner writes a single line of code.

---

## Snippets

Snippets are the ADL reuse mechanism for schema authoring. They allow named YAML fragments to be defined once and composed into schema definitions — keeping schemas DRY, eliminating duplication, and making structural changes in one place.

Snippets can only be used in three ADL model types: `model: schema`, `model: schemasnippet`, and `model: schemasnippets`. There is no prescribed file structure — new snippet files can be added as needed.

Snippet expansion happens at parse time — before any validation or generation. The resolved YAML is what Foundry and the schema validator see. Snippets are invisible at runtime.

### ADL Model Types

**`model: schemasnippet`**

Defines a single named schema fragment. The `def` block is the raw schema shape — exactly what will be injected at the point of use:

```yaml
model: schemasnippet
metadata:
  name: space
def:
  type: object
  properties:
    model:
      const: spaceenvironment
    metadata:
      $snippet-extend: default-metadata-ext
      properties:
        extra:
          type: object
          properties:
            archetype:
              const: environment
    $snippets: [labels]
    def:
      type: object
      properties:
        type:
          type: string
        environment:
          type: object
          properties:
            progLang:
              type: string
            platform:
              type: string
            runtime:
              type: string
        duration:
          type: string
        artifacts:
          type: object
          properties:
            model:
              const: bundle
            def:
              type: object
              properties:
                objects:
                  type: array
                  items:
                    type: object
                    properties:
                      relativePath:
                        type: string
                      kind:
                        type: string
                      name:
                        type: string
                      content:
                        type: string
```

**`model: schemasnippets`**

Defines a named collection of fragments in a single artifact. The `def` block is a list of `name` + `value` pairs. Each entry is individually addressable by name — the collection name is the artifact identifier, the individual names are the operator targets:

```yaml
model: schemasnippets
metadata:
  name: common-schema-fragments
def:
  - name: default-metadata
    value:
      metadata:
        type: object
        properties:
          name:
            type: string
        required: [name]
  - name: default-metadata-ext
    value:
      type: object
      properties:
        name:
          type: string
      required: [name]
  - name: metadata-with-kinds
    value:
      metadata:
        type: object
        properties:
          name:
            type: string
          kinds:
            type: array
            items:
              type: string
        required: [name]
```

### Operators

Four operators are available wherever a snippet should be expanded:

**`$snippet`** — injects a single named fragment inline. The fragment replaces the operator at that position.

**`$snippets`** — injects multiple named fragments. Errors at parse time on key conflicts — use only when the fragments are guaranteed to be non-overlapping.

**`$snippet-extend`** — deepmerges a named fragment into the surrounding structure, then allows additional properties to be declared inline on top. Inline values take precedence on conflict.

**`$snippet-each`** — expands a list of named fragments into an array. Each named fragment becomes one element. Use for union types (`anyOf`, `oneOf`) built from a set of named shape variants.

Operators can be used inside snippet definitions themselves — a `schemasnippet` can reference other snippets via any operator, building composite shapes from primitives:

```yaml
model: schemasnippet
metadata:
  name: llmmaterial
def:
  type: object
  properties:
    $snippets: [model-llmmaterial, metadata-with-kinds, labels]
    def:
      type: object
      properties:
        type:
          type: string
        schema:
          type: string
        crafts:
          type: array
          items:
            type: string
        prompts:
          type: array
          items:
            type: string
```

### Usage in `model: schema`

All four operators are available anywhere inside a `model: schema` definition.

**`$snippet`** — inject a single reusable shape:

```yaml
def:
  schema:
    type: object
    properties:
      metadata:
        $snippet: default-metadata
```

**`$snippets`** — merge multiple non-overlapping fragments:

```yaml
def:
  schema:
    type: object
    properties:
      $snippets: [base-properties, audit-properties]
```

**`$snippet-each`** — build a union type from named shape variants:

```yaml
model: schema
metadata:
  name: agent-message
def:
  schema:
    type: object
    properties:
      model:
        const: agMessage
      def:
        type: array
        items:
          anyOf:
            - $snippet-each: [agent-narrative, agent-questions, agent-result-updated, agent-artifact]
```

Adding a new variant means adding a snippet and its name to the list — no structural change to the schema definition.

**`$snippet-extend`** — merge a base shape and extend with local additions:

```yaml
model: schema
metadata:
  name: challenge_withtext_eval_feedback
  extra:
    type: essay
def:
  schema:
    type: object
    properties:
      model:
        const: feedback
      metadata:
        $snippet-extend: default-metadata-ext
        properties:
          extra:
            type: object
            properties:
              archetype:
                const: essay
      def:
        type: object
        properties:
          summary:
            type: string
          strengths:
            type: string
          suggestions:
            type: string
```

`default-metadata-ext` provides the shared metadata shape; the inline `properties.extra` block adds the archetype-specific constraint on top.
