# Context Infrastructure for AI-Assisted Development
## High-Level Design

**Status:** Draft v0.1.1
**Author:** Greg Blotzer
**Date:** May 2026
**Related work:** "The Case for Purpose-Built Context Infrastructure in AI-Assisted Development" (LinkedIn, 2026)

---

## Executive Summary

### Problem

AI-assisted software development at team scale has a context infrastructure gap. The current generation of tools — CLAUDE.md files, session-local memory, RAG over raw documents — was designed for individual developers working alone with a single agent. None of them adequately address what happens when a team of developers, each running multiple agent sessions per day, needs to share, govern, and build on accumulated organizational knowledge over months and years.

The symptoms are familiar to anyone working in agentic development today: context is re-established from scratch at every session, knowledge captured by one developer's session is invisible to the next, agent-written memory files accumulate without curation or trust signals, and there is no mechanism to distinguish a foundational architectural constraint from a junior developer's tentative hypothesis. The cost of this gap is measured in tokens (context re-establishment at every session startup) and in correctness (agents repeating decisions that have already been made and rejected).

### Architectural Premise

This document specifies a belief graph: a structured, multi-dimensional, trust-tiered representation of organizational knowledge derived from AI-assisted development sessions. Beliefs are first-class entities with explicit provenance, lifecycle states, dimension scores (recency, confidence, corroboration, trust level, scope precision, contradiction history, chain integrity), and a human-in-the-loop promotion interface that governs what becomes authoritative.

The premise can be stated negatively: organizational AI context is not a configuration file. It is not a search corpus. It is not a chat history. It is a curated knowledge graph with the same engineering rigor that production databases receive — schema, indexes, consistency model, access control, audit trail, retention policy, migration path. This design treats it that way. The infrastructure is the substrate for AI-assisted development at team scale, not a layer added on top of existing agent orchestration to make it accountable.

### Load-Bearing Design Properties

Five properties carry the architecture's weight. Each is specified in the body of the document with mechanisms, failure modes, and empirically determined parameters explicitly identified:

- **Multi-dimensional scoring with a trust-level hard gate.** Conflict resolution uses a weighted composite of seven dimensions, but trust level functions as a hard gate that prevents low-trust beliefs from automatically overriding high-trust beliefs regardless of other dimension scores. This is the architectural defense against poisoning attacks that maximize content quality to override provenance quality.
- **Aliasing rather than absorption in deduplication.** Deduplicated beliefs become retired alias nodes with supersession edges, not deletions. The reasoning chain integrity for any belief that referenced the deduplicated belief is preserved, and the audit trail is complete.
- **Corroboration semantics that distinguish independent rederivation from confirming echo.** Trust elevation depends on independent rederivations alone; confirming corroborations contribute to scoring at a discounted weight but do not elevate trust. This makes peer-validated trust epistemically meaningful at any agent-to-human ratio.
- **Human-in-the-loop promotion at the boundary of authoritative knowledge.** Agents contribute candidates; humans govern what becomes organizationally authoritative. The promotion interface is the explicit governance boundary between individual learning and collective knowledge.
- **Self-hostable by design.** The context store holds organizational knowledge that may be more sensitive than the codebase itself. Vendor-hosted deployment is a convenience option, not the default.
- **Systems, not agents, in the infrastructure.** The infrastructure is maintained by deterministic systems operations rather than by agents. Agents are content producers whose session output is processed by the infrastructure; they are not components of the infrastructure itself. This commitment is what makes the cost model favorable, the performance targets credible, and the security posture defensible — and it distinguishes this design from contemporary work using similar primitives that takes the opposite position.

### Cost Case

The system shifts cost from per-session token consumption to per-belief pipeline operations. Read-side displacement is large and scales with session activity; write-side addition scales with belief production and is bounded by extraction selectivity. The shift is favorable across deployment regimes, with two distinct cost stories:

- At human-favoring agent ratios (below ~1:3), the cost case is unambiguous — read displacement dominates write addition by an order of magnitude or more.
- At agent-favoring ratios (above ~1:3), the cost case depends on extraction selectivity. Architectural mechanisms that function as quality controls at low ratios function as economic controls at high ratios. Without selective extraction, write-side cost grows faster than read-side displacement.

The cost displacement requires that the belief graph be trusted as authoritative. Normalization quality is on the critical path for the cost case to hold.

### Security Posture

The belief graph will be attacked. This is treated as a certainty, not a risk to be mitigated to acceptable levels. The defensive architecture specifies cryptographic provenance chains, anomaly detection on dimension velocity and trust drift, multi-layer throttling, scope compartmentalization, and a migration review gate that prevents pre-migration contamination of source files from being imported as ground truth.

The conventional security argument that distribution reduces risk does not apply to this artifact. The distributed local model for AI session context — N developer machines each holding unmanaged organizational knowledge — creates a larger, less visible, and less mitigable attack surface than a well-designed centralized belief graph. The 2026 GitHub TeamPCP breach demonstrated this attack vector in production: a single poisoned developer tool, targeting AI configuration files with zero-width Unicode injection, gave attackers access to 3,800 internal repositories.

### What Remains Empirically Determined

The design specifies its calibration parameters explicitly rather than burying them as implementation details. Dimension weights, threshold values, throttling rates, the regime crossover ratio between human-favoring and agent-favoring deployments, the normalization model cost-quality tradeoff, the corroboration discount factor, and the read displacement ratio achievable in practice are all flagged as requiring empirical calibration against deployment data. The design is complete; the parameters are not, and pretending otherwise would be intellectually dishonest.

### Relationship to Adjacent Work

This design is closest to Karpathy's LLM Wiki pattern in its core premise — that pre-synthesized structured context is a better substrate for LLM reasoning than retrieval over raw sources — but generalizes that premise from personal scale to team scale, where conflict resolution, trust derivation, provenance integrity, and human-in-the-loop promotion become load-bearing requirements that the personal-scale design does not need to address. MEMO and Understand Anything are validating prior work; this document extends them into team-scale governance. Claude Code team memory sync addresses the simpler problem of session continuity for individual developers within a single tool.
Semantica (Hawksight-AI, v0.4.0 released April 2026) is the strongest contemporary work using similar architectural primitives — context graphs, provenance chains, decision tracking, semantic deduplication, temporal snapshots. The convergent use of these primitives across independent design efforts suggests they are the architecturally correct substrate for this category of work. The architectural disagreement is about where agents belong in the infrastructure: Semantica positions as an accountability layer added on top of existing agent orchestration with agentic skills performing extraction, validation, and reasoning inside the infrastructure; this design positions the infrastructure as the substrate with deterministic systems operations performing the same functions. Section 1.4 develops the engagement in full.

---

## 1. System Overview

### 1.1 Purpose

This document describes the high-level architecture of a purpose-built context infrastructure system for AI-assisted software development. The system provides persistent, shared, governed storage of the knowledge that accumulates when developers and AI agents work together on a codebase — knowledge that currently evaporates at session boundaries, siloes on individual machines, and is irreconcilable across team members.

The system is not a replacement for source control. Code lives in Git. The context infrastructure is a companion layer that captures the reasoning that produced the code — the decisions made, the constraints learned, the approaches tried and rejected — and makes that reasoning available to future sessions across the entire team.

### 1.2 Scope

This design covers:

- Extraction of belief candidates from raw AI session data
- Storage and maintenance of a structured belief graph
- Merge strategies for reconciling divergent belief states
- Provenance tracking from raw session to belief graph entry
- The human-in-the-loop promotion interface
- Security model and trust architecture
- Integration with existing AI coding tools
- Storage tiering and latency requirements

This design does not cover:

- Model training or fine-tuning
- Source control or code review workflows
- Specific vendor implementations of AI coding assistants
- The full position paper argument for why this infrastructure is needed (see related work above)

### 1.3 Guiding Principles

**Context is a first-class artifact.** The reasoning that produces code is as valuable as the code itself. The system treats context with the same rigor applied to source control — versioned, auditable, and governed.

**Incremental over periodic.** Context accumulates belief by belief across sessions rather than being regenerated from scratch. This keeps costs manageable and keeps the belief graph current.

**Human judgment at the promotion boundary.** Agents contribute to the belief graph but human review governs what becomes authoritative organizational knowledge. The promotion interface is where individual learning becomes collective knowledge.

**Self-hostable by design.** The context store holds organizational knowledge that may be more sensitive than the codebase itself. Organizations must be able to run this infrastructure on their own hardware under their own governance policies. Vendor-hosted deployment is a convenience option, not the default.

**Cost shift, not cost addition.** The system displaces context establishment cost from session-tier operations to write-path pipeline operations. The displacement is favorable across deployment regimes but depends on extraction selectivity at agent-favoring ratios. Honest cost accounting is preferable to claiming the system is free.

**Properties over implementations.** This document specifies what the system must do and what constraints it must satisfy. Specific technology choices (database vendors, serialization formats, transport protocols) are implementation decisions that should be driven by empirical data from real deployments.

**Systems, not agents, in the infrastructure.** The infrastructure for AI-assisted development is maintained by deterministic systems operations, not by agents. Agents are content producers whose session output is processed by the infrastructure; they are not components of the infrastructure itself. This commitment is what makes the cost model favorable (write-path operations are bounded systems cost rather than agent cost), what makes the performance targets credible (database operations are sub-100ms; agent operations are not), and what makes the security posture defensible (the trust-derivation perimeter is a systems boundary that agent-produced content cannot cross). Contemporary work using similar primitives makes the opposite choice; see Section 1.4 for the engagement with that work.

### 1.4 Relationship to Adjacent Systems

**MEMO (MIT/NUS/Singapore-MIT Alliance, 2026):** The closest existing research system. Validates the architectural direction of separating memory from reasoning and using a dedicated memory model as a queryable knowledge store. MEMO is designed for static document corpora; this system extends that direction to handle dynamic session accumulation, team sharing, human-in-the-loop promotion, and interpretable provenance. The MEMO authors explicitly identify attribution mechanisms and access controls as future work; this design addresses both.

**Understand Anything (Lum1104, 2026):** An open-source Claude Code plugin that uses a multi-agent pipeline to synthesize a knowledge graph from static codebase analysis. Validates the knowledge graph as the right representational primitive for codebase understanding. Addresses the orientation problem (what is this codebase) rather than the accumulation problem (what has the team learned about this codebase over time). The knowledge-graph.json schema serves as a reference for the belief graph node and edge structure, extended with dimension scores, provenance pointers, and belief lifecycle properties.

**Claude Code team memory sync (Anthropic, 2026):** A flat key-value store with server-wins conflict resolution, scoped to repository and synchronized to Anthropic-hosted infrastructure. Addresses session continuity for individual developers within a single tool. Does not address team-scale merge semantics, provenance, structured belief representation, or organizational control of the context store.

**Karpathy's LLM Wiki (2026):** A widely-discussed personal knowledge base pattern in which an LLM maintains a synthesized markdown wiki over raw source files, governed by a natural-language `agents.md` instruction file. Validates the architectural direction of separating raw input from structured working representation and delegating curation to the agent. Optimized for the single-user case where there is no merge problem, no adversarial threat model, and no need for team-scale governance. This design generalizes the same architectural premise — pre-synthesized structured context as the working substrate — to multi-contributor team contexts, where conflict resolution, trust derivation, provenance integrity, and human-in-the-loop promotion become load-bearing requirements that the personal-scale design does not need to address.

**Semantica (Hawksight-AI, 2026):** The strongest contemporary work using similar architectural primitives — context graphs, provenance chains (W3C PROV-O compliant), decision tracking, semantic deduplication, and temporal snapshots applied to AI agent infrastructure. Production v0.4.0 released April 2026. The convergent use of these primitives across independent design efforts suggests they are the architecturally correct substrate for this category of work.
The architectural disagreement lies elsewhere. Semantica's integration model places agents inside the infrastructure: extraction, ingestion, validation, deduplication, and reasoning are performed by agentic skills added to the host environment (Claude Code, Cursor, Codex, and others). The system is positioned as an accountability layer added on top of existing agent orchestration — agents do their work and Semantica captures provenance so the work can be explained and audited.
This design takes the opposite position, articulated as a guiding principle in Section 1.3. Agents produce session content as their primary work; the infrastructure ingests, normalizes, deduplicates, and resolves conflicts as deterministic systems operations. The choice is consequential. Agent-maintained infrastructure inherits the cost, latency, reliability, and failure-mode characteristics of agents — including hallucination, prompt injection susceptibility, and non-deterministic decision-making — and propagates them into the maintenance layer of the context system. Systems-maintained infrastructure has the operational characteristics of conventional software services: deterministic, auditable, performance-predictable. The cost model in Section 8, the performance assumptions in Section 4.7.7, the trust derivation in Section 4.5, and the security posture in Section 9 all depend on the systems-not-agents commitment. Both projects address real problems with overlapping primitive toolkits; they make opposite architectural choices about where agents belong in the infrastructure stack.

---

## 2. Primitives

The system is built on five primitives. All higher-level concepts are derived from these.

### 2.1 Session

A bounded interaction between a contributor and an AI coding assistant, associated with a specific project scope, a contributor identity, and a defined start and end time.

**Properties:**
- `session_id` — globally unique identifier
- `contributor_id` — authenticated identity of the human participant
- `project_scope` — the repository or project context within which the session occurred
- `started_at` — ISO 8601 timestamp
- `ended_at` — ISO 8601 timestamp
- `tool` — the AI coding assistant used (Claude Code, Cursor, Copilot, etc.)
- `raw_log_ref` — pointer to the raw session log in the exchange store

The session is the unit of provenance. Every belief candidate can be traced to the session that produced it.

### 2.2 Exchange

A single turn within a session — one developer message and one agent response, with associated tool calls and results.

**Properties:**
- `exchange_id` — unique within the session
- `session_id` — parent session reference
- `sequence_number` — position within the session
- `timestamp` — when the exchange occurred
- `developer_message` — the developer's input
- `agent_response` — the agent's output
- `tool_calls` — any tool calls made during the exchange, with inputs and outputs
- `belief_candidates_extracted` — list of belief candidate IDs extracted from this exchange

The exchange is the unit of extraction. Belief candidates are extracted from exchanges, not from sessions as a whole.

### 2.3 Memory Entry

An explicitly persisted piece of information, either agent-written to a memory file or human-promoted to a shared context document. This is the current state of the art — CLAUDE.md entries, LangGraph checkpoints, team memory sync entries.

**Properties:**
- `entry_id` — unique identifier
- `content` — the persisted text
- `source` — agent-written or human-promoted
- `project_scope` — the scope to which this entry applies
- `created_at` — timestamp
- `raw_source_ref` — pointer to the session or exchange that produced it, if available

Memory entries are inputs to the extraction pipeline. The system can ingest existing memory entries from current tooling as a migration path.

### 2.4 Project Scope

The codebase or repository context within which beliefs are valid. Beliefs do not automatically transfer across project scopes.

**Properties:**
- `scope_id` — unique identifier
- `name` — human-readable project name
- `repository_ref` — pointer to the source repository
- `parent_scope` — optional reference to a parent scope (for monorepos or related projects)
- `access_policy` — which contributor identities can read and write to this scope

The project scope is the unit of partitioning for the belief graph. Merge operations are scoped to a project scope. Cross-scope promotion requires explicit authorization.

### 2.5 Contributor

The authenticated identity associated with a session — human or agent.

**Properties:**
- `contributor_id` — unique identifier
- `type` — human or agent
- `identity_ref` — pointer to the authentication system record
- `trust_history` — summary of this contributor's belief contribution track record
- `role` — the contributor's role within the project scope (developer, reviewer, agent, pipeline)

The contributor is the unit of trust. Dimension scores in the belief graph are partly a function of contributor history. Agent contributors are treated differently from human contributors in the trust model.

---

## 3. Layer 1: Context Snapshot

### 3.1 Responsibility

The context snapshot layer extracts structured belief candidates from raw session data. It transforms the unstructured, chronological record of a session into discrete, typed, scoped belief candidates that can be evaluated for promotion into the belief graph.

### 3.2 Inputs and Outputs

**Input:** Raw session logs in JSONL format from AI coding assistant session directories (e.g., `~/.claude/projects/`), supplemented by existing memory entries from current tooling.

**Output:** Structured belief candidate objects with initial dimension scores, typed content, scope assignments, normalized declarative content, and raw provenance pointers.

### 3.3 Pipeline Flow

The extraction pipeline runs asynchronously — never in the session critical path. Pipeline operations do not block active session exchanges.

#### 3.3.1 Trigger Model

The extraction pipeline is triggered by three mechanisms in priority order:

**Compaction events (primary).** AI coding assistant context compaction events are the natural primary trigger. Compaction already represents a session boundary where the agent's working context is summarized and reset. This is the lowest-friction moment to both process outbound belief candidates and pull inbound belief graph updates. The compaction event becomes the natural unit of sync — sessions don't have a defined end for sync purposes, they have a series of compaction checkpoints. A developer working in one long session syncs at every compaction event regardless of session duration. This sidesteps the session boundary problem almost entirely.

**Time-based window (fallback).** For sessions where compaction does not occur within a configurable window (default: 2 hours), the pipeline processes exchanges from the elapsed window on a rolling schedule. This handles long sessions that compact infrequently without requiring developer action.

**On-demand (explicit).** A developer-invoked command triggers immediate extraction and merge, independent of compaction or time window. Used before handoff to a teammate, before a major merge event, or when a critical upstream belief is promoted and the developer wants it reflected immediately.

#### 3.3.2 Sync Cycle at Compaction

At each compaction event, the pipeline executes a two-phase sync cycle:

**Phase 1 — Outbound.** Process exchanges from the session window since the last sync checkpoint. Extract belief candidates, normalize content, submit candidates to the merge pipeline. The sync cursor for this session is updated to the current graph state after submission.

**Phase 2 — Inbound.** Pull the belief graph delta since the last sync cursor. Apply delta to the local session cache. If the delta contains beliefs that conflict with beliefs the current session has been working under, surface the conflict to the developer as a notification before the compaction completes. The developer decides whether to accept the incoming belief, hold their current working belief, or flag for team review. Conflicts are not resolved automatically.

The outbound phase runs first so that if the session produced a belief that conflicts with something already in the graph, the conflict is visible in the inbound phase and surfaced to the developer in a single notification.

**Delta pull, not full reload.** The inbound pull retrieves only the belief graph delta since the last sync cursor, not the full graph state. This keeps the compaction window short and predictable. The sync cursor is a pointer to the last known graph state; the delta contains only what changed since that pointer.

#### 3.3.3 Pipeline Stages

