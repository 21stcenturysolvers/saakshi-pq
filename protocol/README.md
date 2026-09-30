# P0: Protocol freeze

Status: next. Nothing in this directory describes implemented software.

P0 fixes the protocol on paper before any P1 code depends on it. A frozen protocol is a written definition of the records, states, roles and interfaces that the rest of the system is built against. After the freeze, a change needs a recorded decision (see [Change control](#change-control)).

## Scope

In scope for the public P0 documents:

- Record types (PREPARE and COMPLETE) and their lifecycle.
- The release agent state machine, at the level of states and transitions.
- Roles and the interfaces between components: sender and envelope service, recipient enclave, ledger, escrow and arbiters, forensic pipeline.
- Failure behaviour: no ledger record means no plaintext; a failed watermark self-check means no release.
- What the signed evidence package shows, and the three possible decisions: ATTRIBUTED, LEAD, NO ATTRIBUTION.

Out of scope for this public repository: key-derivation details, parameter and capacity budgets, detection thresholds, payload layouts and test vectors. These are held in the team's private specification.

## Planned documents

| Document | Purpose | Status |
| --- | --- | --- |
| README.md | Scope, exit criteria, change control | This file |
| records.md | PREPARE and COMPLETE records and their lifecycle | Planned |
| release-flow.md | Release agent states and transitions | Planned |
| interfaces.md | Roles and component interfaces | Planned |
| decision-log.md | Recorded decisions and their reasons | Planned |

## Exit criteria

P0 is complete when all of the following hold. None is met yet.

- [ ] Record types and their lifecycle are written down and reviewed by at least two team members.
- [ ] The release agent state machine has no undefined state or transition.
- [ ] Every interface between components names its caller, callee and failure behaviour.
- [ ] Each rule in the claims boundary can be traced to a document in this directory.
- [ ] The private specification and these public documents have been checked for consistency.
- [ ] Every open question is either resolved in the decision log or assigned to a later phase.

## Change control

After the freeze, any change to a frozen document is made in a pull request that adds an entry to the decision log stating what changed, why, and which later work it affects.
