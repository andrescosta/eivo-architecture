# Learning Activities

## Purpose

This spec covers the full stack of interactive learning activities in Eivo — from the learner's UI interaction down to code execution. It describes the layered architecture, the ADL objects, the APIs, and the lifecycle of a learning session.

## Layered Architecture

```
LearningSession  (facade — no business logic)
       │
       ├── ActivityCoordinator  (one per session — exercises, attempts, validation, execution)
       │         │
       │         ├── RunnerletService  (runnerd accessor — runnerlets only)
       │         │         │
       │         │         └── runnerd  (internal execution service)
       │         │
       │         └── SnippetService  (runnerd accessor — snippets only)
       │                   │
       │                   └── runnerd  (same service, POST /run)
       │
       └── (bookmarks, EiBot discussions, content rendering — existing)
```

### LearningSession

Pure facade. No business logic. Creates `ActivityCoordinator` on session open. Delegates all exercise and execution calls to it. Queries `ActivityCoordinator` when feeding EiBot discussions.

### ActivityCoordinator

One instance per learning session. Coordinates the full exercise lifecycle:

- Owns the exercises map (registered after rendering)
- Owns exercise attempts and tracks history
- Owns validators per exercise kind
- Validates answers and produces result data
- Delegates execution to `RunnerletService` (service exercises) or `SnippetService` (i/o exercises)
- Exposes APIs for other components to query exercise state
- Future: publishes events to UserProfile and Analytics

Created by `LearningSession` on init. Knows about eivolets, sessions, exercises, DocumentDB. Does not know about EiBot — it exposes data, `LearningSession` feeds EiBot.

### RunnerletService

Pure runnerd accessor for runnerlets. No knowledge of eivolets, sessions, or DocumentDB. Owns:
- The `(eivoletId, userId, name)` → runnerlet ID mapping (Redis)
- All runnerd HTTP calls for runnerlet lifecycle
- Transparent reschedule on `409`
- Working directory awareness

### SnippetService

Pure runnerd accessor for snippets. No knowledge of eivolets, sessions, or DocumentDB. Wraps `POST /run` — fire and forget, returns output.

---

## ADL Objects

### `model: runnerlet`

The execution unit for multi-file programs and service exercises. Sent to `POST /runnerlet`. Represents a full execution environment with session context, working directory, and initial files.

```yaml
model: runnerlet
metadata:
  name: {exercise name — unique within the eivolet}
def:
  session:
    eivoletId: {eivoletId}
    userId: {userId}
  domain:
    progLang: typescript
    platform: react
  ttl:
    idleSeconds: 600
    hardSeconds: 2700
  files:
    - path: src/App.tsx
      content: |
        export default function App() {
          return <h1>Hello</h1>
        }
    - path: src/utils.ts
      content: |
        export const greet = (name: string) => `Hello, ${name}`
```

`metadata.name` + `session.eivoletId` + `session.userId` form the working directory path in the working PVC. runnerd creates the working directory on `POST /runnerlet` and deletes it on GC.

### `model: snippet`

The execution unit for single inline code. Sent to `POST /run`. Synchronous, no lifecycle.

```yaml
model: snippet
metadata:
  name: snip-abc123
def:
  domain:
    progLang: python
  engine: isolated      # optional — overrides substrate default
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

### `model: substrate`

Describes the execution environment for a domain. Resolved by runnerd from the request's `domain` — callers never reference it directly. Produced by the onboarding pipeline.

```yaml
model: substrate
metadata:
  name: typescript-react
def:
  domain:
    progLang: typescript
    platform: react
  engine: container
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

**`engine`** — execution mechanism:
- `container` — containerd-based container (current)
- `mvm` — Firecracker microVM (future)
- `isolated` — isolate/cgroups/namespaces, no image
- `process` — plain process, no isolation

**`filesystem`** — declares the filesystem. `type` defaults to `standard`, `properties.type` defaults to `overlay`. Data substrates use `properties.type: tmpfs`.

**`sourceDir`** — where the working directory is mounted inside the container.

**`endpoints`** — service substrates only. Frontend proxies this port and monitors for readiness.

**`commands`** — `serve`, `test`, `build` — typed commands the frontend invokes.

---

## Storage

### Support PVCs (substrate — permanent, shared)

Produced by the onboarding pipeline. Same for all learners:

- `substrate-{name}-scaffold-v1` — scaffold files, read-only
- `substrate-{name}-deps-v1` — dependencies, read-only
- `substrate-{name}-plugins-v1` — Theia plugins, read-only

### Working PVC (session — ephemeral)

One shared PVC (`eivo-working`) with subdirectories per session:

