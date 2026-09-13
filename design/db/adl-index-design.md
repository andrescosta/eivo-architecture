# ADL Index — Technical Design

## Foundations

### The Library Storage is a Tree

The Library storage is a single tree. Every ADL object published to the platform is a node in that tree:

```
namespace (e.g. lingv)
└── eivolet (e.g. french_a1)
    └── aggregate (e.g. frenchA1Syllabus)
        └── aggregate (e.g. unit1_greetings)
            ├── template
            ├── llmmaterial
            └── spaceenvironment
```

There are no separate trees per Eivolet — it is one tree. The namespace is the root. The filesystem layout in `ADLFileStorage` is a direct materialization of this tree on disk. Every node has an absolute address from the namespace root down.

### Two Orthogonal Concerns

The design separates two completely independent concerns:

**`path` — filesystem address.**
The `path` array is the absolute address of a node within the Library tree — e.g. `['lingv', 'french_a1', 'greetings', 'lesson_1']`. Given any node returned from the index, `path` tells you exactly where to find the full document on disk. No secondary lookup, no resolution step — directly usable as the address argument to `ADLStorage.get()`.

**Index — grouping engine.**
Because every node in the entire Library tree is indexed with its document projection, any experience can build any grouping it needs — by model, by invariant, by kind, by tag, by culture, or any combination — scoped to any subtree. The index does not prescribe how content is grouped. Experiences own that entirely through their query. Lingv groups Eivolets by `domain.subject`. Another experience could group by `cultures`, by `kinds`, or by any other dimension. The index supports all of them equally.

---

## Overview

The ADL Index is a queryable index of ADL documents that enables experiences to discover content without scanning the filesystem. It is a separate concern from the content store (`ADLStorage`) — the content store owns persistence of the full ADL tree, the index owns discoverability.

`ADLDocumentDB` coordinates both: it writes to the content store and the index on publish, and removes from both on delete. Experiences query the index to get matching nodes as a tree, then load full content from the content store as needed.

---

## Index Document

Every indexed node is a projection of its ADL document — enough to query, nothing more. Labels carry only `culture` and `tags` — no `title` or `overview`. Content is always loaded from storage when needed for rendering.

```yaml
model: ...
metadata:
  name: ...
  namespace: ...
  kinds: [...]
  extra: {...}
tags: [...]
labels:
  - culture: ...
    tags: [...]
def:
  invariants:               # Eivolet only
    cultures: [...]
    domain:
      subject: ...
      properties: {...}
```

`def.invariants` is only present on `Eivolet` nodes. All other nodes omit it. `objective` and `audience` are intentionally excluded — free text, not useful as structural query fields.

---

## Spec Serialization

`toIndexDocument()` is added to the `Spec` hierarchy. Each level adds its own fields — same pattern as `toYaml()`.

```typescript
// Base — model + metadata fields
export abstract class Spec {
  public toIndexDocument(): AnyData {
    return {
      model: this.model,
      metadata: {
        name: this.metadata.name,
        namespace: this.metadata.namespace,
        kinds: this.metadata.kinds,
        extra: this.metadata.extra,
      },
    };
  }
}

// Adds tags and labels (culture + tags only, no title/overview)
export abstract class SpecLabeled extends Spec {
  public toIndexDocument(): AnyData {
    return {
      ...super.toIndexDocument(),
      tags: this.tags,
      labels: this.labels?.map((l) => ({
        culture: l.culture,
        tags: l.tags,
      })),
    };
  }
}

// Eivolet adds invariants
export class Eivolet extends SpecLabeled {
  public toIndexDocument(): AnyData {
    return {
      ...super.toIndexDocument(),
      def: {
        invariants: this.def?.invariants
          ? {
              cultures: this.def.invariants.cultures,
              domain: this.def.invariants.domain,
            }
          : undefined,
      },
    };
  }
}
```

---

## Address Types

Shared types used across `ADLStorage`, `ADLIndex`, and `ADLDocumentDB`.

