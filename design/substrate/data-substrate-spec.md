# Data Substrate Spec

## Concept

A data substrate is any execution environment where the primary learning artifact is **data state** rather than code execution output. The learner interacts with a stateful system — reading, writing, transforming data — and the result of their work is the state of that system, not a program's stdout.

This is fundamentally different from code execution substrates (runnerd). The output is not a stream of bytes — it is structured data that needs to be visualized, browsed, and reasoned about.

## Concrete Example: Databases

Databases are the canonical data substrate. The learner writes SQL queries or uses a database API. The substrate is a running database engine (PostgreSQL, MySQL, SQLite, MongoDB, Redis, etc.) pre-loaded with seed data.

What makes a database substrate different from a code substrate:

- **Output is structured** — query results are tables, documents, key-value pairs — not stdout
- **State accumulates** — inserts, updates, deletes, schema changes persist within the session
- **Seeded starting state** — the database is pre-populated with data relevant to the exercise
- **Reset is a data operation** — resetting means restoring the data to the seeded state, not restarting a process
- **Schema browser** — the learner needs to see the database structure alongside the query interface
- **Multiple operations** — not a single file execution but an interactive session with many queries

## Data Substrate Categories

Any substrate where state accumulates and persists within a session:

| Category | Examples | State |
|---|---|---|
| Relational | PostgreSQL, MySQL, SQLite | Tables, rows, schema |
| Document | MongoDB | Collections, documents |
| Key-value | Redis | Keys, values, data structures |
| Graph | Neo4j | Nodes, edges, properties |
| Time-series | InfluxDB, TimescaleDB | Measurements, tags, fields |
| Search | Elasticsearch | Indices, documents, mappings |
| File system | Any | Files, directories, permissions |
| Message queue | Kafka, RabbitMQ | Topics, messages, consumer state |

## Common Properties

All data substrates share:

- **Seeded initial state** — starts from a known, exercise-specific data set
- **Stateful writes** — changes accumulate within a session, visible across operations
- **Ephemeral by design** — state is discarded at session end, next session starts fresh from seed
- **Abuse protection** — operation limits, storage limits, query complexity limits, resource caps
- **Structured output** — results visualized as tables, trees, graphs — not streamed as text
- **Different UX** — the learning interface is not a code editor + terminal

## Technical Model

### In-Memory Filesystem

Data substrates run entirely in memory. The database engine uses a tmpfs/ramfs mount for its data directory — no disk I/O, no persistent storage.

```
Container start → database initializes on tmpfs → loads seed data into memory → ready
Session end     → container stops → memory gone → clean state guaranteed
Reset           → restart container → memory gone → reload seed data → ready
```

Benefits:
- **Reset is instant** — clearing memory is faster than any disk operation
- **Ephemeral by nature** — memory is gone when the container stops, no leaks
- **Fast** — database operations on tmpfs are significantly faster than disk
- **No overlay complexity** — no overlayfs seeded layers needed
- **Abuse prevention is natural** — container memory limit caps total database size

The seed data is baked into the container image. On startup the database loads it into the in-memory data directory. All writes go to memory only.

### TTL

Two TTL values control the session lifecycle:

| TTL | Default | Behavior |
|---|---|---|
| Idle TTL | 10-15 min | Container destroyed after N minutes of no queries |
| Hard TTL | 45-60 min | Container destroyed regardless of activity |

Longer than code execution substrates — SQL exercises take more time between operations. Both values are configurable per substrate.

**Return after TTL expiry:** new session starts fresh from seed. In-memory initialization is fast — starting fresh is painless. UX communicates clearly: "Your session expired — starting fresh."

### Abuse Prevention

Handled at the runnerd level via container memory limits — no extra quota system needed:

- **Storage abuse** — container `resources.memoryMiB` caps total database size. If the learner fills memory, the database rejects writes naturally.
- **Query timeout** — runnerd enforces a per-operation timeout at the container level.

### Session Lifecycle

