# Agent Deliberation Journal (ADJ) — v0

**Status:** Draft  
**Date:** 2026-04-11  
**Schema namespace:** `https://adj-manifest.dev/schemas/`

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

---

## 1. Purpose and Non-Goals

### 1.1 Purpose

ADJ defines a portable, append-only record format for agent deliberations that
supports calibration scoring, audit, and cross-deliberation learning. It
specifies the entry types a journal writes, the scoring contract that converts
(confidence, outcome) pairs into calibration scores, and the minimum query
surface that consumers code against.

ADJ is designed for federated deployment without central coordination; see
Section 13.

ADJ is a peer of ADP, not a dependency. ADP defines how agents decide together;
ADJ defines how those decisions are recorded, queried, and scored over time.
The two specs compose — ADP's CalibrationSource interface is implemented by
ADJ's query contract — but each is useful standalone. ADJ can record any
logged-decision system, not only ADP deliberations. ADP without ADJ degrades
gracefully to equal-weight voting plus domain authority.

### 1.2 Design Principle

The journal records the decision and the result separately and asynchronously.
Calibration is the function that joins them. This separation is the
load-bearing design choice: outcomes often arrive hours or days after a
decision is committed, and the scoring system must handle that latency without
blocking the deliberation flow.

### 1.3 Non-Goals

ADJ does **not** specify:

- **Storage backend.** Implementations choose their substrate — SQLite,
  Postgres, JSONL files, event-sourced log, whatever fits. A directory of
  JSONL files is a valid conformant journal.
- **Query language.** ADJ defines a minimum query contract (Section 7), not
  a query language.
- **Retention policy.** How long entries are kept, how they are archived.
- **Identity.** How agent identifiers are issued or verified.
- **Transport.** How the query contract is exposed (HTTP, gRPC, library calls,
  MCP tool).

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Journal** | A conformant store of ADJ entries. May be a file, a database, or a service. |
| **Entry** | A single typed record in the journal. Append-only — once written, never mutated or deleted. |
| **Deliberation record** | The complete set of entries for a single `deliberation_id`, from `deliberation_opened` through `deliberation_closed` and optionally `outcome_observed`. |
| **Outcome** | A measurable result observed after a deliberation was committed. Binary (success/failure), graded (0–1), or absent (no measurable outcome). |
| **Calibration score** | A score in [0, 1] for a (agent, domain) pair, computed from historical (confidence, outcome) pairs using a proper scoring rule. Higher is better calibrated. |
| **Proper scoring rule** | A scoring function that is maximized when the agent's stated confidence equals the true probability of success. Brier score and log-loss are proper scoring rules. |
| **Condition trace** | Per-agent metrics on dissent condition quality: falsification ratio, specificity, amendment frequency. |
| **Scoring window** | A time range or sample count over which calibration and condition quality are computed. |
| **Decision class** | A domain label (e.g., `code.correctness`, `security.policy`) that partitions calibration scores. Matches ADP domain claims. |
| **Evidence ref** | A URI pointing to supporting data. The `journal:` scheme (Section 3.6) resolves to journal entries. |
| **Outcome reporter** | An agent that writes `outcome_observed` entries. Subject to its own calibration on outcome reporting accuracy. |
| **Ground truth** | An outcome signal that is not subject to further calibration — external system confirmations (CI results, incident closure, human sign-off). |

---

## 3. Entry Types

ADJ is a log of typed entries, not a relational schema. Every entry shares a
common envelope and carries type-specific fields.

### 3.0 Common Envelope

Every entry MUST include these fields:

```json
{
  "entry_id": "adj_01HMXM...",
  "entry_type": "deliberation_opened",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:32:00.000Z",
  "prior_entry_hash": null
}
```

| Field | Description |
|---|---|
| `entry_id` | Unique identifier for this entry. MUST be globally unique within the journal. |
| `entry_type` | Discriminator. One of the five types defined below. |
| `deliberation_id` | Links the entry to its deliberation. All entries in a deliberation share this value. |
| `timestamp` | ISO 8601 timestamp of when the entry was written. |
| `prior_entry_hash` | SHA-256 hash of the previous entry in this deliberation's sequence, or `null` for the first entry. RECOMMENDED for tamper evidence; not required for conformance. |

### 3.1 `deliberation_opened`

Written when a deliberation begins. One per deliberation.

```json
{
  "entry_id": "adj_01HMXM7A",
  "entry_type": "deliberation_opened",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:32:00.000Z",
  "prior_entry_hash": null,

  "decision_class": "code.correctness",
  "action": {
    "kind": "merge_pull_request",
    "target": "github.com/acme/api#4471",
    "parameters": { "strategy": "squash" }
  },
  "participants": [
    "did:adp:test-runner-v2",
    "did:adp:security-scanner-v3",
    "did:adp:style-linter-v1"
  ],
  "config": {
    "max_rounds": 3,
    "participation_floor": 0.50
  }
}
```

| Field | Description |
|---|---|
| `decision_class` | Primary domain of the deliberation. Used for context; calibration scoring uses each agent's claimed domain from their proposal. |
| `action` | The action being deliberated. Same structure as ADP's proposal action. |
| `participants` | List of agent IDs expected to submit proposals. |
| `config` | Deliberation configuration (max rounds, participation floor, weight cap, etc.). |
| `delegations` | Optional. If domain delegation (ADP §5.5) was activated, lists the delegation clusters, their members, and their internal deliberation IDs. |

### 3.2 `proposal_emitted`

Written when an agent submits a proposal. One per agent per deliberation.

```json
{
  "entry_id": "adj_01HMXM8B",
  "entry_type": "proposal_emitted",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:32:09.221Z",
  "prior_entry_hash": "sha256:a1b2c3...",

  "proposal": { }
}
```

The `proposal` field contains the full ADP proposal object (ADP Section 3),
including dissent conditions and the initially empty revisions array. The
proposal is stored verbatim — the journal does not interpret or transform it.