For each triggered extraction window, the pipeline executes the following stages in sequence:

1. **Log ingestion.** Read JSONL records from the session log for the extraction window. Validate record structure. Route malformed records to the quarantine queue. Continue processing valid records.

2. **Exchange classification.** Apply extraction heuristics to each valid exchange. Classify exchanges by heuristic type. Record confidence score for each classification. Exchanges below the minimum classification confidence threshold are discarded and logged.

3. **Multi-heuristic resolution.** Exchanges that match multiple heuristics produce multiple belief candidates, one per matching heuristic type. All candidates from the same exchange share a `source_exchange_id` and reference each other via a `sibling_candidates` array. This preserves the co-production context for provenance purposes.

4. **Scope inference.** For each candidate, infer scope from the files open during the exchange, components referenced in the conversation, and explicit scope signals in the developer's messages. Candidates where scope cannot be determined with sufficient confidence receive a null scope and are routed to the human review queue for manual scope assignment after normalization.

5. **Content normalization.** Rewrite raw exchange content into normalized declarative belief statements using an LLM-assisted normalization process (see Section 3.5). Record a `normalized_confidence` score. Low-confidence normalizations are flagged and include the raw exchange content in the review queue entry so reviewers can correct the normalization.

6. **Schema validation.** Validate each belief candidate against the candidate schema. Schema validation failures are quarantined with classification.

7. **Deduplication check.** Check each candidate against existing beliefs in the graph and pending candidates in the current batch using approximate nearest neighbor search on belief embeddings. Near-duplicates are merged rather than submitted as separate candidates (see Section 5.5 for deduplication details).

8. **Candidate submission.** Submit validated, deduplicated candidates to the merge evaluator. The merge evaluator is responsible for promotion decisions (see Section 5).

#### 3.3.4 Idempotency

If the same session window is submitted for extraction more than once — due to retry, duplicate trigger, or operator action — the pipeline checks for existing candidates with matching `source_session_id` and `source_exchange_id` before creating new ones. Reprocessing a session window produces no duplicate candidates. This is a hard guarantee enforced at the pipeline level, not a best-effort behavior.

### 3.4 Extraction Heuristics

Not all exchanges contain belief-relevant content. The extraction process identifies high-signal exchanges using the following heuristics, applied in priority order. An exchange may match multiple heuristics and produces one candidate per match.

**Correction events.** Exchanges where the developer explicitly contradicts or redirects the agent. Signal phrases include "that's wrong," "actually," "no, we don't," "we decided not to," and similar explicit corrections. These are the highest-signal belief candidates because they represent direct human assertions of ground truth. Initial confidence: 0.5 (elevated above default due to explicit human correction signal).

**Constraint assertions.** Exchanges where the developer states something about how the system works, what patterns are used, or what approaches are prohibited. Often appear as context-setting at the start of a session or in response to agent suggestions that violate unstated constraints. Initial confidence: 0.4.

**Decision records.** Exchanges where an approach is chosen over alternatives with the reasoning made explicit. The agent proposes multiple options, the developer selects one and explains why. These map directly onto ADRs and are the most valuable for provenance. Initial confidence: 0.4.

**Rejection events.** Exchanges where an approach is dismissed, particularly with reasoning. Capturing why something was rejected is as important as capturing what was chosen. Initial confidence: 0.35.

**Tacit knowledge elicitation.** Exchanges where the developer explains something to the agent that they haven't written down elsewhere — how a component actually works, what a pattern is for, why something is structured the way it is. These often appear in response to agent questions or misunderstandings. Initial confidence: 0.3.

**What remains empirically determined.** The specific signal phrases for each heuristic, the minimum classification confidence threshold, the initial confidence values per heuristic type, and the relative weighting of heuristic types in downstream scoring. The schema and heuristic categories are fixed; their thresholds are calibrated from deployment data.

### 3.5 Content Normalization

Raw exchange content is rewritten into normalized declarative belief statements before being stored as belief candidates. Normalization runs as an LLM-assisted process using a smaller, faster model than the session model. It runs asynchronously as part of the extraction pipeline and never in the session critical path.

#### 3.5.1 Required Properties of Normalized Content

**Self-contained.** The normalized statement must be readable and meaningful without reference to the session that produced it. No pronouns without clear referents, no implicit references to prior conversation context, no "the above approach" or "as mentioned earlier."

**Declarative and system-centric.** Stated as a fact about the system, not as a conversational fragment. "The project's Lambda functions have a 15-second timeout limit" not "I told Claude the timeout is 15 seconds" or "yeah so the timeout thing."

**Heuristic-type-aware.** Normalization produces different statement forms for different heuristic types. A constraint assertion normalizes as a rule ("X is required / prohibited / preferred"). A rejection event normalizes as a record of what was tried and why it failed ("Approach X was rejected because Y"). A decision record normalizes as a choice with reasoning ("Approach X was selected over Y because Z"). The normalization model receives the heuristic type classification as part of its prompt.

**Specific.** Values, names, version numbers, constraint boundaries, and other specific content from the original exchange must be preserved. "IAM wildcards are prohibited in all policies" is not equivalent to "there are some IAM restrictions." Normalization must not soften, generalize, or omit the specific content of the original exchange.

#### 3.5.2 What Normalization Preserves and Discards

| Preserves | Discards |
|---|---|
| Semantic content of the exchange | Conversational fillers and hedges |
| Specific values, names, constraints | Personal pronouns and voice |
| Type classification | Present-tense conversational framing |
| Provenance pointer to raw exchange | Irrelevant context from adjacent exchanges |
| Scope assignment | Session-specific shorthand |

#### 3.5.3 Normalization Quality Signals

The normalization process produces a `normalized_confidence` score (float, 0.0–1.0) representing the normalization model's self-reported confidence that the output correctly represents the original exchange. This score is one signal among several, not the sole quality gate.

**Why confidence alone is insufficient.** LLM-produced confidence scores are imperfectly calibrated. Models can be overconfident on subtly-wrong outputs and underconfident on stylistically-unusual-but-correct outputs. Treating a single confidence number as the routing gate produces both false negatives (low-quality normalizations reach the merge evaluator) and false positives (high-quality normalizations are unnecessarily routed to human review). Calibration improves with model maturity but never reaches the point where a single score is sufficient.

**Complementary quality signals.** The normalization pipeline computes three additional signals alongside `normalized_confidence`:

- **Specific-value preservation check.** Numeric values, version strings, identifiers, and named entities from the raw exchange must appear in the normalized output. Failure indicates lossy normalization regardless of confidence score. This check is deterministic and fast.
- **Raw-to-normalized embedding similarity floor.** The cosine similarity between the raw exchange embedding and the normalized output embedding must exceed a configurable floor. A normalized output that is semantically distant from the raw exchange is a hallucination indicator, even when self-reported confidence is high.
- **Multi-pass agreement (optional, cost-gated).** For high-stakes belief types (constraint, decision), the normalization pipeline may run two independent passes and compare outputs. Strong disagreement between passes routes to human review regardless of either pass's individual confidence. This signal is most useful at low candidate volumes; in agent-favoring regimes it must be gated by extraction selectivity to remain economical.

**Routing decision.** A candidate is routed to human normalization review if any of: `normalized_confidence` below threshold, specific-value-preservation check fails, embedding similarity below floor, or multi-pass agreement (when used) shows strong disagreement. Routing on any-signal-fails is conservative by design — false positives in the routing decision are operational cost, false negatives are silent quality degradation in the belief graph.

**Raw exchange preservation.** Raw exchange content is retained in the provenance store regardless of any quality signal value. The normalization is the working representation; the raw exchange is the archival record. This preservation is what enables re-normalization (Section 3.5.4) without loss of source material.

#### 3.5.4 Normalization Model Lifecycle

Normalization model versions are part of the system's correctness boundary, not a deployment detail. A model upgrade changes the normalized form of equivalent raw exchanges, which affects content hashing, deduplication, and the dedup fast path. The architecture must accommodate model upgrades without requiring full graph rebuilds.

**Identity is a UUID, not a content hash.** Belief identity is a stable UUID assigned at first normalization. The normalized content is a property of the belief, not the identity of it. This matches how database-of-record systems handle entities whose representation may evolve over time. A belief whose normalized content is re-generated by a newer model retains its UUID; the prior normalized form is preserved in provenance. This is a foundational design choice that ripples through deduplication (Section 5.5) and merge decision evaluation (Section 5.6.2).

**Per-belief model version recording.** Every belief candidate and every promoted belief records the `normalization_model_version` used to produce its current normalized content. Provenance records capture the model version at each normalization event, enabling forensic reconstruction of which beliefs were produced under which model.

**Re-normalization strategies.** Two strategies are supported, and deployments should choose based on belief graph maturity and operational tolerance for migration cost:

- **Eager re-normalization (for hot tier).** Beliefs in the hot storage tier (Section 4.6) are re-normalized in batches over a configurable migration window after a model upgrade. Each re-normalization is treated as a content update on the existing belief, not a new belief. The prior normalized form is captured in provenance. Eager re-normalization is appropriate for the hot tier because hot-tier beliefs are actively queried and their normalized form is on the critical path for session context quality.
- **Lazy re-normalization (for warm and cold tiers).** Beliefs in warm and cold tiers are re-normalized on access. The first query that surfaces a belief produced under a prior model version triggers re-normalization, which is performed asynchronously while the prior content serves the immediate query. Lazy re-normalization avoids the cost of migrating beliefs that may never be accessed again.

**Hybrid as default.** Production deployments should default to hybrid — eager for hot, lazy for warm and cold — and adjust based on operational data. Tier-uniform strategies are available but rarely optimal.

**Migration completeness.** A model upgrade is considered complete when no hot-tier beliefs remain on the prior model version, regardless of warm and cold tier state. Operations dashboards must expose per-model-version belief counts segmented by tier, and the migration window must be configurable per deployment.

**Cost interaction.** Re-normalization is a write-path cost (Section 8.6). Eager strategies trade upfront migration cost for predictable post-migration quality; lazy strategies smooth the cost over time at the price of inconsistent quality during the transition. The choice is also a security choice: under prolonged eager re-normalization, the pipeline is processing high volumes of low-novelty content, which raises the noise floor for anomaly detection (Section 9.3.3) and may require temporary calibration adjustments.

### 3.6 Belief Candidate Schema

```json
{
  "candidate_id": "uuid",
  "content": "normalized declarative belief statement",
  "raw_content_ref": "pointer to original exchange content in provenance store",
  "normalized_confidence": 0.85,
  "normalization_model_version": "model-id@version",
  "normalization_quality_signals": {
    "specific_value_preservation": true,
    "embedding_similarity": 0.91,
    "multi_pass_agreement": null
  },
  "scope": {
    "project_scope_id": "uuid",
    "component": "optional — module, service, or file path this belief applies to",
    "level": "project | component | function",
    "inferred": true,
    "inference_confidence": 0.75
  },
  "type": "constraint | decision | pattern | rejection | observation",
  "source_session_id": "uuid",
  "source_exchange_id": "uuid",
  "sibling_candidates": ["candidate_id", "candidate_id"],
  "extracted_at": "iso8601",
  "raw_provenance_ref": "pointer to original JSONL record",
  "initial_dimensions": {
    "recency": 1.0,
    "confidence": 0.4,
    "corroboration": 1,
    "trust_level": "agent_derived",
    "scope_precision": 0.75,
    "contradiction_history": []
  }
}
```

**Schema changes from initial draft:** Added `raw_content_ref`, `normalized_confidence`, `normalization_model_version`, `normalization_quality_signals`, `scope.inferred`, `scope.inference_confidence`, and `sibling_candidates`. Removed `trust_level` enum values that belong to the belief graph layer rather than the candidate layer. Scope precision is now computed from the scope inference confidence rather than defaulting to 0.0. The `normalization_model_version` and `normalization_quality_signals` fields support the lifecycle and quality model described in Sections 3.5.3 and 3.5.4. `multi_pass_agreement` is null when the multi-pass check did not run for this candidate type.

### 3.7 Failure Modes

The extraction pipeline defines explicit behaviors for all failure cases. Silent drops are never acceptable — every failure produces a record.

| Failure Type | Behavior | Recovery Path |
|---|---|---|
| JSONL parse error | Quarantine raw record with error classification, continue processing | Administrator review queue |
| Schema validation failure | Quarantine candidate with error classification, continue processing | Administrator review queue |
| Scope inference failure | Produce candidate with null scope and low scope precision | Human review queue for manual scope assignment |
| Normalization quality signal failure | Flag candidate on any signal failure (low confidence, specific-value preservation failure, embedding similarity below floor, or multi-pass disagreement), attach raw exchange content and the specific failing signal | Human review queue for normalization review |
| Duplicate session submission | Idempotent — no duplicates produced, existing candidates unchanged | No recovery needed |
| Pipeline infrastructure failure | Queue session window for retry with exponential backoff | Dead letter queue after configurable maximum retries |
| Normalization model unavailable | Queue candidates for normalization retry, do not submit unnormalized candidates to merge evaluator | Retry when model available |

**Dead letter queue.** Sessions that exhaust retry attempts without successful processing enter the dead letter queue. Dead letter entries do not block processing of other sessions. Administrators are notified of dead letter entries on a configurable cadence. Manual reprocessing is available after the underlying issue is resolved.

**Quarantine queue.** Malformed records and schema validation failures accumulate in the quarantine queue. Quarantine entries are reviewed by administrators on a regular cadence. Patterns in quarantine failures — sudden increases, concentration in specific tools or session types — are signals for extraction pipeline maintenance.

### 3.8 Key Design Decisions

**Compaction as sync checkpoint.** Using compaction events as the primary extraction trigger decouples sync from the ambiguous concept of session end. Sessions have compaction checkpoints, not ends. This is an architectural decision that simplifies the timing model significantly.

**Asynchronous pipeline.** All extraction pipeline operations run asynchronously. The session critical path is never blocked by extraction, normalization, or submission. This is a hard constraint — extraction latency must not degrade session performance.

**LLM-assisted normalization.** Normalization requires semantic understanding that rule-based approaches cannot reliably provide. The normalization model is a separate, lighter-weight model from the session model, running asynchronously. Model selection, prompt design, and confidence calibration are implementation decisions; the requirement for LLM assistance and asynchronous execution are architectural constraints.

**Multi-heuristic produces multiple candidates.** An exchange matching multiple heuristics produces multiple typed candidates linked by `sibling_candidates`. The alternative — a single candidate with multiple types — would complicate the merge strategy and make type-specific querying unreliable. The volume cost of multiple candidates is accepted in exchange for clean type semantics.

**Raw content always preserved.** Normalized content is the working representation. Raw exchange content is always preserved in the provenance store regardless of normalization confidence. Normalization is lossy by design; the archival record is not.

**What remains empirically determined.** Extraction heuristic thresholds and signal phrases, normalization confidence threshold, scope inference confidence threshold, time-based window duration, retry backoff parameters, and dead letter queue notification cadence. All are configurable parameters requiring calibration from deployment data.

---

## 4. Layer 2: The Belief Graph

### 4.1 Responsibility

The belief graph is the central data structure of the system — the shared, persistent, governed store of what the team and its agents know about a project. It receives promoted belief candidates from the extraction layer, maintains their dimension scores over time, responds to queries from agent sessions and human reviewers, and enforces the lifecycle transitions that keep the active graph current and trustworthy.

### 4.2 Node Types

**Belief node.** The primary entity. Represents a single discrete piece of project knowledge — a constraint, a decision, a pattern, a rejection, or an observation. Each belief node carries the full dimension schema, a lifecycle state, and a reference to its provenance chain.

**Contributor node.** Represents a human or agent contributor. Connected to belief nodes via contribution edges that record the nature of the contribution (authored, confirmed, contradicted, retired).

**Session node.** Represents a session in which beliefs were generated or modified. Connected to exchange nodes and to belief nodes via provenance edges.

**Scope node.** Represents a project scope or component scope. Belief nodes are connected to scope nodes to enable scope-partitioned queries.

### 4.3 Edge Types

**Provenance edge.** Connects a belief node to the session and exchange that produced it. Directed, immutable once written. The provenance chain is the audit trail.

