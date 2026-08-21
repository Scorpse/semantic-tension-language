# Schema Feedback from Real Agent Usage

**Source:** Kumidai — an agent-native project-management platform where autonomous
coding/review agents (Claude, Codex, Kimi, Zed, Cursor) communicate **only in STL**
inside work-item comments.
**Sample:** 543 STL statements from live multi-agent traffic.
**Date:** 2026-08-21.
**Status:** field report for the schema maintainers. The schemas are early and expected
to evolve with agent usage — this documents where real usage already diverges from
`docs/schemas/software-*`, so the divergence can inform the next revision.

> This is **input to the STL project**, not a change request against any one schema
> file. Decide disposition upstream.

---

## TL;DR

Autonomous software agents, left to write STL freely, converged on a **CI-gate /
review / orchestration dialect** that the current `software-*` schemas do not model:

1. They tag every statement with **`kind=`** (an event/outcome verb) — a field that
   **does not exist** in any `software-*` schema. The schema's required semantic field
   is `relation` (a structural-edge enum). Agents almost never emit `relation`.
2. They express **point-in-time events/outcomes** (a gate passed, a validation failed,
   a merge was skipped), not just **structural edges** between entities. STL relations
   model edges well; there is no first-class construct for an *outcome*.
3. **`status` is unenumerated but highly patterned**, and some success values embed
   alarming substrings (`exit_0_no_errors`, `5_pass_3_warn_0_fail`) that naive consumers
   misread as failures. An enumerated `outcome` axis would remove the ambiguity.
4. Two urgency axes coexist: schema **`severity`** (critical/high/medium/low/info) vs
   agent **`priority`** (0–3). Agents use `priority`; schema governs `severity`.
5. A whole **multi-agent orchestration layer** (`claim` / `release` / `handoff` /
   `owner` / `notify` / `obligation`) has no schema home.

---

## 1. `kind` vs `relation`

- `relation` is `required` in every `software-*` schema and is an **enum** of structural
  verbs: `owns, requires, implements, contains, depends_on, calls, exposes, reads,
  writes, tests, builds, deploys, releases, verifies, rolls_back, threatens, mitigates,
  detects, violates, …`.
- **`kind` appears in 543/543 statements. `relation` appears in 0.** Agents adopted
  `kind=` as their primary semantic tag (seeded by the host's agent guide, then extended
  by analogy).
- Critically, the `kind` values are **not** structural relations — they are
  **events/outcomes**: `gate_pass`, `gate_fail`, `merge_skip`, `qa_fail`, `validation`,
  `verification`, `release`, `cleanup`, `handoff`. A relation says *"Build **deploys_to**
  Environment"*; agents needed to say *"the preflight **gate failed**"* — a different
  shape.

**Observed `kind` vocabulary (45 distinct):**

| count | kind | count | kind | count | kind |
|--:|---|--:|---|--:|---|
| 95 | gate_pass | 45 | validation | 37 | no_change |
| 36 | finding | 33 | cleanup | 29 | release |
| 28 | verification | 28 | merge_skip | 25 | required_action |
| 18 | resolves | 16 | claim | 13 | evidence |
| 11 | independence | 10 | routing | 9 | selection |
| 9 | promote | 9 | merge_pass | 9 | gate_fail |
| 8 | lane_summary | 8 | handoff | 6 | invariant |
| 5 | pending_review | 5 | merge_fail | 5 | limitation |
| 4 | worktree_cleanup | 4 | reviewer_patch | 4 | qa_fail |
| 4 | acceptance | 3 | run_outcome | 3 | outcome |
| 3 | no_patch | 3 | correction | 2 | run_summary |
| 2 | risk | 2 | proves | 2 | contradiction |
| 2 | bounce | 2 | blocks | 1 | patch |
| 1 | non_independent_check | 1 | lane_report | 1 | known_failure |
| 1 | decision | 1 | constraint | | |

Note the naming drift for one concept: `qa_pass` (the schema-guide example) became
`gate_pass` / `merge_pass` / `verification` / `acceptance` in the wild.

**Possible dispositions (upstream call):**
- Add a first-class **event/outcome** construct distinct from `relation` — e.g.
  `[Gate:Preflight] -> [WorkItem:X] ::mod(relation="verifies", outcome="fail", …)`, or a
  parallel `event=` axis.
- Or bless `kind` as an official optional field with a documented, open taxonomy.
- Either way, the "relation is the only semantic field" assumption doesn't survive
  contact with autonomous agents.

---

## 2. Node types: agents coined a CI/review lifecycle

Left tokens (`[Type:id]`) actually used, with counts:

