# Roadmap

Tracks two separate stacks: **big items** (strategic, mostly undesigned — need a scoping pass before any implementation starts) and **small items** (tactical, scoped or at least clearly bounded — pickable up directly). Nothing here is active work by default; an item moves off this list once it's actually started, and back on (with updated status) if it stalls.

## Target milestone

**eivo 0.1 alpha**, three ordered phases, not parallel workstreams: (1) "the programming stuff" — domain onboarding infrastructure, in progress, not closed; (2) onboarding; (3) content generation — explicitly the last phase before alpha.

## Big items

- **Eibot updates to use Skills, and to create examples and exercises.** The main item: update Eibot itself to consume Skills and to actually generate examples/exercises through them, porting everything already built in `eivo/prompts/crafts/scripted/template/template-writer.yaml` and `.../companion/companion-writer.yaml` (Service/Database shapes, console-file requirements, `<APITestCases>`, the `apiOnly` domain mark, every MUST/MUST NOT rule accumulated through real generation debugging) into that Skills-driven capability — replacing the current prompt-assembly mechanism. Not scoped: which craft content maps to which Skill(s), whether this is a 1:1 port or a redesign, how Eibot consumes a Skill differently than today's pipeline, sequencing relative to Eivotron's own build.
  - Subtask: **`SKILL.yaml` `appliesTo` field** — scope a skill *kind* to the object types it applies to (challenge/template/companion), alongside the existing `def.domain` field. Inert today (one universal content-authoring skill kind); pays off once a second, genuinely type-specific kind shows up here (e.g. grading-only knowledge for challenges vs. authoring knowledge for templates/companions). Cheap to add now.
  - **Eibot capability menu** — a catalog exposing which `progLang`/`platform`/database combinations are actually onboarded *and* paired end-to-end (not just declared in `domains.yaml`), so content generation stops requesting combinations that don't really exist (e.g. `express-react` picked for a plain-React example, or a hallucinated platform). Not scoped or designed.
- **Eivotron** — new repo/agent ("the boss," autonomous) that generates platform content going forward, same shape as `eivo-substrates`/Eiva (agent + Skills + Tools), replacing hand-maintained craft YAML as the primary content-generation mechanism. `prompts/crafts`'s own remaining job narrows to just exercises/challenges once this lands.
- **Skills as a general-purpose registry** — `skills-pvc` is meant to serve consumers beyond Eivotron/Eivot eventually, with multiple skill *kinds* per domain, not just today's one universal content-authoring skill.
- **Closed environments** — learners shouldn't be able to add runtime dependencies once an environment is running; everything baked in ahead of time via onboarding. Directional only — enforcement mechanism (network policy? read-only mounts? something else?) is a separate, not-yet-started project.
- **Versioning** — deliberately deferred during active development: every image/PVC is `:latest`, overwritten in place, no protection for an already-running session against a content swap mid-flight. Needs a real design before release.
- **Auth** — replace the current platform auth with an ingress layer backed by an OIDC provider. Named only — no detail yet on the current architecture, which OIDC provider, or a migration path.


## Small items

Status as of 2026-09-12:

- **"Gym," service-style substrates** — scoped, not big: use the already-generated, already-graded API exercises (`LLMMaterial`s carrying `<APITestCases>`) as Gym content directly, no per-exercise timing restriction — an overall constraint only, if any. Now scoped rather than open because the underlying exercise-generation and grading work (this file's first three "done" items above) is complete.
- Postgres / full-stack Next.js example content authoring — infra is done, content isn't — **pending**
- Shrink the Theia IDE image — **pending**
- Theia iframe error/retry mechanism (external detect+retry for load failures) — **pending**
- Full `CLAUDE.md` pass for log-narration and general verbosity — 4 spot-fixes done, not a full pass — **pending**
- **"Gym," React-style substrates** — undefined, still needs a scoping pass, but no longer big: described as "a stream of exercises with no solution and no time constraint," untimed/self-paced. No overall shape decided.
- Runnerlet status → frontend, to gate the iframe against a session that's mid-restart/erroring/torn down (today that surfaces as a confusing raw in-iframe failure instead of a clear disabled/reconnecting state). Architecture already settled: Redis as the status store, `runnerd` as sole writer (fast async write, no held connections), Eivo reads it, non-authoritative (falls back to querying `runnerd` directly if a record is missing). Incidentally also fixes a real gap — the reaper today deletes `lastError` detail on teardown, losing diagnostics. **Not started**, design decided.

**Not a task — explicitly out of scope:** grafting a test-results UI onto the `ProgrammingExampleWorkbench` surface. The workbench is companion/example-only and is never used for test cases; that's intentional and stays that way.

- ~~`eivo`'s grading driver (`waitReady`, walk `APITestCaseData` through `procUrl`, compare response)~~ — **done**
- ~~Comparator logic (structural JSON diff / exact string match)~~ — **done**
- ~~Validate the shipped `<APITestCases>` shape via real end-to-end generation~~ — **done**, including two real lessons and two real bugs found and fixed from that output
