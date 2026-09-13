# eivo-substrates — Overview

## What It Is

`eivo-substrates` is an autonomous agent-driven pipeline that onboards programming domains into the Eivo platform. It generates all the infrastructure assets required to support a domain — container images, bootstrapper scripts, Kubernetes manifests, and ADL objects — and provisions them into the cluster without human intervention.

A domain is a programming language or language+framework combination (e.g. `java`, `java-spring`, `typescript-react`, `python`). Once onboarded, a domain can be used by Eivo to deliver challenges, spaces, and code execution exercises to learners.

## Why It Exists

Eivo supports multiple programming domains across challenges (Theia-based IDE environments), spaces (persistent development environments), and code execution (program evaluation). Each domain requires a specific set of infrastructure assets to function. Manually creating and maintaining these assets for 54+ domains is not sustainable. `eivo-substrates` automates the entire process — from asset generation to cluster provisioning — driven by a single source of truth.

## Connection to Eivonator

`eivo-substrates` is the infrastructure bootstrap layer that Eivonator depends on. When Eivonator detects demand for a new programming domain (via job market trends, GitHub activity, or learner signals), it adds the domain to `domains.yaml` and triggers a named run. The pipeline generates and provisions all required assets automatically. Eivonator can then produce Eivolets for that domain without any manual infrastructure work.

## Source of Truth

Everything is driven by `domains.yaml` — a single file with four sections:

**`def.families`** — shared base images for runtimes that must be downloaded rather than pulled from Docker Hub (jvm, node, dotnet). The generation stage builds these once and reuses them across domains.

**`def.runtimes`** — the runtimes to install in the agent container. Each entry is a language toolchain. Entries with `members` declare groups where the base installs a shared dependency (JDK, Erlang, dotnet) and members add only their specific toolchain on top.

**`def.domains`** — flat list of all domains. Each entry is self-contained and declares:
- `for` — the functions the domain supports: `environment-domain-ide`, `service`, `program`
- `domain` — `progLang` and optionally `platform` (the framework)
- `runtime`, `family`, `imageStyle`, `installLib` — how to build the container image
- `packageManager` — drives `initJob.commands` in the crdtemplate
- `scaffold`, `service`, `commands` — required for `environment-domain-ide` and `service`
- `io` — required for `program`
- `priority` — controls which domains are onboarded first (max 3 per priority group)

## The `for` Field

The `for` field is the core concept. It declares what a domain enables:

- **`environment-domain-ide`** — a full Theia-based environment: scaffold PVC, deps PVC, plugins PVC, and a crdtemplate that the Environment Operator uses to spin up the pod. Used for challenges and spaces.
- **`service`** — a long-lived container process listening on a port, lifecycle managed by runnerd. The container runs `entrypoint.sh` which maps `RUNNERD_COMMAND` to platform commands (serve, test, build).
- **`program`** — a one-time code execution container managed by runnerd. The container runs `run.sh` which executes a single file and exits with its output.

A domain can declare multiple `for` values. Generation produces the union of assets for all declared functions.

## Stages

The pipeline runs as three stages, each with its own `CLAUDE.md`:

### agent (`agent/CLAUDE.md`)

Generates the `Dockerfile` at the repo root — the bootstrap image the agent itself runs inside. Contains all runtimes from `def.runtimes` plus Docker, kubectl, and Claude Code. Run once; rebuild only when `def.runtimes` changes.

The Dockerfile has two sections:
- **FIXED** — always the same: debian base, core tools, kubectl, Node.js, Claude Code
- **RUNTIMES** — one labeled block per runtime entry; grouped by base/member relationships

### generation (`generation/CLAUDE.md`)

Generates all assets for each domain, organized by `for` value:

**`environment-domain-ide`:**
- `work/runs/{run}/generation/{name}/scaffold/` — infrastructure files baked into the image and placed in the scaffold PVC
- `work/runs/{run}/generation/{name}/infra/pvc.yaml` — three PVCs (scaffold, deps, plugins), all `ReadOnlyMany`
- `work/runs/{run}/substrates/{name}-crdtemplate.yaml` — Kubernetes CRD template for the Environment Operator
- `work/runs/{run}/substrates/{name}-onboarding.yaml` — onboarding entry point; PVC names are the single source of truth