`WorkItem`(208), `Gate`(121), `State`(92), `Verifier`(69), `Artifact`(68),
`Finding`(62), `Run`(56), `Agent`(44), `Claim`(42), `Cleanup`(38), `Validation`(33),
`Payload`(33), `Lane`(30), `Required_Action`(22), `Evidence`(21), `Revision`(17),
`Requirement`(14), `Guard`(11), `Queue`(10), `Patch`(10), `Branch`(10), `Walk`(8),
`Question`(8), `Dependency`(6), `Function`(5).

Only **`Artifact`, `Finding`, `Evidence`, `Requirement`, `Dependency`** map to existing
schema node types. The high-frequency ones — **`Gate`, `Verifier`, `Run`, `Lane`,
`Guard`, `Queue`, `Claim`, `Cleanup`, `Validation`, `State`, `WorkItem`** — are the
**review-gate + multi-agent-run** vocabulary, absent from `software-delivery` (which
models `Test/Build/Pipeline/Release/Deployment`) and `software-assurance`.

Suggests a **`software-review`** and/or **`software-agentcoord`** profile, or an
extension of `software-delivery` with gate/verifier/run entities.

---

## 3. `status` is unenumerated but patterned — and ambiguous

`status` is a free string in all schemas. In practice it carries the **outcome**, with
recurring tokens:

`passed`(57), `completed`(42), `exit_0_no_errors`(33), `testing`(28), `not_created`(22),
`closed`(21), `independent`(11), `pending_independent_review`(10), `blocked`(8),
`failed`(5), `fix_before_merge`(7), `bounced_to_coding`(3), `5_pass_3_warn_0_fail`(6),
`exit_zero`(8), `all_passed`(3), `refuted`(2), `qa_failed`(2) …

**Concrete hazard:** these are *successes* whose strings contain failure substrings:
- `exit_0_no_errors` → contains `error`
- `5_pass_3_warn_0_fail` → contains `fail`
- `merges_without_conflict` → contains `conflict`

Any consumer that infers pass/fail by substring (we had to, downstream) misclassifies
them. This is a strong argument for a **governed `outcome`/`result` enum** —
`pass | fail | warn | skip | blocked | pending | no_change` — kept **separate** from the
descriptive `status` string.

---

## 4. `severity` vs `priority`

- Schema governs **`severity`** = `critical | high | medium | low | info`
  (software-assurance, software-operations).
- Agents emit **`priority`** = `0 | 1 | 2 | 3` (543/543 statements; `0` = urgent, 74×,
  used on `gate_fail` and human-blocking questions). **`severity` never appears.**

Two axes for "how much should a human care." Recommend the schema either (a) map
`priority`↔`severity`, or (b) pick one governed urgency axis and document the other's
role (scheduling vs risk).

---

## 5. Multi-agent orchestration fields have no schema home

Fields seen that encode **agent coordination**, not software structure:

| field | count | meaning |
|---|--:|---|
| `owner` | 69 | who must act next |
| `notify` | 63 | who to ping (`@Adi`, `@claude-code`, …) |
| `obligation` | 58 | `must` / `must_review` / `must_consume` / `should` |
| `claim` / `release` (as kinds) | 45 | work-ownership handshake |
| `run_id` | 80 | groups statements from one agent run/lane |
| `confidence` | 543 | agent's self-rated confidence (0.0–1.0) |
| `rule` | 20 | `empirical` / `logical` (evidence basis) |

This is the "STL as the coordination substrate for autonomous agents" use-case
(cf. `STL-for-AI.md`). It's arguably the fastest-growing usage and has **no** profile.
A **`software-agentcoord`** (or generic `orchestration`) profile covering
claim/release/handoff/owner/notify/obligation/run grouping would capture it.

---

## Recommendations (for the STL project to weigh)

1. **Introduce an outcome/event axis** distinct from `relation`. Relations are edges;
   agents also assert point-in-time outcomes. This is the single biggest gap.
2. **Enumerate `outcome`** (`pass|fail|warn|skip|blocked|pending|no_change`) so consumers
   stop guessing from substrings; keep `status` as free-text detail.
3. **Add review-gate + run entities** to a delivery/review profile: `Gate`, `Verifier`,
   `Check`, `Run`, `Lane`, `Guard`, `Queue`.
4. **Add an agent-coordination profile** for claim/release/handoff/owner/notify/
   obligation/run_id/confidence.
5. **Reconcile `severity` vs `priority`** into one documented urgency model.
6. Either **bless `kind`** as an official optional taxonomy field or provide a canonical
   **`kind → relation+outcome`** mapping so host guides stop reinventing it.

---

### Appendix — how this was consumed downstream

Kumidai renders these comments for humans. Because `kind` is open-ended and `status`
ambiguous, the UI can't use a fixed lookup; it classifies each statement into semantic
tiers (fail / needs-action / needs-review / pass / info / routine) by pattern-matching
`kind` + `status`, with `severity`/`priority` as overrides. That works, but a governed
`outcome` + `severity` axis upstream would let any consumer color/route by field lookup
instead of heuristics — the whole point of a schema.
