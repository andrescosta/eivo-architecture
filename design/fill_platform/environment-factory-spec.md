# Environment Factory — Design Specification

## Context

This document captures the design decisions made for the Eivo environment factory tool.
It is intended to be fed back into a future session to continue implementation.

### Broader Initiative

The environment factory is part of a larger initiative to populate the Eivo platform
with LLM-generated Eivolets for demo purposes. The dependency chain is:

```
1. Environment Factory (this tool)             ← current priority
   → receipts, tar artifacts, Theia images
   → platform environments ready to host content

2. Content Generation Pipeline
   → LLM generates Eivolets per platform
     (Anvil batch pipeline — platform spec → Eivolet Definition → batch execution)
   → Eivolets reference platform environments via domain invariants
     (progLang, runtime, platform)

3. Demo Release
   → Platform populated with curated LLM-generated Eivolets
   → Students open spaces and challenges backed by pre-built environments
   → Environments spin up with pre-installed deps from PVC tar
```

The environment factory is a hard prerequisite for step 2 — Eivolet content generation
for programming platforms requires working environments to back spaces and challenges.
The two-step generation pattern already established (expensive model for authoring,
cheap model like DeepSeek for batch execution via Anvil) applies to the content
generation pipeline in step 2.

---

## Problem Statement

Eivo needs pre-built environments for two purposes:

- **Spaces and Challenges** — a tar of pre-installed dependencies (node_modules, venv, etc.)
  extracted into the student's workspace at instance time. Full Theia IDE. Student works
  on a substantial project over time.
- **Firecracker templates** — a VM snapshot (vm.snap + vm.mem + rootfs.ext4) resumed
  per student instance in ~28ms (Priority 2 — not implemented yet). Lightweight, short-lived,
  single-purpose. Used for small focused exercises — a component, a function, a snippet —
  not full projects.

These are two distinct environment types with different scope, frontend, and lifecycle:

| Concern         | Spaces / Challenges         | Firecracker exercises        |
|-----------------|-----------------------------|------------------------------|
| Use case        | Full projects                | Small focused exercises       |
| Frontend        | Theia IDE                   | CodeMirror + custom explorer  |
| Persistence     | Persistent workspace         | Ephemeral, dies on stop       |
| Startup         | Extract tar from PVC         | Resume VM snapshot (~28ms)    |
| Platforms       | React, Spring, Vue, etc.     | React, TS, Python, etc.       |
| No Theia needed | No                           | Yes — fully embedded in Eivolet UI |

The tool covers **platforms** (React, Spring, Vue, Django, etc.) built on top of
**base language runtimes** (Node, Java, Python). Base language images are maintained
manually and are out of scope for this tool.

---

## Architecture Overview

```
Platform spec (ADL)
      │
      ▼
Phase 1 — Generate (LLM)
  platform spec → LLM → model: receipt
  (saved to ADL store / git)
      │
      ▼
Phase 2 — Execute (Tool)
  model: receipt
    → write files to disk
    → run init commands
    → model: commands (packaging) → tar.gz → PVC
```

The receipt is the **source of truth** — rerunnable, versionable, storable in git.
Phase 2 can be re-executed at any time from the same receipt without hitting the LLM again.

---

## ADL Objects

### `model: receipt` (new)

A first-class ADL object. The LLM produces one per platform in Phase 1.

```yaml
model: receipt
metadata:
  name: react-ts
  namespace: eivo
def:
  platform:
    progLang: typescript
    runtime: node
    platform: react
  baseImage: eivo-env-node        # Theia image to use at runtime
  files:
    - path: package.json
      content: |
        {
          "dependencies": {
            "react": "^18",
            "typescript": "^5",
            "vite": "^5"
          }
        }
    - path: tsconfig.json
      content: |
        { "compilerOptions": { ... } }
  commands:
    - npm install
    - npm cache clean --force
```

**Fields:**
- `def.platform` — mirrors the domain properties shape already used in Eivo invariants
  (`progLang`, `platform`, `runtime`)
