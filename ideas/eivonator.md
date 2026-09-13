https://openrouter.ai/docs/guides/features/plugins/fusion

# Eivonator

## What is it?

Eivonator is an autonomous agent layer that monitors external and internal signals to detect emerging learning demand and content quality issues, driving the full Eivolet production pipeline without human intervention. It is the platform's self-directed editorial intelligence.

## Objective

Keep the Eivo content library ahead of the market and continuously improving — publishing Eivolets on emerging topics before demand peaks, and updating existing ones based on real learner performance data.

## How it works

Eivonator runs as a continuous agent loop across four phases:

### 1. External Signal Collection

Monitors external sources for emerging topic signals:

- **Developer communities** — HackerNews, Reddit (r/programming, r/rust, r/webdev, etc.), dev.to, lobste.rs
- **Job market** — Job postings on LinkedIn, Indeed, and similar platforms. Frequency of a technology in job descriptions is a leading indicator of market demand.
- **Ecosystem activity** — GitHub trending repositories, release announcements, major framework updates
- **Search trends** — Rising queries in developer-focused search contexts

### 2. Internal Signal Collection

Monitors platform data for content quality and engagement signals:

- **Learner engagement** — Time on content, completion rates, return visits per Eivolet and per template
- **Exercise results** — Pass/fail rates, common wrong answers, retry patterns per llmmaterial
- **Challenge performance** — Submission quality scores, evaluation feedback patterns, abandonment rates
- **Competition data** — Head-to-head results, score distributions, skill gap indicators per domain
- **Learner input** — Explicit feedback, difficulty ratings, topic requests, confusion signals from assistance conversations
- **Navigation patterns** — Drop-off points, skipped sections, time distribution across content nodes
- **Spaced repetition signals** — Recall rates over time per exercise type and topic

### 3. Decision

An orchestrating agent evaluates the collected signals and decides:

**New Eivolet** — sustained external signal across multiple sources for a topic not yet covered in the library.

**Update existing Eivolet** — internal signals indicate quality issues:
- High failure rates on specific exercises → regenerate that llmmaterial
- Low engagement on a template → regenerate with different structure or depth
- Learner feedback indicates gaps → add content nodes or expand coverage
- Challenge abandonment → simplify or reframe the challenge prompt

**New version** — major ecosystem change (framework update, language release) warrants a full revision of an existing Eivolet rather than targeted updates.

**Retire** — sustained low engagement with no recovery signal → archive and stop serving.

In all cases the agent first queries the ADL Index via Library to understand what exists before deciding.

### 4. Production

On a positive decision, Eivonator drives the full generation pipeline:

- Drafts or updates the Eivolet Definition — invariants, modelers, crafts, model configuration
- Invokes Anvil to execute the definition and generate content
- Publishes the completed Eivolet to Library
- Tags the Eivolet with origin metadata (`source: eivonator`, signal provenance, trigger type)

## Platform Integration

Eivonator is a consumer of existing platform primitives — it introduces no new infrastructure:

- **ADL Index / Library** — content discovery, publish, and versioning
- **Anvil** — generation pipeline execution
- **Promptbook** — craft and role resolution
- **EiQueue** — token usage accounting per generation run
- **Editorial API** — Eivolet Definition authoring
- **Assistance Services** — source of learner confusion and feedback signals
- **Social / Gaming Services** — source of engagement, competition, and performance data

## Constraints

- Eivonator never publishes without a validation pass — generated Eivolets are schema-validated before Library ingestion
- Human override is always possible — any Eivonator-generated Eivolet can be flagged for editorial review
- Token budgets are enforced per run via EiQueue to prevent runaway generation costs
- Internal signal decisions require a minimum observation window before acting — single session anomalies do not trigger regeneration