```
eivo-working/
└── {eivoletId}/
    └── {userId}/
        └── {name}/          ← session working directory
            ├── src/App.tsx  ← editable files (copies from DocumentDB)
            └── src/utils.ts
```

- Created by runnerd on `POST /runnerlet`
- Seeded with initial files from `model: runnerlet` `def.files`
- Mounted as `sourceDir` in the container
- Deleted by runnerd on GC (TTL expiry)

---

## runnerd API

### `POST /runnerlet`

Creates and starts a runnerlet asynchronously. Returns immediately — container starts in the background.

```
POST /runnerlet    { model: runnerlet } → { id, procUrl, stdioUrl }
```

- `id` — used for all subsequent operations
- `procUrl` — proxies the running service HTTP port
- `stdioUrl` — raw stdout/stderr NDJSON stream

runnerd creates the working directory from `def.files`, mounts the support PVCs and working directory into the container, and starts the process.

### `GET /runnerlet`

Look up a runnerlet by semantic key — used on component mount before the ID is known.

```
GET /runnerlet?userId={userId}&materialId={el1},{el2},{el3}    → { id, status, procUrl, stdioUrl }
```

`materialId` elements are joined by comma — the array is opaque, runnerd splits and uses as the lookup key. Returns `404` if no runnerlet exists for that key.

### `GET /runnerlet/{id}`

Current state of a runnerlet — used for polling once the ID is known.

```
GET /runnerlet/{id}    → { id, status, procUrl, stdioUrl }
```

- `status` — `starting` | `ready` | `stopped`
- GC'd → `409 Needs Schedule`

### `PATCH /runnerlet/{id}`

Applies a partial update to a running runnerlet. Called when the learner clicks "Run" after editing code. Accepts a partial `model: runnerlet` — only the fields provided are applied.

```
PATCH /runnerlet/{id}    { model: "runnerlet", def: { files: [...] } } → 204 No Content
```

Synchronous for file writing — runnerd writes files to the working directory and signals the process to restart (SIGTERM → process exits → bootstrapper restarts), then returns `204`. Does not wait for the process to be ready. After `204`, the client refreshes the iframe — `procUrl` owns the loading/ready/error response while the process comes back up. Additional fields (e.g. `ttl`, `env`) may be patched in the future without API changes.

### `GET /runnerlet/{id}/proc/...`

Proxies directly to the running container's HTTP service.

```
GET/POST/... /runnerlet/{id}/proc/...    → proxied to running process
                                         → starting → 503
                                         → stopped → restart → proxy
                                         → GC'd → 409 Needs Schedule
```

### `GET /runnerlet/{id}/stdio`

Stream stdout/stderr as NDJSON.

```
GET /runnerlet/{id}/stdio    → NDJSON stream
```

```json
{ "kind": "stdout", "text": "..." }
{ "kind": "stderr", "text": "..." }
```

### `POST /runnerlet/{id}/stop`

Stop the container. Assignment retained — restarts on next request if TTL not expired.

```
POST /runnerlet/{id}/stop    → 204 No Content
```

### `POST /run`

Synchronous snippet execution. No lifecycle. Streams output and exits.

```
POST /run    { model: snippet } → NDJSON stream
```

```json
{ "kind": "stdout", "text": "..." }
{ "kind": "stderr", "text": "..." }
{ "kind": "exit", "code": 0 }
{ "kind": "error", "message": "..." }
```

---

## Service Exercise UX Lifecycle

The "Try" button drives the service exercise lifecycle. Button state rules:

- **runnerd unavailable** → button disabled, editor frozen
- **runnerd available, no session** → button enabled
- **runnerd available, session running, code unchanged** → button disabled
- **runnerd available, session running, code changed** → button enabled

"Try" flow:
1. Learner edits code → button enabled (dirty flag set)
2. Learner clicks "Try"
3. If no session → `ActivityCoordinator` calls `RunnerletService.create()` → `POST /runnerlet`
4. If session exists → `ActivityCoordinator` calls `RunnerletService.updateFiles()` → `PUT /runnerlet/{id}/files`
5. Preview refreshes — the dev server restart and page reload IS the feedback
6. Dirty flag cleared → button disabled

TTL expiry is transparent — `RunnerletService` reschedules automatically on `409`.

---

## Runnerlet Lifecycle

```
POST /runnerlet ──► starting ──► ready ──► running
                                             │
                                   idle timeout → stopped
                                             │
                                   next request → starting → ready
                                             │
                                     hard TTL → GC'd (working dir deleted)
```

---

## runnerd Internal Architecture

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

**Scheduler** — stateless, pure placement. Receives `POST /runnerlet`, assigns to a frontend, returns `id` immediately.

