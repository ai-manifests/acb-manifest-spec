# Agent Cognitive Budget Protocol (ACB) — v0

**Status:** Draft (pre-implementation)
**Date:** 2026-04-13
**Schema namespace:** `https://acb-manifest.dev/schemas/`

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

---

## 1. Purpose and Non-Goals

### 1.1 Purpose

ACB defines how a deliberation is *priced* — how the cost of cognition is
posted, how that cost escalates when easy decisions become hard, how it is
distributed across the agents that did the work, and how previously-decided
questions are billed at a discount. It specifies the budget object a requester
posts to a deliberation, the pricing model that converts deliberation events
into draws against that budget, the unlock rule that distinguishes routine
deliberation from contested deliberation, and the settlement contract that
distributes the drawn amount across participants by their journal-evidenced
contribution.

ACB is designed for federated deployment without central coordination; see
Section 12.

ACB is a peer of [ADP](https://adp-manifest.dev) and [ADJ](https://adj-manifest.dev),
not a replacement. ADP defines how agents decide together. ADJ defines how
those decisions are recorded, queried, and scored over time. ACB defines how
the cognitive work of deciding is paid for. The three specs compose: ACB reads
disagreement-magnitude signals from ADP, settles against contribution evidence
in ADJ, and adds one new entry type (`settlement_recorded`) that ADJ adopts in
its v0.1 hook list. Each spec remains independently implementable. ADP and ADJ
without ACB function unchanged. ACB without ADJ degrades to flat-rate pricing
because there is no contribution evidence to settle against.

### 1.2 Design Principle

Cognition is metabolically expensive, and the protocol must price it that way.

The brain is an energy-budgeting organ first and a thinking organ second. It
runs on roughly twenty watts and refuses to spend that budget on deliberate
reasoning unless a prediction-error signal forces the unlock. The default
state is the cheapest available routine that historically produced non-fatal
outcomes. Critical thinking is gated behind evidence that the cheap routine
will not work here. ACB applies that architecture to agent federations.
Agreement is cheap because agreement is evidence the cheap routine sufficed.
Disagreement is expensive because disagreement is evidence more cognition was
actually required. The unlock from cheap to expensive is not controlled by the
agents (who would manufacture disagreement to drive up their own pay) — it is
controlled by the prediction-error signal computed mechanically from
ADP's tally.

Three corollaries follow from this principle and shape every section below:

1. **Payment is calibration, not compensation.** A bad actor who shows up,
   casts a garbage vote, gets no contribution credit, and walks away with
   nothing is self-correcting in a way that flat participation fees are not.
   ACB is a second accountability rail running in parallel with ADJ's
   calibration loop, not a monetization layer bolted onto the deliberation
   stack.
2. **Disagreement must resolve to better calibration to be billable.** The
   brain rewards prediction-error signals only when they *update* the model.
   ACB rewards disagreement only when ADJ's outcome-observed loop later shows
   that the disagreement was load-bearing for a correct decision. Manufactured
   disagreement that fails to improve future predictions degrades the agent's
   draw rate the same way it degrades calibration weight under ADP.
3. **The substrate provider and the agent identity are paid separately
   from the same draw.** The brain does not maintain two energy budgets
   for cycles versus cognition — it draws from one pool and that pool is
   consumed by both. ACB inherits the same property: the requester posts one
   budget, and the protocol distributes draws across whatever substrates the
   deliberation actually touched (compute providers) and whatever agent
   identities contributed to the outcome (epistemic work). One budget, two
   recipient classes, one journal record per settlement.

### 1.3 Non-Goals

ACB does **not** specify:

- **Currency.** ACB budgets are denominated in an abstract scalar called
  *energy units* (EU). Mapping EU to any real-world currency, token, or
  off-protocol settlement medium is out of scope. ACB-conformant implementations
  MAY adopt fiat, stablecoins, ledger credits, or any other unit of account,
  provided the conversion is fixed at budget posting time and recorded in the
  budget object.
- **Custody.** Where the budget is held between posting and settlement, who
  holds the keys, how it is escrowed, how disputes against settlement are
  resolved off-protocol. ACB defines the records that make settlement
  *auditable*; off-protocol custody arrangements determine what those records
  *enforce*.
- **Pricing of substrate cycles.** ACB defines that compute cost is a
  recipient class in settlement. It does not specify how a substrate provider
  reports its cycle cost. That is left to substrate operator agreements,
  which MAY use existing pricing primitives (cloud provider rates, on-chain
  gas markets, fixed per-call fees).
- **Contribution scoring algorithm.** ACB specifies that settlement reads
  contribution signals from ADJ. The exact function that converts ADJ entries
  into per-agent contribution shares is profileable (Section 6.4) — ACB v0
  defines a default profile and lets implementations declare alternatives.
- **Outcome timing.** ACB does not require deliberation outcomes to be
  observed within any specific window. Settlement may be immediate (no outcome
  awaited), deferred (waiting for `outcome_observed`), or two-phase
  (preliminary settlement at deliberation close, final adjustment after
  outcome). The choice is per-budget.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Budget** | A pre-committed pool of *energy units* a requester posts to a deliberation. Defined in Section 3. |
| **Energy unit (EU)** | The abstract scalar in which ACB budgets are denominated. Mapping to real-world currency is out of scope (Section 1.3). |
| **Requester** | The human, system, or agent on whose behalf the deliberation is happening. Posts the budget. Receives any unspent remainder at settlement. |
| **Budget authority** | The principal — typically the requester or its proxy — that signs the budget commitment and the settlement record. The only entity authorized to write `settlement_recorded` for the deliberation. |
| **Cheap routine** | The default pricing track. Applies when agents converge without belief-update rounds and disagreement magnitude is below the unlock threshold (Section 5). Per-participant draw at the cheap-routine rate. |
| **Expensive routine** | The escalated pricing track. Applies when the unlock rule fires. Per-participant draw at the expensive-routine rate, plus per-round multipliers. |
| **Disagreement magnitude** | A scalar in [0, 1] computed from ADP's initial tally that quantifies how much the agents disagree on the proposed action. The unlock signal. Defined in Section 5.1. |
| **Unlock threshold** | The disagreement-magnitude value above which the expensive routine is engaged. Per-budget; default 0.30. |
| **Draw** | A debit against the budget for a specific deliberation event. Tracked by ACB as the deliberation runs. |
| **Settlement** | The terminal write that distributes the total draw across recipients (substrate providers and agent identities) and returns any unspent remainder to the requester. Recorded as a `settlement_recorded` entry in the budget authority's journal. |
| **Contribution signal** | A per-agent score derived from ADJ entries that quantifies how load-bearing each agent was for the deliberation outcome. The basis for distributing the epistemic share of settlement. Defined in Section 6.2. |
| **Habit memory discount** | A per-deliberation discount applied when the question being decided closely resembles one already decided in the federation's journal history. Defined in Section 7. |
| **Substrate provider** | The entity that contributed compute cycles to an agent's participation — a cloud provider, an on-prem cluster, a third-party inference service. May be the same entity as the agent operator, or different. |
| **Settlement profile** | A named contribution-scoring function declared by the budget authority. ACB v0 defines `default-v0`. |

---

## 3. The Budget Object

A budget is the atomic unit of pre-commitment in ACB. It is posted by the
requester at or before deliberation open, immutable once posted, and consumed
by draws as the deliberation runs. The budget object MUST be written as a
journal entry of type `budget_committed` in the budget authority's journal
before any agent draws against it. The corresponding ADP `deliberation_id`
MUST already exist in the deliberation context when the budget is posted.

### 3.1 Schema

```json
{
  "$schema": "https://acb-manifest.dev/schemas/budget/v0.json",
  "budget_id": "bgt_01HMXJ3E9R",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "budget_authority": "did:requester:acme-platform",
  "posted_at": "2026-04-11T14:30:00.000Z",

  "denomination": {
    "unit": "EU",
    "external_unit": "USD",
    "external_rate": 0.0001,
    "rate_source": "fixed-at-posting"
  },

  "amount_total": 12000,

  "pricing": {
    "profile": "default-v0",
    "cheap_routine_rate": 50,
    "expensive_routine_rate": 200,
    "round_multiplier": 1.5,
    "unlock_threshold": 0.30,
    "habit_memory_discount": "default-v0"
  },

  "settlement": {
    "profile": "default-v0",
    "mode": "deferred",
    "outcome_window_seconds": 604800,
    "substrate_share": 0.20,
    "epistemic_share": 0.80,
    "unspent_returns_to": "did:requester:acme-platform"
  },

  "constraints": {
    "max_participants": 8,
    "max_rounds": 4,
    "irrevocable": false
  },

  "signature": "ed25519:6f3a..."
}
```

| Field | Description |
|---|---|
| `budget_id` | Globally unique identifier for this budget. |
| `deliberation_id` | The ADP deliberation this budget funds. MUST match an existing or imminent deliberation. |
| `budget_authority` | The DID of the principal authorizing the budget. The signer of this object and of the eventual `settlement_recorded` entry. |
| `posted_at` | Timestamp of budget commitment. MUST be ≤ `deliberation_opened.timestamp` of the referenced deliberation. |
| `denomination.unit` | MUST be `"EU"`. |
| `denomination.external_unit` | OPTIONAL. The off-protocol unit the budget authority maps EU to (e.g., `"USD"`, `"USDC"`, `"ledger:acme-credits"`). |
| `denomination.external_rate` | OPTIONAL. The fixed conversion rate from EU to `external_unit` at posting time. Recorded once and never mutated. |
| `denomination.rate_source` | OPTIONAL. How the rate was set. `"fixed-at-posting"` is the default; off-protocol oracles MAY define their own values. |
| `amount_total` | The total budget pool in EU. Must be > 0. |
| `pricing.profile` | Named pricing function. `"default-v0"` is defined in Section 4. |
| `pricing.cheap_routine_rate` | Per-participant draw, in EU, when the cheap routine applies. |
| `pricing.expensive_routine_rate` | Per-participant draw, in EU, when the expensive routine applies. |
| `pricing.round_multiplier` | Multiplier applied per belief-update round under the expensive routine. |
| `pricing.unlock_threshold` | Disagreement magnitude above which the expensive routine engages. In [0, 1]. Default 0.30. |
| `pricing.habit_memory_discount` | Named discount function. `"default-v0"` is defined in Section 7. |
| `settlement.profile` | Named settlement function. `"default-v0"` is defined in Section 6. |
| `settlement.mode` | One of: `"immediate"` (settle at `deliberation_closed`), `"deferred"` (await `outcome_observed`), `"two_phase"` (preliminary at close, adjustment after outcome). |
| `settlement.outcome_window_seconds` | OPTIONAL. For `deferred` and `two_phase`, the maximum wait for an outcome before settling without one. After this window, settlement MUST proceed. |
| `settlement.substrate_share` | Fraction of the drawn total allocated to substrate providers. In [0, 1]. |
| `settlement.epistemic_share` | Fraction allocated to agent identities. MUST satisfy `substrate_share + epistemic_share = 1.0`. |
| `settlement.unspent_returns_to` | DID receiving any unspent remainder. Typically the budget authority. |
| `constraints.max_participants` | OPTIONAL hard cap on how many agents may draw against this budget. Enforced by ACB-aware deliberation runners. |
| `constraints.max_rounds` | OPTIONAL hard cap on belief-update rounds. ADP convergence logic SHOULD respect this. |
| `constraints.irrevocable` | If `true`, the budget cannot be cancelled after `deliberation_opened`. Default `false`. |
| `signature` | The budget authority's signature over the canonical serialization of the rest of the object. |

### 3.2 Lifecycle

A budget is in exactly one state at any time:

| State | Definition |
|---|---|
| `posted` | `budget_committed` entry written; no `deliberation_opened` yet. |
| `active` | Deliberation is running. Draws are accruing. |
| `awaiting_outcome` | `deliberation_closed` written; settlement is `deferred` or `two_phase`; outcome window has not closed. |
| `settled` | `settlement_recorded` entry written. Terminal. |
| `cancelled` | Budget was cancelled before deliberation open. Permitted only if `constraints.irrevocable` is `false`. Terminal. |
| `expired` | Outcome window closed without an `outcome_observed` entry. Settlement proceeds without outcome adjustment. |

Transitions:

```
posted ──► active ──► awaiting_outcome ──► settled
   │           │              │
   │           │              └──► expired ──► settled
   │           │
   │           └──► settled            (mode = immediate)
   │
   └──► cancelled                       (only if !irrevocable)
```

### 3.3 Budget Cancellation

A budget MAY be cancelled by writing a `budget_cancelled` entry before
`deliberation_opened`, signed by the budget authority. Cancellation after
`deliberation_opened` is permitted only if the deliberation has not yet
collected any proposals AND `constraints.irrevocable` is `false`. Otherwise
the budget MUST proceed to settlement.

ACB v0 does not specify dispute mechanics for cancelled-but-unsettled budgets.
Off-protocol custody arrangements handle that case.

---

## 4. Pricing Model

### 4.1 The Cheap Routine

When the cheap routine applies, the total draw for a deliberation is:

```
draw = cheap_routine_rate × participant_count × (1 − habit_discount)
```

Where:
- `participant_count` is the number of agents who emitted a `proposal_emitted`
  entry for this `deliberation_id`. Abstainers count.
- `habit_discount` is the discount factor in [0, 1] computed by Section 7.
  Zero means no discount.

The cheap routine MUST apply when ALL of the following hold at
`deliberation_closed`:

1. `round_count = 0` (the deliberation converged on the initial tally).
2. The disagreement magnitude (Section 5.1) computed from the initial tally
   was strictly less than `pricing.unlock_threshold`.
3. The deliberation terminated as `converged` (not `partial_commit` or
   `deadlocked`).

### 4.2 The Expensive Routine

When the expensive routine applies, the total draw is:

```
draw = expensive_routine_rate × participant_count
       × round_multiplier^round_count
       × (1 − habit_discount)
```

The expensive routine MUST apply when ANY of the following hold:

1. Disagreement magnitude at the initial tally was ≥ `pricing.unlock_threshold`.
2. `round_count > 0`.
3. The deliberation terminated as `partial_commit` or `deadlocked`.

The exponential round multiplier reflects the brain analogy: each additional
belief-update round is metabolically more expensive than the last because the
remaining disagreement is, by selection, the disagreement the previous round
failed to resolve.

### 4.3 Worked Pricing

Given default profile values from Section 3.1 (`cheap = 50`, `expensive = 200`,
`multiplier = 1.5`):

| Scenario | Routine | Participants | Rounds | Habit | Draw |
|---|---|---|---|---|---|
| Trivial PR merge, all approve, no discount | cheap | 3 | 0 | 0.00 | 150 EU |
| Same merge, identical to 200 prior decisions | cheap | 3 | 0 | 0.80 | 30 EU |
| Contested PR, 1 round to converge | expensive | 3 | 1 | 0.00 | 900 EU |
| Deeply contested, 3 rounds | expensive | 4 | 3 | 0.00 | 2700 EU |
| Same, but 50% habit discount | expensive | 4 | 3 | 0.50 | 1350 EU |

The cheap-vs-expensive gap is intentional. A deliberation that escalates is
roughly an order of magnitude more expensive than one that does not, because
the cognitive work it represents is roughly an order of magnitude greater.

---

## 5. The Unlock Rule

The unlock rule is the load-bearing piece of ACB and the place where
agreement with the brain analogy is enforced mechanically rather than by
agent honesty.

### 5.1 Disagreement Magnitude

Disagreement magnitude is a scalar in [0, 1] computed from the initial tally
of an ADP deliberation. ACB v0 defines:

```
disagreement_magnitude(tally) =
    1 − | approve_weight − reject_weight | / non_abstaining_weight
```

Where `non_abstaining_weight = approve_weight + reject_weight`.

If `non_abstaining_weight = 0` (everyone abstained), disagreement magnitude is
defined as 1.0 (maximum disagreement — total abstention indicates the cheap
routine has failed to find anyone willing to commit).

This formula has the right shape for the brain analogy:

| Initial tally | Magnitude |
|---|---|
| All agents approve, high confidence | ~0.00 |
| 90% approve, 10% reject | ~0.20 |
| 70% approve, 30% reject | ~0.60 |
| 50/50 split | 1.00 |
| All agents abstain | 1.00 |

A 90/10 split is a low-signal outlier: the cheap routine still works, the
deliberation converges, no escalation. A 70/30 split is genuine epistemic
conflict that warrants the expensive routine.

### 5.2 Unlock Mechanics

When an ADP-aware deliberation runner has collected the initial tally and is
about to evaluate convergence, it MUST compute `disagreement_magnitude` and
emit it as part of the deliberation context. ACB-aware runners read this
scalar and apply the unlock rule:

- If `disagreement_magnitude < pricing.unlock_threshold` AND the convergence
  threshold is met by the initial tally: cheap routine applies. No
  belief-update round is initiated.
- If `disagreement_magnitude ≥ pricing.unlock_threshold` OR convergence fails
  on the initial tally: expensive routine engages. A belief-update round MAY
  be initiated subject to ADP's existing convergence logic.

The unlock is mechanical and cannot be triggered by agent request. Agents
cannot vote for "more deliberation" in order to drive up their share of the
draw. The signal is computed from the tally itself, before any agent has the
opportunity to revise.

### 5.3 Hook into ADP

ACB's only hard requirement on ADP is that the disagreement magnitude be
*computable* from the initial tally. ADP v0 already exposes
`approve_weight`, `reject_weight`, and `abstain_weight` in
`deliberation_closed.final_tally`, which is sufficient for retrospective
computation. Real-time pricing requires the same scalar at the *initial*
tally — before any belief-update rounds — which is a non-disruptive addition
to ADP's convergence step. ACB v0 RECOMMENDS that ADP v0.1 emit a
`tally_observed` event with the same `final_tally` shape at each round
boundary so ACB-aware tooling can subscribe without coupling to ADP internals.

This is the only ADP-side change ACB requires. It is small, useful to ADP
on its own terms (better round-gating reduces wasted belief-update cycles),
and load-bearing for ACB.

---

## 6. Settlement

### 6.1 Settlement Modes

ACB defines three settlement modes, declared per-budget:

| Mode | When settlement is written | Outcome adjustment |
|---|---|---|
| `immediate` | At `deliberation_closed` | None. Settles against tally and ADJ entries through close. |
| `deferred` | At `outcome_observed`, or at outcome-window expiry | Full. Distribution incorporates outcome correctness into contribution scores. |
| `two_phase` | Preliminary at `deliberation_closed`; adjustment at `outcome_observed` or window expiry | Partial. Substrate share is paid at preliminary; epistemic share is held until adjustment. |

`immediate` is appropriate for low-stakes deliberations where the
deliberation itself is the only available signal — there is no measurable
downstream outcome. `deferred` is appropriate when outcomes arrive within a
useful window (CI runs, monitoring alerts, human sign-off). `two_phase` is
appropriate when substrate providers need timely payment but epistemic
contribution can only be scored against outcome.

### 6.2 Contribution Signal

The epistemic share of the draw is distributed across participating agents in
proportion to a per-agent contribution signal computed from ADJ. ACB v0
defines the `default-v0` profile as follows:

For each agent `a` in the deliberation:

```
contribution(a) =
    base_share(a)
  + falsification_bonus(a)
  + load_bearing_bonus(a)
  + outcome_correctness_bonus(a, outcome)
  − dissent_quality_penalty(a)
```

Where:

- **`base_share(a)`** — equal share of 25% of the epistemic pool, distributed
  evenly across all agents that emitted a `proposal_emitted` entry. This is
  the show-up share. Small but non-zero, so cheap routines still pay
  participants something.
- **`falsification_bonus(a)`** — 25% of the epistemic pool, distributed
  proportionally to the count of `round_event` entries with
  `event_kind ∈ {falsification_evidence, amend}` where `agent_id = a` AND the
  evidence was acknowledged (a subsequent `event_kind: acknowledge` or
  `event_kind: revise` from the targeted agent). Unacknowledged falsification
  attempts pay zero to discourage spam.
- **`load_bearing_bonus(a)`** — 25% of the epistemic pool, distributed to
  agents whose vote at `deliberation_closed` was load-bearing for convergence.
  An agent's vote is load-bearing if removing their `final_tally` weight from
  the tally would have changed the termination state (e.g., dropped the
  `approval_fraction` below the threshold). This is a counterfactual computed
  by replay against the journal.
- **`outcome_correctness_bonus(a, outcome)`** — 25% of the epistemic pool,
  applies only when an `outcome_observed` entry exists. Distributed
  proportionally to per-agent calibration delta: the difference between an
  agent's stated `confidence` in their proposal and the realized outcome,
  scored by Brier (lower is better, inverted to a positive bonus). Agents
  whose confidence matched the outcome closely earn more than agents whose
  confidence was poorly calibrated, regardless of which way they voted.
- **`dissent_quality_penalty(a)`** — A subtracted floor in [0, 0.25 × pool]
  applied to agents whose contribution to the deliberation degraded
  calibration. Triggered when an agent's `condition_amended` entries in this
  deliberation correlate with their long-running condition quality score from
  ADJ §6 falling below a configurable floor (default: 0.4). The penalty
  redistributes to agents above the floor.

The shares sum so that the four bonus pools collectively account for 100% of
the epistemic pool, with the dissent-quality penalty acting as a redistribution
within the pool rather than an additional draw.

**Empty-bonus-pool rule.** When a bonus pool has no eligible recipients — no
agent acknowledged any falsification, no agent was load-bearing, no
`outcome_observed` entry exists at settlement time — the pool MUST distribute
equally across all participants instead of being lost. Without this rule the
draw arithmetic does not close (substrate + epistemic distributions sum to
strictly less than `draw_total`) and the requester would be over-charged
relative to what the protocol actually delivered. The cheap routine in
particular almost always hits this case for the load-bearing and outcome
pools, so the rule is load-bearing for cheap-routine settlement.

This function is intentionally tunable. Any settlement profile that passes
the conformance tests in `acb-validate` may declare itself ACB-compliant.

### 6.3 Substrate Share

The `substrate_share` of the draw is distributed across substrate providers
in proportion to reported cycles. ACB v0 does not specify how substrate
providers report cycles — they MAY use a `substrate_report` envelope that
ACB-aware tooling consumes, or they MAY rely on off-protocol agreements with
the agent operator who runs them.

If no substrate reports are submitted, the substrate share is added to the
epistemic share for that deliberation.

### 6.4 The Settlement Record

Settlement is written as a single `settlement_recorded` entry in the budget
authority's journal. The entry MUST follow the ADJ common envelope (Section
3.0 of ADJ) so that ACB settlements inherit ADJ's append-only, hash-chained,
replayable properties without requiring a parallel log infrastructure.