- `def.baseImage` — reference to the pre-built Theia image for this runtime
- `def.files` — files to write to disk before running commands (LLM-generated)
- `def.commands` — platform-specific init commands (LLM-generated, runs ONCE during build)

The LLM only produces `files` and `commands` — the platform-specific parts.
Packaging is handled separately by `model: commands`.

### `model: commands` (existing, new instance)

One instance per packaging target. Single-purpose — one artifact per output type.

```yaml
model: commands
metadata:
  name: challenge-packaging
  namespace: eivo
def:
  commands:
    - tar czf deps.tar.gz node_modules package.json tsconfig.json
    - # upload to PVC
```

When Firecracker template support is added (Priority 2), a second artifact is added:

```yaml
model: commands
metadata:
  name: template-packaging
  namespace: eivo
def:
  commands:
    - # snapshot vm
    - # upload to PVC
```

No changes to existing artifacts — new packaging target = new artifact.

---

## Phase 1 — LLM Receipt Generation

**Input:** Platform spec — the domain properties already present in Eivo values artifacts:

```yaml
platform:
  progLang: typescript
  runtime: node
  platform: react
```

**LLM task:** Given the platform spec, produce:
- The dependency files (`package.json`, `tsconfig.json`, `requirements.txt`, etc.)
- The install commands (`npm install`, `pip install`, etc.)

**LLM does NOT produce:**
- Packaging commands (owned by `model: commands`)
- Firecracker snapshot commands (owned by `model: commands`)
- Plugin lists (owned by the Theia image build, out of scope)

**Output:** A `model: receipt` artifact saved to the ADL store.

---

## Phase 2 — Receipt Execution

The tool loads the receipt and executes it mechanically:

```
1. Write receipt.files to disk
2. Run receipt.commands  (npm install / pip install / etc.)
3. Load challenge-packaging commands
4. Run packaging commands → deps.tar.gz
5. Upload tar to PVC under /assets/<platform-name>/v<n>/bare/deps.tar.gz
```

### PVC Layout

```
/assets/
  react-ts/
    v1/
      manifest.json       ← platform name, created_at, checksum
      bare/
        deps.tar.gz
  spring-boot/
    v1/
      manifest.json
      bare/
        deps.tar.gz
  python-django/
    v1/
      ...
```

Versioned so rebuilding a platform doesn't break running environments using the old artifact.

---

## Theia Image Architecture

### What is NOT the tool's concern

Theia image building is manual, separate from this tool entirely.

### Structure

```
/
  Dockerfile                    ← single shared Dockerfile, parameterized by PLATFORM
  lerna.json
  package-lock.json
  .npmrc
  lingv/                        ← shared Theia extension, same across all platforms
  browser-app/                  ← shared browser app config
  theia-images/
    node/
      package.json              ← Theia + node/ts plugins
      plugins/                  ← pre-downloaded plugins
    java/
      package.json              ← Theia + java plugins
      plugins/
    python/
      package.json              ← Theia + python plugins
      plugins/
```

### Shared Dockerfile

```dockerfile
ARG BUILD_IMAGE
ARG RUNTIME_IMAGE
ARG PLATFORM

FROM ${BUILD_IMAGE} AS builder
WORKDIR /home/theia

COPY package.json lerna.json package-lock.json .npmrc ./
COPY theia-images/${PLATFORM}/package.json ./package.json
COPY lingv/package.json ./lingv/package.json
COPY lingv/tsconfig.json ./lingv/tsconfig.json
COPY browser-app/package.json ./browser-app/package.json
COPY browser-app/gen-webpack.config.js ./browser-app/gen-webpack.config.js
COPY browser-app/gen-webpack.node.config.js ./browser-app/gen-webpack.node.config.js
COPY browser-app/webpack.config.js ./browser-app/webpack.config.js
COPY lingv/src ./lingv/src

RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline

RUN PARENT_APP_ORIGIN=http://lingv.jobico.local npm run build:browser

FROM ${RUNTIME_IMAGE}
RUN groupadd --system theia && useradd --system --gid theia theia
WORKDIR /home/theia

COPY --chown=theia:theia theia-images/${PLATFORM}/plugins /home/theia/plugins
COPY --chown=theia:theia .theia /home/theia/.theia
COPY --chown=theia:theia --from=builder /home/theia/lerna.json /home/theia/
COPY --chown=theia:theia --from=builder /home/theia/package.json /home/theia/
COPY --chown=theia:theia --from=builder /home/theia/node_modules /home/theia/node_modules
COPY --chown=theia:theia --from=builder /home/theia/browser-app /home/theia/browser-app

USER theia
ENV THEIA_DEFAULT_PLUGINS=local-dir:/home/theia/plugins
EXPOSE 3000
CMD ["npm", "run", "start:browser"]
```