As the proposal accumulates amendments and revisions during belief-update
rounds, the updated state is captured in `round_event` entries, not by
mutating this entry.

### 3.3 `round_event`

Written for each state-machine transition during a belief-update round. Many
per deliberation.

```json
{
  "entry_id": "adj_01HMXM9C",
  "entry_type": "round_event",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:34:15.882Z",
  "prior_entry_hash": "sha256:d4e5f6...",

  "round": 1,
  "event_kind": "falsification_evidence",
  "agent_id": "did:adp:test-runner-v2",
  "target_agent_id": "did:adp:security-scanner-v3",
  "target_condition_id": "dc_ss_01",
  "payload": {
    "evidence_refs": ["journal:dlb_01HMXJ3E9R/evidence/test-run-9912"],
    "argument": "Test run 9912 includes path coverage for all 3 auth module code paths."
  }
}
```

| Field | Description |
|---|---|
| `round` | Belief-update round number (1-indexed). |
| `event_kind` | One of: `falsification_evidence`, `acknowledge`, `reject`, `amend`, `revise`, `challenge_tier`, `tier_response`, `timeout`, `outcome_contested`, `calibration_divergence`. |
| `agent_id` | The agent that produced this event. |
| `target_agent_id` | The agent whose condition or proposal is targeted. `null` for self-directed events (revise, timeout). |
| `target_condition_id` | The dissent condition being addressed. `null` for non-condition events. |
| `payload` | Event-specific data. Structure varies by `event_kind`. |

### 3.4 `deliberation_closed`

Written when a deliberation reaches a terminal state. One per deliberation.

```json
{
  "entry_id": "adj_01HMXMAC",
  "entry_type": "deliberation_closed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:35:30.004Z",
  "prior_entry_hash": "sha256:g7h8i9...",

  "termination": "converged",
  "round_count": 1,
  "tier": "partially_reversible",
  "final_tally": {
    "approve_weight": 0.89,
    "reject_weight": 0.00,
    "abstain_weight": 0.64,
    "total_weight": 1.53,
    "approval_fraction": 1.00,
    "participation_fraction": 0.582,
    "threshold": 0.60
  },
  "weights": {
    "did:adp:test-runner-v2": 0.71,
    "did:adp:security-scanner-v3": 0.64,
    "did:adp:style-linter-v1": 0.18
  },
  "committed_action": {
    "kind": "merge_pull_request",
    "target": "github.com/acme/api#4471",
    "parameters": { "strategy": "squash" }
  }
}
```

| Field | Description |
|---|---|
| `termination` | One of: `converged`, `partial_commit`, `deadlocked`. |
| `round_count` | Number of belief-update rounds executed (0 if converged on first tally). |
| `tier` | Final reversibility tier (may differ from initial if a tier challenge succeeded). |
| `final_tally` | The tally that produced the terminal state. |
| `weights` | Map of agent_id to computed weight. Enables tally re-verification. |
| `committed_action` | The action committed for execution, or `null` for deadlock. For `partial_commit`, this is the reversible subset. |

### 3.5 `outcome_observed`

Written when a measurable result is observed after a committed action. Zero or
one per deliberation. May be written hours or days after `deliberation_closed`.

```json
{
  "entry_id": "adj_01HMZP2D",
  "entry_type": "outcome_observed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-14T09:15:00.000Z",
  "prior_entry_hash": "sha256:j0k1l2...",

  "observed_at": "2026-04-14T09:12:00.000Z",
  "outcome_class": "binary",
  "success": true,
  "evidence_refs": [
    "ci:github-actions/run/8835001",
    "monitoring:datadog/alert/clear/991204"
  ],
  "reporter_id": "did:adp:ci-monitor-v1",
  "reporter_confidence": 0.95,
  "ground_truth": false
}
```

| Field | Description |
|---|---|
| `observed_at` | When the outcome was actually observed (may differ from journal write time). |
| `outcome_class` | `"binary"` (success is boolean) or `"graded"` (success is a number in [0, 1]). |
| `success` | `true`/`false` for binary, a number in [0, 1] for graded. |
| `evidence_refs` | URIs pointing to the evidence that established the outcome. |
| `reporter_id` | The agent that reported this outcome. Subject to its own calibration. |
| `reporter_confidence` | The reporter's confidence in the accuracy of this outcome observation. |
| `ground_truth` | If `true`, this outcome is treated as authoritative and not subject to reporter calibration. Used for external system signals (CI pass/fail, human confirmation). |

If no measurable outcome exists, no `outcome_observed` entry is written. The
deliberation is excluded from calibration scoring.

### 3.6 Journal URI Scheme

ADP proposals reference journal entries via the `journal:` URI scheme.
ADJ defines the resolution semantics.

**Format:**

```
journal:<deliberation_id>/<path>
```

**Examples:**

| URI | Resolves to |
|---|---|
| `journal:dlb_01HMXJ3E9R` | Full deliberation record |
| `journal:dlb_01HMXJ3E9R/entry/adj_01HMXM9C` | A specific entry by ID |
| `journal:dlb_01HMXJ3E9R/proposal/prp_01HMXK4F7G` | A proposal entry by proposal_id |
| `journal:dlb_01HMXJ3E9R/evidence/test-run-9912` | An evidence artifact attached to a round_event |
| `journal:dlb_01HMXJ3E9R/outcome` | The outcome_observed entry, if any |

Resolution is implementation-defined — implementations translate `journal:`
URIs to their underlying storage addresses. The scheme is a logical namespace,
not a transport protocol.

ADP §3 `evidence_refs` and `justification.evidence_refs` SHOULD use `journal:`
URIs when referencing data stored in a journal. Non-journal evidence (CI
systems, external monitoring) uses implementation-specific URI schemes.

---

## 4. The Outcome Contract

The outcome entry is where ADJ does the work ADP cannot. It binds a
deliberation to a measurable result, asynchronously and after the fact.

### 4.1 Outcome Types

