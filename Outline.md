# Outline

ADJ v0 — outline
1. Purpose & non-goals. Purpose: define a portable, append-only record of agent deliberations that supports calibration scoring, audit, and cross-deliberation learning. Non-goals: storage backend, query language, retention policy, identity. ADJ defines the record format and the scoring contract; implementations choose their substrate (SQLite, Postgres, JSONL files à la PostMortem, event-sourced log, whatever fits). Explicitly compatible with the JSONL inflection-detection work — a journal can be a directory of JSONL files and still be conformant.
2. Terminology. Entry, deliberation record, outcome, calibration score, condition trace, scoring window, decision class, evidence ref. Tight glossary, ~10 terms.
3. The entry types. ADJ is a log of typed entries, not a relational schema. Five entry kinds:

deliberation_opened — references the ADP deliberation_id, lists participants and decision class
proposal_emitted — full proposal object (the append-only thing from ADP §3), versioned
round_event — vote, revise, abstain, challenge_tier, falsification — one entry per state-machine transition
deliberation_closed — terminal state (converged / partial-commit / deadlock), final tally, committed action
outcome_observed — the consequential entry — what actually happened after commit, written later (sometimes much later) by an outcome reporter

The split between deliberation_closed and outcome_observed is the load-bearing design choice: the journal records the decision and the result separately and asynchronously. Calibration is the function that joins them.
4. The outcome contract. This is where ADJ does the work ADP can't. An outcome entry binds a deliberation to a measurable result: { deliberation_id, observed_at, outcome_class, success: bool|float, evidence_refs[], reporter_id }. Outcomes can be binary (PR merged cleanly / caused regression), graded (incident resolved in N minutes vs estimated M), or null (no measurable outcome — the deliberation is recorded but doesn't feed calibration). The reporter is itself an agent with its own calibration on outcome reporting, which prevents the obvious gaming vector of self-reported success.
5. Calibration scoring. The math ADP §4 depends on. For each (agent, decision_class) pair, the journal computes a calibration score from the agent's historical (confidence, outcome) pairs using a proper scoring rule — Brier score by default, with log-loss as alternative. Returns the {value, sample_size, staleness} triple ADP's CalibrationSource interface expects. Spec defines the default scoring rule, the staleness clock (time since last scored deliberation in this class), and how new agents bootstrap. Decay is not applied here — that's ADP's job, per our earlier decision.
6. Condition quality scoring. The other thing ADP defers. For each agent, the journal tracks:

Falsification ratio: of dissent conditions published, what fraction were actually tested in their deliberations
Specificity score: heuristic measure of condition concreteness (presence of thresholds, named entities, measurable predicates)
Amendment frequency: how often conditions get amended mid-deliberation, which signals either poor initial specification or healthy responsiveness depending on whether amendments correlate with successful convergence

These are the metrics that catch the "p99 latency in a room with no metrics access" defection pattern. They don't gate participation, they just feed back into the agent's effective authority over time.
7. Query contract. Not a query language — a minimum set of queries any conformant journal must answer:

getCalibration(agent_id, decision_class) -> CalibrationScore
getDeliberation(deliberation_id) -> full record
getConditionTrace(agent_id, window) -> condition quality metrics
getOutcome(deliberation_id) -> outcome | null

Implementations expose these however they like (HTTP, gRPC, library calls, MCP tool). The contract is the surface ADP and other consumers code against.
8. Append-only and integrity. ADJ entries MUST be append-only. Spec recommends but doesn't mandate hash-chaining (each entry references the hash of the previous entry in its deliberation), because some implementations will want it for tamper-evidence and others won't need it. Either way, amendments are new entries that reference prior entries by id, never mutations of existing entries. This is the property that makes the journal trustworthy as a calibration substrate — you can't retroactively improve your track record.
9. Worked example. Same PR-merge deliberation from ADP §8, but shown as the journal sees it: the sequence of entries written from deliberation_opened through outcome_observed three days later when CI shows no regression. Reader sees how the same event stream that drove the ADP state machine becomes the data structure that scores the participating agents. This is the moment the two specs visibly compose.
10. Compliance levels. L1: write conformant entries. L2: serve the query contract. L3: compute calibration and condition-quality scores. Lets a journal exist as just a log (L1) before becoming a scoring engine (L3). Matches ADP's own tiered compliance.
11. Relationship to other specs. ADP consumes ADJ via the CalibrationSource interface. mcp-manifest declares that an agent participates in ADJ and points at its journal endpoint (or declares it writes to a shared one). PostMortem-style JSONL session journals are a valid L1 implementation — explicitly called out, because that's the bridge to your existing work and the credibility anchor for the spec.
12. Open questions. Who reports outcomes when no agent is naturally positioned to? (Outcome-reporter agents as a role.) How are multi-deliberation outcomes attributed when several decisions contributed to one observed result? (Attribution weights, deferred to v0.1.) How do journals federate across organizations without leaking proprietary deliberation content? (Hashed summaries + selective disclosure, definitely v0.1+.)