**Runnerlet Frontend** — owns assigned runnerlets. Manages container lifecycle, working directory, PVC mounts, file updates, idle stop, TTL GC.

**Runnerlet** — the running container. No knowledge of scheduler or frontend.

### Assignment Store

```go
type AssignmentStore interface {
    Write(key string, info Assignment) error
    Read(key string) (Assignment, error)
    Delete(key string) error
}
```

In-memory (single machine) or Redis (multi-machine).

### Deployment

**Single machine:** Scheduler + Frontend in same process, in-memory store.

**Kubernetes:**
```
Scheduler (stateless Deployment)
     │
     └──► Redis (assignment store)
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 Frontend   Frontend   Frontend
 (DaemonSet, mounts containerd socket)
```

### Service Modules

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
```

`runnerd` lives in `eivo-sandbox`.

---

## Security

- Namespace isolation (PID, network, filesystem, IPC) per container
- cgroup resource limits from `substrate.resources`
- seccomp/AppArmor syscall filtering
- Default-deny egress; substrates may declare an allowlist
- overlayfs prevents substrate file modification by program code
- Working directory isolated per session — no cross-session access

---

## Observability

- Per-runnerlet metrics: starts, restarts, idle stops, request count, file updates
- Per-frontend metrics: active containers, memory usage, assignment count
- Structured logs per runnerlet id; output not logged

---

## Implementation Guide

### Flow 1: Learner Opens an Eivolet

```
1. UI loads the eivolet page
2. LearningSession.getOrSet(materialId, culture, sessionFetcher)
   - Checks sessionsCache (LRU, in-memory)
   - Cache miss → creates new LearningSession
   - init():
     - Loads eivolet from library
     - Refreshes workbook (bookmarks, eibits)
     - Creates ActivityCoordinator
3. LearningSession returns to the handler
4. Handler renders the page with initial content
```

No runnerlet started yet — lazy, on demand.

---

### Flow 0: Example Component Mounts

```
1. Component mounts with (materialId, userId)
2. Component calls GET /runnerlet?userId={userId}&materialId={joined}
3. Response:
   a. 404 (no session) → "Run" button enabled, preview empty
   b. { status: "starting" } → "Run" disabled, loading indicator shown
      - Poll GET /runnerlet/{id} every 2s until ready
      - On ready → hide loading, load iframe from procUrl
      - On timeout → show "Retry" button
   c. { status: "ready" } → "Run" disabled, iframe loads from procUrl
      - Start background poll every 30s (catch critical failures)
      - On failure detected → show "Retry" button
   d. { status: "stopped" } → "Run" button enabled, preview empty
4. procUrl is owned by runnerd — it returns appropriate response for any container state
   (loading page when starting, error page when failed, proxied service when ready)
```

### Flow 2: Learner Views a Service Exercise (React example)

```
1. UI renders the exercise component (editor + "Try" button + preview)
2. Handler calls LearningSession.renderContentObjects(templateName)
   - Renders the template
   - Extracts exercises → calls ActivityCoordinator.registerExercises(exercises)
3. UI shows editor with initial code (from ADL), preview is empty
4. "Try" button is enabled (no session yet)
```

---

### Flow 3: Learner Clicks "Try" (First Time)

```
1. UI calls handler: POST /api/exercise/{name}/try  { files: [...] }
2. Handler → LearningSession → ActivityCoordinator.try(eivoletId, userId, name, files)
3. ActivityCoordinator → RunnerletService.create(eivoletId, userId, name, domain, files)
4. RunnerletService builds model: runnerlet
5. RunnerletService → POST /runnerlet → runnerd
   - runnerd creates working directory in working PVC
   - Seeds with files from request
   - Mounts support PVCs (scaffold, deps) + working directory
   - Starts container in background
   - Returns { id, procUrl, stdioUrl }
6. RunnerletService stores (eivoletId, userId, name) → id mapping in Redis
7. Returns { procUrl, stdioUrl } to ActivityCoordinator → handler → UI
8. UI loads preview iframe from procUrl
9. UI polls GET /api/exercise/{name}/status until ready
10. "Try" button disabled (code unchanged)
```

---

### Flow 4: Learner Edits Code and Clicks "Run" Again

```
1. Learner edits code in editor → dirty flag set → "Run" button enabled
2. UI calls handler: POST /api/exercise/{name}/run  { files: [...] }
3. Handler → LearningSession → ActivityCoordinator.try(materialId, userId, files)
4. ActivityCoordinator → RunnerletService.patch(materialId, userId, { files })
5. RunnerletService resolves id from Redis
6. RunnerletService → PATCH /runnerlet/{id} → runnerd
   - runnerd writes files to working directory
   - runnerd signals process restart (SIGTERM → bootstrapper restarts process)
   - runnerd returns 204 (does not wait for process ready)