```typescript
export interface NodeAddress {
  namespace: string;
  parents?: string[];
}

export type LocatedSpec   = NodeAddress & { document: Spec };
export type LocatedParent = NodeAddress & { document: Parentable<Spec> & Spec };
export type LocatedNode   = NodeAddress & { name: string };
```

---

## ADLStorage Interface

Pure persistence — no filtering, no querying. `ADLIndex` owns all discoverability.

```typescript
export interface ADLStorage {
  initializeNamespace(namespace: string): Promise<void>;

  put(located: LocatedSpec): Promise<void>;         // always upsert
  remove(located: LocatedNode): Promise<void>;
  exists(located: LocatedNode): Promise<boolean>;
  get(located: LocatedNode): Promise<ADLContentModels | undefined>;
  getWithIndex(
    index: IndexDocument,
    strategy?: LoadingStrategy,
  ): Promise<(Parentable<ADLContentModels> & ADLContentModels) | undefined>;

  putAttachment(attachment: { owner: LocatedNode; document: Spec }): Promise<void>;
  getAttachment(attachment: { owner: LocatedNode; model: string }): Promise<ADLContentModels | undefined>;
  removeAttachment(attachment: { owner: LocatedNode; model: string }): Promise<void>;
}

export type LoadingStrategy = 'full' | 'metadata';
```

### Filesystem Layout

```
basePath/
└── lingv/                              ← namespace directory
    ├── lingv.namespace.yaml            ← namespace node
    ├── french_a1/                      ← eivolet directory (container)
    │   ├── french_a1.eivolet.yaml
    │   ├── _attachments/               ← attachments (e.g. EivoletDef)
    │   │   └── trainning_french.eivoletdef.yaml
    │   └── greetings/                  ← aggregate directory (container)
    │       ├── greetings.aggregate.yaml
    │       └── lesson_1/               ← aggregate directory
    │           ├── lesson_1.aggregate.yaml
    │           ├── my_template.template.yaml    ← leaf — file in parent dir
    │           └── my_exercise.llmmaterial.yaml ← leaf — file in parent dir
```

**Key conventions:**
- Containers (`eivolet`, `aggregate`, `namespace`) — own directory: `name/name.model.yaml`
- Leaves (`template`, `llmmaterial`, `spaceenvironment`) — file directly in parent: `name.model.yaml`
- Attachments — `_attachments/` directory under owner container
- `_attachments` is a reserved segment — throws if used as a node name
- Leaf `put` validates no file with same name but different model exists — would cause `get` ambiguity

---

## ADLIndex Interface

```typescript
export interface ADLIndex {
  index(located: LocatedSpec): Promise<void>;
  index(located: LocatedParent): Promise<void>;
  remove(located: LocatedNode): Promise<void>;
  query(address: NodeAddress, query?: IndexQuery): Promise<IndexDocument[]>;
}
```

### IndexQuery — Query By Example

A partial document that mirrors the `toIndexDocument()` projection. Fields present are matched — absent fields ignored. New fields are queryable without interface changes.

```typescript
export interface IndexQueryFieldMatch {
  value: unknown;
  matchIfAbsent?: boolean;  // match if field absent OR equals value
}

export interface IndexQuery {
  document?: {
    model?: string;
    metadata?: {
      kinds?: string[];
      extra?: Record<string, IndexQueryFieldMatch | unknown>;
    };
    tags?: string[];
    labels?: { culture?: string; tags?: string[] }[];
    def?: {
      invariants?: {
        cultures?: string[];
        domain?: {
          subject?: string;
          properties?: Record<string, string>;
        };
      };
    };
  };
  depthLimit?: number;
  limit?: number;
  offset?: number;
}
```

### IndexDocument

The index always returns a tree. Every node carries its path (directly usable with `ADLStorage.get()`), its projection, and its children. Ancestors of matching nodes are always included — no orphaned nodes.

```typescript
export interface IndexDocument {
  path: string[];              // full path from namespace root
  document: Record<string, unknown>;
  children: IndexDocument[];
}
```

---

## Postgres Implementation — PostgresADLIndex

