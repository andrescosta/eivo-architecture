# model: eibooklet (example kind) — Spec

## Concept

An **eibooklet** is a named, addressable, tree-positioned artifact attached to the
Eivolet material that is not part of the reading path's content — you jump to
it, you don't read through it. Its meaning comes from where it sits in the tree:
an eibooklet under a page relates to that page, under a section to the section,
under the Eivolet root to the whole material (scope-by-position).

The **programming example** is the first concrete eibooklet kind. It is a
multi-file code artifact that demonstrates one or more concepts from the Eivolet.

## Tree Position

`model: eibooklet` is a content node alongside `section` and `page`. Position
expresses the author's intent — no platform constraint on placement:

```
eivolet/
├── aggregate (unit 1)
│   ├── template
│   ├── llmmaterial
│   └── eibooklet [example, programming, runnable]   ← scoped to this unit
├── aggregate (unit 2)
│   └── template
└── eibooklet [example, programming, runnable]       ← synthesizes the whole Eivolet
```

## kinds

`metadata.kinds` is a **set** of facets, cumulative from general to specific:

- `[example]` — an addressable example eibooklet
- `[example, programming]` — a programming example
- `[example, programming, runnable]` — a programming example with live execution

Each added kind narrows the eibooklet; they compose rather than replace. `kinds`
selects the renderer and drives other consumers (Eibooklets TOC groups on `example`;
EiBot discussion flows branch on `programming`). Each consumer asks "has kind X?"
independent of the rest.

## Storage

Eibooklet content is stored via **contentgroup** — the same mechanism as space
environments (eibooklets are not leaf templates). This is a persistence concern,
invisible above the storage layer.

## ADL Eibooklet

The eibooklet is a thin shell — `kinds`, `labels`, and two named template fields.
All example content lives inside `<ProgrammingExample>` in `def.object`.

```yaml
model: eibooklet
metadata:
  name: react-hooks-examples
  namespace: eivo
  kinds: [example, programming, runnable]
labels:
  - culture: en-US
    title: React Hooks Examples
    overview: Demonstrates common React hooks through isolated, runnable components.
  - culture: fr
    title: Exemples React Hooks
    overview: Démontre les hooks React courants à travers des composants isolés et exécutables.
def:
  summary:
    format: markdown
    content:
      - culture: en-US
        body: >-
          Explore the hooks used in this section hands-on. <EibookletLink/>
      - culture: fr
        body: >-
          Explorez les hooks utilisés dans cette section. <EibookletLink/>
  object:
    format: markdown
    content:
      - culture: en-US
        body: |-
          <ProgrammingExample lang="typescript" platform="react" name="abc123">
            <Narrative>
              This example shows how useState drives re-renders. Change the
              initial value or add a reset button and watch how the component
              responds.
            </Narrative>
            <File path="src/App-Counter.tsx" role="scaffold" runnable title="Counter" description="Runs the Counter component.">
              {`import { Counter } from './Counter';
export default function App() { return <Counter />; }`}
            </File>
            <File path="src/Counter.tsx" role="subject" title="Counter Component" description="A counter using useState. Notice how state updates trigger a re-render.">
              {`import { useState } from 'react';
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}`}
            </File>
            <File path="src/Toggle.tsx" role="context" title="Toggle Component" description="A toggle switch built with useState.">
              {`import { useState } from 'react';
export function Toggle() {
  const [on, setOn] = useState(false);
  return <button onClick={() => setOn(v => !v)}>{on ? 'On' : 'Off'}</button>;
}`}
            </File>
          </ProgrammingExample>
      - culture: fr
        body: |-
          <ProgrammingExample lang="typescript" platform="react" name="abc123">
            <Narrative>
              Cet exemple montre comment useState déclenche les re-rendus.
            </Narrative>
            <File path="src/App-Counter.tsx" role="scaffold" runnable title="Compteur" description="Exécute le composant Compteur.">
              {`import { Counter } from './Counter';
export default function App() { return <Counter />; }`}
            </File>
            <File path="src/Counter.tsx" role="subject" title="Composant Compteur" description="Un compteur avec useState. Observez comment les mises à jour d'état déclenchent un re-rendu.">
              {`...`}
            </File>
          </ProgrammingExample>
```

## def.summary