Resolved by the in-memory model:

- Always ephemeral — memory is gone when the container stops
- Starting fresh is fast — seed data loads in milliseconds
- **Session start** — on first query, container starts on demand
- **Session end** — idle TTL or hard TTL, whichever comes first
- **Reset** — explicit reset button restarts the container, reloads seed data instantly
- **Multi-tab** — each tab gets its own independent session

## Open Questions

### UX Lifecycle Communication
- How does the learner know their state is ephemeral?
- How is TTL expiry communicated without frustrating the learner mid-exercise?
- What does the "session expired" recovery experience look like?

### The Database UX Component

A dedicated component — not `ProjectWorkbench`. Needs:
- Query editor (SQL, MongoDB query language, Redis commands, etc.)
- Results visualization (table for SQL, tree for documents, etc.)
- Schema/structure browser
- Session state indicator
- Reset affordance

## Example: PostgreSQL Substrate

A concrete `model: substrate` for a PostgreSQL data substrate:

```yaml
model: substrate
metadata:
  name: postgresql
  namespace: eivo
def:
  domain:
    progLang: sql
    platform: postgresql
  engine: container
  image: reg.jobico.local/runnerd-postgresql:latest
  resources:
    memoryMiB: 256      # covers both PostgreSQL process and in-memory data
    cpuMillis: 250
  env:
    POSTGRES_DB: exercise
    POSTGRES_USER: learner
    POSTGRES_PASSWORD: learner
    PGDATA: /dev/shm/pgdata    # tmpfs — all data in memory
  sourceDir: /exercise         # where seed SQL files are mounted
  filesystem:
    properties:
      type: tmpfs              # in-memory filesystem
      sizeLimit: 200Mi         # bounded within container memoryMiB
    structure:
      - path: /pgdata          # PostgreSQL data directory — in memory
      - path: /exercise
        readOnly: true         # seed SQL files — read-only bind mount
  endpoints:
    - port: 5432
      healthPath: ""           # TCP check only — no HTTP health path
  commands:
    - type: serve
      run: docker-entrypoint.sh postgres
  ttl:
    idleSeconds: 600           # 10 minutes idle
    hardSeconds: 2700          # 45 minutes hard limit
```

Key decisions reflected in this substrate:

**`PGDATA: /dev/shm/pgdata`** — `/dev/shm` is a tmpfs mount present in every Linux container. PostgreSQL writes its data directory here — all data lives in memory, nothing touches disk.

**`memoryMiB: 256`** — covers PostgreSQL's shared_buffers + working memory + the in-memory data. The container memory limit IS the storage limit. If a learner's data fills the 256MB, PostgreSQL naturally rejects further writes.

**`sourceDir: /exercise`** — seed SQL files (schema + data) are mounted here read-only. The PostgreSQL entrypoint runs these files on startup to initialize the database.

**`endpoints.port: 5432`** — the frontend proxies this port. The client connects via the `procUrl` which proxies to port 5432. No HTTP health path — health is a TCP connection check.

**`ttl`** — longer than code substrates. 10 minutes idle, 45 minutes hard. A learner writing complex queries needs more time between operations.

The container image is built from `postgres:alpine` with the bootstrapper entrypoint that:
1. Starts PostgreSQL with `PGDATA` on `/dev/shm`
2. Runs seed files from `/exercise` on first start
3. Handles SIGTERM gracefully

## Relationship to runnerd

Data substrates use the same container lifecycle as runnerd (`model: container`, scheduler/frontend/runnerlet) but the interaction model is different:

- The runnerlet is a long-running database process, not a dev server
- The `procUrl` proxies the database protocol or a thin HTTP API over it
- The `stdioUrl` shows database logs
- Reset is a new operation not present in runnerd — restores the overlay to the seeded state

## Status

**Design not started.** This spec is a stub capturing the open questions and known constraints. A dedicated design session is needed before implementation can begin. The primary blocker is the session lifecycle decision.
