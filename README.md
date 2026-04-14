# adj-manifest specification

A portable, append-only journal format for agent deliberations — enabling calibration scoring, audit, and cross-deliberation learning.

## The Problem

Agents make decisions together (via [ADP](https://adp-manifest.dev) or otherwise), but there is no standard for how those decisions are recorded, how outcomes are tracked, or how the gap between predicted and actual results feeds back into future decision-making.

## The Solution

ADJ defines five entry types that capture the full lifecycle of a deliberation:

```json
{ "entry_type": "deliberation_opened", "deliberation_id": "dlb_01HMXJ3E9R", "participants": ["did:adp:test-runner-v2", "did:adp:security-scanner-v3"] }
{ "entry_type": "proposal_emitted", "deliberation_id": "dlb_01HMXJ3E9R", "proposal": { "vote": "approve", "confidence": 0.86, "..." : "..." } }
{ "entry_type": "round_event", "deliberation_id": "dlb_01HMXJ3E9R", "event_kind": "falsification_evidence", "..." : "..." }
{ "entry_type": "deliberation_closed", "deliberation_id": "dlb_01HMXJ3E9R", "termination": "converged" }
{ "entry_type": "outcome_observed", "deliberation_id": "dlb_01HMXJ3E9R", "success": true, "observed_at": "2026-04-14T09:12:00Z" }
```

The split between `deliberation_closed` and `outcome_observed` is the load-bearing design choice: decisions and results are recorded separately and asynchronously. Calibration scoring joins them.

## Contents

| File | Description |
|------|-------------|
| [spec.md](spec.md) | Full specification (v0) |
| [schema/v0.json](schema/v0.json) | JSON Schema for journal entry validation |
| [examples/](examples/) | Reference journal entries |
| [ci/](ci/) | CI workflow templates for entry validation |

## How It Composes

```
mcp-manifest   declares what an agent can do
ADP            declares how agents agree on doing it together
ADJ            declares how those agreements are recorded and scored
```

ADP produces deliberation events. ADJ stores them. When outcomes arrive, ADJ computes calibration scores. ADP reads those scores to weight future votes. The loop closes.

## Quick Start

### For journal implementers

1. Choose a substrate (JSONL files, SQLite, Postgres, event log)
2. Write conformant entries per the [entry types](spec.md#3-entry-types)
3. Serve the [query contract](spec.md#7-query-contract) for consumers

### For ADP agents

1. Write deliberation events to a journal as they occur
2. Implement CalibrationSource by calling `getCalibration(agent_id, domain)` on the journal
3. Write `outcome_observed` entries when results are available — manually, or via an OutcomePlugin that watches CI systems, monitoring platforms, or other external sources automatically

## Examples

- [deliberation-opened.json](examples/deliberation-opened.json) — start of a PR merge deliberation
- [outcome-observed.json](examples/outcome-observed.json) — binary outcome recorded 3 days later
- [deliberation-closed.json](examples/deliberation-closed.json) — converged deliberation with full tally

## Compliance Levels

| Level | Name | What it does |
|-------|------|-------------|
| L1 | Writer | Writes conformant entries. A directory of JSONL files qualifies. |
| L2 | Query Provider | Serves the query contract (getCalibration, getDeliberation, etc.). |
| L3 | Scoring Engine | Computes calibration and condition quality scores. The full loop. |

PostMortem-style JSONL session journals are a natural Level 1 implementation.

## Reference Implementation

A C# reference library implementing journal types, calibration scoring, and the query contract is at [adj-ref-lib-csharp](https://git.marketally.com/ai-manifests/adj-ref-lib-csharp). TypeScript and Python ports live at [adj-ref-lib-ts](https://git.marketally.com/ai-manifests/adj-ref-lib-ts) and [adj-ref-lib-py](https://git.marketally.com/ai-manifests/adj-ref-lib-py).

## Status

**v0 — Draft.** Feedback welcome via issues.
