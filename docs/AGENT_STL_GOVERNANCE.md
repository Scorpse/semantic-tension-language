# Agent-Generated STL Governance

**Status:** Proposed v0.1  
**Evidence:** [`SCHEMA_FEEDBACK_FROM_AGENT_USAGE.md`](SCHEMA_FEEDBACK_FROM_AGENT_USAGE.md)

This policy governs STL produced or consumed by autonomous software agents. It prevents local prompt conventions from silently becoming incompatible dialects.

## Producer rules

1. Declare the schema/profile and version used for every durable STL stream.
2. Validate statements before publishing them.
3. Use `relation` for graph semantics. `kind` may describe an event but does not replace the edge relation.
4. Use structured fields for machine decisions. Never encode pass/fail only inside `status`.
5. Keep urgency dimensions separate: `priority` is scheduling order; `severity` is impact/risk.
6. Unknown experimental fields require a documented owner, meaning, and promotion plan.

## Consumer rules

1. Route and color by governed fields, never by substring searches in `status` or descriptions.
2. Preserve unknown fields during round trips.
3. Reject or quarantine statements that fail the declared profile; do not silently repair durable records.
4. Record the parser version and profile version used for validation.

## Vocabulary lifecycle

New agent vocabulary moves through four stages:

| Stage | Requirement |
|---|---|
| Observed | Real examples and frequency counts |
| Candidate | Defined semantics, owner, collision analysis, proposed profile |
| Stable | Schema constraint, valid/invalid examples, parser tests, changelog entry |
| Deprecated | Replacement and migration window |

A field cannot become stable from prompt usage alone. Promotion requires evidence from at least one durable workload and a review of producer and consumer behavior.

## Current dispositions

| Observed construct | Governance disposition |
|---|---|
| `kind` | Candidate descriptive event taxonomy; not a substitute for `relation` |
| `outcome` | First functional candidate: closed enum for machine decisions |
| `status` | Free-text operational detail; never authoritative for outcome |
| `priority` | Candidate scheduling axis `0..3`; distinct from severity |
| `severity` | Existing risk/impact axis |
| `owner`, `notify`, `obligation`, `run_id` | Candidate agent-coordination profile |
| `Gate`, `Verifier`, `Run`, `Lane`, `Guard`, `Queue` | Candidate review/orchestration profile |

## Change discipline

- One functional semantic change per branch.
- Documentation-only governance changes use a separate branch.
- Parser changes belong in `STL-TOOLS`; schemas and protocol definitions belong here.
- Every semantic change includes migration guidance and representative valid/invalid examples.
- Experimental taxonomies remain open; stable decision fields use closed enums.