7. Client receives 204 → refreshes iframe src
8. iframe hits procUrl → runnerd proxy returns loading response while process restarts → service when ready
9. Dirty flag cleared → "Run" button disabled
```

---

### Flow 5: Session Expired — Learner Clicks "Try"

```
1. UI calls handler: POST /api/exercise/{name}/try  { files: [...] }
2. ActivityCoordinator → RunnerletService.updateFiles(...)
3. RunnerletService → PUT /runnerlet/{id}/files → runnerd → 409 Needs Schedule
4. RunnerletService transparently reschedules:
   - POST /runnerlet with current files → new { id, procUrl, stdioUrl }
   - Updates Redis mapping
5. Returns new { procUrl, stdioUrl } to UI
6. UI reloads preview from new procUrl — learner sees nothing unusual
```

---

### Flow 6: Learner Runs a Code Snippet (Python example)

```
1. UI renders snippet exercise (code editor + "Run" button + output panel)
2. Handler calls LearningSession.renderContentObjects(templateName)
   - ActivityCoordinator.registerExercises(exercises)
3. Learner writes code, clicks "Run"
4. UI calls handler: POST /api/exercise/{name}/run  { code, stdin }
5. Handler → LearningSession → ActivityCoordinator.run(name, code, stdin)
6. ActivityCoordinator → SnippetService.run(domain, code, stdin)
7. SnippetService builds model: snippet
8. SnippetService → POST /run → runnerd → NDJSON stream
9. Handler streams output to UI
10. UI shows stdout/stderr in output panel, exit code when done
```

---

### Flow 7: Learner Submits an Exercise Answer

```
1. Learner submits answer (fill-blank, multiple choice, etc.)
2. UI calls handler: POST /api/exercise/{name}/validate  { answer }
3. Handler → LearningSession → ActivityCoordinator.validateAnswer(name, answer)
4. ActivityCoordinator:
   - Looks up exercise by name
   - Selects validator by exercise.kind
   - Calls validator.validateAnswer(exercise, answer)
   - Records attempt in exerciseAttempts[name]
   - Returns result
5. Handler returns result to UI
6. UI shows feedback
```

---

### Flow 8: Learner Discusses an Exercise with EiBot

```
1. Learner opens EiBot discussion on an exercise
2. UI calls handler with exercise name
3. Handler → LearningSession.discussNamedComponentWithEibot('exercise', materialId, name)
4. LearningSession queries ActivityCoordinator.getAttempts(name) → exerciseAttempts
5. LearningSession → EiBotAssistant.createOrLoad(..., { exerciseAttempts })
   - EiBot receives exercise context including attempt history
6. Returns EiBot session data to UI
7. Learner chats with EiBot about the exercise, informed by their attempts
```

---

### Flow 9: Working Directory Cleanup (CronJob)

```
1. CronJob triggers on schedule (every 15 minutes)
2. Lists all subdirectories in working PVC: {eivoletId}/{userId}/{name}/
3. For each directory:
   a. Calls LearningSession API: is session (eivoletId, userId, name) active?
   b. LearningSession checks Redis for session key
   c. Key exists → skip (session still active)
   d. Key missing → delete the directory
4. Job completes, logs deleted directories
```

## Working PVC Cleanup Job

A Kubernetes CronJob (TypeScript) that periodically removes orphaned session directories from the working PVC. Same pattern as the challenge init job.

**When it runs:** On a schedule (e.g. every 15 minutes).

**What it does:**
1. Lists all subdirectories in the working PVC (`{eivoletId}/{userId}/{name}/`)
2. For each directory, calls the LearningSession API to check whether the session is still active in Redis
3. If the session no longer exists → deletes the directory

**Why needed:** runnerd deletes the working directory on GC (normal TTL expiry). The CronJob handles orphaned directories from edge cases — runnerd restarts, crashed frontends, or sessions that were never explicitly terminated.

**Implementation:** TypeScript, uses existing LearningSession infrastructure, no new services. Deployed as a `CronJob` in Kubernetes.

---

## Known Gaps

### OS-Level Substrates

Assembly, bootloader, kernel concepts requiring bare-metal access. Would require QEMU/KVM engine and machine-level substrate fields. Maps onto the `engine: mvm` path already planned.

### ActivityCoordinator Events

Future: `ActivityCoordinator` publishes events on exercise completion and validation for UserProfile (progress, stats) and Analytics. No event system currently in scope.

### ActivityCoordinator Events

Future: `ActivityCoordinator` publishes events on exercise completion and validation for UserProfile (progress, stats) and Analytics. No event system currently in scope.
