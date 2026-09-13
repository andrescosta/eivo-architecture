# Build System Specification: Lingv Monorepo

## 1. Architecture Overview

The project uses a **layered Makefile orchestrator** sitting on top of a **pnpm monorepo**. This allows for a unified command interface while respecting the unique build requirements of frontend apps, backend services, Kubernetes operators, and shared libraries.

### Core Components

* **Root Orchestrator (`/Makefile`)**: The entry point for all global operations.
* **Common Configuration (`/common.mk`)**: Defines global variables (Registry, Versioning, Namespaces) and the cascading trigger logic.
* **Project-specific Makefiles**: Located in `apps/`, `services/`, and `external/`, handling localized Docker or local dev logic.

---

## 2. Versioning Strategy

We use a **Unified Git-SHA Versioning** system.

* The variable `VERSION` is automatically derived: `$(shell git rev-parse --short HEAD)`.
* This version is injected into:
* **Docker Tags**: `reg.jobico.local/app-name:sha-12345`
* **NPM Packages**: During the `release-libs` cycle.
* **K8s Deployments**: Injected via `sed` into the `IMAGE_PLACEHOLDER` field.



---

## 3. Command Lifecycle (The Cascades)

The system uses **Dependency Cascading** defined in `common.mk`. One command triggers the prerequisites automatically:

| Target | Dependencies | Description |
| --- | --- | --- |
| `make all` | `refresh` | Full system synchronization. |
| `make refresh` | `deploy` | Deploys and restarts K8s pods. |
| `make deploy` | `push` | Pushes images to registry and applies manifests. |
| `make push` | `build` | Builds Docker images and pushes them. |
| `make build` | `build-packages` | Compiles TS packages then builds Docker images. |

---

## 4. Documentation Pipeline

Documentation is treated as code. Markdown files are transformed into high-quality artifacts stored in `workspace/blobs/`.

* **Standard Docs (`/docs`, `/specs`)**: Converted via **Pandoc** using `pdf.sh -d`.
* **Presentations (`/slides`)**: Converted via **Marp** into PDF and PPTX using `pdf.sh -s` and `ppt.sh`.

---

## 5. Developer Guide (Quick Start)

### Local Development (No Docker)

Use these for rapid iteration with hot-reloading:

* `make dev-all-local`: Runs everything (Lingv + APIs).
* `make dev-apis-local`: Runs only Realtime and Cloud services.

### Shared Library Management

Internal libraries (`packages/core`, `packages/commons`) are built automatically during `make build`, but publishing to the NPM registry is an **explicit manual step**:

* `make release-libs`: Bundles, bumps version to Git-SHA, and runs `pnpm publish`.

### The "Grand Master" Command

* `make all-all`: The complete CI/CD simulator. Generates all documentation, builds the workspace, creates Docker images, pushes them, and deploys to the cluster.

---

## 6. Directory Map

| Path | Responsibility | Build Tool |
| --- | --- | --- |
| `/apps` | Frontend & Core Logic | Docker / Vite |
| `/services` | APIs & Utility Jobs | Docker / Node.js |
| `/packages` | Shared Libraries | pnpm / TS |
| `/external` | Submodules & Operators | Kustomize / Go |
| `/scripts` | Automation Logic | Bash |
| `/workspace` | Documentation & Blobs | Marp / Pandoc |

---