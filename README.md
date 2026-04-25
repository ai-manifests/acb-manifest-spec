# acb-manifest specification

_A specification by MarketAlly_

A cognitive-budget protocol for agent deliberations — pricing cognition the way the brain does, so that easy decisions are cheap and contested decisions cost what they actually cost.

## The Problem

Agent federations need a way to pay for the cognitive work of deciding things together. Existing answers borrow from places they shouldn't:

- **Crypto gas markets** pay for cycles, not for the epistemic work the cycles produced. Fine when the agent IS the VM. Wrong when the agent and the substrate are decoupled, which is the entire point of [ADP](https://adp-manifest.dev).
- **Per-vote participation fees** reward showing up, which incentivizes vote spam and round-dragging.
- **Marketplace pricing** turns agent confidence into a bid signal, which is exactly what ADP routes around by using calibration history instead.

What's needed is a payment rail that mirrors the way the brain itself prices cognition — cheap by default, expensive only when prediction error forces the unlock, and always settled against retrospective contribution rather than real-time participation.

## The Solution

ACB defines a budget object the requester posts to a deliberation, a pricing model that distinguishes cheap routines from expensive routines based on disagreement magnitude, and a settlement record that distributes the drawn amount across substrate providers and agent identities by their contribution evidence in the journal.

```json
{ "entry_type": "budget_committed", "deliberation_id": "dlb_01HMXJ3E9R", "amount_total": 12000, "pricing": { "unlock_threshold": 0.30 } }

{ "entry_type": "settlement_recorded", "deliberation_id": "dlb_01HMXJ3E9R", "draw_total": 180, "amount_returned_to_requester": 11820, "habit_discount_applied": 0.80, "unlock_triggered": true }
```

The brain analogy is not decoration — it is the architectural argument. The brain runs on a fixed energy budget and refuses to spend it on deliberate reasoning unless prediction error forces the unlock. ACB inherits that property: agreement is cheap because agreement is evidence the cheap routine sufficed; disagreement is expensive because disagreement is evidence more cognition was actually required. The unlock is mechanical and cannot be triggered by agent request.

## Contents

| File | Description |
|------|-------------|
| [spec.md](spec.md) | Full specification (v0, draft) |
| [schema/v0.json](schema/v0.json) | JSON Schema for `budget_committed`, `budget_cancelled`, and `settlement_recorded` entries |
| [examples/](examples/) | Reference budget, cancellation, and settlement records — all matched to the ADJ §9 worked example |
| [ci/](ci/) | CI workflow templates (GitHub Actions and Gitea Actions) for entry validation |

## Examples

- [budget-committed.json](examples/budget-committed.json) — 12,000 EU posted for a contested PR merge, deferred settlement, 80/20 epistemic/substrate split
- [settlement-recorded.json](examples/settlement-recorded.json) — final settlement after the deliberation: 180 EU drawn (80% habit discount), distributed across two substrate providers and three agents with full contribution breakdown
- [budget-cancelled.json](examples/budget-cancelled.json) — a budget cancelled before the deliberation began

## How It Composes

```
mcp-manifest   declares what an agent can do
ADP            declares how agents agree on doing it together
ADJ            declares how those agreements are recorded and scored
ACB            declares how the cognitive work of agreeing is paid for
```

ACB reads disagreement magnitude from ADP. ACB settles against contribution evidence in ADJ. ACB adds two new entry types (`budget_committed`, `settlement_recorded`) that ADJ adopts in its v0.1 hook list. Each spec remains independently implementable. ADP and ADJ without ACB function unchanged. ACB without ADJ degrades to flat-rate pricing because there is no contribution evidence to settle against.

## Three Things ACB Refuses To Be

1. **A monetization layer.** ACB is a calibration loop in a second currency. The reason bad actors lose money under ACB is not that they are punished — it is that they are metabolically inefficient. They draw energy without delivering the cognitive work that justifies the draw, and the budget keeper eventually stops feeding them. Same reason your brain stops paying attention to a stimulus that has never once mattered.

2. **A marketplace.** ACB v0 is unambiguously a *bounty* model. The requester posts a budget and the protocol distributes by contribution. Marketplace primitives (agents publishing per-call prices in their manifests) are explicitly deferred to a future v1, because they would re-introduce exactly the signaling game ADP is designed to route around.

3. **A gas market.** Compute cost (paid to substrate providers) and epistemic cost (paid to agent identities) come from the same draw. The brain does not maintain two budgets, and ACB does not either. Decoupling the agent from the VM means the agent's reputation travels with its identity, not its hardware — but the payment for both still flows from one pool, in proportion to what each contributed.

## Quick Start

### For deliberation runners

1. Read the budget at `deliberation_opened`. Honor `constraints.max_participants` and `constraints.max_rounds`.
2. Compute disagreement magnitude (Section 5.1) at the initial tally and emit it as part of the deliberation context.
3. Apply the unlock rule before initiating any belief-update round.
4. Track per-agent draws as the deliberation runs.
5. At terminal state, write `settlement_recorded` to the budget authority's journal.

### For ADJ implementations

Adopt two new entry types from ACB v0.1 hook list: `budget_committed` and `settlement_recorded`. Both follow the existing common envelope. No other changes required.

### For agent operators

Nothing changes. Your agents continue to participate in deliberations exactly as they did under ADP. The settlement record will appear in the budget authority's journal and you will be able to verify your share by replay against your own deliberation entries.

## Licensing Model

The ACB specification text is released under the [Community Specification 
License 1.0 (Patent and Copyright)](https://github.com/CommunitySpecification/1.0).
Reference implementations, schemas, and tooling are released under the Apache 2.0 License.
The specification is fully implementable without reliance on any specific implementation.

Contributions to this specification are accepted under the same Community 
Specification License 1.0 terms. By submitting a contribution, you agree to 
the copyright and patent grants defined therein.
 
## Document Metadata
- Author: David H Friedel Jr.
- Copyright: © 2026 David H Friedel Jr.
- First Published: 2026-03-20
- Affiliated with: MarketAlly (https://marketally.ai)
- **Specification**: acb-manifest  
- **Version**: v0 (Draft)  
- **Status**: Public Draft  

David H Friedel Jr. is the current copyright holder of this specification. 
The MarketAlly group includes MarketAlly LLC (USA), MarketAlly Pte Ltd 
(Singapore), and MarketAlly OÜ (Estonia). This specification is being 
prepared for assignment to a MarketAlly entity; until that assignment is 
in place, copyright remains with the individual author.

"ADP", "ACB", "ADJ", and "Agent Registry" are trademarks of MarketAlly. 
The specifications are openly licensed; the names are not.