# Runnerd

## Purpose

An execution service for framework-based programs and code snippets. Decouples scheduling (placement decisions) from execution (container lifecycle). Scales from one machine to many without architectural change.

Two execution kinds:
- **Long-running** — a process that stays alive. Scheduled via `/runnerlet/schedule`. Always has an output stream; optionally exposes an HTTP port.
- **Run-and-exit** — a process that runs, produces output, and exits. Called via `/run` (synchronous).

## Long-Running Architecture

### Components

```
Client
  │
  ├──► Scheduler        (placement, assignment, returns runnerlet id)
  │         │
  │         └──► Assignment Store (in-memory or Redis)
  │
  └──────────────────────────► Runnerlet Frontend (owns assignment, lifecycle)
                                      │
                                      └──► Runnerlet (the running container)
```

### Scheduler

Stateless. Receives `POST /runnerlet/schedule`, decides which Runnerlet Frontend to assign the key to, writes the assignment to the store, notifies the frontend, and returns the runnerlet `id` immediately — without waiting for the container to start.

Pure placement engine. Has no knowledge of whether containers are running.

### Runnerlet Frontend

Owns assigned keys. When an assignment arrives from the scheduler, it starts the container immediately in the background. When a request arrives for an assigned key:

- Container running → handle request directly
- Container stopped, assignment valid → start container → handle request
- Assignment expired (GC'd) → 409 Needs Schedule

Manages the full container lifecycle: start, idle stop, restart on demand, destroy on TTL expiry.

### Runnerlet

The running container. A long-running process that:
- Always produces stdout/stderr accessible via the output URL
- Optionally exposes an HTTP port (for service-style substrates)

Has no knowledge of the scheduler or frontend.

## Request Models

### `model: program`

A complete program. The `source` field determines where the files come from.

**Filesystem-based** — files resolved from `metadata.name` on the local filesystem:

```yaml
model: program
metadata:
  name: prog-abc123
def:
  domain:
    progLang: typescript
    platform: react
  ttl: 900
  source:
    type: fs          # default — files on filesystem, resolved from metadata.name
  stdin: "hello world"
  args:
    - --verbose
  env:
    DEBUG: "true"
```

**Inline** — files provided directly in the request:

```yaml
model: program
metadata:
  name: prog-abc123
def:
  domain:
    progLang: python
  source:
    type: inline
    files:
      - path: main.py
        content: |
          from utils import greet
          print(greet("world"))
      - path: utils.py
        content: |
          def greet(name):
            return f"hello, {name}"
  stdin: "world"
```

Future `source` types: `git`, `object-storage`.

### `model: snippet`

A single inline piece of code. For multiple files, use `model: program` with `source.type: inline`.

```yaml
model: snippet
metadata:
  name: snip-abc123
def:
  domain:
    progLang: python
  runner: isolated      # optional — overrides substrate default
  code: |
    import sys
    print(sys.stdin.read())
  stdin: "hello world"
  args:
    - --input
    - foo
  env:
    DEBUG: "true"
```

`stdin` and `args` are per-execution inputs used when the substrate command reads them. `env` is merged in order: substrate `env` first, then request `def.env` — request values overwrite substrate values for matching keys.

## Substrate (`model: substrate`)

Describes the execution environment for a domain. The frontend resolves it from the request's domain — callers never reference it directly.

The onboarding pipeline produces one substrate per (progLang, platform) pair.

```yaml
model: substrate
metadata:
  name: typescript-react
  namespace: eivo
def:
  domain:
    progLang: typescript
    platform: react
  engine: container     # container | mvm | isolated | process
  image: reg.jobico.local/runnerd-typescript-react:latest
  resources:
    memoryMiB: 512
    cpuMillis: 500
  env:
    NODE_ENV: development
    PORT: "3000"
  sourceDir: /app/src
  filesystem:
    structure:
      - path: /app/deps
        readOnly: true
      - path: /app/scaffold
        readOnly: true
  endpoints:
    - port: 3000
      healthPath: /
  commands:
    - type: serve
      run: npm run dev
    - type: test
      run: npm test -- --watchAll=false
    - type: build
      run: npm run build
```

**`engine`** — the default execution mechanism. Can be overridden per session at the API level:
- `container` — containerd-based container (P1, current implementation)
- `mvm` — Firecracker microVM (P3, future — hardware kernel boundary)
- `isolated` — process isolated via isolate/cgroups/namespaces, no container image
- `process` — plain process on the frontend node, no isolation

The substrate declares the appropriate default. The caller overrides when needed.

**`filesystem`** — declares the filesystem for the substrate. Three fields:
- `type` — the implementation; defaults to `standard` if absent
- `properties` — implementation-specific configuration; `properties.type` defaults to `overlay` if absent. Use `type: tmpfs` with optional `sizeLimit` for in-memory data substrates.
- `structure` — paths within the filesystem and their access mode

Most substrates only need `structure`. Data substrates declare `properties.type: tmpfs` to run entirely in memory.

**`sourceDir`** — where program files are mounted inside the container. Everything outside `sourceDir` is read-only by mount policy. File permissions inside `sourceDir` are set by the platform (`444` read-only, `644` editable). The frontend does not modify them.

**`endpoints`** — optional. Present for service-style substrates. The frontend proxies this port and monitors it for readiness.

**`commands`** — typed commands the frontend can invoke inside the container:
- `serve` — starts the long-running process
- `test` — runs and exits, exit code is the result
- `build` — runs and exits, output is the result

The same program files are shared across two consumers:
- **runnerd** — mounted at `sourceDir`
- **Challenge (Theia)** — same files as read-only symlinks in the workspace

## API

### `POST /runnerlet/schedule`

Schedules an asynchronous execution — service, process, or anything that runs in the background. Returns immediately — the container starts in the background.

```
POST /runnerlet/schedule    { YAML request object } → { id, procUrl, stdioUrl }
```

- `id` — used for all subsequent operations on this runnerlet
- `procUrl` — always present. The running process's exposed interface: for service-style substrates this proxies the HTTP service; for process-style substrates it renders the program output. Works for programs and snippets alike — the client always uses this URL to see what the process produced.
- `stdioUrl` — always present. Raw stdout/stderr NDJSON stream. For some substrates this is the same content as `procUrl` (CLI, build); for others it is the process logs (React dev server logs vs the running app).

The client never needs to know which substrate is behind the runnerlet — it always gets both URLs and uses them appropriately.

### `GET /runnerlet/{id}`

Get the current state of a runnerlet.

```
GET /runnerlet/{id}    → { id, status, procUrl, stdioUrl }
```

- `status` — `starting` | `ready` | `stopped`

**If the runnerlet has been GC'd** (TTL expired) → `409 Needs Schedule`. The client must call `POST /runnerlet/schedule` again.

### `GET /runnerlet/{id}/proc/...`

The running process's exposed interface. Proxied directly to the running container — path and query string are forwarded as-is.

```
GET/POST/... /runnerlet/{id}/proc/...    → proxied to running process
                                         → starting → 503 (client retries)
                                         → stopped, assignment valid → restart → proxy
                                         → GC'd → 409 Needs Schedule
```

### `GET /runnerlet/{id}/stdio`

Stream raw stdout/stderr of the running process as NDJSON. Stdout and stderr are delivered as separate frames — the client can display them combined or separately.

```
GET /runnerlet/{id}/stdio    → NDJSON stream
```

```json
{ "kind": "stdout", "text": "..." }
{ "kind": "stderr", "text": "..." }
```

Container not running → 404. Runnerlet GC'd → `409 Needs Schedule`.

### `POST /runnerlet/{id}/stop`

Explicitly stop the runnerlet container. The assignment is retained — the container restarts on the next request if the TTL has not expired.

```
POST /runnerlet/{id}/stop    → 204 No Content
```

Runnerlet GC'd → `409 Needs Schedule`.

### `POST /run`

Synchronous run-and-exit execution. No runnerlet, no lifecycle. Runs immediately, streams output, exits.

```
POST /run    { YAML request object } → NDJSON stream
```

```json
{ "kind": "stdout", "text": "..." }
{ "kind": "stderr", "text": "..." }
{ "kind": "exit", "code": 0 }
{ "kind": "error", "message": "..." }
```

## Assignment Store

Abstraction over the assignment backing store. The scheduler writes; the frontend reads on cold start.

```go
type AssignmentStore interface {
    Write(key string, info Assignment) error
    Read(key string) (Assignment, error)
    Delete(key string) error
}
```

**In-memory** — single machine, zero infrastructure.
**Redis** — multi-machine, shared contract between scheduler and frontends.

Same code, different wiring. No Redis required to start.

## Runnerlet Lifecycle

```
assigned ──► starting ──► ready ──► running
                                      │
                            idle timeout → stopped
                                      │
                            next request → starting → ready
                                      │
                              hard TTL → GC'd
```

**Assigned** — scheduler assigns the key to the frontend. Container starts immediately in the background.

**Starting** — container created from substrate image, process started.

**Ready** — process running and accepting requests. For substrates with `endpoints`: port listening + health path responding.

**Running** — container live. Output stream available. Service requests proxied if `endpoints` declared.

**Stopped** — idle timeout reached. Container stopped, assignment retained. Restarts on next request if TTL not expired.

**GC'd** — hard TTL reached. Container destroyed, assignment deleted. All endpoints return `409 Needs Schedule`.

## Filesystem

| Layer | Content | Enforced by |
|---|---|---|
| Base OS + runtime | Language engine, package manager | overlayfs (read-only image layer) |
| Dependencies | `node_modules`, Maven repo | overlayfs (read-only image layer) |
| Scaffold | Framework boilerplate | overlayfs (read-only image layer) |
| Substrate directories | `substrate.filesystem` entries | Read-only bind mount (frontend) |
| Program files (`sourceDir`) | Resolved from `metadata.name` | File permissions set by platform |

The frontend's only filesystem responsibility: mount substrate directories read-only.

## Security

- Namespace isolation (PID, network, filesystem, IPC) per container
- cgroup resource limits from `substrate.resources`
- seccomp/AppArmor syscall filtering
- Default-deny egress; substrates may declare an allowlist
- overlayfs prevents substrate file modification by program code

## Deployment

### Single Machine

```
Scheduler + Runnerlet Frontend in same process
Assignment store: in-memory
No Redis required
```

### Multiple Machines

```
Scheduler (stateless, N replicas)
     │
     └──► Redis (assignment store)
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 Frontend   Frontend   Frontend
 (+ containers)
```

Scheduler places assignments across frontends based on capacity. The runnerlet `id` encodes the assigned frontend address — the client hits the right frontend directly after scheduling.

### Kubernetes Mode

```
OS → k8s → pod → containerd → runnerd frontend
```

Scheduler and frontend can run as separate Deployments or combined in one. Frontend mounts the containerd socket via hostPath. Inner containers created in the `runnerd` containerd namespace.

## Known Gaps

### OS-Level Substrates

The current substrate model covers application-level platforms (React, Spring, Python, etc.) running inside containers. OS-level learning content — assembly programming, bootloader development, kernel concepts requiring bare-metal access — is an identified gap.

This would require:
- A different execution engine (QEMU/KVM rather than containerd)
- Different isolation model (full VM, not container namespaces)
- Different substrate fields (machine config, CPU architecture, boot image rather than container image and scaffold)

This maps naturally onto the `model: mvm` engine path already planned in the runnerd architecture. When OS-level Eivolets are needed, the substrate model and runnerd engine will need extending to support machine-level virtualization substrates.

## Observability

- Per-runnerlet metrics: starts, restarts, idle stops, request count
- Per-frontend metrics: active containers, memory usage, assignment count
- Structured logs per runnerlet id; output not logged

## Service Modules

```
runnerd/
├── cmd/
│   ├── scheduler/       # placement, assignment, id generation
│   └── frontend/        # runnerlet lifecycle, proxy, output streaming
├── internal/
│   ├── scheduler/       # placement algorithm, assignment store client
│   ├── frontend/        # container lifecycle, proxy, idle/TTL management
│   ├── container/       # containerd client
│   ├── substrate/       # substrate ADL resolution by domain
│   ├── store/           # AssignmentStore: in-memory + Redis implementations
│   ├── api/             # shared HTTP types, request decoding
│   └── netns/           # network namespace, egress rules
└── build/               # substrate image build tooling
```

`runnerd` lives in `eivo-sandbox`.