**Confirmation edge.** Connects a contributor node to a belief node when the contributor's session produces a candidate that deduplicates to the belief. Each confirmation edge carries a `confirmation_type` property — either `confirming` (the contributor's session had the belief in its inbound context) or `independent` (the contributor's session produced the candidate without prior knowledge of the belief). Classification is determined at merge evaluation time, not asserted by the contributor (see Section 5.5). Independent confirmations are the strong evidence signal; confirming confirmations contribute at a discounted weight (see Section 4.5).

**Contradiction edge.** Connects a contributor node to a belief node when the contributor explicitly contradicts the belief. Decrements confidence, adds to contradiction history, and may trigger human review routing.

**Supersession edge.** Connects a newer belief node to an older one when the newer belief replaces the older. The superseded belief is retired but the supersession edge preserves the relationship for provenance traversal.

**Dependency edge.** Connects related beliefs — a decision belief referencing the constraint beliefs that drove it, a rejection belief referencing the approach that was rejected and the constraint that made it unacceptable.

### 4.4 Belief Lifecycle

Each belief node has a defined lifecycle state. Lifecycle state governs queryability, weight in agent reasoning, and storage tier placement.

#### 4.4.1 States

| State | Description | Queryable | Agent Weight | Storage Tier |
|---|---|---|---|---|
| Candidate | Extracted, awaiting merge evaluation | No | N/A | Extraction pipeline |
| Provisional | Promoted at lowest trust tier | Yes | Low, flagged | Hot |
| Active | In active graph above provisional | Yes | Normal | Hot |
| Flagged | Conflict or anomaly detected | Yes | Reduced, flagged | Hot |
| Retired | No longer active organizational knowledge | Audit only | None | Warm |
| Archived | Long-term retention | Audit only | None | Cold |

#### 4.4.2 Valid Transitions

```
Candidate ──→ Provisional     (merge decision: promote)
Candidate ──→ [discarded]     (merge decision: reject — logged, never enters graph)
Provisional ──→ Active        (trust tier elevation via confirmation or policy)
Active ──→ Flagged            (conflict detected or anomaly triggered)
Flagged ──→ Active            (conflict resolved, human review cleared)
Flagged ──→ Retired           (conflict resolved by retiring this belief)
Active ──→ Retired            (superseded, explicitly retired, or recency floor reached)
Provisional ──→ Retired       (low-confidence belief not validated within retention window)
Retired ──→ Archived          (warm tier retention window expires — time-based automatic)
Archived ──→ [permanent]      (human action required for deletion — no automatic process)
```

All lifecycle transitions generate a provenance record. The transition record includes the triggering event, the contributor or system process responsible, and the previous and new states.

#### 4.4.3 Flagged State Behavior

Beliefs in the Flagged state remain queryable to support continuity of agent operation — removing a belief from query results mid-session would be more disruptive than surfacing it with a reduced weight and a flag. Consuming agents receive the flag as part of the belief's metadata and should reflect appropriate uncertainty in their outputs when building on flagged beliefs.

The promotion interface surfaces flagged beliefs prominently in the review queue with the full conflict or anomaly context. Flagged beliefs that remain unresolved beyond a configurable deadline are escalated to scope administrators.

### 4.5 Dimension Schema

Each belief node carries a dimension vector that governs how it is weighted during retrieval and merge operations.

**Recency** (float, 0.0–floor). Initialized at 1.0. Decays linearly over time at a belief-type-dependent rate. Linear decay is appropriate for codebase knowledge: a constraint about authentication handling that has been stable for two years is more reliable than one written last week — it has been validated by continued operation. Exponential decay would incorrectly penalize the most durable beliefs. The floor is belief-type-dependent and non-zero — a long-stable architectural constraint retains meaningful weight even after significant recency decay.

**Linear decay formula:** `recency = max(floor, 1.0 - (elapsed_days / half_life_days))`

Default half-life and floor values by belief type:

| Belief Type | Half-life (days) | Floor |
|---|---|---|
| Constraint | 365 | 0.4 |
| Decision | 180 | 0.3 |
| Pattern | 180 | 0.3 |
| Rejection | 90 | 0.2 |
| Observation | 30 | 0.1 |

These are initial defaults requiring empirical calibration. The floor for constraints is deliberately high — a long-standing constraint that nobody has contradicted or retired is still authoritative organizational knowledge even if it was established years ago.

**Confidence** (float, 0.0–1.0). Reflects the accumulated evidence for this belief across sessions and contributors. Increases with each independent confirmation. Decreases with each contradiction. Initialized based on how the belief entered the graph:

| Entry Path | Initial Confidence |
|---|---|
| Agent-derived (extraction) | 0.3 |
| Policy-promoted | 0.5 |
| Peer-validated (independent rederivation threshold reached) | 0.7 |
| Human-reviewed | 0.85 |
| Foundational (human-authored directly) | 0.95 |

**Corroboration** (two integers, each minimum 0; total minimum 1). The breadth of evidence for this belief, decomposed by the epistemic strength of each confirmation. Distinct from confidence in that it measures breadth rather than depth.

- `independent_rederivation_count`: distinct contributors whose session produced a candidate that deduplicated to this belief *without* having had the belief in their inbound context. Each such contributor independently reached the same belief from their own context. This is the strong evidence signal — the same conclusion reached from multiple independent paths.
- `confirming_corroboration_count`: distinct contributors whose session had the belief in inbound context and produced a candidate that deduplicated to it. The contributor confirmed by not contradicting, which is weak evidence — silent assent, lack of attention, and uncritical agreement with a wrong belief all produce the same signal.

Each contributor counts once per category. A belief confirmed fifty times by the same contributor counts as 1 in whichever category applies to that contributor's earliest confirmation. Subsequent confirmations from the same contributor do not change either count.

**Why the distinction matters.** A confirmation count that does not distinguish independence from echo can be inflated trivially: every agent that reads the graph will see existing beliefs and naturally produce candidates that deduplicate to them. Without the distinction, peer-validated trust elevates based on the graph confirming itself — the echo-chamber failure mode. Maintaining the distinction is what makes peer-validated trust epistemically meaningful.

**Peer-validated trust elevation.** A belief is automatically elevated to the peer-validated trust tier when `independent_rederivation_count` reaches a configurable threshold (default: 3). The threshold is over independent rederivations alone; confirming corroborations contribute to the corroboration *dimension score* but do not count toward trust elevation. This prevents trust elevation from being driven by graph echo at any agent-to-human ratio.

**Corroboration dimension score.** The scoring function (Section 5.6.1) consumes a normalized corroboration value derived from both counts with the confirming count weighted at a configurable fraction (default: 0.3) of the independent count. See Section 5.6.1 for the formula.

**Trust level** (enum). Derived from the provenance chain, not asserted by the contributor. Values: `agent_derived`, `policy_promoted`, `peer_validated`, `human_reviewed`, `foundational`. Trust level is the most powerful dimension in conflict resolution and the most carefully guarded against manipulation. Trust level can only increase through validated events — it cannot be set directly by any contributor.

**Scope precision** (float, 0.0–1.0). How specifically this belief applies. A belief scoped to a specific function is more precise than one scoped to a module, which is more precise than one scoped to the whole project. More specific beliefs take precedence over less specific ones in conflict resolution when both are relevant to the same query.

**Contradiction history** (array). A timestamped record of each contradiction this belief has received, including the contributing session and the nature of the contradiction. A belief that has survived multiple contradictions is more robust than one that has never been challenged. The size of this array and the recency of its most recent entry are both inputs to the merge strategy's conflict resolution scoring.

**Chain integrity** (float, 0.0–1.0). Reflects the health of the belief's upstream reasoning chain — whether the beliefs it depends on are themselves active, accurate, and uncompromised. Stored and updated on events rather than computed at query time, maintaining consistency with the stored-and-event-driven model used for all other dimensions and enabling uniform query performance across all dimension-based operations.

Initialized at promotion time based on the minimum chain_integrity of the belief's dependencies, weighted by their trust levels. A belief with no upstream dependencies initializes at 1.0. A belief whose reasoning chain includes a provisional dependency initializes lower than one whose reasoning chain is entirely foundational.

Updated by the retirement cascade: when an upstream belief is retired, chain_integrity is decremented on all downstream beliefs within the configurable hop limit, with the decrement magnitude dampened by hop distance. When an upstream belief is reinstated, chain_integrity is incremented on downstream beliefs by the reverse of the retirement cascade.

**Hop limit** (configurable integer, default: 3). The maximum dependency graph distance over which chain_integrity updates propagate. Beyond the hop limit, no automatic update occurs — the connection is too indirect for the upstream event to reliably affect downstream belief quality. Beliefs beyond the hop limit receive an informational annotation rather than a dimension update.

### 4.6 Storage Model

The belief graph uses a hybrid tier transition model: activity-based hot-to-warm transitions, time-based warm-to-cold transitions, event-based retirement transitions, and human-action-only archival.

#### 4.6.1 Hot Tier — Active Belief Graph

**Contents:** All beliefs in Provisional, Active, or Flagged states.

**Performance target:** Sub-100ms query response for scope queries at session startup. This is the hard latency constraint that governs the hot tier's technology choices. The size and concurrency assumptions backing this target, and the failure modes that exceed them, are specified in Section 4.7.7.

**Technology:** Property graph database with indexes on scope, type, trust level, lifecycle state, and recency. Content-addressed node storage (beliefs identified by a hash of their normalized content) enables efficient diffing — checking whether two graphs share a belief is a hash comparison rather than a semantic search.

**Transition to warm (activity-based):** A belief moves from hot to warm when it has not been queried or modified within a configurable inactivity window (default: 90 days) AND its lifecycle state is Active or Provisional. Flagged beliefs do not move to warm on inactivity — they require explicit resolution.

**Transition to warm (event-based):** Retirement immediately moves a belief to warm regardless of activity. Retirement is an explicit signal that the belief is no longer active organizational knowledge and should not delay on the inactivity window.

**Size management:** The hot tier is bounded by belief retirement. Beliefs that reach the recency floor without being contradicted are retired rather than accumulating indefinitely. The active graph does not grow without limit.

#### 4.6.2 Warm Tier — Recent Provenance

**Contents:** Retired beliefs, full provenance event log for the configurable retention window (default: 365 days), access event log.

**Performance target:** Query latency of seconds is acceptable. Warm tier queries are audit and investigation queries, not session-critical queries.

**Technology:** Append-only, immutable event log. Time-series optimized storage. Queryable by belief ID, contributor ID, time window, and event type.

**Transition to cold (time-based):** After the warm tier retention window expires, beliefs and their associated provenance records move to cold storage automatically. This transition is system-initiated, not human-initiated. The retention window is a governance parameter set by scope administrators.

**Transition to cold (event-based):** Distillation of session history moves the affected provenance records to cold immediately, regardless of the warm retention window.

#### 4.6.3 Cold Tier — Archive

**Contents:** All archived beliefs, provenance records older than the warm retention window, distilled session histories.

**Performance target:** Latency of minutes acceptable. Cold tier access is rare and deliberate.

**Technology:** Immutable object storage. Complete. No record is ever deleted from cold storage by an automated process.

**Transition to deleted (human action only):** Deletion from cold storage requires explicit administrator action with documented justification. No automated process removes data from cold storage. This is a governance requirement, not an implementation preference — the cold tier is the system's complete historical record and the foundation of its forensic capability.

### 4.7 Query Patterns

The belief graph must support six primary query patterns. Each has defined performance requirements and index implications.

#### 4.7.1 Scope Query (Session Startup)

**Purpose:** Return active beliefs relevant to a session's project and component scope, ordered by relevance.

**Inputs:** `project_scope_id`, optional `component_scope`, optional `belief_types[]`, optional `minimum_trust_level`

**Output:** Ranked list of belief nodes with full dimension vectors and lifecycle states. Provisionally-trusted and flagged beliefs included with metadata indicating their status.

**Latency target:** Sub-100ms. This is the hot path — every session startup depends on this query. The index strategy for the hot tier must treat this as the primary optimization target.

**Index requirements:** Composite index on `(project_scope_id, lifecycle_state, trust_level, recency)`. Secondary index on `(component_scope, belief_type)`.

#### 4.7.2 Type Query

**Purpose:** Return beliefs of a specific type within a scope. Used by the promotion interface, anomaly detection, and administrative tooling.

**Inputs:** `belief_type`, `project_scope_id`, optional `lifecycle_state`, optional `time_window`

**Output:** List of matching belief nodes with dimension vectors.

**Latency target:** Sub-second.

#### 4.7.3 Provenance Query (Backward Traversal)

**Purpose:** Given a belief, return its full provenance chain — every event that contributed to its current state, back to its originating exchange.

**Inputs:** `belief_id`

**Output:** Ordered sequence of provenance records from origin to current state, spanning hot and warm tiers as needed.

**Latency target:** Seconds acceptable. Used for audit and investigation, not session-critical operations.

**Note:** Full provenance traversal may span hot and warm tiers. The query must be able to cross tier boundaries transparently to the caller.

#### 4.7.4 Conflict Query

**Purpose:** Given a belief candidate, return existing beliefs that may conflict with it. Used by the merge evaluator during candidate evaluation.

**Inputs:** `candidate_content_embedding`, `project_scope_id`, `belief_type`, optional `component_scope`

**Output:** List of potentially conflicting beliefs ranked by semantic similarity, with dimension vectors.

**Latency target:** Sub-second within a project scope partition. The conflict query runs on every candidate submission — its latency directly affects merge throughput.

**Index requirements:** Approximate nearest neighbor index on belief content embeddings, partitioned by project scope.

#### 4.7.5 Delta Query (Sync)

**Purpose:** Return all beliefs that changed since a given sync cursor. Used by the compaction sync cycle to populate inbound belief updates for active sessions.

**Inputs:** `project_scope_id`, `sync_cursor` (pointer to last known graph state)

**Output:** List of belief nodes that were created, modified, or retired since the cursor, with their current states and dimension vectors.

**Latency target:** Sub-second for typical delta sizes (tens to low hundreds of beliefs). The delta query runs at every compaction event for every active session — it must be fast enough not to meaningfully extend the compaction window.

**Index requirements:** Modification timestamp index on belief nodes, partitioned by project scope.

#### 4.7.6 Anomaly Query

**Purpose:** Identify beliefs with anomalous dimension patterns within a time window and scope. Used by the anomaly detection layer in Section 9.

**Inputs:** `project_scope_id`, `time_window`, `anomaly_type` (velocity | conflict_cluster | trust_drift | provenance_gap)

**Output:** List of beliefs or belief patterns matching the anomaly criteria, with supporting evidence.

**Latency target:** Minutes acceptable. Anomaly queries run on a scheduled basis, not in the session critical path.

#### 4.7.7 Performance Assumptions and Failure Modes

The latency targets specified throughout Section 4.7 derive from explicit assumptions about belief graph size and concurrency. This subsection states the assumptions and the failure modes that exceed them.

**Size assumptions per project scope:**

- Typical: 100–1000 active beliefs in the hot tier
- Mature: up to 10,000 active beliefs in the hot tier
- Beyond mature: above 10,000 active beliefs, the assumptions break and additional architectural work is required (see failure modes below)

The scope query latency target (sub-100ms, Section 4.7.1) is comfortable at typical scope sizes and tight at the mature boundary. The composite index on `(project_scope_id, lifecycle_state, trust_level, recency)` produces an indexed range scan whose cost scales with result-set size rather than total graph size; the dominant cost at large scopes is dimension scoring for ranking, which is bounded by a multiply-add per dimension per belief in the result set.

The conflict query latency target (sub-second, Section 4.7.4) is comfortable across the full assumed range. Modern approximate nearest neighbor libraries (FAISS, hnswlib, and equivalents) handle ANN queries over 100,000 vectors at sub-100ms latency. A 10,000-belief project scope partition is well within budget. The downstream conflict detection cost (type rules in the fast path, LLM-assisted comparison in the slow path) is separate from query latency and is bounded by the candidate batch size, not by graph size.

The delta query latency target (sub-second, Section 4.7.5) is bounded by typical delta size, not by total graph size. Compaction occurs frequently enough that deltas are typically tens to low-hundreds of beliefs. The query is a range scan on a modification timestamp index; the dominant cost is materialization of the result set into the response payload.

**Concurrency assumptions:**

- Compaction is per-session and triggered by session events (Section 3.3.2), not by a time-aligned external trigger. Compactions across a development team are naturally desynchronized — there is no synchronization point that causes 50 developers to compact simultaneously. This is load-bearing for the concurrency story and explicitly relied upon by the latency targets.
- Average compaction rate: at 50 developers with ~10 compactions per day per developer, the system experiences ~500 compactions per day, or roughly 6 per minute on average. Peak burst capacity: 10–20 concurrent compactions during peak hours.
- Read scaling: scope and delta queries can scale horizontally with read replicas of the hot tier. Read replication lag is bounded by the consistency model choice (Section 11 open question).
- Write scaling: merge evaluator writes are partitioned by project scope. Conflict detection requires consistency within a scope partition but allows full parallelism across partitions. Bursts of 10–20 concurrent compactions are absorbed by partitioning across the project scopes represented in the burst.

**Failure modes that exceed the assumptions:**

- *Scope size above 10,000 active beliefs.* The scope query at session startup becomes the bottleneck. Mitigations: precompute composite ranking scores at write time and store them as an indexed property (the scoring function is deterministic and can be evaluated at promotion time), or apply a top-N cutoff at the DB layer before returning to the application. Either mitigation degrades the freshness of dynamic dimensions like recency, requiring periodic recomputation.
- *Very large deltas.* A session whose sync cursor predates the previous compaction by days (developer returning from vacation, long-running branch work, extended outage recovery) may have a delta in the thousands of beliefs. The query remains fast; the response payload is the bottleneck. Mitigation: page the delta response or stream beliefs in chunks during the compaction window.
- *Conflict query partition imbalance.* A single project scope that accumulates significantly more beliefs than peer scopes can produce ANN partition imbalance and degrade conflict query latency for that scope. Mitigation: sub-partition large scopes by component scope, accepting that conflict detection across sub-partitions requires either a fan-out query or acceptance that cross-sub-partition conflicts are caught by the background consistency check (Section 5.4.4) rather than at submission time.
- *Time-aligned compaction triggers (anti-pattern).* If a deployment introduces a time-aligned compaction trigger (every hour on the hour, CI run on a schedule, scheduled batch job) the natural desynchronization assumption breaks. The merge evaluator must handle the resulting burst either by absorbing it through partitioned write capacity or by token-bucketing at the API layer. Deployments should prefer event-triggered compaction over time-triggered compaction.

**What remains empirically determined.** The actual achievable latency at the mature boundary depends on the specific property graph database, the specific ANN library, and the deployment's hardware profile. The numbers in this subsection are design targets that production deployments should validate against their own measurements before committing to size and concurrency limits.

### 4.8 Key Design Decisions

**Linear decay over exponential.** Codebase knowledge exhibits the opposite of recency bias — stable constraints and architectural decisions gain credibility over time by virtue of surviving. Linear decay with a non-zero floor correctly models this: decay is constant, old beliefs retain meaningful weight, and the floor prevents beliefs from being effectively retired by age alone. Exponential decay would incorrectly accelerate the deprecation of the most durable beliefs.

**Activity-based hot-to-warm transition.** Beliefs that are actively queried should remain in the hot tier regardless of age. A constraint that is queried every session because agents need it is active organizational knowledge even if it was established years ago. Activity-based transition correctly keeps these beliefs accessible at hot-tier latency.

**Human-action-only archival.** Deletion from cold storage requires explicit human authorization. This is a governance requirement. The cold tier is the system's complete historical record and forensic foundation. No automated process should be able to remove it.

**Flagged beliefs remain queryable.** Removing a flagged belief from query results mid-session would create unexpected agent behavior and potentially more disruption than surfacing the belief with reduced weight and appropriate metadata. Flagged beliefs stay queryable with reduced weight until human review resolves them.

**Queryability in both directions.** The belief graph must answer "what does the system know about X" (forward query, scope query) and "why does it know that" (backward query, provenance traversal). Both query patterns need efficient indexes. The forward query is session-critical; the backward query is audit-critical.

**Content-addressed node storage.** Identifying beliefs by a hash of their normalized content enables efficient diffing between belief graph states. Checking whether two graphs share a belief is a hash comparison rather than a semantic search. This matters for the delta query and for the merge strategy's deduplication step.

**What remains empirically determined.** Decay half-life and floor values per belief type, confidence initialization values per trust tier, the independent rederivation threshold for peer-validated promotion, the confirming corroboration discount factor, the contradiction threshold for automatic flagging, the inactivity window for hot-to-warm transition, and the warm retention window. All are configurable parameters requiring calibration from deployment data.

---

## 5. Layer 3: The Merge Strategy

### 5.1 Responsibility

The merge strategy reconciles divergent belief states that have developed independently from a common ancestor — across sessions, across developers, and across periods of offline operation. It is the hardest layer technically and the one where the consequences of getting it wrong are most severe. A bad merge produces a subtly degraded belief graph that may pass all validation checks and still cause agents to behave incorrectly in ways that are difficult to detect.

### 5.2 The Three Divergence Types

The merge strategy must distinguish between three fundamentally different types of conflict, because each requires a different response.

**Additive divergence.** Two sessions learned different things that are both true and should both survive the merge. Session A learned that the project uses DynamoDB single-table design. Session B learned that the project's Lambda functions have a 15-second timeout constraint. These beliefs are about different things and both are valid. The merge operation adds both to the shared graph.

**Contradictory divergence.** Two sessions developed incompatible beliefs about the same thing. Session A believes that IAM wildcards are permitted for S3 read operations. Session B believes that IAM wildcards are prohibited in all cases. These cannot both be true. The merge operation detects the conflict and routes it to human review.

**Temporal divergence.** One session's beliefs are outdated relative to another's because the codebase changed in between. Session A's belief about the authentication pattern reflects a state of the system that was refactored away two weeks ago. Session B's belief reflects the current state. The merge operation uses recency and provenance to identify the stale belief and retire it.

### 5.3 Pipeline Sequence

The merge pipeline executes the following stages for each submission batch, in order. Stage ordering is not configurable — each stage depends on the output of the previous.

#### 5.3.1 The Durable Queue

A durable message queue sits between the extraction pipeline and the merge evaluator. This is an architectural requirement, not an optimization. The queue provides:

**Durability.** Candidates survive session termination, infrastructure failures, and extended merge evaluator unavailability. Candidates are written to the queue when extracted; the merge evaluator reads from the queue when available. The queue is the system's resilience boundary.

**Ordered processing.** The queue preserves submission timestamp. On recovery the merge evaluator processes from the oldest unprocessed position — not from arbitrary order. This restores the ordering guarantee that is lost during outages.

**Candidate expiry.** Candidates that have been in the queue longer than the staleness threshold are expired rather than processed on recovery. Submitting candidates from weeks ago after an extended outage would trigger the staleness discount, route most to human review, and consume pipeline capacity with low-value work. Expiry is cleaner. Expired candidates are logged in the queue audit trail and are not silently dropped — they are available for administrative review.

**Recovery flood protection.** When the merge evaluator recovers from an extended outage, the queue may contain a large backlog. Draining the full backlog immediately can overwhelm the pipeline — a DDoS-by-recovery attack exploits this by building a large backlog and then stopping, letting the recovery flood do the damage. The merge evaluator drains the queue at a configurable throttled rate, well below sustained throughput capacity, leaving headroom for new submissions from live sessions.

**Priority ordering on recovery drain.** During backlog drain, candidates are processed in priority order: foundational and human-reviewed candidates first, then peer-validated, then policy-promoted, then agent-derived. This improves belief graph quality during the recovery window even if the full backlog takes time to drain.

#### 5.3.2 Stage Sequence

```
[Durable Queue] ← candidates written here by extraction pipeline

Merge evaluator reads from queue in submission timestamp order:

1. Queue read and expiry check      — expire candidates beyond staleness threshold, log expired
2. Normalization verification       — confirm all candidates are normalized (reject unnormalized)
3. Intra-batch deduplication        — resolve near-duplicates within the batch before graph comparison
4. Embedding generation             — generate content embeddings for all candidates
5. Topical proximity search         — find existing beliefs within similarity radius (Stage 1 conflict detection)
6. Semantic compatibility check     — assess compatibility for proximity matches (Stage 2 conflict detection)
7. Staleness assessment             — compute divergence between session sync cursor and current graph
8. Scoring                          — compute composite score for each candidate
9. Decision                         — apply scoring thresholds to produce merge decisions
10. Execution                       — execute promote, update, flag, or reject for each candidate
11. Provenance recording            — write provenance records for all decisions
12. Dimension propagation           — update dimension scores on affected existing beliefs
13. Watermark advancement           — advance the graph's processed watermark timestamp
14. Sync cursor update              — advance the session's sync cursor to current graph state
```

Stages 5 and 6 together constitute conflict detection. Stage 7 feeds into Stage 8. The pipeline is transactional at the batch level — if any stage fails, the full batch is requeued for retry rather than partially applied. The durable queue retains the batch until the merge evaluator explicitly acknowledges successful completion.

#### 5.3.3 The Graph Watermark

The graph maintains a processed watermark — the timestamp of the last candidate successfully committed. The watermark is advanced at stage 13 of every successfully processed batch. It is published in submission acknowledgment responses and used by sessions to compute true divergence on reconnection after an outage.

During intermittent availability, the graph may process some candidates and not others. The watermark identifies the boundary between processed and unprocessed state. On recovery, the merge evaluator resumes from the queue position corresponding to the watermark rather than reprocessing already-committed candidates.

#### 5.3.4 Partial Recovery Mode

Rather than waiting for the full backlog queue to drain before accepting new submissions, the merge evaluator enters partial recovery mode immediately on restoration of availability. In partial recovery mode:

- New submissions from live sessions are accepted and queued normally
- The backlog drains at the throttled drain rate
- New submissions receive processing priority over backlog candidates of equivalent trust level
- The partial recovery mode status is visible in the submission response and in the API health endpoint

Partial recovery mode ends when the backlog queue depth returns to the normal operating range. The transition is automatic and does not require operator intervention.

### 5.4 Conflict Detection

Conflict detection runs in two stages: topical proximity search identifies candidates that are about the same topic as existing beliefs, and semantic compatibility assessment determines whether topically similar pairs are compatible or contradictory.

#### 5.4.1 Stage 1: Topical Proximity Search

Approximate nearest neighbor search on belief content embeddings, partitioned by project scope and belief type. Returns all existing beliefs within a configurable similarity radius for each incoming candidate. The output is a ranked list of proximity matches — not a conflict determination, only a topical relationship signal.

The similarity radius is configurable and belief-type-aware. Constraint beliefs use a tighter radius (fewer false proximity matches accepted) because a constraint that partially overlaps another constraint is more likely to be a genuine conflict than two observations that partially overlap.

#### 5.4.2 Stage 2: Semantic Compatibility Assessment

For each proximity match pair returned by Stage 1, the compatibility assessment determines whether the two beliefs are compatible or contradictory. This runs in two sub-stages:

**Type-based rule evaluation (fast path).** Deterministic rules based on belief types and normalized content structure catch unambiguous cases without LLM involvement.

Examples of type-based rules:
- A constraint stating "X must be Y" and a constraint stating "X must not be Y" → contradictory
- A decision and a rejection about the same approach → compatible (decision records choice, rejection records exclusion)
- Two patterns describing different implementations of the same interface → compatible
- Two constraints about the same property with different values → contradictory

Pairs resolved by type-based rules produce a high-confidence compatibility determination. Pairs not resolved proceed to LLM-assisted comparison.

**LLM-assisted semantic comparison (slow path).** For pairs that pass through type-based rules without a clear determination, an LLM-assisted comparison generates a compatibility assessment. The model receives: both normalized belief statements, their types, their scopes, the type-based rules that did not resolve the question, and any relevant context from their provenance chains. It returns a structured response:

```json
{
  "assessment": "compatible | contradictory | uncertain",
  "confidence": 0.85,
  "explanation": "human-readable explanation of the assessment",
  "supporting_evidence": ["specific phrases or properties that drove the assessment"]
}
```

The explanation becomes part of the review queue entry when the conflict is flagged. Reviewers see the system's reasoning alongside the conflicting beliefs, significantly reducing the cognitive load of the review task.

#### 5.4.3 Confidence-Stratified Routing

The combination of the Stage 1 similarity score and the Stage 2 compatibility assessment produces a routing decision:

| Similarity | Compatibility | Confidence | Routing |
|---|---|---|---|
| High | Compatible | High | Deduplication — proceed to aliasing |
| High | Contradictory | High | Flag — hard conflict, high priority review |
| High | Uncertain | Any | Flag — soft conflict, standard priority review |
| Low | Compatible | Any | Promote — no conflict detected |
| Low | Contradictory | High | Flag — non-obvious conflict, standard priority review |
| Low | Uncertain | Any | Promote with soft flag metadata |

**Soft flag behavior.** A low-similarity, uncertain potential conflict is promoted rather than held — the evidence for conflict is too weak to justify blocking promotion. A `potential_conflict_ref` field in the promoted belief's metadata records the potential conflict and the assessment that produced it. If subsequent sessions generate stronger evidence of the conflict, the soft flag is escalated to a hard flag without losing the provenance of when the potential conflict was first detected.

#### 5.4.4 Non-Obvious Conflict Detection (Background Process)

Per-submission conflict detection catches topically similar contradictions reliably. Cross-topic contradictions — where two beliefs are logically contradictory but dissimilar in embedding space — are not reliably detectable at submission time without full-graph analysis.

A scheduled background consistency check traverses the full belief graph using deeper semantic analysis than the per-submission pipeline can afford, looking for logical contradictions across the full graph. Contradictions found by the background check are flagged with lower urgency than per-submission conflicts, since they have likely been present in the graph for some time without causing visible problems.

This is an explicitly named limitation: per-submission conflict detection is fast and reliable for topically similar contradictions. Cross-topic contradictions are caught eventually by the background check, not immediately at submission time.

### 5.5 Deduplication

Deduplication identifies belief candidates that represent the same underlying fact as existing beliefs or other candidates in the same batch. The key design principle: **deduplication creates aliases, not deletions**. Deduplicated candidates are never removed — they become retired alias nodes with supersession edges pointing to the canonical belief. This preserves reasoning chain integrity and audit trail completeness.

#### 5.5.1 Why Aliasing Rather Than Absorption

If a candidate A is simply absorbed into existing belief B — updating B's dimensions and discarding A — any other belief in the submission batch that references A in its reasoning chain now has a dangling pointer. The reasoning chain is broken and the audit trail is incomplete.

The correction event case makes this concrete. If A was extracted as a correction event — the developer explicitly contradicting something — A carries stronger epistemic weight than a regular observation. If A is absorbed into B without preserving A as a node, the reasoning chain for any decision belief that depended on A now appears to rest on B's history, which might be a years-old observation. The fact that the reasoning was grounded in a fresh human correction is lost.

Aliasing preserves this: A exists as a retired node, A's provenance record shows it was a correction event, A has a supersession edge to B, and reasoning chain traversal from any belief that references A can follow the full chain: reference → A (correction event, session X, exchange Y) → supersession → B (canonical, full history).

#### 5.5.2 Intra-Batch Deduplication

Before cross-graph deduplication, the submission batch is checked for internal near-duplicates. Two candidates in the same batch that are near-duplicates of each other are resolved within the batch before any graph comparison.

**Canonical selection in intra-batch deduplication.** The candidate with the higher initial confidence score becomes canonical. On tie, the candidate from the earlier exchange in the session becomes canonical. This rule is arbitrary but deterministic, which is what matters for reproducibility and auditability.

**Sibling relationship preservation.** If an exchange produced sibling candidates (multiple types from the same exchange), and one sibling is deduplicated against another candidate in the batch while its siblings are not, the sibling relationship is preserved across all three nodes. The deduplicated sibling becomes an alias node. Its siblings' `sibling_candidates` arrays retain the alias node's ID. The alias node is reachable for reasoning chain traversal.

#### 5.5.3 Cross-Graph Deduplication

After intra-batch deduplication, each remaining candidate is compared against the existing belief graph using the same approximate nearest neighbor search as Stage 1 of conflict detection. Candidates with similarity above the deduplication threshold — distinct from the conflict detection proximity threshold, typically higher — are treated as near-duplicates of the matched existing belief.

**Deduplication is type-aware.** Two candidates are only considered potential duplicates if they share the same belief type or if one is an observation that subsumes into a constraint or decision of the same content. A constraint and a decision can both be true statements about the same system property — they are not duplicates.

#### 5.5.4 Cross-Version Deduplication

Belief identity is a stable UUID, not a content hash (Section 3.5.4). The deduplication fast path must accommodate the case where a candidate's normalized content does not exactly match any existing belief's current normalized content but represents the same underlying belief produced under a different normalization model version.

The deduplication match cascade is:

1. **Exact content match for current normalization model version.** Fastest path. The candidate's normalized content exactly equals an existing belief's current normalized content, and both share the same `normalization_model_version`. Route to update.
2. **Exact content match against any prior normalized form of an existing belief.** The candidate's normalized content exactly equals a historical normalized form of an existing belief (preserved in provenance from a prior re-normalization). Route to update of the existing belief. The candidate is treated as confirming a belief already in the graph.
3. **Embedding proximity above deduplication threshold.** The candidate's content embedding is sufficiently close to an existing belief's content embedding regardless of normalization model version. Route to the standard dedup check (Section 5.4.2 type rules followed by Section 5.4.2 LLM-assisted comparison if needed).
4. **No match.** Candidate proceeds to merge decision as a potentially new belief.

Steps 1 and 2 are the fast path and resolve the majority of cross-version dedup cases without invoking the slow path. Step 3 handles the case where the new model produces a meaningfully different normalized form that does not match any historical form exactly. The embedding model used in step 3 must be model-version-independent (Section 11 open question).

**The alias node schema addition.** Retired alias nodes carry a `canonical_belief_id` field pointing to the canonical belief, in addition to the supersession edge in the graph structure. This enables efficient lookup: a traversal that reaches an alias node can find the canonical belief directly from node data without needing to query for outgoing supersession edges.

```json
{
  "belief_id": "uuid",
  "lifecycle_state": "retired",
  "retirement_reason": "deduplicated",
  "canonical_belief_id": "uuid",
  "provenance": {
    "event_type": "deduplicated",
    "canonical_belief_id": "uuid",
    "absorbed_at": "iso8601",
    "dimension_updates_applied_to_canonical": true,
    "original_extraction_type": "correction_event | constraint_assertion | ...",
    "original_confidence": 0.5
  }
}
```

**Dimension update flow.** Dimension updates from the deduplicated candidate — corroboration increment, confidence nudge, trust level if higher — flow to the canonical belief, not to the alias node. The alias node's dimensions are frozen at the values they had at deduplication time.

**Corroboration classification at dedup time.** When a candidate deduplicates against an existing belief, the merge evaluator classifies the corroboration as `independent` or `confirming` (Section 4.5) before incrementing either count. The classification test is:

1. Compare the contributing session's most recent sync cursor at the time of candidate generation against the belief's promotion timestamp.
2. If the sync cursor predates the belief's promotion timestamp, the session could not have had the belief in inbound context — classify as `independent`.
3. If the sync cursor postdates the belief's promotion timestamp, check whether the belief was in the session's scope at the time of that sync. If not, the session would not have pulled the belief even though it existed — classify as `independent`. If yes, the session had the belief available — classify as `confirming`.
4. The classification is recorded on the confirmation edge (Section 4.3) and on the provenance record for the dedup event.

The classification check uses the same sync cursor data already maintained for staleness discount computation in the scoring function. No new tracking infrastructure is required.

**Per-contributor uniqueness.** Each contributor is counted once per category for a given belief. If a contributor's first dedup against a belief was classified as `independent` and a later dedup from the same contributor would classify as `confirming`, the contributor's first classification stands and the later dedup does not increment any count. This prevents a contributor from contributing to both counts and prevents repeated confirmations from the same contributor from inflating either count.

**Epistemic property preservation.** The alias node's provenance record preserves the epistemic properties of the original candidate — particularly its extraction heuristic type and correction event classification. These properties are visible to reasoning chain traversal even though the canonical belief receives the dimension updates.

### 5.6 The Multi-Dimensional Scoring Function

For each belief candidate that has passed deduplication and conflict detection without being aliased or flagged, the system computes a composite score that determines the merge decision.

#### 5.6.1 Structure

The scoring function executes in two stages. Stage 1 is a trust level gate. Stage 2 is a weighted composite of the remaining dimensions.

**Stage 1: Trust level gate.** If the incoming candidate's trust level is more than one tier below the trust level of any belief it would update or conflict with, automatic override is prohibited regardless of other dimension scores. The decision routes directly to Flag. This is a hard rule enforced before Stage 2 runs.

*Rationale:* A foundational or human-reviewed belief cannot be automatically overridden by an agent-derived candidate, regardless of how high the agent-derived candidate's other dimension scores are. Trust level encodes the integrity of the provenance chain, not just the content quality. A trust level gate prevents a sophisticated poisoning attack from using high-content-quality beliefs to override high-provenance-integrity beliefs.

**Stage 2: Weighted composite.** For candidates that pass the trust level gate, the composite score is computed in two parts. First, the weighted dimension sum:

```
composite = (w_recency × recency)
          + (w_confidence × confidence)
          + (w_corroboration × normalized_corroboration)
          + (w_scope_precision × scope_precision)
          + (w_chain_integrity × chain_integrity)
          - (w_contradiction × contradiction_penalty)
```

Then the staleness discount is applied to the full composite:

```
score = composite × staleness_discount
```

Where:
- `normalized_corroboration` = `min((independent_rederivation_count + confirming_discount × confirming_corroboration_count) / corroboration_ceiling, 1.0)` — independent rederivations are full-weight, confirming corroborations are discounted (default `confirming_discount`: 0.3), and the sum is bounded by the corroboration ceiling to prevent a single high-corroboration belief from dominating the score. See Section 4.5 for the rationale behind the discount.
- `contradiction_penalty` = function of contradiction history size and recency of most recent contradiction
- `chain_integrity` = the candidate's upstream reasoning chain health score (see Section 4.5) — candidates whose reasoning chains include incorrectly-retired or heavily-flagged upstream beliefs score lower regardless of their own content quality
- `staleness_discount` = function of divergence between session sync cursor and current graph state, range 0.0–1.0 (1.0 = no staleness, 0.0 = maximum staleness discount)
- `w_*` = configurable weights, must sum to 1.0 before staleness discount

The two-step formulation is deliberate: the staleness discount must apply to the entire composite, not only to the contradiction term. A single-line formulation with implicit operator precedence could be misread as discounting only the contradiction penalty, which would produce the opposite of the intended behavior — stale sessions would receive a *smaller* contradiction penalty rather than a smaller overall score. Implementations must apply staleness discount to the composite as the final step.

#### 5.6.2 Decision Thresholds

The composite score maps to merge decisions at defined thresholds:

| Condition | Decision |
|---|---|
| Exact content match with existing belief for current normalization model version | Update — no scoring needed |
| Exact content match against any prior normalized form of an existing belief (Section 5.5.4) | Update — no scoring needed |
| Score below `reject_threshold` | Reject |
| Score between `reject_threshold` and `promote_threshold` | Flag |
| Score at or above `promote_threshold`, no conflict detected | Promote |
| Score at or above `promote_threshold`, conflict detected | Flag |
| Trust level gate triggered | Flag |

The `reject_threshold` and `promote_threshold` values are empirically determined. Initial deployments should set `promote_threshold` conservatively high — more decisions routed to human review — and lower it as deployment data validates the scoring function's reliability.

#### 5.6.3 Inputs and Outputs

**Inputs:**
- Candidate dimension vector (from extraction)
- Existing belief dimension vector (if updating an existing belief)
- Conflict detection result (compatible | contradictory | uncertain, with confidence)
- Staleness measure (divergence between session sync cursor and current graph state)
- Trust level gate result (passed | triggered)

**Output:**
- Merge decision (promote | update | flag | reject)
- Composite score (float)
- Decision rationale (structured record of which factors drove the decision)
- For flag decisions: the specific reason for flagging and the review queue priority

The decision rationale is part of the provenance record for every merge decision. Reviewers in the Flag queue see not just the conflicting beliefs but the specific scoring factors that produced the Flag decision.

### 5.7 Merge Decision Execution

#### 5.7.1 Promote

The candidate is new, scores above the promote threshold, and has no detected conflict. A new belief node is created at Provisional lifecycle state with the candidate's dimension vector. Chain_integrity is initialized at promotion time: a belief with no upstream dependencies initializes at 1.0; a belief with dependency edges initializes at the minimum chain_integrity of its dependencies weighted by their trust levels. A provenance record is written recording the promotion decision, the composite score, the initial chain_integrity value and its derivation, and the contributing session and exchange.

#### 5.7.2 Update

The candidate matches an existing belief by content hash. The existing belief's dimension scores are updated to reflect the new evidence. The existing belief remains in its current lifecycle state. A provenance record is written recording the dimension changes, the contributing session, and the composite score.

#### 5.7.3 Flag

The candidate cannot be automatically resolved. The candidate is held in a pending state — not promoted, not rejected. A review queue entry is created containing the candidate, the conflicting beliefs (if any), the conflict detection assessment and explanation, the composite score and rationale, and the trust level gate result if triggered. The review queue entry has a priority level based on the nature of the flag: trust level gate triggers are high priority, high-confidence hard conflicts are standard priority, soft conflicts and uncertain assessments are low priority.

Flagged candidates remain in pending state until a reviewer acts. A configurable deadline triggers escalation if no action is taken.

#### 5.7.4 Reject

The candidate scores below the reject threshold. The candidate is not added to the active graph. A rejection record is written to the provenance store with the composite score and the specific dimensions that drove the rejection. Rejection records are visible in the quarantine queue for administrator review. Patterns in rejection — systematic rejection of candidates from a specific contributor or session type — are signals for extraction pipeline or scoring calibration.

### 5.8 The Stale Gradient Problem

When a session has been offline for an extended period and the shared belief graph has evolved significantly, the session's belief candidates are computed relative to a starting point that no longer reflects the current graph state. Applying these candidates without correction can degrade the graph quality — a belief that was accurate when the session started may have been superseded by the time it is submitted for merge.

The federated learning literature characterizes this as the stale gradient problem. The `staleness_discount` in the scoring function is the primary mitigation: candidates from sessions with high sync cursor divergence are scored with a discount that reduces their effective composite score, making it less likely they override current high-confidence beliefs.

**Staleness threshold.** When the divergence between a session's sync cursor and the current graph state exceeds a configurable threshold, the staleness discount reaches zero — the candidate's score is zeroed and the decision routes automatically to Flag, bypassing scoring entirely. This is the rejection threshold for stale updates: candidates that are too far behind the current graph state are held for human review rather than applied automatically.

**Staleness measure.** The staleness measure is computed as a function of: the number of beliefs in the session's scope that changed since the session's sync cursor, weighted by the trust levels of the changed beliefs. A session whose sync cursor predates one foundational belief change is more stale than one that predates ten provisional belief changes.

**Initial deployments** should set the staleness threshold conservatively — stale candidates require human review sooner rather than later — and relax it as operational experience informs what staleness levels produce acceptable merge quality.

### 5.9 Key Design Decisions

**Pipeline is transactional at batch level.** A batch either fully succeeds or is fully retried. Partial application of a batch leaves the belief graph in an inconsistent state relative to the session's provenance record. Transactional execution at the batch level prevents this.

**Deduplication creates aliases, not deletions.** Deduplicated candidates become retired alias nodes with supersession edges to the canonical belief. This preserves reasoning chain integrity and audit trail completeness. No belief that ever entered the pipeline is truly deleted — it becomes unreachable in normal operations but remains traversable for provenance purposes.

**Conflict detection is two-stage and confidence-stratified.** Fast type-based rules handle unambiguous cases. LLM-assisted comparison handles ambiguous cases. Confidence stratification routes low-confidence potential conflicts to promotion with soft flags rather than blocking on uncertain evidence.

**Non-obvious conflicts are a known limitation.** Per-submission conflict detection catches topically similar contradictions reliably. Cross-topic contradictions require the background consistency check. This is an explicit architectural acknowledgment, not an oversight.

**Trust level gate is a hard rule, not a weighted factor.** The gate cannot be overridden by high scores in other dimensions. This prevents sophisticated poisoning attacks from using content quality to override provenance integrity.

**Decision rationale is always recorded.** Every merge decision — promote, update, flag, reject — generates a provenance record with the scoring factors that produced it. This is essential for debugging, calibration, and audit. A belief graph whose decisions are not explained is not auditable.

**What remains empirically determined.** The scoring function weights (including `w_chain_integrity`), the promote and reject thresholds, the staleness threshold and staleness measure formula, the similarity radius for proximity search, the deduplication similarity threshold, the corroboration ceiling for score normalization, the hop limit for chain_integrity cascade, and the LLM model used for semantic compatibility assessment. All are configurable parameters requiring calibration from deployment data.

---

## 6. Layer 4: The Provenance Layer

### 6.1 Responsibility

The provenance layer maintains the chain of custody from raw session exchange to belief graph entry. It answers the question "why does the system know that?" with a traceable, auditable chain of evidence.

### 6.2 The Trust Derivation Model

Trust level is a derived property, not an asserted one. No contributor — human or agent — sets their own trust level. The system derives it from verifiable provenance attributes.

**Agent-derived.** The belief was extracted from a session exchange without human review. The contributing session is authenticated but the belief content has not been validated by a human.

**Policy-promoted.** The belief was automatically promoted under a human-authored promotion policy. The policy itself has provenance — who authored it, when, and what it governs. The belief's trust derives from the policy's trust.

**Peer-validated.** The belief has been independently rederived by multiple contributors across multiple sessions without contradiction. "Independently rederived" is the operative phrase: trust elevation depends on `independent_rederivation_count` exceeding the threshold (default: 3), not on the total corroboration count. The distinction is what makes peer-validated trust resistant to graph echo — see Section 4.5 for the full epistemic argument.

**Human-reviewed.** A human reviewer explicitly confirmed the belief's accuracy through the promotion interface. The reviewer's identity is part of the provenance record.

**Foundational.** A human authored the belief directly, not derived from an agent session. These are treated as ground truth for conflict resolution.

### 6.3 Provenance Record Schema

```json
{
  "provenance_id": "uuid",
  "belief_id": "uuid",
  "event_type": "extracted | promoted | confirmed | contradicted | retired | merged | distilled",
  "timestamp": "iso8601",
  "contributing_session_id": "uuid | null",
  "contributing_exchange_id": "uuid | null",
  "contributor_id": "uuid",
  "previous_dimensions": { },
  "new_dimensions": { },
  "policy_ref": "uuid | null",
  "reviewer_id": "uuid | null",
  "notes": "optional human-readable explanation"
}
```

Every dimension change generates a provenance record. The full provenance chain for a belief is the ordered sequence of provenance records with that belief's ID, from extraction to current state.

### 6.4 Provenance as Security Control

The provenance layer is the primary defense against belief poisoning attacks. An attacker who introduces a malicious belief into the graph needs to forge a plausible provenance chain for it — authenticated session, plausible contributor, consistent dimension history. Forging a convincing provenance chain is significantly harder than simply writing a malicious belief to a flat context file.

The specific attack surface and mitigations are detailed in Section 9 (Security Model).

### 6.6 Retirement Semantics

Retirement removes a belief from the active graph but its consequences extend to any belief that referenced it in a reasoning chain. The retirement reason determines the downstream response.

#### 6.6.1 Retirement Reasons

**`incorrect`.** The belief was wrong from the start — the developer misremembered, the agent misunderstood, or normalization was faulty. This is the most serious retirement reason. Any belief whose reasoning chain includes an incorrect belief was built on a false foundation. The downstream cascade generates high-priority flags on direct dependents, standard-priority flags on two-hop dependents, and informational annotations on three-hop dependents. Beyond the hop limit, no automatic cascade.

**`superseded`.** The belief was correct when produced but the system evolved and it is no longer accurate. The retirement record includes a `superseded_by` reference pointing to the new canonical belief. Downstream cascade generates standard-priority flags on direct dependents and informational annotations on two-hop dependents. The downstream beliefs were correct given the state of the system when they were produced — the question is whether their conclusions remain valid under the new belief. Review is important but not urgent.

**`redundant`.** The belief is retired because a more precise or authoritative belief now covers the same ground. Not wrong, not outdated — refined. Downstream cascade generates informational annotations only, no flags. Beliefs that depended on a redundant belief almost certainly remain valid under the more specific replacement.

#### 6.6.2 The Cascade Dampening Model

Flag intensity decreases with each hop in the dependency graph. This prevents a single incorrect retirement from generating hundreds of high-priority flags across a mature belief graph with rich dependency relationships.

| Hop Distance | Retirement Reason: incorrect | Retirement Reason: superseded | Retirement Reason: redundant |
|---|---|---|---|
| Direct (1 hop) | High-priority flag | Standard-priority flag | Informational annotation |
| 2 hops | Standard-priority flag | Informational annotation | No action |
| 3 hops | Informational annotation | No action | No action |
| Beyond hop limit | No action | No action | No action |

The hop limit is the configurable parameter defined in Section 4.5. The cascade traverses dependency edges in the forward direction from the retired belief, up to the hop limit. The traversal is asynchronous — it does not block the retirement operation and does not need to complete before the retirement provenance record is written.

Chain_integrity dimension updates follow the same cascade path and dampening model, applied simultaneously with flag generation. One traversal produces both the flags and the chain_integrity decrements.

#### 6.6.3 The Retirement Provenance Record

Retirement generates a provenance record with `retirement_reason` as a required field. The record additionally contains: the `superseded_by` reference if applicable, the cascade scope (how many beliefs were flagged or annotated), and the reviewer identity if the retirement was human-initiated.

```json
{
  "provenance_id": "uuid",
  "belief_id": "uuid",
  "event_type": "retired",
  "timestamp": "iso8601",
  "contributor_id": "uuid",
  "retirement_reason": "incorrect | superseded | redundant",
  "superseded_by": "belief_id | null",
  "cascade_scope": {
    "high_priority_flags": 3,
    "standard_priority_flags": 7,
    "informational_annotations": 12,
    "hop_limit_applied": 3
  },
  "reviewer_id": "uuid | null",
  "notes": "human-readable explanation of why the belief was retired"
}
```

#### 6.6.4 Reinstatement

A retired belief can be reinstated if review determines the retirement was incorrect. Reinstatement is a defined operation with its own provenance record and review cascade.

**Reinstatement provenance record.** Records the belief being reinstated, the reviewer who authorized reinstatement, the original retirement reason that is being reversed, and a reference to the retirement provenance record being reversed.

**Reinstatement cascade.** Reinstatement reverses the flags and chain_integrity decrements generated by the original retirement cascade, using the same hop limit and dampening model. Beliefs that were flagged due to the retirement are cleared. Chain_integrity scores are restored.

**The reinstatement window problem.** Reinstatement cannot automatically undo downstream modifications that happened during the retirement window — beliefs that were reviewed, modified, or retired themselves because of the flagging. Those changes need human review. The reinstatement operation therefore generates a secondary review cascade: for each direct dependent that was flagged and subsequently modified or retired during the retirement window, a review queue entry is created asking reviewers to assess whether the modification remains appropriate given the reinstated belief.

The secondary cascade is the only part of reinstatement that requires human action beyond authorizing the reinstatement itself. The primary cascade (flag reversal, chain_integrity restoration) is automated.

### 6.7 Distillation Semantics

When session history is distilled for storage efficiency, the distillation operation generates a provenance record that explicitly records what was compressed. Beliefs whose full reasoning chain was compressed are flagged as `distilled` in their provenance records. Consumers of the belief graph can query for distilled beliefs and treat them with appropriately reduced confidence.

Distilled session history moves to cold storage rather than being deleted. The distillation provenance record contains a pointer to the cold storage location, enabling retrieval when full provenance is required for investigation.

### 6.8 Key Design Decisions

**Immutability.** Provenance records are never modified after being written. If a provenance record is found to be incorrect, a correction record is appended that references the incorrect record and explains the correction. The original incorrect record remains in the log.

**Completeness over performance.** The provenance store is write-path optimized for completeness. Every dimension change is recorded even when this creates storage overhead. The audit capability the complete provenance record provides is not recoverable after the fact if records are omitted.

**What remains empirically determined.** Retention windows for each provenance tier, distillation criteria, cold storage retrieval latency requirements, and the minimum provenance chain length required for each trust tier.

---

## 7. Layer 5: The Promotion Interface

### 7.1 Responsibility

The promotion interface is the human-in-the-loop surface — the mechanism by which individual session learning becomes collective team knowledge, and by which the quality and trust of the belief graph is maintained over time.

### 7.2 The Scaling Problem

At low agent-to-human ratios, individual belief review is feasible. As the ratio increases, review volume grows faster than human review capacity. The promotion interface must scale without either creating a bottleneck that starves the belief graph of new content or degrading to rubber-stamp review that provides no real quality control.

The solution separates the human role from content review to policy authorship. Humans define promotion policies that govern which beliefs are automatically promoted under what conditions. The system validates belief candidates against policies automatically. Only candidates that fall outside policy boundaries — novel types, high conflict scores, trust level anomalies — reach human review.

### 7.3 The Policy Authoring Interface

A promotion policy is a human-authored rule that governs automatic promotion for a defined category of belief candidates.

**Policy schema:**
```json
{
  "policy_id": "uuid",
  "name": "human-readable policy name",
  "scope": "project_scope_id",
  "authored_by": "contributor_id",
  "created_at": "iso8601",
  "conditions": {
    "belief_types": ["constraint", "pattern"],
    "minimum_trust_level": "agent_derived",
    "maximum_conflict_score": 0.2,
    "minimum_scope_precision": 0.5,
    "contributor_role_required": null
  },
  "promotion_result": {
    "trust_level": "policy_promoted",
    "confidence_cap": 0.6,
    "requires_review_after_days": 90
  }
}
```

Policies are themselves versioned artifacts with provenance. When a belief is automatically promoted under a policy, the policy ID and version are part of the belief's provenance record. If a promoted belief is later found to be wrong, the policy that permitted its promotion can be identified and revised.

### 7.4 The Review Queue Interface

Belief candidates that fall outside policy boundaries are routed to the review queue. Reviewers see:

- The belief candidate's normalized content
- The exchange that produced it (with context)
- The full provenance chain
- Any existing beliefs the candidate conflicts with
- The system's explanation of why automatic promotion was not applied
- The dimension scores and how they compare to policy thresholds

Reviewer actions:

**Confirm.** The belief is accurate. Promote to the appropriate trust tier based on reviewer role. Update dimension scores.

**Modify.** The belief is approximately right but needs correction. The reviewer edits the normalized content. The original content and the modification are both preserved in the provenance record.

**Retire.** The belief is wrong or outdated. Move to the provenance store with a retirement record explaining why.

**Escalate.** The reviewer is uncertain and the belief has high potential impact. Route to a higher-authority reviewer with a mandatory response deadline.

**Defer.** The reviewer needs more information before deciding. The candidate remains in the queue with a note.

### 7.5 The Anomaly Detection Interface

At scale, human attention is the scarcest resource. The anomaly detection interface directs human attention to patterns in the belief graph that warrant investigation rather than individual beliefs.

**Belief velocity anomalies.** A sudden spike in belief candidates about a particular component or topic. Could indicate rapid legitimate learning or an agent operating under an adversarial prompt.

**Conflict clustering.** Multiple beliefs in conflict with each other concentrated in a short time window. Could indicate a codebase undergoing rapid change or a poisoning attempt.

**Trust level drift.** Beliefs from a previously reliable contributor suddenly scoring poorly on consistency with existing high-confidence beliefs. Could indicate a compromised session.

**Provenance gaps.** Beliefs that cannot be traced cleanly to a session and exchange. Could indicate a direct write attempt bypassing the extraction pipeline.

Anomaly detection findings route to a separate investigation queue, distinct from the belief review queue. The investigation queue is for patterns, not for individual beliefs.

### 7.6 The Confidence Tier Model

Not all beliefs require the same human review rigor. The confidence tier model creates a structured progression from agent-generated content to authoritative organizational knowledge.

| Tier | Trust Level | Promotion Path | Agent Weight |
|---|---|---|---|
| Provisional | agent_derived | Automatic, no policy required | Low — flagged when consumed |
| Policy-promoted | policy_promoted | Automatic under matching policy | Moderate |
| Peer-validated | peer_validated | Automatic at independent rederivation threshold | Moderate-high |
| Human-reviewed | human_reviewed | Explicit reviewer confirmation | High |
| Foundational | foundational | Human-authored directly | Highest — ground truth for conflict resolution |

Agents consuming the belief graph see the confidence tier and weight their reasoning accordingly. An agent building on a foundational belief can be more confident in its output than one building on a provisional belief. This confidence propagation makes it possible for downstream consumers — human reviewers, CI/CD pipelines, other agents — to understand how much trust to place in agent-generated work without reviewing every belief it relied on.

### 7.7 Key Design Decisions

**Low-friction promotion as a hard constraint.** A promotion interface that requires significant effort per belief will not be used consistently. The target interaction time for a standard belief review is under thirty seconds. Anything that consistently takes longer will be skipped under deadline pressure, degrading the graph.

**Partial adoption and honest representation.** Not all beliefs will be reviewed. The system should be explicit about the coverage of its promotion interface — what percentage of extracted beliefs have been reviewed, what the trust tier distribution looks like, and how those numbers are trending. A belief graph that represents its completeness honestly is more useful than one that implies full coverage it doesn't have.

**What remains empirically determined.** The policy schema parameters that produce good automatic promotion rates in practice, the review queue priority ordering, the anomaly detection thresholds, and the confidence tier weights for agent reasoning. All are configurable parameters requiring calibration.

---

## 8. Cost Model

### 8.1 Framing: Cost Shift, Not Cost Addition

The system does not add operational cost to AI-assisted development — it shifts cost from per-session token consumption to per-belief pipeline operations. Whether the shift is favorable depends on the read-to-write ratio in the deployment.

Context establishment in current tooling is paid on every session startup, every compaction, and every cold-start in CI. Each payment consumes session-tier model tokens — the most expensive class of LLM operation in the system. The cost is invisible because it is bundled into session billing rather than appearing as a separate line item, but it scales linearly with session count and is paid in full regardless of how much of the loaded context is relevant to the work performed.

The belief graph displaces this cost. A scoped query returns ranked-relevant beliefs only — typically a small fraction of the bulk context that would otherwise be loaded. The displacement compounds across all sessions in the deployment. In return, the system adds pipeline operations that scale with belief production: normalization, embedding generation, conflict detection, and cascade traversals. These are real costs but they are bounded, write-path, and amortizable across all subsequent reads of each belief.

### 8.2 The Human-Favoring Regime

At agent-to-human ratios below approximately 1:3, the cost case is unambiguously favorable. Belief production is bounded by human cognitive output — developers produce high-signal beliefs at the rate they can think through problems. Read activity scales with session count, which is much larger than write volume even before any agent multiplier. Read displacement dominates write addition by an order of magnitude or more.

In this regime, the dominant cost optimization is keeping normalization lightweight (a smaller model than the session model, as specified in Section 3.5) and ensuring embeddings are cached rather than recomputed.

### 8.3 The Agent-Favoring Regime

At ratios above approximately 1:3, the cost case still holds but depends on extraction selectivity. Agents generate belief candidates at much higher per-session rates than humans, do not self-limit on what is "worth recording," and corroborate each other's beliefs trivially. Without active filtering at the front end of the pipeline, write-side cost grows faster than read-side displacement and the system becomes a net cost increase.

The architecture already contains the mechanisms required for selective extraction. In the agent-favoring regime they shift from quality controls to economic controls:

- The minimum classification confidence threshold (Section 3.3.3) becomes the primary write-side cost lever
- Intra-batch deduplication (Section 5.5.2) prevents superlinear cost growth when many agents submit similar candidates concurrently
- Corroboration semantics that distinguish independent rederivation from confirming corroboration (Section 4.5) — peer-validated trust elevation depends on the independent count alone, and confirming corroborations contribute at a discounted weight to the corroboration dimension score. This prevents agent-to-agent corroboration cascades from inflating trust or generating expensive write operations with no epistemic value
- Per-session and per-contributor throttling (Section 9.3.6 layers 1 and 2), elsewhere framed as security controls, also function as cost controls bounding runaway pipeline spend from misconfigured agents

At extreme ratios, the merge pipeline's economics resemble those of a search indexing system more than those of a knowledge management system. Filtering low-value input cheaply at ingest time becomes more important than demoting low-value beliefs at query time. This is a structural shift in the system's operating model that calibration must reflect.

### 8.4 Normalization Quality as a Cost-Critical Path Dependency

The cost displacement only occurs if downstream agent sessions trust the belief graph as their authoritative context source. If normalization quality is poor enough that agents re-establish context from raw sources anyway, the system pays both the pipeline cost and the original context establishment cost. Normalization quality is on the critical path for the cost case to hold.

This dependency reinforces the design properties specified in Section 3.5 — particularly the requirement that the raw exchange is preserved regardless of normalization confidence, enabling re-normalization when the normalization model improves without loss of source material.

### 8.5 Operational Visibility

The cost shift moves spending from invisible (token consumption hidden in session activity) to visible (pipeline operations on a billing line). Operations teams must budget for the extraction pipeline as a distinct cost center even though the net deployment cost is lower than the unmanaged-context baseline.

Deployments should expose three metrics in operational telemetry:

- **Write-path cost per belief produced** — normalization, embedding generation, and conflict detection cost, attributed per promoted belief
- **Read displacement ratio** — session tokens saved by belief graph queries relative to the baseline session token consumption without the belief graph
- **Net cost trend over time** — write-path cost trending against read displacement, segmented by deployment regime

Tracking these makes the cost case auditable and allows calibration decisions in the extraction pipeline to be evaluated against their economic impact, not just their precision.

### 8.6 Cost Optimization Levers in the Architecture

Several existing design properties are also cost optimizations and are documented here to make their economic function explicit.

| Mechanism | Section | Cost Function |
|---|---|---|
| Lighter-weight normalization model | 3.5 | Avoids session-tier token spend on every candidate |
| Fast-path-first conflict detection | 5.4.2 | Keeps the majority of proximity-matched pairs out of the LLM-assisted comparison path |
| Delta-only inbound sync | 3.3.2 / 4.7.5 | Avoids full graph reload at every compaction event |
| Tiered storage (hot/warm/cold) | 4.6 | Object storage cost for cold tier scales with retention, not access |
| Intra-batch deduplication | 5.5.2 | Resolves near-duplicates before cross-graph operations |
| Aliasing rather than absorption in deduplication | 5.5 | One canonical belief receives all subsequent corroboration updates rather than dimension updates propagating to multiple similar beliefs |
| Per-session and per-contributor throttling | 9.3.6 | Bounds runaway pipeline spend from misconfigured agents (also a security control) |

### 8.7 Key Design Decisions

**Cost displacement, not cost elimination.** The system reduces total cost in any realistic deployment but does not eliminate cost. Pipeline operations are a real and bounded line item. Honest representation of this is preferable to claiming the system is free.

**Regime-dependent calibration.** The same architecture serves both human-favoring and agent-favoring regimes, but the calibration of front-end filters differs significantly. Deployments must identify their regime and tune accordingly.

**Normalization quality is on the cost critical path.** The displacement argument requires that the belief graph is trusted as authoritative. This is a quality requirement with direct economic consequences, not just a usability concern.

**Cost mechanisms are dual-purpose.** Several architectural properties function simultaneously as quality controls, security controls, and cost controls. The agent-favoring regime shifts the relative weight of these functions toward cost without changing the underlying mechanism.

**What remains empirically determined.** The crossover ratio at which the regime distinction becomes operationally significant, the normalization model cost-quality tradeoff curve, the actual read displacement ratio observed in practice for typical project scopes, and the per-belief pipeline cost at different scales.

---

## 9. Security Model

### 9.1 Design Posture

The belief graph will be attacked. This is not a risk to be mitigated to acceptable levels — it is a certainty to be designed for from the start. The system is explicitly designed on the assumption that sophisticated, persistent attackers will attempt to poison, exfiltrate, and manipulate the belief graph once it holds organizational value. Security is not a feature to be added later.

This section specifies the threat model, the defensive mechanisms, the incident response architecture, and the organizational controls required to operate a belief graph securely. Implementations that omit any of these components should not be considered production-ready.

---

### 9.2 The Threat Model

#### 9.2.1 Asset Characterization

The belief graph holds organizational knowledge that is categorically more sensitive than the codebase it describes in several specific ways:

**Reasoning over artifacts.** Source code shows what was built. The belief graph shows why, what alternatives were rejected, what security constraints were deliberately designed in, and what the team explicitly decided not to do. An attacker with read access to a mature belief graph understands the system's design intent, its known weaknesses, and the constraints that shaped its security posture — without needing to reverse engineer any of it.

**Non-rotatability.** Compromised credentials can be rotated. Compromised source code can be audited and patched. Compromised organizational reasoning cannot be un-known. Once an attacker has exfiltrated a belief graph, the intelligence value of that graph persists indefinitely regardless of subsequent defensive measures.

**Governance gap.** Most organizations have no current policy, monitoring, or classification covering AI session context. The belief graph formalizes and centralizes knowledge that currently exists in an ungoverned, unmonitored state. The act of centralizing it makes it a more visible target — but the knowledge it represents was always present and always valuable to an attacker.

#### 9.2.2 Attacker Profiles

**External attacker with developer machine access.** Demonstrated by the TeamPCP GitHub breach (May 2026). Gains access via poisoned developer tooling, credentials on compromised machines, or supply chain compromise. Seeks to exfiltrate session histories and context stores alongside source code and credentials. The belief graph is an additional high-value target that current incident response procedures don't account for.

**Insider threat.** A developer with legitimate access who exfiltrates organizational knowledge on departure or sells access. The distributed local model has no defense against this — departing developers take their accumulated context with them through normal channels. The centralized model enables access revocation, audit logging, and detection of anomalous bulk reads.

**Sophisticated persistent attacker targeting the belief graph directly.** Understands the belief graph architecture and attempts to introduce malicious beliefs through legitimate-looking interactions. Goal is to influence agent behavior in ways that are difficult to detect — introducing subtle security vulnerabilities, directing agents away from secure patterns, or creating exploitable gaps between what the agent believes and what is true.

**Automated model-assisted attacker.** Uses capable AI models to probe the belief graph systematically — identifying belief patterns, crafting inputs designed to produce specific belief candidates through normal-looking interactions, and attempting to identify the provenance chains that would need to be forged to give a malicious belief high trust. The capability of this attack scales with model capability and will grow over time.

#### 9.2.3 Attack Vectors

**Direct poisoning through normal interaction.** The attacker participates in a legitimate session and crafts exchanges designed to produce high-scoring belief candidates that, when promoted, alter agent behavior in exploitable ways. This attack bypasses most defenses because it looks like normal usage.

**Provenance spoofing.** The attacker attempts to forge a plausible provenance chain for a high-trust-level belief — authenticated session, plausible contributor, consistent dimension history. The goal is to make a malicious belief appear to have passed human review or to have been authored directly by a trusted human contributor.

**Promotion policy manipulation.** The attacker gains access to the policy authoring interface and creates or modifies policies to automatically promote malicious belief candidates that would otherwise be flagged for human review.

**Context rot exploitation.** The attacker prevents legitimate belief updates from reaching the graph — through denial of service against the extraction pipeline, by introducing confusion that causes human reviewers to reject valid updates, or by targeting the developers most likely to produce corrective belief candidates. The goal is to widen the gap between what the agent believes and the current state of the system, then exploit that gap.

**Bulk exfiltration.** The attacker gains read access to the belief graph and extracts its contents. May be accomplished through compromised credentials, exploitation of the API layer, or access to the underlying storage infrastructure.

**Cold storage exfiltration.** The full provenance history and distilled session archives in cold storage may contain more sensitive information than the active graph. An attacker targeting cold storage can reconstruct the full history of organizational decision-making.

---

### 9.3 Defensive Architecture

#### 9.3.1 Authentication and Authorization

**Identity verification for all write operations.** Every belief candidate submission, policy authoring action, and review queue action must be authenticated to a specific contributor identity. Anonymous or pseudonymous writes are not permitted at any trust level. Authentication must use a cryptographically strong mechanism — API keys scoped to project and role at minimum, hardware security module backed identity for high-trust-level operations.

**Role-based access control with least privilege.** Access to the belief graph is governed by role:

| Role | Read | Submit Candidates | Review Queue | Author Policies | Admin |
|---|---|---|---|---|---|
| Agent (session) | Scoped to project | Scoped to session | No | No | No |
| Developer | Scoped to project | Yes | Yes | No | No |
| Reviewer | Scoped to project | Yes | Yes | No | No |
| Policy Author | Scoped to project | Yes | Yes | Yes | No |
| Admin | Organization-wide | Yes | Yes | Yes | Yes |
| Pipeline Agent | Scoped to project | Scoped to pipeline | No | No | No |

No role has direct write access to the belief graph itself. All writes go through the extraction pipeline and merge evaluator. This is a hard architectural constraint, not a configuration option.

**Cross-scope isolation enforcement.** Access controls are enforced at the scope level. A contributor with access to Project A cannot read or submit to Project B without explicit authorization. Cross-scope promotion requires authorization from both scope administrators.

**Credential rotation policy.** All credentials used to access the belief graph must be rotatable without service interruption. Session credentials are time-limited. Pipeline credentials are scoped to specific pipeline identities and rotated on a defined schedule. Compromised credentials can be revoked without affecting other access.

#### 9.3.2 The Provenance Integrity Mechanism

The provenance layer is the primary defense against poisoning attacks. Its integrity depends on the following properties being enforced:

**Cryptographic signing of provenance records.** Every provenance record is signed with the contributor's identity key at creation time. The signature covers the full record content including the belief content, dimension values, timestamp, and contributor identity. A forged provenance record requires forging the contributor's signature — which requires access to their private key.

**Tamper-evident log structure.** The provenance store uses a hash-chained append-only structure. Each record contains the hash of the previous record. Tampering with any record invalidates all subsequent hashes, making tampering detectable. The chain head hash is published to an external notary at regular intervals, preventing retroactive rewriting of the chain.

**Provenance completeness verification.** Before any belief is elevated to human-reviewed or foundational trust level, the system verifies that the belief's full provenance chain is intact and consistent — no gaps, no hash mismatches, no signature verification failures. Incomplete or inconsistent provenance chains cannot produce high-trust beliefs.

**Out-of-band notarization for high-trust events.** Human review confirmations, policy authorizations, and foundational belief creation events are notarized out-of-band — recorded in a separate append-only log that is independently auditable and not modifiable through the belief graph's own API. This prevents an attacker who gains full control of the belief graph API from retroactively legitimizing malicious beliefs.

#### 9.3.3 Anomaly Detection

Anomaly detection operates continuously on the belief graph's write stream and the active graph's state. It does not prevent all attacks — it provides early warning that enables human investigation before an attack propagates.

**Belief velocity monitoring.** Track the rate of belief candidate submissions per project scope, per contributor, and per belief type. Alert when rates deviate significantly from the established baseline for that scope and contributor. Velocity spikes that do not correlate with known development activity (new feature work, major refactoring) warrant investigation.

**Content drift detection.** Monitor for systematic shifts in the content of belief candidates from specific contributors or sessions — particularly shifts that move beliefs toward less restrictive security constraints, reduced IAM boundaries, or relaxed validation patterns. These are signatures of a direct poisoning attempt.

**Conflict pattern analysis.** Monitor for clusters of conflicting belief candidates submitted within narrow time windows. Isolated conflicts are normal. Clusters of conflicts about the same component or constraint category, particularly from new or unfamiliar contributors, are a poisoning signature.

**Trust escalation monitoring.** Alert when beliefs move through trust tiers faster than the expected validation timeline, or when a single reviewer confirms an unusually high volume of beliefs in a short period. Both patterns may indicate a compromised review account or a policy that is too permissive.

**Provenance chain anomalies.** Monitor for belief candidates with unusual provenance properties — sessions of abnormally short duration producing high volumes of high-confidence candidates, exchanges with content patterns inconsistent with legitimate development work, candidates submitted from sessions that cannot be corroborated by the session log.

**Geographic and temporal access anomalies.** Alert on read access to the belief graph from unexpected locations, at unexpected times, or by contributor identities whose access patterns deviate significantly from their established baseline. Bulk reads — a contributor accessing a large fraction of the belief graph's content in a short period — warrant immediate investigation.

#### 9.3.4 The Human Review Gate as Security Control

The promotion interface's human review gate is not only a quality control mechanism — it is a security control. Beliefs cannot reach high-trust tiers without human review, which means that even a successful automated poisoning attempt is limited in its influence until it survives human review.

The review queue interface must present reviewers with the information they need to detect manipulated belief candidates:

- The full session context, not just the extracted belief, so reviewers can see whether the exchange that produced the belief looks legitimate
- The contributor's history of previous belief submissions and their accuracy track record
- Any anomaly detection flags associated with this candidate or its contributing session
- The potential downstream effects of this belief on agent behavior if promoted

Reviewers should be trained to treat the belief review process as a security function, not just a quality function. The review queue is where social engineering attacks on the belief graph are caught.

#### 9.3.5 Scope Compartmentalization

Beliefs in one project scope cannot influence beliefs in another without explicit cross-scope promotion requiring administrator authorization. This limits the blast radius of any single poisoning success to the scope in which it occurred.

Within a scope, belief types can be further compartmentalized. Security constraint beliefs can require a higher minimum trust level for promotion than pattern or observation beliefs. This means an attacker targeting security constraints faces a higher bar than one targeting less sensitive belief categories.

Sensitive belief categories — security constraints, IAM policies, authentication patterns, data handling requirements — can be flagged as requiring mandatory human review regardless of promotion policy. These beliefs never enter the automatic promotion path.

#### 9.3.6 Multi-Layer Throttling

Rate limiting alone is insufficient in an agent-heavy deployment. The distinction between legitimate high-volume submission and an attack is not reliably detectable at the API layer alone. The throttling model uses four layers operating simultaneously, each catching a different attack profile.

**Layer 1: Per-session rate limits.** Protects against burst attacks from a single compromised session. The limit should be calibrated against legitimate session behavior — setting it too low breaks high-volume agent workflows. Returns a 429 with `Retry-After` header for legitimate clients. Does not reveal threshold values in the response body.

**Layer 2: Per-contributor adaptive limits.** Tighter than per-session limits. Catches a single compromised identity submitting from multiple sessions by linking all sessions to the contributor identity. Adaptive: contributors with a long history of high-quality submissions that pass review earn more generous limits over time. New contributors, contributors with recent review failures, and contributors whose recent submissions show unusual patterns operate under tighter constraints until they establish a track record.

**Layer 3: Scope circuit breaker.** A hard limit on total inflow to a project scope regardless of session or contributor count. When total submission volume across all sessions exceeds the threshold, automatic promotion is suspended and all subsequent submissions in that scope route to enhanced review. This is the primary defense against distributed attacks where load is spread across many sessions to stay under per-session limits. The circuit breaker threshold is calibrated against the scope's legitimate peak activity — a scope with a large multi-agent CI pipeline has a much higher legitimate peak than one with three human developers.

**Layer 4: Content-based routing to enhanced review.** Suspicious submissions are accepted but routed to a high-scrutiny queue rather than rejected. A candidate that passes schema validation but has anomalous content properties — very high initial confidence assertions, scope claims inconsistent with the session's file access patterns, normalized content that diverges significantly from the raw exchange content — receives a 202 response from the attacker's perspective while being held for enhanced review internally. This prevents the attacker from knowing their submissions are being scrutinized and from adjusting pacing to avoid detection.

**Anomaly detection feedback loop.** When the anomaly detection layer identifies a velocity anomaly for a specific contributor or session, it feeds back into Layer 1 and Layer 2 throttling — temporarily reducing limits for the affected contributor or session while investigation proceeds. Detection triggers throttling; throttling does not wait for investigation to complete.

---

### 9.4 Availability Attacks and Recovery

The belief graph is a target for availability attacks — not just poisoning — because preventing legitimate beliefs from reaching the graph degrades agent behavior without requiring the attacker to introduce malicious content. A DDoS against the merge pipeline creates the same degradation as a poisoning attack from a different direction.

#### 9.4.1 Threat Characterization

**DDoS against the merge pipeline.** High-volume submission from compromised sessions can overwhelm the merge evaluator, causing 503 responses for legitimate sessions. Sessions operate in degraded mode with stale cached beliefs. The belief graph's quality degrades as legitimate candidates accumulate in the durable queue without processing.

**DDoS-by-recovery.** The attack doesn't need to sustain. It needs to last long enough to build a large backlog in the durable queue. When the attack stops, the recovery flush of accumulated candidates overwhelms the pipeline, extending the effective outage. The attacker triggers the recovery flood and steps away.

**Intermittent availability during attack.** Spotty throughput during a DDoS creates partial state — some candidates process, some don't. Sessions have divergent sync cursor states. The graph's state during the attack window is uncertain. When availability is restored, the staleness calculation for each session is uncertain because neither the session nor the graph has a clean picture of what processed during the attack.

#### 9.4.2 Primary Mitigations

**Durable queue as resilience boundary.** The durable message queue between the extraction pipeline and merge evaluator (Section 5.3.1) is the primary resilience mechanism for availability attacks. Candidates are durable from the moment they enter the queue — they survive merge evaluator unavailability, infrastructure failures, and extended DDoS without being lost. The queue absorbs the attack; the merge evaluator resumes from where it stopped.

**Graph watermark for intermittent availability.** The processed watermark (Section 5.3.3) identifies the boundary between what the graph has and hasn't processed during a period of spotty availability. On recovery, the merge evaluator resumes from the watermark position rather than attempting to reconstruct which candidates processed during the attack window.

**Throttled recovery drain.** The merge evaluator drains the backlog queue at a configurable throttled rate on recovery, well below sustained throughput capacity. This leaves headroom for new submissions from live sessions and prevents the recovery flood from overwhelming the pipeline. The drain rate is a configurable operational parameter calibrated against the pipeline's measured sustained throughput.

**Priority ordering on recovery drain.** During backlog drain, foundational and human-reviewed candidates process before provisional agent-derived candidates. This improves belief graph quality during the recovery window even when full backlog processing takes time.

**Partial recovery mode.** New submissions from live sessions receive processing priority over backlog candidates of equivalent trust level. The system returns to useful operation immediately on availability restoration, not after the full backlog has drained.

**Scope circuit breaker as flood protection.** The scope circuit breaker (Section 9.3.6 Layer 3) doubles as recovery flood protection. If the recovery drain itself threatens to overwhelm the pipeline, the circuit breaker can be triggered manually by an administrator to slow the drain further.

#### 9.4.3 Session Recovery After Outage

Sessions that experienced unavailability during an attack reconnect with a potentially stale sync cursor. The `last_acknowledged_watermark` field in the submission request (Section 10.1.2) provides the merge evaluator with enough information to compute the true divergence between the session's state and the graph's current state, even when the session's sync cursor predates the outage.

Recovery procedure for a session reconnecting after an outage:

1. Session pulls the current graph delta using its last sync cursor
2. Delta response includes the current `graph_watermark`
3. Session computes divergence between its `last_acknowledged_watermark` and the current `graph_watermark`
4. If divergence is within the staleness threshold, session submits its accumulated backlog with the `last_acknowledged_watermark` field populated
5. If divergence exceeds the staleness threshold, session receives a 409 and routes its backlog to human review rather than automatic submission

---

### 9.5 Incident Response

When a poisoning event is detected or suspected, the following response sequence applies:

**Isolation.** Immediately suspend automatic promotion for the affected scope. All subsequent belief candidates in that scope are routed to human review regardless of policy. This prevents further propagation while investigation proceeds.

**Belief graph snapshot.** Take an immutable snapshot of the current belief graph state before any remediation. This preserves evidence and provides a rollback point.

**Provenance traversal.** Starting from the suspected malicious belief or beliefs, traverse the provenance graph backward to identify the contributing sessions, contributors, and exchanges. Forward from the malicious beliefs, identify all beliefs that were influenced by or corroborated the malicious beliefs — these are potentially contaminated and must be reviewed.

**Contributor investigation.** For each contributor whose session contributed to the potentially malicious belief chain, audit their full contribution history within the affected scope. Look for patterns consistent with a systematic poisoning attempt rather than an isolated error.

**Affected belief retirement.** Retire malicious beliefs and any beliefs whose trust scores were inappropriately elevated by association with malicious contributions. Retirement is recorded in the provenance store with a full explanation — the provenance record serves as the incident log.

**Policy review.** If a promotion policy permitted malicious beliefs to be automatically promoted, the policy must be suspended and reviewed before being reinstated. Policy authors should be notified and the policy's promotion history audited.

**Rollback if necessary.** If the scope of contamination is large enough that targeted retirement is insufficient, the belief graph for the affected scope can be rolled back to the pre-incident snapshot. Beliefs generated during the contaminated period are re-evaluated from their original belief candidates under enhanced review.

**Post-incident provenance audit.** After remediation, conduct a full audit of the provenance chain for the affected period to verify that the remediation was complete and that no contaminated beliefs survived the process.

---

### 9.6 The Self-Hosting Requirement

The belief graph must be self-hostable. The reasoning is architectural, not ideological.

The organizational knowledge the belief graph holds is subject to the organization's data governance policies, regulatory obligations, and security requirements. A vendor-hosted belief graph places that knowledge under terms of service that can change, in infrastructure the organization cannot audit, subject to access by parties the organization has not authorized.

Specific requirements for a compliant self-hosted deployment:

**Data residency.** All belief graph data — active graph, warm provenance, cold archive — must reside on infrastructure the organization controls. No belief content, provenance records, or access logs may be transmitted to external services except through explicitly authorized integrations.

**Key management.** Cryptographic keys used for provenance signing must be managed by the organization, not the software vendor. Hardware security module integration is recommended for high-trust-level key operations.

**Audit log sovereignty.** Audit logs must be stored on infrastructure the organization controls and must not be accessible to the software vendor. The organization must be able to provide audit logs to regulators or auditors without vendor intermediation.

**Portable export.** The belief graph must be exportable in an open format that enables migration to alternative implementations without loss of belief content, dimension history, or provenance records. Export must be available on demand, not subject to vendor approval.

**Vendor access controls.** If the vendor provides support services that require access to the organization's belief graph, that access must be governed by the organization's access control policies, logged in the organization's audit log, and terminable by the organization at any time.

---

### 9.7 Regulatory and Compliance Considerations

The belief graph may contain reasoning about systems that handle personal data, health information, financial records, or other regulated information categories. Organizations operating in regulated industries should assess whether:

**GDPR and equivalent regulations.** If the belief graph contains reasoning about how personal data is processed, it may itself constitute processing of personal data under GDPR and equivalent frameworks. Data subject rights — access, rectification, erasure — may apply to belief graph content. The provenance layer's immutability requirement and the erasure requirement appear to be in tension, but the architectural answer is cryptographic erasure (also called crypto-shredding), which resolves the tension cleanly:

- **Per-contributor keys for raw exchange content.** Raw exchange content in the provenance store is encrypted at rest with a key derived from a per-contributor key. When a contributor exercises right-to-erasure, their per-contributor key is destroyed. The ciphertext remains on disk but becomes mathematically inert — recovery requires the destroyed key.
- **Project-scope keys for belief content.** Normalized belief content is encrypted with project-scope keys. This granularity supports project-level erasure (decommissioning a project, transferring control to another organization) without requiring per-contributor key infrastructure for every belief.
- **Hashes in the provenance chain.** Provenance records store hashes of ciphertext, timestamps, and chain pointers. Hashes survive key destruction — the chain still verifies after erasure, the chain of custody is preserved, but the content referenced by the hashes is irrecoverable.

The EU Data Protection Board's Guidelines 02/2023 on technical and organizational measures explicitly recognize cryptographic deletion as a valid erasure technique under appropriate conditions. The UK ICO's position is similar. The mechanism is well-understood in regulated industries and has been used in financial services and healthcare data architectures for over a decade. Specific jurisdictional interpretation should still be confirmed with counsel — the legal interpretation of cryptographic erasure is an active question in some non-EU jurisdictions.

**What cryptographic erasure does not solve.** The technique addresses encrypted content but cannot address personal data that has been written in plaintext into the normalized belief content itself. Section 3.5.1's "declarative and system-centric" normalization property is the primary control here — properly normalized belief content does not identify individuals ("the project's Lambda functions have a 15-second timeout" rather than "Engineer X said the timeout is 15 seconds"). When normalization fails and a belief's content directly contains personal data, the erasure path is re-normalization to remove the identifying information (Section 3.5.4 lifecycle infrastructure supports this) or belief retirement with a redaction tombstone. Both mechanisms already exist in the architecture and serve the erasure use case for this edge case.

**Operational cost.** Per-contributor key infrastructure is a real operational expense — keys must be generated, stored in a hardware security module or equivalent, backed up, rotated on a schedule, and destroyed on erasure events. Deployments in regulated industries should budget for key management infrastructure as a distinct cost center alongside the extraction pipeline operational costs described in Section 8.5. Deployments not subject to data subject rights frameworks can operate without per-contributor keys, treating crypto-erasure as a feature enabled only when regulatory requirements demand it.

**EU AI Act documentation requirements.** The EU AI Act requires documentation of high-risk AI systems including descriptions of the logic used to make decisions. The belief graph's provenance layer is a candidate mechanism for satisfying this requirement — it provides a traceable record of the reasoning that influenced AI-assisted development decisions. Organizations subject to the Act should assess whether and how the belief graph satisfies applicable documentation obligations.

**Sector-specific regulations.** Healthcare organizations (HIPAA), financial services organizations (SOC 2, PCI DSS), and government contractors (FedRAMP, ITAR) face sector-specific requirements that may apply to belief graph content and operations. Compliance assessment should be conducted before deploying a belief graph in regulated contexts.

**Discovery risk.** The belief graph's comprehensive record of architectural decisions, security considerations, and rejected approaches may be subject to discovery in litigation. Organizations should assess retention policies with legal counsel before deployment. The argument for having a governed, policy-based retention system — rather than unmanaged session logs scattered across developer machines — is that it makes discovery obligations predictable and manageable rather than open-ended.

---

### 9.8 The Distributed Local Model is Not Safer

The conventional security intuition that distribution reduces risk does not apply to this specific artifact. The distributed local model for AI session context creates a larger, less visible, and less mitigable attack surface than a well-designed centralized belief graph.

In the distributed model: N developer machines hold sensitive organizational knowledge. Each has a different security posture. None is monitored for belief graph access. None has access control governing who can read the accumulated context. None has an audit log of who accessed what and when. Departing developers take their accumulated context through normal channels with no process to capture or revoke it.

In the centralized model: one system holds the same knowledge. It can be defended with enterprise-grade controls. Access is authenticated, authorized, and logged. Compromise is detectable. Affected credentials can be revoked. Contaminated beliefs can be identified and retired.

The centralized model's failure modes — a single high-value target, larger blast radius per successful attack — are real but they are known failure modes with known mitigations. The distributed model's failure modes are novel, largely unrecognized, and currently unmitigated in every organization actively using AI-assisted development.

The GitHub TeamPCP breach demonstrated this in production: a single poisoned developer tool gave attackers access to 3,800 internal repositories. The attack specifically targeted AI coding assistant configuration files — CLAUDE.md and .cursorrules — with zero-width Unicode injection because those files are trusted by agents and invisible to security tooling. The attack produced no CVE for this specific vector and was missed by standard scanners. This is not a hypothetical. It is a documented attack on the distributed local model that the centralized model — with its access controls, audit logging, and anomaly detection — would have had significantly better defenses against.

This threat directly motivates the migration review gate specified in Section 10.3.3. Organizations migrating from current tooling cannot assume their source files are clean. Importing CLAUDE.md content directly into the foundational trust tier — as a naïve migration design would do — would propagate any pre-migration contamination into the highest authority position in the new system. The migration design specified here imports at `policy_promoted` and requires an administrator review decision before cutover for every imported entry, with the review gate as the architectural control that prevents poisoned content from surviving migration as ground truth.

---

## 10. Integration Points

### 10.1 Agent Session Interface

The agent session interface is the primary runtime integration point — every session startup and every session submission passes through it. It must satisfy the session startup latency budget (target: sub-5-seconds for typical project scopes; size and concurrency assumptions backing all latency targets are specified in Section 4.7.7) and handle the compaction sync cycle described in Section 3.3.2.

#### 10.1.1 Session Startup Query

Retrieves active beliefs relevant to the session's scope. Used at session start and at each compaction sync cycle for inbound delta pulls.

```
GET /v1/beliefs
  ?project_scope_id={uuid}           required
  &component_scope={path}            optional — narrows to component-level beliefs
  &belief_types[]={type}             optional — filter by type, repeatable
  &minimum_trust_level={enum}        optional — filter by minimum trust tier
  &limit={int}                       optional — default 50, max 200
  &sync_cursor={cursor}              optional — if provided, returns delta since cursor

Response 200:
{
  "beliefs": [                       // ranked by relevance score
    {
      "belief_id": "uuid",
      "content": "normalized belief statement",
      "type": "constraint | decision | pattern | rejection | observation",
      "scope": { ... },
      "dimensions": { ... },         // full dimension vector including chain_integrity
      "lifecycle_state": "provisional | active | flagged",
      "trust_level": "enum",
      "flags": []                    // active flags if lifecycle_state is flagged
    }
  ],
  "flagged_beliefs": [],             // beliefs with active flags, surfaced separately
  "sync_cursor": "opaque cursor",    // advance this cursor on next delta pull
  "graph_state_hash": "hash",        // for staleness detection at submission time
  "delta_only": false                // true if sync_cursor was provided
}
```

The `flagged_beliefs` array is separate from the main beliefs array so consuming agents can apply appropriate uncertainty to flagged content without needing to filter the main list. Both arrays are returned in every response — agents decide how to weight flagged beliefs in their reasoning.

#### 10.1.2 Belief Candidate Submission

Submits belief candidates extracted from a session for asynchronous merge evaluation. Called at each compaction sync cycle as the outbound phase, and on-demand when explicitly triggered.

```
POST /v1/candidates
Body:
{
  "session_id": "uuid",
  "sync_cursor": "opaque cursor",    // session's sync cursor at submission time
  "graph_state_hash": "hash",        // hash from last startup query — staleness check
  "candidates": [
    {
      // belief candidate objects per Section 3.6 schema
    }
  ]
}

Response 202 (accepted for async processing):
{
  "submission_id": "uuid",           // for status tracking
  "queued_count": 12,
  "immediately_rejected_count": 1,   // unnormalized, schema invalid, etc.
  "estimated_processing_seconds": 30,
  "graph_watermark": "timestamp",    // current processed watermark — advance local record
  "recovery_mode": false             // true if merge evaluator is in partial recovery mode
}

Response 409 (sync cursor diverged beyond staleness threshold):
{
  "error": "sync_cursor_stale",
  "current_graph_state_hash": "hash",
  "graph_watermark": "timestamp",
  "divergence_measure": 0.87,        // normalized 0.0-1.0
  "action_required": "pull_delta_and_resubmit | request_human_review"
}
```

The 409 response makes the staleness threshold visible at the API layer. A session whose sync cursor has diverged beyond the threshold receives an explicit error rather than a silent bad merge. The `action_required` field indicates whether the session should pull a delta and resubmit (divergence below the hard staleness limit) or route to human review (divergence above it).

Sessions must record the `graph_watermark` from each successful 202 response as `last_acknowledged_watermark`. This value is provided in subsequent submission requests and used by the merge evaluator to compute true divergence after outages.

The submission request body should include `last_acknowledged_watermark`:

```json
{
  "session_id": "uuid",
  "sync_cursor": "opaque cursor",
  "graph_state_hash": "hash",
  "last_acknowledged_watermark": "timestamp",
  "candidates": [ ... ]
}
```

#### 10.1.3 Submission Status

Checks the status of an asynchronous submission.

```
GET /v1/candidates/{submission_id}

Response 200:
{
  "submission_id": "uuid",
  "status": "queued | processing | complete | failed",
  "processed_count": 10,
  "promoted_count": 6,
  "updated_count": 2,
  "flagged_count": 1,
  "rejected_count": 1,
  "errors": []                       // for failed submissions
}
```

#### 10.1.4 Error States

| Status Code | Error | Meaning | Recovery |
|---|---|---|---|
| 401 | `unauthorized` | Invalid or expired session credential | Refresh credential and retry |
| 403 | `forbidden` | Contributor lacks access to project scope | Request scope access or check role |
| 409 | `sync_cursor_stale` | Sync cursor diverged beyond staleness threshold | Pull delta and resubmit, or route to human review |
| 422 | `invalid_candidates` | Candidate schema validation failure | Fix schema errors and resubmit |
| 429 | `rate_limited` | Per-session, per-contributor, or scope circuit breaker limit exceeded | Back off and retry after `Retry-After` header — threshold values not disclosed in response body |
| 503 | `service_unavailable` | Belief graph unavailable | Degrade gracefully — session continues without sync, record last acknowledged watermark for reconnection |

The 503 response is critical: the session must continue operating even when the belief graph is unavailable. Degraded operation means the session uses its last cached belief graph state without attempting to sync until the service recovers. The session should not fail or block because the belief graph is temporarily unavailable.

### 10.2 CI/CD Pipeline Interface

Pipeline agents connect to the belief graph using a service account contributor identity with read access and submission access (no direct writes, no review queue access, no policy authoring). Pipeline sessions receive the same belief graph context as developer sessions at the same API endpoints.

Pipeline-generated belief candidates are automatically marked with the pipeline contributor identity and receive lower initial trust scores than human-session-derived candidates. Pipeline submissions include a `pipeline_run_id` field in the submission body that links the submission to the specific pipeline run for audit purposes.

**Pipeline-specific error handling.** Pipeline agents that receive a 503 (service unavailable) should not fail the pipeline run — they should log the unavailability and continue with the last cached belief state. Failing a pipeline run because the belief graph is temporarily unavailable would make the belief graph a critical path dependency for CI/CD, which is not the intended deployment model.

### 10.3 Migration from Current Tooling

Migration from current tooling follows a three-phase process. Each phase is independently completable — organizations can stop after any phase and operate in a stable state.

#### 10.3.1 Phase 1: Bulk Import

One-time ingestion of existing memory entries from current tooling sources: CLAUDE.md files, LangGraph checkpoints, team memory sync entries, and any other structured context documents the organization maintains.

**Trust assignment.** Imported entries enter as `policy_promoted` beliefs — the second-lowest trust tier — regardless of how they were originally created. The rationale is twofold:

- *Conceptual accuracy.* Imported entries ARE policy-promoted: they were brought into the belief graph by an administrator's decision to run the migration policy, not by independent verification or human review of the content. Assigning them a higher tier would misrepresent how they got into the graph.
- *Security.* Source files in current tooling (CLAUDE.md, .cursorrules, and equivalent) are routinely written by agents and have been demonstrated targets of poisoning attacks (Section 9 — the GitHub TeamPCP breach). Importing them directly into the foundational tier — the system's ground truth for conflict resolution — would inject any pre-migration contamination into the highest authority position in the new system. Importing at `policy_promoted` keeps imported content useful (it contributes to session context immediately) while preventing imported entries from automatically dominating conflicts against beliefs at higher tiers.

Imported entries remain queryable from day one and contribute to session context retrieval. They are subject to normal conflict resolution semantics — beliefs at `peer_validated`, `human_reviewed`, or `foundational` can override imported entries through the standard trust-level gate (Section 5.6.1 Stage 1). Imported entries can elevate to `peer_validated` through the normal `independent_rederivation_count` threshold, which is correctly gated by the corroboration classification rules (Section 4.5) — sessions reading migrated beliefs and producing dedup candidates are classified as `confirming`, not `independent`, so imported beliefs cannot echo-elevate.

**Migration review status.** Each imported belief carries a `migration_review_status` field with one of four values:

- `pending` — the initial state; the entry has been imported but not reviewed
- `retained` — an administrator has reviewed the entry and confirmed it should remain at `policy_promoted` with no change
- `elevated` — an administrator has reviewed the entry and elevated it to `human_reviewed` or `foundational` based on review judgment
- `retired` — an administrator has reviewed the entry and decided it should not survive migration; the entry is retired with a migration-decision provenance record

The status field exists only on imported beliefs. Beliefs created through normal session extraction after Phase 1 do not carry this field.

**Provenance record.** Each imported belief carries a provenance record with `event_type: imported`, the source tool and file path, the ingest timestamp as the creation date, the initial `migration_review_status: pending`, and a migration note identifying the migration batch. This preserves auditability even though the original session provenance is not available. Subsequent administrator actions on the entry (review decisions, elevations, retirements) produce additional provenance records linked to the original import.

**Idempotency.** Bulk import can be re-run without creating duplicate beliefs. Duplicate detection during import is type-aware — an existing CLAUDE.md entry that maps to the same belief as a previously imported entry is aliased rather than duplicated. The import operation produces a summary report of new beliefs created, aliases created, and entries that could not be parsed.

**Failure modes.** Import failures generate quarantine records for manual review, not silent drops. A failed import does not block the import of other entries in the same batch.

#### 10.3.2 Phase 2: Parallel Running Period

The belief graph runs alongside existing tooling for a configurable period (recommended minimum: 30 days). This period validates that the belief graph is producing useful context before existing tooling is deprecated.

During parallel running:
- New beliefs accumulate in the belief graph through normal session extraction
- Existing tooling continues to operate and update existing context files
- Developer sessions query both — the belief graph for structured beliefs, existing tooling for legacy context
- Discrepancies between the belief graph and existing tooling are surfaced to developers as learning opportunities for tuning the extraction pipeline and promotion policies
- Imported entries with `migration_review_status: pending` are surfaced in a dedicated migration review queue, allowing administrators to review entries incrementally rather than as a bulk activity at cutover

The parallel running period is the primary empirical calibration window. Extraction heuristic thresholds, promotion policy conditions, and scoring function weights should be actively tuned against the difference between what the belief graph captures and what developers consider important.

**Migration review during parallel running.** The migration review queue exposes imported entries with `pending` status, ranked by frequency of access from sessions during the parallel period. Frequently-accessed entries surface first — administrators review the entries that are actually being used before the entries that may have been dead in the source files. Each review produces a status transition (`retained`, `elevated`, or `retired`) and an admin-action provenance record. Reviewing all entries is not required during Phase 2, but doing so spreads the review workload over the parallel period rather than concentrating it at cutover.

#### 10.3.3 Phase 3: Cutover

Existing tooling is deprecated as the primary context source. The belief graph becomes the sole context store for participating sessions.

**Migration review gate.** Cutover is gated on the migration review status of all imported entries. Cutover is permitted only when zero imported beliefs remain in `migration_review_status: pending`. Each pending entry must be resolved to one of `retained`, `elevated`, or `retired` before cutover can complete.

The gate exists for two reasons:

- *Security.* The gate is the architectural control against pre-migration contamination of source files surviving into the new system. An administrator must have considered each entry before the system treats it as anything more than a candidate for organizational knowledge.
- *Curation.* Migration is a natural moment to prune. Source files in current tooling accumulate entries that were useful once and have not been removed because removal requires effort and judgment. The review gate forces that judgment to happen, and many organizations will find that a meaningful fraction of their imported entries should be retired rather than carried forward.

The default resolution is `retained`. Most imported entries are appropriately classified as `policy_promoted` and should remain there — they are useful context that came in through an administrator's migration policy, which is exactly what `policy_promoted` means. Elevation to `human_reviewed` or `foundational` is reserved for entries that the administrator has affirmatively verified as accurate and that the organization wants to treat as ground truth. Retirement is reserved for entries that the administrator has determined should not survive migration.

The gate's operational cost is proportional to the size of the imported set. A migration of 50 entries adds 50 review decisions. A migration of 1000 entries adds 1000 review decisions — substantial, but a one-time cost paid at deployment against the long-term security and quality benefit. Phase 2's incremental review approach (Section 10.3.2) is designed to spread this cost over the parallel period rather than concentrating it at cutover.

**Archival of legacy files.** Legacy context files (CLAUDE.md, memory entries, etc.) are archived rather than deleted. They remain accessible for reference and for re-import if needed, but are no longer updated or consulted during session startup. Organizations should retain legacy files for at least one year after cutover for audit and rollback purposes.

**Rollback path.** If the belief graph needs to be taken offline after cutover, the last bulk export of the belief graph can be used to regenerate CLAUDE.md files for each project scope, returning to the legacy tooling model. This rollback path must be tested before cutover is finalized. Note that rollback produces files derived from beliefs that have already passed the migration review gate — the rollback files reflect the organization's post-migration curation decisions, not the pre-migration state of the source files.

### 10.4 Cross-Tool Compatibility

The belief graph is tool-agnostic. An agent session in Claude Code, Cursor, Copilot, or any other tool with MCP support can read from and contribute to the same belief graph through the MCP server interface described in Section 10.1.

Tool-specific adapters may be needed to translate between tool-native session log formats and the JSONL format expected by the extraction pipeline. The adapter is responsible for format translation only — it does not perform extraction, normalization, or merge evaluation. Those operations are performed by the extraction pipeline against the translated JSONL.

---

## 11. Open Questions

The following questions are explicitly deferred to empirical calibration from real deployments or to ongoing research. They are open problems, not design oversights.

**Default dimension weights.** What are the right initial values for the composite scoring function in the merge strategy? How should trust level be weighted relative to recency and corroboration? What weight should chain_integrity carry relative to other dimensions?

**Extraction heuristic thresholds.** What signal phrase patterns most reliably identify the five extraction heuristic categories in real session logs? What is the false positive rate for each heuristic in practice?

**Similarity threshold for deduplication.** What embedding similarity threshold correctly distinguishes near-duplicate beliefs from genuinely distinct beliefs about similar topics? How does this vary by belief type?

**Staleness threshold.** At what level of graph divergence should offline session merge be suspended in favor of human review? How should the staleness measure weight foundational belief changes versus provisional belief changes?

**Consistency model.** Should the active belief graph use strong consistency (all reads see all writes immediately) or eventual consistency (writes propagate with bounded delay)? Strong consistency is more trustworthy but limits write throughput and geographic distribution. The right answer depends on session volume and team distribution.

**Federation protocol.** How should two organizations share beliefs across organizational boundaries? What permission model governs cross-organizational belief promotion? What does portable belief graph export look like in practice?

**Regulatory compliance.** What data protection regulations apply to belief graphs that contain reasoning about systems handling personal data? How does the EU AI Act's documentation and human oversight requirements map onto the belief graph's provenance layer?

**Belief graph capacity limits.** At what point does a belief graph become too large for a single project scope? How should beliefs be partitioned across sub-scopes when the graph exceeds manageable size?

**The agent-as-primary-consumer transition.** As agent-to-human ratios increase in development teams, how should the promotion interface adapt? At what ratio does the policy authoring model become insufficient and what replaces it?

**Embedding model selection.** The deduplication process and conflict detection both depend on belief content embeddings. Which embedding model produces the best semantic similarity results for technical belief content? How is the embedding model updated when a better model becomes available — does re-embedding the full belief graph invalidate existing similarity thresholds? What is the migration path when the embedding model changes?

**Chain integrity hop limit calibration.** What hop limit correctly captures the meaningful propagation distance of belief integrity events without generating excessive cascade updates in mature graphs with rich dependency relationships? Does the right hop limit vary by belief type or retirement reason?

**Retirement cascade dampening calibration.** What flag intensity reduction per hop correctly balances the need to surface meaningful downstream impacts against the risk of overwhelming reviewers with distantly connected flags? How should the dampening model adapt as the belief graph matures and dependency relationships become denser?

**Throttling calibration in agent-heavy deployments.** What per-session, per-contributor, and scope circuit breaker thresholds correctly distinguish legitimate high-volume agent workloads from attack traffic? How should thresholds be calibrated as the agent-to-human ratio in a deployment increases? What contributor trust history window produces the most accurate adaptive limit adjustments? Note that throttling thresholds in agent-favoring regimes are simultaneously security calibrations and cost calibrations — both functions must be considered when tuning.

**Recovery drain rate calibration.** What throttled drain rate correctly balances recovery speed against pipeline stability after an extended outage or DDoS? How should the drain rate adapt based on the size of the backlog and the pipeline's current load? At what backlog depth should partial recovery mode automatically engage?

**Regime crossover ratio.** At what agent-to-human ratio does the cost calibration regime distinction become operationally significant? Is the 1:3 boundary suggested in Section 8 borne out in practice, or does the crossover happen at a different ratio depending on workload characteristics and team-specific extraction selectivity?

**Normalization model cost-quality tradeoff.** What is the empirical relationship between normalization model size and normalized_confidence calibration accuracy? At what point does a lighter-weight normalization model degrade belief graph trustworthiness enough to break the cost displacement case described in Section 8.4? How is this tradeoff affected by belief type — do constraint beliefs require higher normalization quality than observation beliefs?

**Read displacement ratio in practice.** What read displacement ratio is achievable for typical project scopes? How does the ratio vary with belief graph maturity, scope size, and the quality of extraction heuristic calibration? What instrumentation produces the most reliable measurement of session tokens saved relative to a counterfactual baseline?

**Normalization quality signal calibration.** What is the right embedding similarity floor for raw-to-normalized hallucination detection (Section 3.5.3)? Does the floor vary by belief type — do constraint beliefs require a stricter floor than observation beliefs? At what candidate volume does multi-pass agreement become economically defensible for high-stakes types? How is the floor recalibrated when the underlying embedding model changes?

**Cross-version embedding stability.** Section 5.5.4 requires that embedding proximity matching work across normalization model versions. What embedding model properties ensure that cosine similarity between normalized outputs of the same underlying belief produced under different normalization models remains above the dedup threshold? Is this property guaranteed by any current embedding model, or does it require deliberate embedding model selection?

**Confirming corroboration discount factor.** What is the right value for `confirming_discount` (Section 5.6.1)? The default of 0.3 reflects the intuition that confirming corroborations are weak evidence but not zero evidence — a contributor who saw a belief and did not contradict it does provide some signal, especially when that contributor has high human_reviewed trust. Should the discount factor vary by the contributing session's trust level, with confirmations from higher-trust sessions discounted less? At what discount value does the distinction between independent and confirming become operationally meaningful for trust elevation timing?

**Sync-cursor classification edge cases.** Section 5.5 specifies classification based on sync cursor position relative to belief promotion timestamp. Are there session patterns that produce systematic misclassification — for example, sessions that pre-load context from out-of-band sources (project documentation, prior conversations) without that pre-loading appearing in the sync cursor? How is misclassification detected and corrected when it occurs? Is the classification stable enough to use as a trust signal, or does it require periodic audit?

---

## Appendix A: Relationship to Five-Layer Framework

The five-layer framework introduced in "The Case for Purpose-Built Context Infrastructure in AI-Assisted Development" maps onto this HLD as follows:

| Framework Layer | HLD Section | Key Artifact |
|---|---|---|
| Context Snapshot | Section 3 | Belief candidate object |
| Belief Graph | Section 4 | Belief node with dimension vector |
| Merge Problem | Section 5 | Merge decision taxonomy and scoring function |
| Provenance Layer | Section 6 | Provenance record chain |
| Promotion Interface | Section 7 | Policy schema and review queue |

---

## Appendix B: Glossary

**Belief.** A discrete piece of project knowledge stored in the belief graph — a constraint, decision, pattern, rejection, or observation.

**Belief candidate.** A belief extracted from a session log that has not yet been evaluated for promotion into the belief graph.

**Belief staleness.** The gradual degradation of a belief's accuracy as the system it describes evolves without corresponding updates to the stored belief. Distinct from context window degradation (within-session performance loss) which is a model architecture property.

**Chain integrity.** A dimension of the belief graph's dimension vector reflecting the health of a belief's upstream reasoning chain. Stored and updated on events. Initialized at promotion time based on the minimum chain_integrity of upstream dependencies weighted by their trust levels. Decremented by retirement cascades from upstream beliefs and restored by reinstatement cascades.

**Context tax.** The recurring token overhead cost paid by agent sessions that must re-establish context that already existed in previous sessions. Reducible through context infrastructure investment.

**Context rot.** The degradation of a belief graph's overall quality through accumulation of stale, conflicting, or unvalidated beliefs. Addressed through belief retirement, confidence decay, and provenance-linked invalidation.

**Distillation.** The compression of session history into a summary, distinct from squashing in that distillation explicitly records what was discarded and preserves a pointer to the full history in cold storage.

**Foundational belief.** A belief authored directly by a human, not derived from an agent session. Treated as ground truth for conflict resolution.

**Hop limit.** The maximum dependency graph distance over which chain_integrity updates and retirement cascade flags propagate. A configurable integer (default: 3). Beyond the hop limit no automatic cascade occurs — the upstream event is too distant to reliably affect downstream belief quality.

**Promotion.** The process by which a belief candidate moves from the extraction layer into the active belief graph, subject to dimension scoring, deduplication, merge evaluation, and (for policy-exceeding candidates) human review.

**Project scope.** The repository or project context within which beliefs are valid. The unit of partitioning for the belief graph.

**Provenance.** The chain of custody from raw session exchange to belief graph entry. The answer to "why does the system know that?"

**Reinstatement.** The operation that restores a retired belief to active status when its retirement is found to have been incorrect. Generates its own provenance record and a cascade that reverses the flags and chain_integrity decrements produced by the original retirement. Also generates a secondary review cascade for downstream beliefs modified during the retirement window.

**Retirement reason.** A required field on retirement provenance records. Values: `incorrect` (belief was wrong from the start — highest cascade urgency), `superseded` (belief was correct but the system evolved — standard cascade urgency), `redundant` (belief was refined by a more precise replacement — informational annotations only).

**Sync cursor.** An opaque pointer to the last known belief graph state for a session. Used in delta pulls to retrieve only beliefs that changed since the last sync. Updated at each compaction sync cycle. The divergence between a session's sync cursor and the current graph state is the staleness measure used in the scoring function.

**Trust level.** A derived property of a belief reflecting the integrity of its provenance chain. Not asserted by the contributor; computed by the system from verifiable attributes.

---

## Appendix C: References

**MEMO** (MIT/NUS/Singapore-MIT Alliance, 2026).
The closest existing research system to this design. Validates the architectural direction of separating memory from reasoning and using a dedicated memory model as a queryable knowledge store. Cited in Section 1.4 as the validating prior work this design extends from static document corpora to dynamic session accumulation, team sharing, human-in-the-loop promotion, and interpretable provenance. The MEMO authors explicitly identify attribution mechanisms and access controls as future work.
https://arxiv.org/html/2605.15156v2

**Understand Anything** (Lum1104, 2026).
Open-source Claude Code plugin using a multi-agent pipeline to synthesize a knowledge graph from static codebase analysis. Cited in Section 1.4 as validating the knowledge graph as the right representational primitive for codebase understanding. The `knowledge-graph.json` schema serves as a reference for this design's belief graph node and edge structure, extended here with dimension scores, provenance pointers, and belief lifecycle properties.
https://github.com/Lum1104/Understand-Anything

**LLM Wiki** (Andrej Karpathy, 2026).
Widely-discussed personal knowledge base pattern in which an LLM maintains a synthesized markdown wiki over raw source files, governed by a natural-language `agents.md` instruction file. Cited in Section 1.4 as the personal-scale predecessor to this design's team-scale architecture. This design generalizes the LLM Wiki's core premise — pre-synthesized structured context as the working substrate — to multi-contributor team contexts where conflict resolution, trust derivation, provenance integrity, and human-in-the-loop promotion become load-bearing requirements that the personal-scale design does not need to address.
https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

**TeamPCP GitHub breach** (The Hacker News, May 2026).
Documented attack on the distributed local model for AI session context. A single poisoned developer tool, targeting AI coding assistant configuration files (CLAUDE.md and .cursorrules) with zero-width Unicode injection, gave attackers access to 3,800 internal repositories. Cited in the Executive Summary, Section 9.2.2 (attacker profiles), Section 9.8 (distributed model security analysis), and Sections 10.3.1 and 10.3.3 as the documented attack motivating the migration review gate.
https://thehackernews.com/2026/05/github-investigating-teampcp-claimed.html

**Semantica** (Hawksight-AI, 2026).
Open-source framework for context graphs, decision intelligence, and provenance in AI agent systems. Production v0.4.0 released April 2026. Cited in Section 1.3 (guiding principles) and Section 1.4 (related work) as the strongest contemporary work using similar architectural primitives applied to a different architectural commitment about where agents belong in the infrastructure stack.
https://github.com/Hawksight-AI/semantica