```json
{
  "$schema": "https://acb-manifest.dev/schemas/settlement/v0.json",
  "entry_id": "adj_01HMZQ7K",
  "entry_type": "settlement_recorded",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-14T09:30:00.000Z",
  "prior_entry_hash": "sha256:p9q0r1...",

  "budget_id": "bgt_01HMXJ3E9R",
  "settlement_profile": "default-v0",
  "outcome_referenced": "adj_01HMZP2D",

  "draw_total": 900,
  "amount_total": 12000,
  "amount_returned_to_requester": 11100,

  "substrate_distributions": [
    {
      "recipient": "did:substrate:acme-cluster-eu",
      "amount": 120,
      "basis": "cycles",
      "report_ref": "substrate:acme-cluster-eu/report/8821443"
    },
    {
      "recipient": "did:substrate:openai-azure",
      "amount": 60,
      "basis": "cycles",
      "report_ref": "substrate:openai/usage/run-9912"
    }
  ],

  "epistemic_distributions": [
    {
      "recipient": "did:adp:test-runner-v2",
      "amount": 380,
      "contribution_breakdown": {
        "base_share": 60,
        "falsification_bonus": 180,
        "load_bearing_bonus": 100,
        "outcome_correctness_bonus": 40,
        "dissent_quality_penalty": 0
      }
    },
    {
      "recipient": "did:adp:security-scanner-v3",
      "amount": 280,
      "contribution_breakdown": {
        "base_share": 60,
        "falsification_bonus": 60,
        "load_bearing_bonus": 100,
        "outcome_correctness_bonus": 60,
        "dissent_quality_penalty": 0
      }
    },
    {
      "recipient": "did:adp:style-linter-v1",
      "amount": 60,
      "contribution_breakdown": {
        "base_share": 60,
        "falsification_bonus": 0,
        "load_bearing_bonus": 0,
        "outcome_correctness_bonus": 0,
        "dissent_quality_penalty": 0
      }
    }
  ],

  "habit_discount_applied": 0.00,
  "unlock_triggered": true,
  "disagreement_magnitude_initial": 0.42,

  "signature": "ed25519:7a4b..."
}
```