### Table

Uses PostgreSQL `ltree` extension for hierarchical path queries. Node names are encoded using `__HEX__` to satisfy ltree's `[A-Za-z0-9_]` label constraint. Double underscore `__` is reserved and forbidden in raw node names.

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE IF NOT EXISTS adl_nodes (
  id          uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
  path        ltree       NOT NULL,
  document    jsonb       NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE adl_nodes ADD CONSTRAINT adl_nodes_path_unique UNIQUE (path);

CREATE INDEX IF NOT EXISTS adl_nodes_path_gist     ON adl_nodes USING gist(path);
CREATE INDEX IF NOT EXISTS adl_nodes_path_btree    ON adl_nodes USING btree(path);
CREATE INDEX IF NOT EXISTS adl_nodes_document_gin  ON adl_nodes USING gin(document jsonb_path_ops);
```

### ltree Encoding

```typescript
function encodeLabel(label: string): string {
  return label.replace(/[^A-Za-z0-9]/g,
    c => `__${c.charCodeAt(0).toString(16).toUpperCase()}__`,
  );
}
// french_a1 → french__5F__a1
// french-a1 → french__2D__a1
```

### Query Pattern

The query uses a CTE to find matching nodes, then joins to get the full ancestor chain — driven by children, not by trying to find parents:

```sql
WITH nodes_filtered AS (
  SELECT path FROM adl_nodes
  WHERE path <@ $scope           -- or path ~ $lquery for depthLimit
  AND document->>'model' = $model
  AND <other filter conditions>
)
SELECT DISTINCT o.path, o.document
FROM adl_nodes o
JOIN nodes_filtered nf ON o.path @> nf.path  -- o is ancestor of nf
WHERE o.path <@ $scope                        -- scope the ancestors
ORDER BY o.path
```

**ltree operators:**
- `<@` — path is descendant of (or equal to) scope
- `@>` — path is ancestor of (or equal to) node
- `~`  — path matches lquery pattern (used for depthLimit)

**Query examples:**

```sql
-- All Eivolets in namespace by subject
WITH nodes_filtered AS (
  SELECT path FROM adl_nodes
  WHERE path <@ 'lingv'
  AND document->>'model' = 'eivolet'
  AND document->'def'->'invariants'->'domain'->>'subject' = 'language'
)
SELECT DISTINCT o.path, o.document FROM adl_nodes o
JOIN nodes_filtered nf ON o.path @> nf.path
WHERE o.path <@ 'lingv'
ORDER BY o.path;

-- All templates of an Eivolet (with full ancestor chain)
WITH nodes_filtered AS (
  SELECT path FROM adl_nodes
  WHERE path <@ 'lingv.french__5F__a1'
  AND document->>'model' = 'template'
)
SELECT DISTINCT o.path, o.document FROM adl_nodes o
JOIN nodes_filtered nf ON o.path @> nf.path
WHERE o.path <@ 'lingv.french__5F__a1'
ORDER BY o.path;

-- Direct children only (depthLimit = 1)
WITH nodes_filtered AS (
  SELECT path FROM adl_nodes
  WHERE path ~ 'lingv.french__5F__a1.*{1,1}'
  AND document->>'model' = 'aggregate'
)
SELECT DISTINCT o.path, o.document FROM adl_nodes o
JOIN nodes_filtered nf ON o.path @> nf.path
WHERE o.path <@ 'lingv.french__5F__a1'
ORDER BY o.path;
```

---

## ADLDocumentDB

Concrete coordinator class — wires `ADLStorage` and `ADLIndex` together. The only entry point consumers depend on.

```typescript
export class ADLDocumentDB {
  constructor(
    private readonly storage: ADLStorage,
    private readonly index: ADLIndex,
  ) {}

  async initializeNamespace(namespace: string): Promise<void>;  // idempotent

  async put(located: LocatedSpec): Promise<void>;
  async put(located: LocatedParent): Promise<void>;
  async remove(located: LocatedNode): Promise<void>;
  async exists(located: LocatedNode): Promise<boolean>;
  async get(located: LocatedNode): Promise<ADLContentModels | undefined>;
  async find(address: NodeAddress, query?: IndexQuery, strategy?: LoadingStrategy): Promise<ADLContentModels[]>;

  async putAttachment(attachment: { owner: LocatedNode; document: Spec }): Promise<void>;
  async getAttachment(attachment: { owner: LocatedNode; model: string }): Promise<ADLContentModels | undefined>;
  async removeAttachment(attachment: { owner: LocatedNode; model: string }): Promise<void>;
}
```

**`initializeNamespace`** — idempotent. Checks existence before writing. Called by the Library service before any Eivolet is stored. Writes namespace to both storage and index.

**`put`** — always upsert. Transparent to the caller — detects `Parentable` with children and recursively stores the full tree, then indexes. Error management: storage writes first; if index fails, compensates by removing from storage; if compensation fails, logs and rethrows.

**`find`** — queries index, loads from storage. `IndexDocument` never leaks to the caller. Returns assembled `ADLContentModels[]`.

**`remove`** — removes from storage first, then index. If index removal fails, logs (index may be stale but is rebuildable) and rethrows.

---

## EivoletSessionDB

Purpose-built class for the EiBot authoring session. Not an incarnation of `ADLDocumentDB` — a different tool with a different purpose, lifetime, and operational surface.

### Key Strategy

Three Redis keys per node — no virtual filesystem paths, no path parsing:

```
eivsession:<sessionId>:node:<n>      → spec YAML
eivsession:<sessionId>:parent:<n>    → parent name (empty string = root)
eivsession:<sessionId>:children:<n>  → SET of child names
```

Everything addressed by name. `getEivolet` reconstructs the full tree by recursively loading from the `children:` SET. `getNode` is O(1) direct key lookup.

### Two Operation Modes

**Bulk load — `saveEivolet`**
Atomic pipeline, no validation, trusted input. Stores the full Eivolet tree in one `pipeline.exec()`. `EivoletDef` stored as sibling — not in the `children:` SET, so `getEivolet` tree reconstruction skips it naturally. `collectNodes` recursively collects all children into the pipeline.

**Incremental — EiBot-driven**
Single node operations with validation. `storeAggregate`, `storeContentGroup`, `storeSpec` validate existence, topology, and model conflicts before writing. `writeNode` is the single write primitive — does a `readNode` check first to catch model conflicts.

### Interface

```typescript
class EivoletSessionDB {
  // Bulk load
  saveEivolet(eivolet: Eivolet, def?: EivoletDef): Promise<void>;

  // Incremental
  storeAggregate(parentName: string, spec: Aggregate, override?: boolean, invariants?: EivoletInvariants): Promise<void>;
  storeContentGroup(parentName: string, spec: ContentGroup, override?: boolean, invariants?: EivoletInvariants): Promise<void>;
  storeSpec(parentName: string, spec: Spec, override?: boolean, invariants?: EivoletInvariants): Promise<void>;

  // Read
  getEivolet(name: string): Promise<Eivolet | undefined>;
  getNode(name: string): Promise<ADLContentModels | undefined>;
  exists(name: string): Promise<boolean>;

  // Update
  updateNodeMetadata(spec: Spec): Promise<void>;

  // Delete
  deleteNode(parentName: string, spec: Spec): Promise<void>;
  clearAggregateChildren(name: string): Promise<void>;
  clearContentGroupChildren(name: string): Promise<void>;
}
```

### Key Notes

- `EivoletDef` is a sibling of the Eivolet — stored with `parent:defName = eivoletName` but never added to `children:eivoletName`. Loaded explicitly via `getNode(defName)`, not via tree reconstruction.
- `writeNode` validates model conflicts — a node cannot be overwritten with a different model type.
- `collectNodes` is synchronous — builds the full pipeline in one pass before `exec()`.
- TTL applied to every key on every write including `expire` on `children:` SET.
- Topology validation in incremental methods uses `resolveTopologyValidator(invariants)` with sibling names from `children:parentName`.