### Build Script

```bash
for PLATFORM in node java python; do
  docker build \
    --build-arg PLATFORM=$PLATFORM \
    --build-arg BUILD_IMAGE=node:20 \
    --build-arg RUNTIME_IMAGE=node:20-slim \
    -t eivo-env-$PLATFORM .
done
```

Build args per platform:

| Platform | BUILD_IMAGE  | RUNTIME_IMAGE     |
|----------|-------------|-------------------|
| node     | node:20     | node:20-slim      |
| java     | node:20     | eclipse-temurin:25-jre |
| python   | node:20     | python:3.12-slim  |

Note: BUILD_IMAGE is always `node:20` (needed for `theia build`).
RUNTIME_IMAGE varies per language.

---

## CRD Template — Per-Platform Provisioning

The `model: crdtemplate` does **not** need to be generated per platform. One shared
artifact covers all Theia-based platforms. Platform-specific values are injected at
provisioning time via the existing variable substitution mechanism.

### What varies per platform

Only two fields differ across platforms:

```yaml
app:
  image: {&environment.image}      # eivo-env-node, eivo-env-java, eivo-env-python
  command: {&environment.command}  # ["npm", "run", "start:browser"], etc.
```

Everything else — cleaner, initJob, volumes, security context, env vars, resource
limits — is identical across all platforms and stays hardcoded in the single shared
`crdtemplate`.

### Platform config artifact

A `model: values` artifact per platform carries the substitution values:

```yaml
model: values
metadata:
  name: react-ts-env-config
  namespace: eivo
  extra:
    kind: environment-config
def:
  items:
    - value:
        environment:
          image: reg.jobico.local/eivo-env-node:latest
          command: ["npm", "run", "start:browser"]
```

The provisioner loads the right `environment-config` values artifact for the platform
and passes it to `CRDForge` alongside the shared `crdtemplate`. No per-platform
template files, no generation needed.

### Tool output

Phase 2 does NOT generate `crdtemplate` artifacts. It generates only:
- `deps.tar.gz` → PVC (challenge/space artifact)

The `model: values` environment-config artifact is created manually alongside the
receipt — it is a one-time setup per platform, not regenerated by the tool.

---

## What the LLM Generates vs What is Fixed

| Concern                        | Owner                           |
|-------------------------------|--------------------------------|
| Dependency files               | LLM (Phase 1)                   |
| Install commands               | LLM (Phase 1)                   |
| Hello world starter file       | LLM (Phase 1, Firecracker only) |
| Plugin list                    | Manual (package.json per platform) |
| Packaging commands             | model: commands (fixed)         |
| Firecracker snapshot commands  | model: commands (Priority 2)    |
| Theia image build              | Manual build script             |
| PVC upload                     | Tool (Phase 2, fixed)           |
| crdtemplate                    | Single shared artifact (manual) |
| environment-config values      | Manual, one per platform        |

---

## Supported Platforms (Demo Release)

Initial set for the demo:

| Platform     | Base Image      | receipt name    |
|-------------|----------------|-----------------|
| React + TS   | eivo-env-node  | react-ts        |
| JavaScript   | eivo-env-node  | javascript      |
| Python       | eivo-env-python| python          |
| Java (Spring)| eivo-env-java  | spring-boot     |