| Field | Description |
|---|---|
| `entry_type` | MUST be `"settlement_recorded"`. |
| `budget_id` | The budget being settled. |
| `settlement_profile` | The contribution-scoring function used. |
| `outcome_referenced` | OPTIONAL. The `entry_id` of the `outcome_observed` entry incorporated into this settlement. `null` if `mode = immediate` or no outcome was observed before window expiry. |
| `draw_total` | Total amount drawn from the budget for this deliberation. |
| `amount_total` | The original budget amount (echoed for self-containment). |
| `amount_returned_to_requester` | `amount_total − draw_total`. The unspent remainder returned to `unspent_returns_to`. |
| `substrate_distributions` | Per-substrate allocations summing to `draw_total × substrate_share`. |
| `epistemic_distributions` | Per-agent allocations summing to `draw_total × epistemic_share`. Each entry includes a `contribution_breakdown` for audit. |
| `habit_discount_applied` | The discount factor used in pricing, in [0, 1]. |
| `unlock_triggered` | Whether the expensive routine was engaged. |
| `disagreement_magnitude_initial` | The unlock signal that was computed at the initial tally. |
| `signature` | The budget authority's signature over the canonical serialization. |

### 6.5 Replay and Divergence

Settlement is auditable by replay. Any peer with read access to the budget
authority's journal and the deliberation's journal entries (which may live in
the participating agents' own journals) MUST be able to:

1. Verify the `prior_entry_hash` chain.
2. Re-compute the disagreement magnitude from the initial tally.
3. Re-compute each agent's contribution signal from ADJ entries.
4. Re-compute the per-recipient distribution.
5. Verify the budget authority's signature.

If any step fails, the peer MAY publish a `settlement_contested` entry in
its own journal. ACB v0 does not define a resolution mechanism for contested
settlements — that is left to off-protocol arbitration. What ACB v0 *does*
guarantee is that contested settlements are mechanically detectable, the
same way ADJ guarantees calibration divergence is mechanically detectable.
Detection is the protocol's contribution; resolution is the federation's.

This is the elegant fallout from building on ADJ. ACB does not need a
multi-sig handshake at deliberation close, does not need synchronous agent
availability at settlement time, and does not need a separate consensus
mechanism for the settlement record. The budget authority signs the record
once. Any disagreement is caught by replay against ground-truth journal
entries the same way disagreement about calibration is caught.

---

## 7. Habit Memory

The brain compiles repeated decisions into cheap subcortical routines.
Learning to drive is expensive the first time and effectively free the
thousandth time, because the brain refuses to re-pay for cognition it has
already done. ACB applies the same principle to deliberations.

### 7.1 The Discount Function

A new deliberation's habit discount is a function of how closely it resembles
deliberations already in the federation's journal history. The `default-v0`
discount profile defines:

```
habit_discount(d) = min(0.80, similarity(d, history) × stability(history))
```

Where:

- **`similarity(d, history)`** — In [0, 1]. Computed by an action-similarity
  function that compares the proposed action against `committed_action`
  fields in prior `deliberation_closed` entries. ACB v0 defines this loosely:
  matches on `action.kind` plus structural similarity of `action.parameters`.
  Implementations MAY use embedding-based similarity, exact match, or
  domain-specific heuristics, provided the function is deterministic.
- **`stability(history)`** — In [0, 1]. The fraction of similar prior
  deliberations whose outcome was observed and was successful. A pattern that
  has resolved correctly 100 times is stable. A pattern that has resolved
  correctly 50 of 100 times is unstable, and the discount shrinks toward zero.

The maximum discount is 0.80 by default — the cheapest possible habit-routed
deliberation still costs 20% of its undiscounted price, because even
well-compiled habits incur some metabolic overhead.

### 7.2 Why the Cap Matters

A 100% discount would produce a degenerate state: federation-wide consensus
on a class of decisions would drive the cost of those decisions to zero,
which would then mean no agent has any reason to continue evaluating them,
which would mean the federation stops noticing when the habit becomes wrong.
The 0.80 cap is the analogue of the brain's continued (cheap but non-zero)
attention to habitual stimuli — the federation keeps paying enough to ensure
agents are still actually checking, not just rubber-stamping.

### 7.3 Habit Discounts and the Calibration Loop

Habit discounts compound with the calibration loop in the right direction.
A pattern that has resolved correctly 100 times not only earns the maximum
discount — it also means the participating agents have been calibrated by
those outcomes, so their weights are high under ADP, so the deliberation
converges fast, so it stays in the cheap routine, so the draw is small.
A pattern that suddenly stops resolving correctly causes:

1. Outcomes start failing → calibration weights of agents who voted for the
   pattern degrade.
2. Stability drops → habit discount shrinks.
3. Disagreement magnitude rises as agents diverge → unlock fires.
4. The expensive routine engages, the deliberation gets serious cognition
   again, and the federation re-learns the question.

This is exactly the brain's response to a previously-stable prediction
suddenly failing: the cheap routine stops working, the expensive routine
unlocks, the model gets updated. ACB inherits the loop because it builds on
ADJ.

---

## 8. Worked Example: The PR Merge Settlement

This example continues the ADJ §9 worked example (`dlb_01HMXJ3E9R`) with an
ACB budget attached. The deliberation is a contested PR merge that runs one
belief-update round and converges with `committed_action: merge_pull_request`.
Three agents participate. The outcome (CI green) is observed three days
later.

### 8.1 Budget Posted

**Entry — budget_committed:**

```json
{
  "entry_id": "adj_01HMXM9A",
  "entry_type": "budget_committed",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-11T14:30:00.000Z",
  "prior_entry_hash": null,

  "budget_id": "bgt_01HMXJ3E9R",
  "budget_authority": "did:requester:acme-platform",
  "denomination": { "unit": "EU", "external_unit": "USD", "external_rate": 0.0001 },
  "amount_total": 12000,
  "pricing": {
    "profile": "default-v0",
    "cheap_routine_rate": 50,
    "expensive_routine_rate": 200,
    "round_multiplier": 1.5,
    "unlock_threshold": 0.30,
    "habit_memory_discount": "default-v0"
  },
  "settlement": {
    "profile": "default-v0",
    "mode": "deferred",
    "outcome_window_seconds": 604800,
    "substrate_share": 0.20,
    "epistemic_share": 0.80,
    "unspent_returns_to": "did:requester:acme-platform"
  },
  "signature": "ed25519:6f3a..."
}
```

### 8.2 Deliberation Runs

The deliberation proceeds per ADJ §9. Initial tally shows
`approve_weight = 0.71`, `reject_weight = 0.64`, `abstain_weight = 0.18`.

Disagreement magnitude:
```
non_abstaining = 0.71 + 0.64 = 1.35
magnitude = 1 − |0.71 − 0.64| / 1.35 = 1 − 0.052 = 0.948
```

That is well above the 0.30 unlock threshold. The expensive routine engages.
A belief-update round runs (`round_count = 1`). After falsification and
revision, the deliberation converges to `approve` and `deliberation_closed`
is written.

### 8.3 Habit Discount Lookup

The action `merge_pull_request` against this repository has been decided 47
times in the federation's prior journal history, with a 96% outcome success
rate. Similarity is high (`0.85`), stability is high (`0.96`), so:

```
habit_discount = min(0.80, 0.85 × 0.96) = min(0.80, 0.816) = 0.80
```

The maximum discount applies.

### 8.4 Pricing

```
draw = expensive_routine_rate × participants × multiplier^rounds × (1 − discount)
     = 200 × 3 × 1.5^1 × (1 − 0.80)
     = 200 × 3 × 1.5 × 0.20
     = 180 EU
```

Substrate share = `180 × 0.20 = 36 EU`
Epistemic share = `180 × 0.80 = 144 EU`

### 8.5 Outcome Arrives

Three days later, an `outcome_observed` entry is written: `success: true`,
`reporter_id: did:adp:ci-monitor-v1`, `ground_truth: false`,
`reporter_confidence: 0.95`. The deferred settlement now proceeds.

### 8.6 Contribution Computation

For the epistemic share of 144 EU, distributed by `default-v0`:

| Agent | Base (25%) | Falsif. (25%) | Load-bearing (25%) | Outcome (25%) | Penalty | Total |
|---|---|---|---|---|---|---|
| test-runner-v2 | 12 | 24 | 36 | 14 | 0 | 86 |
| security-scanner-v3 | 12 | 12 | 0 | 12 | 0 | 36 |
| style-linter-v1 | 12 | 0 | 0 | 10 | 0 | 22 |

The base share (36 EU total) is split evenly. The falsification bonus
(36 EU) goes to test-runner (acknowledged amend) and scanner (acknowledged
amend by linter); linter contributed no amends. The load-bearing bonus
(36 EU) goes entirely to test-runner because removing their final-round
weight would have dropped the tally below threshold. The outcome bonus
(36 EU) is distributed by per-agent Brier delta — test-runner's confidence
0.86 maps closest to the realized success outcome.

Substrate share (36 EU) is distributed across the two reported cycle
sources at 24 EU and 12 EU.

### 8.7 Settlement Recorded

**Entry — settlement_recorded:**

```json
{
  "entry_id": "adj_01HMZQ7K",
  "entry_type": "settlement_recorded",
  "deliberation_id": "dlb_01HMXJ3E9R",
  "timestamp": "2026-04-14T09:30:00.000Z",
  "prior_entry_hash": "sha256:p9q0r1...",

  "budget_id": "bgt_01HMXJ3E9R",
  "settlement_profile": "default-v0",
  "outcome_referenced": "adj_01HMZP2D",

  "draw_total": 180,
  "amount_total": 12000,
  "amount_returned_to_requester": 11820,

  "substrate_distributions": [
    { "recipient": "did:substrate:acme-cluster-eu", "amount": 24, "basis": "cycles", "report_ref": "..." },
    { "recipient": "did:substrate:openai-azure", "amount": 12, "basis": "cycles", "report_ref": "..." }
  ],
  "epistemic_distributions": [
    { "recipient": "did:adp:test-runner-v2", "amount": 86, "contribution_breakdown": { "base_share": 12, "falsification_bonus": 24, "load_bearing_bonus": 36, "outcome_correctness_bonus": 14, "dissent_quality_penalty": 0 } },
    { "recipient": "did:adp:security-scanner-v3", "amount": 36, "contribution_breakdown": { "base_share": 12, "falsification_bonus": 12, "load_bearing_bonus": 0, "outcome_correctness_bonus": 12, "dissent_quality_penalty": 0 } },
    { "recipient": "did:adp:style-linter-v1", "amount": 22, "contribution_breakdown": { "base_share": 12, "falsification_bonus": 0, "load_bearing_bonus": 0, "outcome_correctness_bonus": 10, "dissent_quality_penalty": 0 } }
  ],

  "habit_discount_applied": 0.80,
  "unlock_triggered": true,
  "disagreement_magnitude_initial": 0.948,

  "signature": "ed25519:7a4b..."
}
```

The requester gets 11,820 EU back. The federation paid 180 EU for the work
of merging a familiar PR with one round of contested deliberation. The
agent that did the falsification work earned the most. The agent that
contributed nothing distinguishable earned only its base share. Without the
habit discount, the same deliberation would have cost 900 EU — five times as
much — and that gap is the federation's accumulated learning showing up on
the bill.

---

## 9. Compliance Levels

Implementations declare compliance level in their manifest.

| Level | Required capabilities |
|---|---|
| **L1 — Budget posting** | Write `budget_committed` entries. Honor `amount_total`. No pricing model required; flat-rate settlement permitted. |
| **L2 — Pricing** | L1 + implement at least the `default-v0` pricing profile. Compute disagreement magnitude. Honor unlock threshold. Write `settlement_recorded` entries. |
| **L3 — Settlement profiles** | L2 + implement at least one `default-v0` settlement profile. Distribute by contribution signals from ADJ. Support `immediate` mode at minimum. |
| **L4 — Full ACB** | L3 + support `deferred` and `two_phase` settlement modes. Implement habit-memory discounts. Support replay-based settlement verification. Honor outcome-window expiry. |

L1 and L2 are useful for closed federations or single-operator setups where
contribution scoring is uninteresting. L4 is required for open federations
where settlement disputes need to be mechanically auditable.

---

## 10. Relationship to Other Specs

### 10.1 ACB and ADP

ACB depends on ADP for the deliberation lifecycle and the disagreement
magnitude signal. ADP does not depend on ACB. ADP-only deployments function
unchanged.

ACB v0 RECOMMENDS one small addition to ADP v0.1: emit `tally_observed`
events at each round boundary so ACB-aware tooling can compute disagreement
magnitude in real time. This is non-disruptive — ADP can implement it as a
hook that no-ops when no listener is attached.

### 10.2 ACB and ADJ

ACB depends on ADJ for storage, replay, contribution evidence, and the
common entry envelope. ADJ adopts two new entry types from ACB in its
v0.1 hook list: `budget_committed` and `settlement_recorded`. Both follow
the ADJ §3.0 common envelope and inherit hash chaining, append-only
guarantees, and replay verification.

ADJ-only deployments function unchanged. Without ACB, the new entry types
are simply unused.

### 10.3 ACB and the Registry

ACB introduces one new federation-level health metric to be tracked by
registry implementations: **budget efficiency**, defined as the ratio of
deliberations that stayed in the cheap routine to total deliberations over
a window. A healthy federation has high budget efficiency — most decisions
are easy, and the expensive routine unlocks only when needed. A federation
with falling budget efficiency is either over-escalating (manufactured
disagreement to drive up draws) or under-escalating (collusion to suppress
disagreement and skim cheap-routine fees without doing the work). The
metric belongs in the registry alongside calibration mobility and Gini
because that is where federation-level health is already observed.

The registry does not need to know about individual budgets, settlements,
or pricing. It just needs one more field in the federation health snapshot.

### 10.4 ACB and mcp-manifest

ACB does not require any change to mcp-manifest. Agents that wish to
publish their substrate-cost preferences MAY do so in an MCP capability
declaration, but ACB v0 does not specify the format. This is a v1 question.

---

## 11. Open Questions

These are intentionally unresolved in v0 and flagged for v1 work, almost
certainly informed by real settlement traffic from a reference deployment.

1. **Pricing tuning.** The default rates, multipliers, and unlock thresholds
   in Section 3.1 are placeholders. Real values will be set by experience.
2. **Substrate cost reporting format.** Section 6.3 punts on how substrate
   providers report cycles. A standard envelope would help interoperability
   but is premature without implementation experience.
3. **Cross-budget pooling.** Should an agent operator be able to maintain a
   running balance across many deliberations rather than receiving micropayments
   per-settlement? Probably yes, but the semantics of pooling, withdrawal, and
   dispute are out of scope for v0.
4. **Marketplace pricing.** ACB v0 is unambiguously a *bounty* model: the
   requester posts, the protocol distributes by contribution. A future v1
   may layer marketplace primitives on top — agents publishing per-call
   prices in their manifests, requesters routing around expensive agents.
   This is intentionally deferred. Marketplace pricing creates a signaling
   game (agents bidding their confidence via their price) that ACB v0 is
   trying to route around by using contribution evidence instead.
5. **Settlement disputes.** Section 6.5 specifies that contested settlements
   are mechanically detectable but punts resolution to off-protocol
   arbitration. A future v1 may define a lightweight dispute protocol — a
   second-pass replay by a registry-elected arbiter, for instance — but the
   v0 position is that the federation health metrics are sufficient pressure.
6. **Calibration of the contribution function itself.** The `default-v0`
   settlement profile makes specific weighting choices (25/25/25/25 across
   four bonus categories). Whether that mix produces the right incentive
   gradient is unknown until there is real settlement data.
7. **Habit-memory similarity function.** Section 7.1 specifies similarity
   loosely. A canonical similarity function may emerge from implementation
   experience, or this may remain pluggable.

---

## 12. Federation

ACB inherits federation properties from ADJ. There is no central settlement
authority. Every budget authority maintains its own journal. Every
participating agent maintains its own journal. Settlement records are
written once per deliberation by the budget authority's journal, signed,
and verifiable by replay against the deliberation entries in the
participating agents' journals.

A federation-wide audit of any settlement requires only:

1. The budget authority's `budget_committed` and `settlement_recorded`
   entries.
2. The participating agents' `proposal_emitted`, `round_event`, and
   `deliberation_closed` entries for the referenced deliberation.
3. Optionally, the `outcome_observed` entry from whichever agent reported
   the outcome.

All of these are journal entries with hash-chained integrity and are
discoverable via the same query contract ADJ §7 already defines. ACB
adds no new transport, no new storage, no new identity, no new consensus
mechanism. The settlement record is just another entry in the same log
that records the decision it settles.

This is what makes ACB deployable today rather than waiting for a
parallel ledger infrastructure to exist. The infrastructure is already
in ADJ.

---

## Appendix A: The Energy Budget as Architecture

The brain analogy is not decoration. It is the architectural argument for
why ACB looks the way it does instead of looking like a marketplace, an
auction, or a gas market. A short version, for readers wondering whether
the analogy is doing real work or just being cited.

The brain runs on a fixed energy budget and refuses to spend that budget
on deliberate reasoning unless prediction error forces the unlock.
Cognition is metabolically expensive, so the default is the cheapest
available routine that has historically produced non-fatal outcomes.
Critical thinking is gated behind evidence the cheap routine will not work.

Every architectural choice in this spec falls out of treating that as
literal:

- **The cheap routine pays a small base rate** because metabolically, the
  brain still spends some calories even on habitual routines. Zero would
  be wrong, because then there is no incentive for the agent to remain
  attentive. A small floor preserves attention.
- **The expensive routine engages on disagreement magnitude, not on agent
  request**, because in the brain, the unlock signal is computed by the
  predictive machinery itself, not requested by the cognitive process that
  benefits from it. An agent voting for "more deliberation" in order to
  earn more would be the analogue of a thought asking the brain to spend
  more glucose on it, which is exactly the failure mode evolution selected
  against.
- **Round multipliers are exponential** because each additional belief-update
  round addresses, by selection, the disagreement the prior round failed to
  resolve. The remaining work is harder and metabolically more costly.
- **Habit memory discounts repeated decisions toward (but not to) zero**
  because the brain compiles repeated decisions into cheap routines but does
  not stop attending to them entirely. Total compilation would mean the
  federation stops noticing when the habit becomes wrong.
- **Settlement is contribution-weighted, not participation-weighted**,
  because the brain rewards prediction errors only when they update the
  model in a way that improves future prediction. Disagreement that does
  not lead to better calibration is noise, and noisy sources get
  metabolically suppressed.
- **The substrate provider and the agent identity are paid from the same
  draw** because the brain does not maintain two energy budgets, one for
  cycles and one for cognition. They are the same metabolic act, billed
  to the same pool, and decoupling them at the protocol layer would be
  pretending the architecture is something it is not.

This is the meaningful claim of ACB and the reason it is a third spec
rather than a feature in the others: pricing cognition is a distinct
concern with its own architectural logic, and that logic happens to be
the logic the only known working example of cognition — the brain —
already uses. The protocol's job is to inherit it faithfully.
