# Testnet Threat Model

## Status

This is a **lightweight, testnet-scoped** threat model. It is not a second
design-level threat model and not a correction to
[`docs/security.md`](../../docs/security.md). It describes what can actually go
wrong on the **one shared testnet deployment** that many unrelated parties use
— and, just as importantly, what is explicitly not a concern there.

> The deployment is not yet live. As of the last review, no Bridgelet contracts
> are deployed to testnet (`../registry/contract-status.md`). This model is
> written ahead of the deployment, describing the environment's threat surface
> rather than asserting anything has happened.

## What this model is, and what it is not

| | [`docs/security.md`](../../docs/security.md) | This document |
|---|---|---|
| Question | Is the design sound? | What can go wrong on the shared testnet environment? |
| Scope | The contracts as specified | One shared deployment, many parties |
| Nature | Static, design-level | Operational, environment-level |
| Time | Stable; changes when the design changes | Changes when anyone touches the deployment |
| Value at stake | Real funds (in a mainnet reading) | Integration credibility, time, reputation |

`docs/security.md` can be entirely correct while the testnet deployment is being
misbehaved by a user of it; nothing in a design document would reflect that.
That gap is the reason this document exists.

## Scope

**In scope:** the shared testnet deployment as a multi-tenant environment; what
unrelated integrators can do to each other; what an integrator can lose even
though no real value is at stake; the failure modes of a public demo; and the
drift of documentation and keys that testnet workflows encourage.

**Out of scope:** real fund theft (there is none to model here);
the correctness of the mainnet-intended design (`docs/security.md`);
the SDK/CLI supply chain (`dependency-audit-notes.md`);
and any legal or compliance question. Abuse patterns and their on-chain
signatures are catalogued in `known-testnet-abuse-patterns.md` — this document
defines the risks those patterns produce, not the patterns themselves.

## What is actually at risk

Testnet holds no real value. A threat model that stopped there would conclude
there is nothing to model. The point of this one is that the value is real but
non-financial.

| Asset | Why it matters | How it is lost |
|---|---|---|
| **Integration credibility** | Conclusions drawn from testnet are carried into mainnet designs, audits, and docs | A griefing pattern is misread as a contract bug, or a false security property is believed |
| **Engineering time** | Debugging a shared-state artefact costs hours that the artefact does not deserve | Contention or griefing is investigated as a code defect |
| **Demo reputation** | A public demo that fails is seen by an audience, and testnet fails in ways localnet does not | Expired account mid-demo, RPC outage, wrong network, someone else's abuse |
| **Shared-deployment liveness** | Everyone builds against the same contracts; one party's abuse degrades all | Nonce churn, index exhaustion, RPC spam |
| **Data hygiene** | Testnet workflows are casual, and casual handling leaks secrets that matter later | A mainnet-intended key is used on testnet (`testnet-vs-mainnet-key-reuse-warning.md`) |
| **Documentation accuracy** | Integrators trust the registry; a reset or redeploy silently invalidates every ID | Stale contract IDs survive a reset (`../registry/contract-status.md`) |

## Actors

- **Other integrators** — not attackers. They cause contention simply by using
  the same contracts at the same time. Most "attacks" turn out to be this.
- **Griefers** — parties who interfere with the shared state for no financial
  gain, because the state is public and writable. The absence of a profit
  motive is why the impact is denial and confusion, not theft.
- **The audience of a public demo** — an actor only in the reputational sense.
- **The upstream operator (SDF)** — resets testnet periodically, destroying all
  accounts and IDs. This is an expected event, not an attack.
- **No actor at all** — ordinary bugs and failure modes. A threat model that
  assumes every failure is deliberate is as misleading as one that assumes none
  are.

## Threat scenarios

| Scenario | Vector | Consequence here | Reference |
|---|---|---|---|
| Benign contention read as a bug | Two integrators sweep at once; the loser sees a nonce or signature failure | Wasted debugging on a non-defect | `known-testnet-abuse-patterns.md` P3; `../runbooks/failed-sweep-signature.md` |
| Index-slot exhaustion | Prior caller exhausts factory slots | `batch_initialize` returns failures that look like a broken factory | `known-testnet-abuse-patterns.md` P1; `../runbooks/account-factory-batch-failure.md` |
| Payment injection | Anyone can record a payment to a public account | Phantom balances; misleading demos and balances | `known-testnet-abuse-patterns.md` P7 |
| Destination repointing / unauthorised upgrade | Shared admin surface, or a sweep before the nonce is locked | A demo's destination changes; a deployment is altered | `known-testnet-abuse-patterns.md` P4, P5 |
| Replay and front-running | Public, signable operations | Duplicated or hijacked work; stuck accounts | `known-testnet-abuse-patterns.md` P2 |
| Resource and RPC spam | Heavy traffic against shared endpoints | Latency and failed submissions for everyone | `known-testnet-abuse-patterns.md` P9; `../runbooks/rpc-outage.md` |
| Public demo failure | Expiry, outage, or wrong network during a live demo | Reputational, not financial | `../runbooks/stuck-ephemeral-account.md`, `../runbooks/wrong-network-passphrase.md` |
| Secret exposure through testnet tooling | Casual key handling in a low-stakes environment | A mainnet-intended key is compromised later | `testnet-vs-mainnet-key-reuse-warning.md` |

## What is explicitly not a concern here

Stated as plainly as the risks, so the model is not read as alarmist:

- **Theft of value.** `EphemeralAccount` moves no SEP-41 tokens; testnet XLM is
  worthless. No scenario above is a fund-theft scenario.
- **Losing testnet XLM.** Re-fund with Friendbot. It is not an incident.
- **Someone else using "your" test accounts.** Addresses and contracts are
  public and shared by design; there is no private testnet.
- **A testnet reset.** Expected and periodic. It invalidates IDs, not people.
- **Correct alerts firing.** An ablation or a real condition is not a false
  positive; record it, do not suppress it.

## Handling findings

- A suspected security-relevant finding while testing against testnet goes
  through `disclosure-process.md` before anything is posted publicly.
- An observation that made you look — including a false alarm — goes in
  `reentrancy-observation-log.md` if it is reentrancy-adjacent, or is triaged
  against `known-testnet-abuse-patterns.md`.
- Step-by-step diagnostics belong to `../runbooks/`; this document says what is
  at risk and what is not, not how to fix it.

## Related documentation

- [`../../docs/security.md`](../../docs/security.md) — the design-level model
  this one deliberately does not duplicate
- [`known-testnet-abuse-patterns.md`](known-testnet-abuse-patterns.md) — the
  patterns the scenarios above are built from
- [`README.md`](README.md) — index and standing rules for this directory
- [`unverified-signature-path-warning.md`](unverified-signature-path-warning.md)
  and [`authorized-controller-confusion.md`](authorized-controller-confusion.md)
  — the two false security properties most likely to enter a design
- [`testnet-vs-mainnet-key-reuse-warning.md`](testnet-vs-mainnet-key-reuse-warning.md)
  — the data-hygiene risk
- [`disclosure-process.md`](disclosure-process.md) — where a finding goes
- [`../registry/contract-status.md`](../registry/contract-status.md) — whether
  anything is actually live

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial lightweight, testnet-scoped threat model |