Full language list (Judge0 CE v1.13.1 — 37 languages) is available for future
challenge/execution support but does not require Theia images or receipts —
those run via Judge0 directly.

---

## Firecracker Integration (Priority 2 — Not Yet Implemented)

Captured here for continuity. When implemented:

### Use case
Small focused exercises — a React component, a Python function, a TS snippet.
Not full projects. Restrictive scope by design.

### Frontend
No Theia. Fully embedded in the Eivolet UI:
- **CodeMirror** — editor
- **Custom explorer** — lightweight file tree, talks to the VM via vsock
- No separate IDE process, no plugins, no external window

The receipt for Firecracker environments does NOT have a `baseImage` field —
there is no Theia to launch. The VM just needs the runtime + starter files.

### Receipt shape (Firecracker variant)
```yaml
model: receipt
metadata:
  name: react-ts-exercise
  namespace: eivo
  extra:
    target: firecracker
def:
  platform:
    progLang: typescript
    runtime: node
    platform: react
  # no baseImage — Theia not used
  files:
    - path: package.json
      content: |
        { ... }
    - path: src/App.tsx        ← hello world starter file
      content: |
        import React from 'react'
        export default function App() {
          return <h1>Hello World</h1>
        }
  commands:
    - npm install
```

The presence of `metadata.extra.target: firecracker` distinguishes this from
a challenge/space receipt. The hello world starter file is included in `files`
for Firecracker receipts — it is excluded for challenge/space receipts.

### Artifact lifecycle
- The same `model: receipt` schema is reused — `target` field drives the output path
- A new `model: commands` artifact `template-packaging` is added
- Phase 2 gains a third output path: Firecracker snapshot stored under
  `/assets/<platform>/v<n>/firecracker/`
- Snapshot lifecycle:
  - Boot VM from base rootfs (runtime only, no Theia)
  - Write receipt files + run receipt commands inside VM
  - Pause VM → `firecracker snapshot/create` → `vm.snap` + `vm.mem` + `rootfs.ext4`
  - Upload to PVC
  - Per-user instance = resume from snapshot via MAP_PRIVATE CoW (~28ms)
  - Working state lives in guest RAM (tmpfs) — evaporates on VM termination
- Requires `/dev/kvm` on the node — already confirmed available

---

## Infrastructure Notes

- Cluster: self-hosted Kubernetes
- `/dev/kvm` confirmed available on node (`crw-rw---- 1 root kvm 10, 232`)
- PVC already in use for environment seeds (`eivo-seeds`, `filesPvcDir: environments-seeds`)
- Existing `model: config` `defaults` artifact has relevant PVC paths:
  - `project.filesDir: /mnt/files`
  - `project.filesPvc: eivo-seeds`
  - `project.filesPvcDir: environments-seeds`

---

## Proposal — Domain Shape Refactoring (Pending)

This is a separate design decision to be applied to the ADL/architecture docs when addressed.
Captured here for continuity.

### Current shape

```typescript
interface ProgrammingDomain {
  subject: string
  properties: {
    progLang: string
    platform?: string
    runtime?: string
  }
}
```

### Proposed shape

```typescript
interface ProgrammingDomain {
  subject: string
  progLang: string        // base — always present for programming Eivolets
  stack?: {               // optional — present when content targets a specific platform/stack
    runtime: string
    libraries: string[]
  }
}
```

### In YAML

```yaml
# bare language — no stack
domain:
  subject: programming
  progLang: python

# platform Eivolet — with stack
domain:
  subject: programming
  progLang: typescript
  stack:
    runtime: node
    libraries:
      - react
      - vite
```

### Rationale

- `progLang` promoted to top-level — it is always present, not an optional property
- `platform` removed — replaced by `libraries` which directly declares what is in the environment
- `stack` groups `runtime` + `libraries` as optional extras — absent for bare language Eivolets
- `libraries` provides a more precise receipt lookup key at provisioning time
- Pre-calculated receipts are unaffected — `libraries` just narrows the lookup, nothing more
- `DomainAccessor` gets a cleaner typed shape — base properties always present, stack optional