| Type | `success` value | Example |
|---|---|---|
| **Binary** | `true` or `false` | PR merged cleanly / caused regression |
| **Graded** | Number in [0, 1] | Incident resolved in N minutes vs estimated M; score = clamp(1 - actual/estimated, 0, 1) |
| **Absent** | No entry written | No measurable outcome; excluded from calibration |

Binary outcomes map to 1.0 (success) or 0.0 (failure) for calibration scoring.
Graded outcomes are used directly.

### 4.2 Outcome Reporters

The reporter is itself an agent with its own calibration — specifically,
calibration on the decision class `outcome.reporting`. This prevents the
obvious gaming vector of self-reported success: an agent that reports its own
outcomes favorably will accumulate poor calibration on outcome reporting, and
its future reports will be discounted.

**Ground truth.** This recursion bottoms out at ground truth: outcomes sourced
from external systems (CI pass/fail, incident closed in PagerDuty, human
sign-off) are marked with `ground_truth: true` and are not subject to reporter
calibration. Ground truth outcomes are the fixed points that anchor the entire
calibration system.

In practice, most production deployments will have a small number of outcome
reporters — often a single CI/monitoring agent — whose reports are treated as
ground truth for automated decisions. The reporter-calibration mechanism
exists for cases where outcomes require judgment (e.g., "did the incident
response succeed?"), not for cases where a boolean signal is available.

### 4.3 Timing

Outcomes may arrive long after the deliberation closes. The calibration system
MUST handle this latency:

- A deliberation without an `outcome_observed` entry is excluded from
  calibration scoring.
- When an outcome is written, calibration scores for all participants are
  recomputed (or incrementally updated).
- Staleness in the CalibrationScore triple is measured from the most recent
  *scored* deliberation (one with an outcome), not the most recent
  participation.

### 4.4 Outcome Contestation

Any agent MAY contest a recorded outcome by submitting an `outcome_contested`
round event with evidence. Contestation is the mechanism that prevents a
compromised outcome reporter (or a false-positive CI signal) from permanently
poisoning calibration data.

**Contestation flow:**

1. An `outcome_observed` entry is written with `ground_truth: true`.
2. Within the contestation window (default: 72 hours), any participating agent
   submits an `outcome_contested` round event referencing the outcome entry ID
   and providing counter-evidence.
3. If agents with combined domain authority ≥ 0.7 for the deliberation's
   decision class support the contestation, the outcome is superseded.
4. A new `outcome_observed` entry is written with `supersedes` referencing
   the original, reflecting the contested result.
5. After the contestation window expires with no challenges, the outcome
   is considered final.

**Round event for contestation:**

```json
{
  "entry_type": "round_event",
  "event_kind": "outcome_contested",
  "agent_id": "did:adp:security-scanner-v3",
  "target_condition_id": null,
  "payload": {
    "contested_entry_id": "adj_01HMZP2D",
    "reason": "CI reported success but production regression detected within 24 hours.",
    "evidence_refs": ["monitoring:datadog/alert/triggered/991205"],
    "proposed_outcome": false
  }
}
```

Uncontested outcomes remain authoritative. Contestation after the window
expires MUST be rejected. Implementations SHOULD log expired contestation
attempts for audit.

---

## 5. Calibration Scoring

This is the math that ADP Section 4 depends on. For each (agent, domain) pair,
the journal computes a calibration score from the agent's historical
(confidence, outcome) pairs using a proper scoring rule.

### 5.1 Default Scoring Rule: Brier Score

The default proper scoring rule is the Brier score, which penalizes both
overconfidence and underconfidence:

```
BS(agent, domain) = (1/N) × Σᵢ (cᵢ − oᵢ)²
```

Where:

- `N` = number of scored deliberations for this (agent, domain) pair
- `cᵢ` = agent's stated `confidence` in deliberation i
- `oᵢ` = observed outcome: 1.0 for success, 0.0 for failure, or [0, 1] for
  graded outcomes

The Brier score ranges from 0 (perfect calibration) to 1 (maximally wrong).
The calibration value inverts this:

```
calibration_value = 1 − BS(agent, domain)
```

This gives a value in [0, 1] where:

| Value | Interpretation |
|---|---|
| 1.00 | Perfect calibration — stated confidence exactly matches outcomes. |
| 0.75 | Uninformative — equivalent to always predicting 0.50 on balanced binary outcomes. |
| 0.50 | Significant miscalibration. |
| → 0 | Systematically wrong — confidently predicting the opposite of what occurs. |

Implementations MAY substitute log-loss or another proper scoring rule. If
substituted, the scoring rule MUST return a value in [0, 1] with the same
polarity (higher = better calibrated).

### 5.2 CalibrationScore Triple

ADJ's query contract returns the triple that ADP's CalibrationSource interface
expects:

```json
{
  "value": 0.85,
  "sample_size": 312,
  "staleness": "P18D"
}
```

| Field | Derivation |
|---|---|
| `value` | `1 − BS(agent, domain)`, computed over all scored deliberations in the scoring window. |
| `sample_size` | Count of scored deliberations for this (agent, domain) pair. |
| `staleness` | Duration since the most recent scored deliberation for this (agent, domain) pair. ISO 8601 duration. |

**Scoring window.** Implementations MAY limit the scoring window (e.g., last
365 days, last 1000 deliberations). If windowed, older deliberations are
excluded from the computation but not deleted from the journal.

### 5.3 Which Confidence is Scored

Each agent's confidence is scored against the outcome in the agent's **claimed
domain**, not the deliberation's primary decision class. The test-runner's
calibration on `code.correctness` is updated; the scanner's calibration on
`security.policy` is updated. This matches ADP's `getScore(agent_id, domain)`
interface.

If an agent participates in a deliberation but the outcome is in a different
domain than claimed, the agent's calibration on their claimed domain is still
updated. The outcome is the same for all agents; the domain partitioning
determines which calibration bucket the score lands in.

### 5.4 Bootstrap

An agent with no calibration history in a domain receives the bootstrap
default:

```json
{ "value": 0.5, "sample_size": 0, "staleness": "PT0S" }
```

ADP's weighting function applies a sample-size discount
(`value × (1 − 1/(1 + sample_size))`), which means a bootstrap agent has
effective calibration weight of zero. Authority is earned through
participation, not declared.

### 5.5 Worked Calibration Update

After the PR merge outcome (success, binary), updating the test-runner's
calibration on `code.correctness`:

```
Prior:  cal = 0.85, BS = 0.15, N = 312
Agent's confidence: 0.86
Outcome: 1.0 (success)
Brier contribution: (0.86 − 1.0)² = 0.0196

Updated Brier sum:  312 × 0.15 + 0.0196 = 46.8196
Updated N:          313
New BS:             46.8196 / 313 = 0.1496
New cal:            1 − 0.1496 = 0.8504 ≈ 0.85
New staleness:      P0D
```

With 312 prior samples, a single deliberation barely moves the score. This is
the intended behavior — calibration is a long-term signal, not a reactive one.

---

## 6. Condition Quality Scoring

ADP defers dissent condition quality assessment to the journal. ADJ tracks
three per-agent metrics that catch defection patterns like publishing
unfalsifiable conditions or habitually vague declarations.

### 6.1 Falsification Ratio

```
falsification_ratio(agent, window) =
    conditions_tested / conditions_published
```

Where `conditions_tested` counts dissent conditions whose `tested_in_round` is
non-null (someone attempted falsification), and `conditions_published` counts
all dissent conditions the agent published across deliberations in the window.

| Ratio | Interpretation |
|---|---|
| > 0.7 | Conditions are testable and routinely tested. Healthy. |
| 0.3 – 0.7 | Mixed — some conditions may be hard to test. Neutral. |
| < 0.3 | Conditions are habitually untestable. Possible defection. |

A low falsification ratio does not prove defection — the agent may operate in
domains where evidence is genuinely scarce. The metric provides signal, not
verdicts.

### 6.2 Specificity Score

A heuristic measure of condition concreteness. Implementations SHOULD compute
a specificity score in [0, 1] based on the presence of:

- **Numeric thresholds** — "if latency p99 > 200ms" scores higher than
  "if latency is high"
- **Named entities** — "if auth module" scores higher than "if any module"
- **Measurable predicates** — "if coverage delta is negative" scores higher
  than "if quality decreases"
- **Temporal bounds** — "within 24 hours" scores higher than unbounded
  conditions

The exact heuristic is implementation-defined. The spec defines what to
measure, not how to compute it, because natural language analysis is
domain-specific.

### 6.3 Amendment Frequency

```
amendment_frequency(agent, window) =
    total_amendments / conditions_published
```

Amendment frequency is ambiguous in isolation:

- **High frequency + high convergence rate** → healthy responsiveness.
  The agent refines conditions in response to evidence. Good.
- **High frequency + low convergence rate** → poor initial specification.
  The agent publishes conditions it cannot sustain. Problematic.

Implementations SHOULD report amendment frequency alongside the convergence
rate of deliberations the agent participated in.

### 6.4 Feeding Back Into Authority

Condition quality metrics do not directly gate participation — any agent can
join a deliberation regardless of its condition quality. However,
implementations SHOULD feed condition quality into the agent's effective
authority over time:

- An agent with a habitually low falsification ratio (< 0.3 over 50+
  deliberations) SHOULD see its domain authority discounted.
- An agent with high amendment frequency and low convergence rate SHOULD
  see its calibration score penalized.

The exact feedback mechanism is implementation-defined.

---

## 7. Query Contract

ADJ defines a minimum set of queries any conformant journal MUST support (at
Level 2 compliance, Section 10). Implementations expose these however they
choose — HTTP endpoints, gRPC services, library calls, MCP tools.

### 7.1 Required Queries

```
getCalibration(agent_id, domain) → CalibrationScore
```

Returns the calibration score triple (Section 5.2) for an (agent, domain)
pair. Returns the bootstrap default (Section 5.4) if no history exists. This
is the query that implements ADP's CalibrationSource interface.

```
getDeliberation(deliberation_id) → DeliberationRecord
```

Returns the complete set of entries for a deliberation, ordered by timestamp.
Includes all entry types from `deliberation_opened` through `outcome_observed`.

```
getConditionTrace(agent_id, window) → ConditionQualityMetrics
```

Returns condition quality metrics (Section 6) for an agent over a scoring
window. The window parameter specifies a time range or sample count.

```
getOutcome(deliberation_id) → OutcomeEntry | null
```

Returns the `outcome_observed` entry for a deliberation, or null if no outcome
has been recorded.

```
listDeliberationsSince(since, limit) → DeliberationRecord[]
```

Returns full deliberation records (all entries per deliberation, ordered by
timestamp) for every deliberation whose `deliberation_closed` entry has a
timestamp at or after `since`. Ordered newest-first. `limit` caps the
response size; implementations MUST return at most `limit` records and
SHOULD set a sensible server-side maximum (the reference implementations
use 500).

This is the batch equivalent of calling `getDeliberation` once per
deliberation ID. It is REQUIRED at Level 2 so registries and aggregators
can compute federation-wide metrics in a single round trip per agent
rather than `N × M` round trips when walking `N` deliberations across `M`
peer journals.

The HTTP binding is `GET /adj/v0/deliberations?since=<iso8601>&limit=<int>`.
The response envelope is:

```json
{
  "since": "2026-04-10T00:00:00Z",
  "limit": 200,
  "total": 42,
  "records": [
    {
      "deliberationId": "dlb_01HMXJ3E9R",
      "entries": [ /* same shape as getDeliberation(id) */ ]
    }
  ]
}
```

Returns `records` newest-first by `deliberation_closed.timestamp`.
Deliberations with no `deliberation_closed` entry (still in progress) are
excluded — open deliberations are not billable history. An agent that has
never closed a deliberation in the window returns `{ total: 0, records: [] }`.

### 7.2 Recommended Queries

These are not required for Level 2 conformance but are useful for monitoring
and tooling:

```
getAgentHistory(agent_id, window) → DeliberationSummary[]
```

Returns summaries of deliberations the agent participated in.

```
getCalibrationHistory(agent_id, domain, window) → CalibrationScore[]
```

Returns a time series of calibration scores, enabling trend analysis.

### 7.3 Federation Health Queries

These queries support ADP's federation health metrics (ADP Section 5.6).
They are RECOMMENDED for Level 3 conformance.

```
getWeightDistribution(window) → { agentId: string, domain: string, weight: number }[]
```

Returns the weight of all active agents (those who participated in at least
one deliberation within the window), enabling Gini coefficient computation.

```
getIntegrationRate(agent_id) → { deliberationsToThreshold: number, currentWeight: number, medianWeight: number }
```

Returns how many deliberations a new agent has participated in and whether
its weight has reached meaningful levels relative to the median.

```
getDeliberationEfficiency(window) → { avgRounds: number, quorumFailureRate: number, avgParticipants: number, delegationRate: number }
```

Returns aggregate deliberation statistics over a time window, enabling trend
detection for scaling degradation.

```
getCalibrationMobility(window) → { agentId: string, weightChange: number }[]
```

Returns per-agent weight changes over a window. Low mobility indicates a
calcified hierarchy where the learning loop has stalled.

### 7.4 Signed Calibration Snapshot (Well-Known Endpoint)

Agents at Level 3 compliance SHOULD expose a signed calibration snapshot at
`/.well-known/adp-calibration.json`. The snapshot is a per-agent summary
covering every decision class the agent claims authority over, signed with
the same Ed25519 key published in the agent's manifest.

The purpose of this endpoint is to allow cross-org trust bootstrapping and
third-party tamper evidence *without* requiring a peer to walk the full
journal or contact a central registry. A peer that wants to verify another
agent's calibration fetches the well-known file, verifies the signature
against the manifest public key, and is done — one HTTPS call plus one
signature check.

**Response envelope:**

```json
{
  "agentId": "did:adp:test-runner-v2",
  "publicKey": "82d5d49d701cb3260b730e8021a6235f4c607c745f7db20353a189de6d683dd5",
  "computedAt": "2026-04-13T10:30:00.000Z",
  "snapshots": [
    {
      "domain": "code.correctness",
      "calibrationValue": 0.9804,
      "sampleSize": 1,
      "journalHash": "3a8f1e2d9c7b...",
      "computedAt": "2026-04-13T10:30:00.000Z",
      "signature": "ed25519:7a4b1c..."
    }
  ]
}
```

| Field | Description |
|---|---|
| `agentId` | Top-level envelope: the agent whose calibration is being published. MUST match the `agent_id` in the agent's manifest. |
| `publicKey` | Hex-encoded Ed25519 public key. MUST match the `publicKey` in the agent's manifest. Clients MAY cross-check this. |
| `computedAt` | When the envelope was built. Typically the most recent snapshot's `computedAt`. |
| `snapshots[].domain` | The decision class this snapshot applies to. One entry per declared decision class with non-empty calibration history. |
| `snapshots[].calibrationValue` | The Brier-based calibration score (Section 5) in [0, 1]. |
| `snapshots[].sampleSize` | Number of (confidence, outcome) pairs contributing to this value. |
| `snapshots[].journalHash` | Hex-encoded SHA-256 of the scoring pairs used to compute this value. Canonical form: for each pair ordered by outcome timestamp, append `<deliberation_id>:<confidence>:<outcome>|`, then hash the resulting UTF-8 string. This format is deterministic and matches what audit implementations compute when walking the journal directly, enabling direct comparison between an agent's signed snapshot and a third-party replay. |
| `snapshots[].computedAt` | When this specific snapshot was computed. |
| `snapshots[].signature` | Ed25519 signature over a canonical string, hex-encoded. See canonicalization below. |

**Signature canonicalization.** Each snapshot is signed individually so a
peer can verify a single (agent, domain) pair without trusting the rest of
the envelope. The signed message is the UTF-8 encoding of the pipe-joined
string:

```
<agentId>|<domain>|<calibrationValue>|<sampleSize>|<journalHash>|<computedAt>
```

`calibrationValue` MUST be serialized to exactly four decimal places
(matching the Brier precision in Section 5). `computedAt` MUST be the same
ISO 8601 string that appears in the JSON. This is a minimal canonicalization
chosen for implementability across language ecosystems without JSON-canonicalization libraries.

**Properties this delivers.** An agent that rewrites its journal to fake a
better calibration value would need to produce a new snapshot with a new
signature — but a third party (registry or peer) that archived the prior
signed snapshot can present the divergence mechanically. The journal hash
binds the published value to a specific set of scoring pairs, so an agent
cannot claim the same calibration was computed from a different journal state
without producing two conflicting signatures over the same (agentId, domain)
pair. The registry pattern for catching this is described in ADJ §8.3.

**Registry archival.** A registry MAY periodically fetch this endpoint for
every registered agent, verify the signatures, and store the snapshots as
rolling history. The audit surface then has three values to compare for any
(agent, domain): reported (the value the agent currently serves via
`getCalibration`), signed (from the well-known snapshot), and computed (from
replaying the journal). All three MUST agree within tolerance; any
divergence between them is evidence of either tampering or implementation
drift and SHOULD be flagged.

**Not required.** This endpoint is a Level 3 SHOULD, not a MUST. Agents
that do not publish signed snapshots remain conformant at Level 2. Peers
that want tamper evidence from such agents fall back to walking the journal
via `listDeliberationsSince` (Section 7.1) and recomputing.

---

## 8. Append-Only and Integrity

### 8.1 Append-Only Guarantee

ADJ entries MUST be append-only. Once written, an entry MUST NOT be mutated
or deleted. This is the property that makes the journal trustworthy as a
calibration substrate — you cannot retroactively improve your track record.

Corrections are new entries that reference prior entries by `entry_id`:

```json
{
  "entry_type": "outcome_observed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "supersedes": "adj_01HMZP2D",
  "reason": "Initial CI signal was a false positive; regression confirmed after manual review.",
  "success": false
}
```

The `supersedes` field links the correction to the entry it replaces. Both
entries remain in the journal. The most recent entry (by `entry_id` or
timestamp) is authoritative for calibration scoring.

### 8.2 Hash Chaining

Hash chaining — each entry referencing the SHA-256 hash of the previous entry
in its deliberation — is RECOMMENDED but not REQUIRED.

Hash chaining provides tamper evidence: any modification to a historical entry
breaks the chain. This is valuable for multi-party scenarios where agents
from different organizations share a journal and need to verify that records
have not been retroactively altered.

Implementations that do not need tamper evidence MAY set `prior_entry_hash`
to `null` for all entries.

### 8.3 JSONL Compatibility

A journal MAY be implemented as a directory of JSONL files — one JSON object
per line, one file per deliberation or per time window. This is explicitly
compatible with PostMortem-style session journals and provides the simplest
possible conformant implementation (Level 1).

---

## 9. Worked Example: The PR Merge Journal

The same PR merge from ADP Section 8, shown as the journal records it.

### 9.1 Entry Sequence

**Entry 1 — deliberation_opened:**

```json
{
  "entry_id": "adj_01HMXM7A",
  "entry_type": "deliberation_opened",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:32:00.000Z",
  "prior_entry_hash": null,
  "decision_class": "code.correctness",
  "action": { "kind": "merge_pull_request", "target": "github.com/acme/api#4471", "parameters": { "strategy": "squash" } },
  "participants": ["did:adp:test-runner-v2", "did:adp:security-scanner-v3", "did:adp:style-linter-v1"],
  "config": { "max_rounds": 3, "participation_floor": 0.50 }
}
```

**Entries 2–4 — proposal_emitted (one per agent):**

Three `proposal_emitted` entries, each containing the full ADP proposal object.
(See ADP Section 8.2 for the proposal content.)

**Entry 5 — falsification evidence (test-runner → scanner, dc_ss_01):**

```json
{
  "entry_id": "adj_01HMXM9C",
  "entry_type": "round_event",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:34:15.882Z",
  "prior_entry_hash": "sha256:d4e5f6...",
  "round": 1,
  "event_kind": "falsification_evidence",
  "agent_id": "did:adp:test-runner-v2",
  "target_agent_id": "did:adp:security-scanner-v3",
  "target_condition_id": "dc_ss_01",
  "payload": {
    "evidence_refs": ["journal:dlb_01HMXJ3E9R/evidence/test-run-9912"],
    "argument": "Test run 9912 includes path coverage for all 3 auth module code paths identified in scan 4410."
  }
}
```

**Entry 6 — falsification evidence (test-runner → scanner, dc_ss_02):**

Same shape, targeting `dc_ss_02` with security test suite evidence.

**Entry 7 — acknowledge (scanner falsifies dc_ss_01):**

```json
{
  "entry_id": "adj_01HMXM9E",
  "entry_type": "round_event",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:34:44.221Z",
  "prior_entry_hash": "sha256:e5f6g7...",
  "round": 1,
  "event_kind": "acknowledge",
  "agent_id": "did:adp:security-scanner-v3",
  "target_agent_id": null,
  "target_condition_id": "dc_ss_01",
  "payload": { "reason": "Evidence shows full path coverage of all 3 flagged code paths." }
}
```

**Entry 8 — acknowledge (scanner falsifies dc_ss_02):**

Same shape for `dc_ss_02`.

**Entry 9 — revise (scanner, reject → abstain):**

```json
{
  "entry_id": "adj_01HMXM9G",
  "entry_type": "round_event",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:35:22.109Z",
  "prior_entry_hash": "sha256:f6g7h8...",
  "round": 1,
  "event_kind": "revise",
  "agent_id": "did:adp:security-scanner-v3",
  "target_agent_id": null,
  "target_condition_id": null,
  "payload": {
    "prior_vote": "reject",
    "new_vote": "abstain",
    "reason": "Both dissent conditions (dc_ss_01, dc_ss_02) falsified by test-runner evidence. No remaining basis for rejection."
  }
}
```

**Entry 10 — deliberation_closed (converged):**

```json
{
  "entry_id": "adj_01HMXMAC",
  "entry_type": "deliberation_closed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:35:30.004Z",
  "prior_entry_hash": "sha256:g7h8i9...",
  "termination": "converged",
  "round_count": 1,
  "tier": "partially_reversible",
  "final_tally": {
    "approve_weight": 0.89,
    "reject_weight": 0.00,
    "abstain_weight": 0.64,
    "total_weight": 1.53,
    "approval_fraction": 1.00,
    "participation_fraction": 0.582,
    "threshold": 0.60
  },
  "weights": {
    "did:adp:test-runner-v2": 0.71,
    "did:adp:security-scanner-v3": 0.64,
    "did:adp:style-linter-v1": 0.18
  },
  "committed_action": { "kind": "merge_pull_request", "target": "github.com/acme/api#4471", "parameters": { "strategy": "squash" } }
}
```

**Entry 11 — outcome_observed (3 days later):**

```json
{
  "entry_id": "adj_01HMZP2D",
  "entry_type": "outcome_observed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-14T09:15:00.000Z",
  "prior_entry_hash": "sha256:j0k1l2...",
  "observed_at": "2026-04-14T09:12:00.000Z",
  "outcome_class": "binary",
  "success": true,
  "evidence_refs": ["ci:github-actions/run/8835001", "monitoring:datadog/alert/clear/991204"],
  "reporter_id": "did:adp:ci-monitor-v1",
  "reporter_confidence": 0.95,
  "ground_truth": true
}
```

### 9.2 Calibration After Outcome

Once the outcome is recorded, the journal updates calibration for each
participant:

| Agent | Domain | Confidence | Outcome | Brier | Prior cal (N) | Updated cal (N+1) |
|---|---|---|---|---|---|---|
| test-runner-v2 | code.correctness | 0.86 | 1.0 | 0.0196 | 0.85 (312) | 0.8504 (313) |
| security-scanner-v3 | security.policy | 0.79 | 1.0 | 0.0441 | 0.83 (187) | 0.8258 (188) |
| style-linter-v1 | code.style | 0.62 | 1.0 | 0.1444 | 0.72 (89) | 0.7215 (90) |

All three agents under-predicted success. The test-runner's score barely moves
(312 samples absorb one data point). The scanner and linter show similar
stability. Over dozens of deliberations, systematic over- or under-prediction
becomes visible and shifts weights meaningfully.

### 9.3 The Composition Moment

This is where the two specs visibly compose. The event stream that drove
ADP's state machine — proposals, falsification, revision, convergence — is
exactly the data structure that scores the participating agents. The journal
doesn't interpret the deliberation differently than the orchestrator did; it
just keeps the record and runs the math when the outcome arrives.

ADP defines how agents decide. ADJ defines how they learn from deciding.

---

## 10. Compliance Levels

| Level | Name | Requirements |
|---|---|---|
| **Level 1** | Writer | MUST write conformant entries per Section 3. MUST respect append-only (Section 8). MAY use JSONL files, databases, or any other substrate. |
| **Level 2** | Query Provider | MUST meet Level 1. MUST serve the required queries in Section 7.1. MAY expose them via any transport. |
| **Level 3** | Scoring Engine | MUST meet Level 2. MUST compute calibration scores using a proper scoring rule (Section 5). MUST compute condition quality metrics (Section 6). MUST return valid CalibrationScore triples. |

A Level 1 journal is just a log — it records deliberations faithfully but
does not answer queries or compute scores. This is the entry point for
implementations that want audit trails without the scoring overhead.
PostMortem-style JSONL session journals are a natural Level 1 implementation.

A Level 2 journal answers queries, enabling ADP orchestrators to look up
deliberation history and outcomes.

A Level 3 journal is the full calibration engine that ADP's weighting
function depends on. It computes the scores, serves them through the
CalibrationSource interface, and closes the learning loop.

---

## 11. Relationship to Other Specs

```
ACB                  ← cognitive budget, settlement (extends ADJ via v0.1 hook)
ADJ — this spec      ← journal entries, calibration scoring, query contract
ADP                  ← proposal → weight → converge → commit (consumes ADJ)
mcp-manifest         ← declares journal participation and endpoint
A2A / AGNTCY         ← transport for query contract
PostMortem           ← existing JSONL format, natural L1 implementation
```

| Spec | Relationship |
|---|---|
| **ADP** | ADP consumes ADJ via the CalibrationSource interface. ADP writes deliberation events; ADJ stores them. ADP's weighting function reads calibration scores from ADJ's query contract. The composition is bidirectional: ADP produces the data, ADJ scores it, ADP uses the scores. |
| **ACB** | ACB extends ADJ in v0.1 by adding three optional entry types — `budget_committed`, `budget_cancelled`, `settlement_recorded` — that follow the §3.0 common envelope and inherit hash chaining, append-only guarantees, and replay verification. ACB requires no other ADJ changes. ADJ-only deployments function unchanged; the new entry types are simply unused. See §11.2. |
| **mcp-manifest** | An agent's mcp-manifest declares that it participates in ADJ and points at its journal endpoint (or declares it writes to a shared journal). The `adj_journal` field in mcp-manifest is the discovery mechanism. |
| **PostMortem** | PostMortem's single-agent JSONL session format is a valid Level 1 ADJ implementation. The journal spec retroactively describes the existing tool's record format, with L2 and L3 adding query and scoring capabilities. This is the bridge to existing work and the credibility anchor: the spec describes something that already runs, not a greenfield design. |
| **A2A / AGNTCY** | If the query contract is exposed as a network service, transport is delegated to A2A/AGNTCY. ADJ defines the queries, not how they travel. |

### 11.1 The Four-Spec Family

mcp-manifest, ADP, ADJ, and ACB form a composable stack:

- **mcp-manifest** declares what an agent can do.
- **ADP** declares how agents agree on doing it together.
- **ADJ** declares how those agreements are recorded and scored.
- **ACB** declares how the cognitive work of agreeing is paid for.

Each spec has a reference implementation and a worked example. Each is useful
standalone and more useful together.

### 11.2 v0.1 Hook for ACB: Three New Entry Types

ACB v0 adds three entry types that ADJ adopts in its v0.1 hook list. Each
follows the ADJ §3.0 common envelope — `entry_id`, `entry_type`,
`deliberation_id`, `timestamp`, `prior_entry_hash` — so they live in the
same journal as the existing five entry types and inherit hash chaining,
append-only guarantees, and replay verification.

| Entry type | Written by | When |
|---|---|---|
| `budget_committed` | Budget authority's journal | At or before `deliberation_opened` for the referenced deliberation. |
| `budget_cancelled` | Budget authority's journal | Before `deliberation_opened`, or before any proposal if `constraints.irrevocable` is false. Mutually exclusive with `settlement_recorded`. |
| `settlement_recorded` | Budget authority's journal | After `deliberation_closed` and (for deferred/two-phase modes) after `outcome_observed` or outcome-window expiry. One per budget. |

The full schemas live in the ACB spec; ADJ does not duplicate them. ADJ
implementations that wish to support ACB MUST treat these three values as
valid `entry_type` discriminators and apply the common-envelope checks.
ADJ implementations that do not wish to support ACB MAY reject them as
unknown entry types — both behaviors are conformant.

The split between `budget_committed`, `deliberation_*`, and
`settlement_recorded` is the same load-bearing design choice as the split
between `deliberation_closed` and `outcome_observed`: pre-commitment, work,
and post-hoc settlement are recorded separately and asynchronously, with
the journal's hash chain making the sequencing auditable. Settlement does
not require a synchronous close-of-deliberation handshake. The budget
authority signs the record once. Any disagreement is caught by replay
against the ground-truth deliberation entries the same way calibration
divergence is.

---

## 12. Open Questions

### 12.1 Outcome Attribution

When multiple deliberations contribute to a single observed result (e.g.,
three sequential PRs all contributed to a production incident), how are
outcomes attributed? Options:

- **Full attribution.** All contributing deliberations receive the same
  outcome. Simple but noisy.
- **Weighted attribution.** Each deliberation receives a fraction of the
  outcome proportional to its estimated contribution. Accurate but requires
  a causal model.
- **Nearest-in-time.** The most recent deliberation before the outcome is
  attributed. Simple heuristic.

Deferred to v0.1.

### 12.2 Outcome Reporter Recursion

The outcome reporter is an agent with its own calibration (Section 4.2). If
the reporter's calibration depends on outcomes, and outcomes are reported by
reporters whose calibration depends on outcomes, the system appears recursive.

In practice, the recursion bottoms out at **ground truth** — external system
signals that are not subject to reporter calibration:

- CI pass/fail (binary, automated, no judgment)
- Incident closure in monitoring systems (PagerDuty, Datadog)
- Human confirmation (manual sign-off)

Ground truth outcomes (`ground_truth: true`) are the fixed points. Reporter
calibration only applies to outcomes that involve judgment. Most production
deployments will have ground truth for the vast majority of outcomes.

### 12.3 Journal Federation

How do journals federate across organizations without leaking proprietary
deliberation content? Options:

- **Hashed summaries.** Share calibration scores and condition quality metrics
  without sharing deliberation content. Enables cross-org weighting.
- **Selective disclosure.** Share full records for specific deliberations with
  consent.
- **Aggregated scores only.** Share only the CalibrationScore triple. Minimal
  information, sufficient for weighting.

Deferred to v0.1+.

### 12.4 Calibration Gaming via Outcome Timing

An agent that participates only in easy, predictable deliberations will
accumulate a high calibration score without demonstrating competence on
hard decisions. Mitigations:

- Track the distribution of decision classes and reversibility tiers an
  agent participates in.
- Weight calibration updates by decision difficulty (irreversible decisions
  contribute more to calibration than reversible ones).
- Report calibration alongside a "decision breadth" metric.

### 12.5 Null Outcome Rate

An agent that habitually opts out of calibration (ADP `calibration_at_stake:
false`) or participates only in deliberations that never produce outcomes
avoids the scoring loop entirely. The journal SHOULD track per-agent:

- Outcome rate: fraction of participated deliberations that produced outcomes.
- Opt-out rate: fraction of deliberations where `calibration_at_stake` was false.

Agents with systematically low outcome or high opt-out rates SHOULD see
authority discounted, per ADP Section 11.6.

---

## 13. Federation

ADJ journals are per-agent, not shared. Each agent owns its epistemic history,
carries it across deployments, and presents it to peers on request. This
design is federation-native: no central scoreboard, no shared substrate, no
lock-in point.

### 13.1 Journal as a Service

An agent's journal endpoint is declared in its ADP manifest
(`.well-known/adp-manifest.json`, field `journal_endpoint`). The endpoint
serves the ADJ query contract (Section 7) over HTTPS.

Peers query calibration scores by fetching:

```
GET https://agent.example.com/adj/v0/calibration?agent_id=did:adp:test-runner-v2&domain=code.correctness
```

The response is the CalibrationScore triple:

```json
{ "value": 0.85, "sample_size": 312, "staleness": "P18D" }
```

Implementations MAY expose the full query contract (getDeliberation,
getConditionTrace, getOutcome) or restrict access to calibration queries only.
The minimum for federation is `getCalibration`.

### 13.2 Trust Without a Central Authority

An agent reports its own calibration. This is trustworthy because:

1. **Append-only guarantee.** Journal entries cannot be retroactively modified
   (Section 8). Hash chaining makes tampering detectable.
2. **External evidence.** Outcome entries reference external evidence
   (`evidence_refs` pointing at CI runs, incident tickets, monitoring
   signals). These are independently verifiable.
3. **Replayable logs.** Any peer can request the full deliberation record
   (via `getDeliberation`) and recompute the calibration score from the raw
   (confidence, outcome) pairs. If the recomputed score disagrees with the
   reported score, the agent is lying.
4. **Ground truth anchoring.** Outcomes marked `ground_truth: true` reference
   signals that neither the reporter nor the scored agent controls (CI
   systems, PagerDuty, human sign-off).

Trust is trust-but-verify: accept the reported score, spot-check by replaying
the log, discount agents whose scores don't survive replay.

### 13.3 Portability

An agent's journal is portable. When an agent moves to a new organization,
its JSONL files (or database export) move with it. The new organization's
ADP orchestrator queries the agent's journal endpoint and gets calibration
history earned elsewhere.

This is the advantage of per-agent journals over a shared substrate. A shared
journal ties agents to the organization that runs it. Per-agent journals with
a standard query contract let agents carry their track record across
deployments — the same way a developer carries their commit history across
employers.

PostMortem-style JSONL session files are literally the thing an agent serves
from its journal endpoint. The existing tool is the reference implementation
of the federated substrate, not just a local one.

### 13.4 Privacy and Selective Disclosure

Not all deliberation content should be shared across organizational
boundaries. Federation supports tiered disclosure:

| Level | What is shared | Use case |
|---|---|---|
| **Scores only** | CalibrationScore triple | Sufficient for weighting. No deliberation content crosses the boundary. |
| **Summaries** | Scores + condition quality metrics + deliberation counts | Enables auditing without content disclosure. |
| **Full records** | Complete deliberation logs with proposals and outcomes | Full transparency. Requires explicit consent. |

Implementations SHOULD default to scores-only for cross-organization queries
and full records for intra-organization queries.

---

*ADJ v0 is a draft specification. Feedback and implementation experience
will inform v1. File issues at the adj-manifest-spec repository.*