The eibooklet's in-flow presence in the reading stream. Ordinary MDX containing
**`<EibookletLink/>`** — a real component bound at render time to the object's
full id (`buildBoundedComponents(materialId)` pattern, scoped to this one
component). On click, calls the open-eibooklet action with the bound id: a
**view switch** that takes the full surface. The authored body is never mutated.

## def.object

The full example content. A single `<ProgrammingExample>` component per
culture. The component carries everything — domain, files, roles, narrative.
Localization is at the template level: each culture entry has its own
`<ProgrammingExample>` body with translated `<Narrative>`, `title`, and
`description` attributes. File content (`path`, `role`, code) is identical
across cultures.

## Two Tiers of Example

`example` (a kind) appears at two scales, distinguished by location:

- **Standalone eibooklet** — a `model: eibooklet` node in the tree (this spec).
  Positional: reached via its link-template in the reading flow and from the
  Eibooklets TOC.
- **Inline example** — a `<ProgrammingExample>` component embedded directly in
  a template body, beside exercises and diagrams. In-template, single-culture,
  no separate ADL eibooklet.

## Inline Example — `<ProgrammingExample>`

Authored directly in the template MDX body. Follows the same component pattern
as `<ProgrammingExercise>`. Culture is the template's culture — no localization
array, attributes are plain strings.

```
<ProgrammingExample lang="typescript" platform="react" name="<name>">
  <Narrative>
    This example shows how useState drives re-renders. Change the initial
    value or add a reset button and watch how the component responds.
  </Narrative>
  <File path="src/App.tsx" role="scaffold" runnable>
    {`import { Counter } from './Counter';
export default function App() { return <Counter />; }`}
  </File>
  <File path="src/Counter.tsx" role="subject" title="Counter Component" description="A counter using useState. Notice how state updates trigger a re-render.">
    {`import { useState } from 'react';
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}`}
  </File>
</ProgrammingExample>
```

### Attributes

| Attribute | Required | Notes |
|---|---|---|
| `lang` | yes | `progLang` — derived from template context or `get-programming-values` |
| `platform` | no | optional framework qualifier |
| `name` | yes | unique randomly generated identifier |

### Children

**`<Narrative>`** — optional. A brief framing paragraph introducing what the
example shows and what the learner should look at or try. Plain text. Omit for
short, self-evident examples. Essential for multi-file or multi-entry-point
examples where orientation helps.

**`<File>`** — one per file, in order. Attributes:

| Attribute | Required | Applies to |
|---|---|---|
| `path` | yes | all |
| `role` | yes | all (`scaffold`, `subject`, `context`) |
| `runnable` | no | `scaffold` only — boolean attribute |
| `title` | no | `subject`, `context`, runnable `scaffold` |
| `description` | no | `subject`, `context`, runnable `scaffold` |

Same role semantics as the standalone eibooklet. Non-runnable `scaffold` files
carry no `title` or `description` — they are never surfaced to the learner.

## Reader Integration

### In the reading stream

An `eibooklet` node renders its link-template like a page — it has its own slot in
the skeleton-first lazy fill stream. The heavy content lives in the contentgroup
and renders only when the eibooklet view is active.

### Eibooklets TOC

Navigation surface distinct from Contents and Workbook:

- **Contents** — reading path (`section` + `page`, excludes `eibooklet`)
- **Workbook** — learner's layer (eibits, notes, highlights, discussions)
- **Eibooklets TOC** — addressable eibooklets attached to the material

Eibooklets TOC groups positional eibooklets by tree position (under their parent
section or at Eivolet root), showing title + overview. In-template eibooklets
(exercises, diagrams) are a future addition to this surface.

### Rendering

`kinds` selects the renderer. `[example, programming, runnable]` renders through
the runnable programming-example renderer — the **read/run mode** of the
programming-exercise component (multi-file + run, no solve, no grade). Role tags
drive file foregrounding: `subject` is focal, `context` is supporting.

Composes with size presets: large when opened as full-surface view, compact when
referenced in EiBot chat.

### EiBot

Discussing an eibooklet is a natural EiBot starting point. Files-with-roles become
context; roles tell EiBot what is focal (`subject`) vs incidental (`context`).

## Status

ADL eibooklet design settled. Open (per reader spec §10 and §12):
- `eibooklet` node kind end-to-end: contentgroup storage, three-kind tree in reader,
  Eibooklets TOC implementation
- Read/run example renderer
- Example discussion `kind` for EiBot
- Non-programming example domains (unexplored)
- Inline example rendering (in-place in page body vs full-surface on click)