**`service`:**
- `work/runs/{run}/{name}/entrypoint.sh` — maps `RUNNERD_COMMAND` to platform commands
- `work/runs/{run}/{name}/health.sh` — port readiness check
- `work/runs/{run}/generation/{name}/Dockerfile` — service container image
- `work/runs/{run}/substrates/{name}.yaml` — substrate ADL object with endpoints and commands

**`program`:**
- `work/runs/{run}/{name}/run.sh` — executes a single file and returns output
- `work/runs/{run}/generation/{name}/Dockerfile` — program container image
- `work/runs/{run}/substrates/{name}.yaml` — substrate ADL object with io invocation

**All domains:**
- `work/runs/{run}/generation/{name}/README.md` — platform identity and build instructions
- `work/runs/{run}/generation/images/{family}/Dockerfile` — family base image (once per family, for `imageStyle: command` domains)

### onboarding (`onboarding/CLAUDE.md`)

Validates generated files and provisions the cluster. Two sequential phases per domain:

**Phase 1 — Validate and Fix** *(log: `validate`)*: Dockerfile syntax, bash syntax, YAML validity, PVC name consistency, plugin existence on Open VSX. Agent fixes issues and retries until clean.

**Phase 2 — Provision:**
1. `build_family` — build and push family base images (skip if already in registry)
2. `build` — build and push the platform image
3. `test` — run the bootstrapper tests inside the built container
4. `plugins` — download `.vsix` files from Open VSX
5. `apply_pvcs` — `kubectl apply` the PVC manifests
6. `scaffold_pvc` — populate the scaffold PVC
7. `deps_pvc` — populate the deps PVC and verify install exits 0
8. `plugins_pvc` — populate the plugins PVC

## Named Runs

Every agent invocation is a named run specified in the prompt:

```
run: full-run-1 — run all domains at priority 0
run: java-spring-fix-1 — regenerate dockerfile for java-spring service
run: node-family-1 — regenerate family image for node
run: full-run-1 — resume
```

All output and the log for a run live under `work/runs/{run-name}/`. The agent reads the log at startup and skips completed steps. Runs are scoped — a run targeting one domain or one `for` type leaves everything else untouched.

The log at `work/runs/{run-name}/log.yaml` tracks every step:

```yaml
agent:
  status: done | failed | pending
generation:
  domains:
    typescript-react:
      bootstrapper: done | failed | pending
      dockerfile: done | failed | pending
      ...
  families:
    node:
      dockerfile: done | failed | pending
onboarding:
  domains:
    typescript-react:
      validate: done | failed | pending
      build: done | failed | pending
      ...
```

## Priority and Dev Workflow

Each domain has a `priority` field. No more than 3 domains share the same priority. The agent processes domains in priority order within a run.

Priority 0 is the dev priority — a small representative batch covering all `for` types, validated manually in Eivo before automating the rest. Once the priority-0 batch is solid, higher priorities follow.

## runnerd Integration

runnerd is the execution service that consumes the substrate ADL objects generated by this pipeline. It reads `work/runs/{run}/substrates/{name}.yaml` to know how to run a domain:

- `service` domains — runnerd starts the container, runs `entrypoint.sh`, manages the lifecycle, and proxies the HTTP port
- `program` domains — runnerd starts the container, runs `run.sh` with the learner's file, captures output, and exits

The `engine: container` field on every substrate tells runnerd to use containerd for execution. Future engines (`mvm`, `isolated`) are defined but not yet active.

## Eivo Platform Integration (Pending)

These integration tasks connect the onboarded domains to the Eivo product:

- **Replace Judge0** — runnerd replaces Judge0 CE as the code execution backend for `program` domains
- **Test challenges with new assets** — validate challenges work with `environment-domain-ide` assets and with `program`-only domains
- **Exercises based on runnerd services** — add exercise types in learning templates that leverage `service` domains
- **Examples as ADL objects** — external multi-file ADL examples and inline template examples for EiBot authoring

## Assets Reference

| Path | Purpose |
|---|---|
| `assets/generation/lib/github.sh` | Downloads runtime release assets from GitHub |
| `assets/generation/lib/uri.sh` | Downloads runtime assets from direct URLs |
| `assets/bootstrappers/lib/bootstrap.sh` | Library sourced by all bootstrapper scripts |
| `specs/runnerd-spec.md` | runnerd API and substrate format reference |
| `specs/generation-lib-spec.md` | Install library (`github.sh`, `uri.sh`) reference with examples |
