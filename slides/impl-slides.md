---
marp: true
theme: default
paginate: true
header: ''
footer: ''
style: |
  section {
    background-color: #f5f5f5;
    color: #2c3e50;
    font-size: 20px;
    padding: 60px 80px;
  }
  section::after {
    text-align: center;
    font-size: 11px;
    color: #95a5a6;
    text-transform: uppercase;
    letter-spacing: 1px;
  }
  header {
    font-size: 11px;
    color: #7f8c8d;
    text-transform: uppercase;
    letter-spacing: 2px;
    text-align: center;
    width: 100%;
  }
  footer {
    font-size: 11px;
    color: #95a5a6;
    text-transform: uppercase;
    letter-spacing: 1px;
    text-align: center;
    width: 100%;
  }
  h1 {
    font-size: 48px;
    font-weight: 700;
    color: #2c3e50;
    margin-bottom: 40px;
  }
  h2 {
    font-size: 22px;
    font-weight: 700;
    color: #2c3e50;
    margin-top: 30px;
    margin-bottom: 15px;
  }
  p {
    font-size: 18px;
    line-height: 1.6;
    color: #34495e;
    margin-bottom: 20px;
  }
  strong {
    color: #2c3e50;
    font-weight: 700;
  }
  ul {
    margin-left: 2rem;
    margin-top: 0.5rem;
  }
  li {
    font-size: 18px;
    line-height: 1.6;
    color: #34495e;
    margin-bottom: 8px;
  }
---

<!-- _header: "" -->
<!-- _footer: "" -->

# Everything as Code

Not just the code — everything. One repository, one workflow, one source of truth.

---

<!-- _header: "EVERYTHING_AS_CODE" -->
<!-- _footer: "CODE → EXTERNAL → DOCUMENTATION → WORKSPACE" -->

# The Project

The platform spans one central repository with two satellite components linked as submodules.

**Code** — Platform, SDK, and services.

**External** — Environment infrastructure and writing editor as submodules.

**Documentation** — Technical specifications and architecture documents.

**Workspace** — Presentations, planning, and all project assets.

---

<!-- _header: "EVERYTHING_AS_CODE" -->
<!-- _footer: "APPS → SERVICES → PACKAGES" -->

# The Platform

A PNPM monorepo in TypeScript where packages power services, and services power apps.

**apps** — Learning experiences and content generation tooling.

**services** — Cloud API, Real-time coordination, and Environment jobs.

**packages** — Eivo's SDK, Content generation and rendering pipelines, and Shared infrastructure.

---

<!-- _header: "EVERYTHING_AS_CODE" -->
<!-- _footer: "CONFIGS → PROMPTS → ENVIRONMENTS → OPERATIONS" -->

# Supporting Assets

Assets beyond code, required for the platform to operate.

**Configs** — Platform configuration, Facet specifications, and CRD templates.

**Prompts** — LLM generation and evaluation prompts for Editorial and Assistance services.

**Environments** — Pre-built assets and scaffolding for environment provisioning.

**Operations** — Makefiles and scripts for deployment, database management, and seeding.

**External** — Environment infrastructure and writing editor as submodules.

---

<!-- _header: "EVERYTHING_AS_CODE" -->
<!-- _footer: "SPECIFICATIONS → ARCHITECTURE → PRESENTATIONS → PLANNING" -->

# Workspace

Everything that defines, explains, and communicates the platform lives in the repository alongside the code.

**Specifications** — ADL, SDK, and platform contracts.

**Functional Documentation** — Capabilities, experiences, and platform features.

**Architecture** — System design, component interactions, and infrastructure.

**Presentations** — Slide decks for architecture, functionality, and stakeholder communication.

**Planning** — Ideas, designs, and roadmaps.

---

<!-- _header: "" -->
<!-- _footer: "" -->

# The Stack

The technologies behind each component.

---

<!-- _header: "THE_STACK" -->
<!-- _footer: "LINGV → ANVIL → CLOUD_API → REALTIME → JOB" -->

# Apps & Services

**Lingv** — Reference experience that utilises Next.js, Auth.js, Socket.IO.

**Anvil** — Content generation CLI that utilises NestJS, nest-commander.

**Cloud API** — Backend services built on NestJS modular monolith, TypeORM, JWT authentication.

**Realtime** — Live collaboration service built on Socket.IO, dual server and worker output.

**Job** — Environment initialization job built on TypeScript, webpack.

---

<!-- _header: "THE_STACK" -->
<!-- _footer: "SDK → GENERATION → RENDERING → INFRASTRUCTURE" -->

# Packages

**Core SDK** — A pure TypeScript SDK that implements the platform's business logic and capabilities.

**Facets SDK** — The UI framework and navigation layer, built with React and Next.js server components.

**Crafter** — The multi-provider LLM integration library, leveraging Vercel AI SDK, Zod, and Mustache.

**Foundry** — The content generation orchestrator, coordinating Crafter to execute LLM workflows.

**ContentKit** — The content rendering library, integrating next-mdx-remote-client, CodeHike, and CodeMirror.

**Contracts** — Shared data structures and DTOs, relying on class-transformer and js-yaml.

**Commons** — Shared utilities and infrastructure abstractions, integrating the Kubernetes client and OIDC.

---

<!-- _header: "THE_STACK" -->
<!-- _footer: "SANDBOX → OPERATOR → THEIA → WRITING_EDITOR" -->

# External Components

Three components across two repositories, integrated as submodules.

**Sandbox** — Environment infrastructure.

- **Operator** — A Kubernetes operator written in Go using Kubebuilder, responsible for provisioning environments on demand and managing their full lifecycle from creation to cleanup.
- **Theia** — A customized Eclipse Theia IDE written in TypeScript, deployed as a container inside each provisioned environment and embedded in experiences via iframe, integrating Core SDK for authentication, challenge state, and file submission.

**Writing Editor** — A full rich text editor built on TipTap and ProseMirror in TypeScript with React, providing formatting, word count, and draft persistence for writing challenges. Published as a private package and consumed through the Facets SDK layer.