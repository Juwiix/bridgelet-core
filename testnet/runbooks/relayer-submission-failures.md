# Runbook: Diagnosing Relayer-Side Failures on the Gas-Free Claim Path

## Overview

`SweepController::claim()` is the gas-free path: the recipient signs a Soroban
auth entry off-chain, and a relayer submits the transaction and pays every fee.
That split creates a failure class that does not exist on the signed-sweep path
— the recipient did everything right, and the claim still fails because of
something on the relayer's side of the transaction.

This runbook covers **relayer-submission failures only**: an unfunded relayer,
a fee that does not cover the transaction, and network problems between the
submitter and testnet. Failures in the recipient's own signed auth entry are
explicitly out of scope here — they look different, and the final section shows
how to tell them apart before going down the wrong diagnostic path.

## Symptom

Any of the following on the claim path:

- The CLI or SDK errors before a transaction hash is returned.
- Submission is rejected with a fee-class result (`txInsufficientFee`,
  `txInsufficientBalance`).
- The relayer's endpoint is unreachable, times out, or returns HTTP errors.
- A hash was returned, but no outcome ever appears.
- The claim fails even though the recipient's auth entry was signed correctly.

## Prerequisites

- Testnet identities with XLM, kept separate by role
  (`../config/soroban-cli-identity-setup.md`):
  - `testnet-relayer` — submits the claim, pays all fees;
  - `testnet-investigator` — read-only queries for this diagnosis.
- Contract IDs from `../registry/contract-status.md`.
- The transaction hash, if one was produced. Preserve it exactly
  (`../monitoring/explorer-links.md`).
- Access to the same RPC endpoint the claim was submitted against
  (`../docs/network-config.md`).

## First split: whose failure is this?

Diagnose the side before the cause. The stage at which the failure appears
decides which half of the two-party flow to investigate:

| Stage | Observation | Side | Go to |
|---|---|---|---|
| Before submission | CLI/SDK error, no hash returned | Relayer | Common causes below |
| Before submission | `Error(Auth, InvalidAction)` or a bare auth failure | Recipient | [Not this runbook](#recipient-side-failures-not-this-runbook) |
| After acceptance | Named contract error (table below) | Account state | [Claim-reachable contract errors](#claim-reachable-contract-errors) |
| After submission | Hash exists, outcome unknown | Either | [Cause 4](#cause-4-submission-timeout-outcome-unknown) |

A failure the recipient can reproduce with a different, funded relayer is not
a relayer failure. A failure that follows the relayer across different
recipients is not a recipient failure.

## Common causes

### Cause 1: Relayer out of funds

**Symptom**: submission rejected with `txInsufficientBalance`, or the balance
cannot cover the inclusion fee plus the Soroban resource fee. Nothing was
simulated or executed.

**Diagnosis**:

```bash
stellar keys balance testnet-relayer --network testnet
```

**Resolution**: fund the relayer and resubmit:

```bash
stellar keys fund testnet-relayer --network testnet
```

Friendbot is rate-limited per address
(`../examples/integration-test-harness-notes.md`); a rate-limit response is
not an outage. The proposed funding guardrails in
`../monitoring/balance-thresholds.md` are a 5 XLM hard floor and a 50 XLM
working target for relayer identities. No automated balance check or top-up
exists in this repository — the check above is manual.

### Cause 2: Fee does not cover the transaction

**Symptom**: rejected at submission with `txInsufficientFee`, or a
fee-class rejection even though the relayer's balance looks fine for a plain
payment.

**Diagnosis**:

1. Confirm which fee is being paid. A claim carries two parts:
   - the **inclusion fee** in the envelope's `fee` field, priced per operation
     against the network minimum; and
   - the **Soroban resource fee** returned by simulation (`minResourceFee`),
     which covers execution and rent for the contract storage the call touches.
2. An envelope assembled by hand with only the inclusion fee, without the
   simulated resource fee, fails at submission even when the relayer holds
   plenty of XLM. The same applies to an envelope whose resource fee was
   simulated against different arguments than the one submitted.
3. Compare the fee actually in the envelope with the fee the last simulation
   returned.

**Resolution**: re-simulate against the exact arguments being submitted, set
the envelope fee from that result, and resubmit. The network-level step in
`insufficient-reserve-errors.md` covers the resource-fee/rent mechanics in
detail; an underfunded source failing at submission implicates the network
layer, not the contracts.

### Cause 3: Relayer network issues

**Symptom**: connection errors, HTTP timeouts, or unhealthy responses from the
endpoint — no hash, or an error that mentions the transport rather than the
transaction.

**Diagnosis**:

```bash
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' \
  "$STELLAR_SOROBAN_RPC_URL"
```

Compare the returned sequence with an earlier successful check. Then check the
official [Stellar status page](https://status.stellar.org/) and record both its
advisory state and your direct endpoint evidence — one network location failing
is a suspected incident until broader reachability is established.

Also confirm the submission targets the right network. A wrong passphrase is a
configuration fault, not an upstream outage: it presents as transactions that
are rejected outright, and it is diagnosed in
`wrong-network-passphrase.md`.

**Resolution**: if the endpoint is down, follow `rpc-outage.md` — including
its instruction to pause dependent automation and to handle in-flight
transactions before recovery. Do not treat an RPC failure as a contract
failure.

### Cause 4: Submission timeout, outcome unknown

**Symptom**: the submitter never returned a definitive result. The transaction
may or may not have been accepted.

**Diagnosis**:

1. Recover the hash (from the submitter's log or the constructed envelope).
2. Look it up directly — Horizon at
   `$STELLAR_HORIZON_URL/transactions/<HASH>`, or an explorer link from
   `../monitoring/explorer-links.md` — before doing anything else.
3. Classify the result: final (success or failure), or genuinely unknown.

**Resolution**: if the hash is final, the outcome is already decided — proceed
to the contract-error table below if it failed. If it is still unknown,
**do not build a new envelope for the same intent.** Look up the stored hash
again; only an identical signed envelope is a safe retry. The rules are in
`../examples/idempotency-considerations.md`.

## Claim-reachable contract errors

If the transaction was accepted and failed with a named contract error, the
relayer did its job — the cause is account state or destination configuration,
not submission. These are the errors `claim()` can actually produce:

| Error | Source | Meaning | Fix |
|---|---|---|---|
| `UnauthorizedDestination` (13) | `SweepController` | Locked mode; recipient does not match the locked destination | Use the locked destination, or change it while the nonce is still 0 |
| `NotInitialized` (2) | `EphemeralAccount` | Wrong or uninitialized ephemeral account ID | Check the ID against `../registry/contract-status.md` |
| `AlreadySwept` (7) | `EphemeralAccount` | Duplicate submission of a finished claim | Expected on retry; confirm status instead of resubmitting |
| `NoPaymentReceived` (10) | `EphemeralAccount` | No payment recorded, or the summed amount is zero | `record_payment()` first |
| `AccountExpired` (11) | `EphemeralAccount` | Past `expiry_ledger` | Expired accounts go to `recovery_address` |
| `Unauthorized` (8) | `EphemeralAccount` | The stored controller is not the caller | Deployment wiring fault — see escalation |

The `get_info()` call at the top of `claim()` traps if the ephemeral account
address is not an initialized contract, which presents as a host error rather
than a named error.

> The HTTP mapping in `../integration/error-code-to-http-status-mapping.md`
> lists `AccountNotReady` (5) and `AccountExpired` (6) against sweep failures
> generally. On the `claim()` path the account-state failures surface as the
> `EphemeralAccount` codes above (10, 11), not as controller codes 5 and 6.
> Trust the error the transaction actually returned.

## Recipient-side failures (not this runbook)

For the boundary: these indicate the recipient's auth entry, and the fix
belongs to the signer, not the relayer.

- **Bare auth failure, no contract error** — the recipient did not sign, or
  signed a different contract, function, or argument set. Every argument in
  the entry is bound: `recipient` and `ephemeral_account` must match the
  submitted invocation exactly.
- **`Error(Auth, InvalidAction)`** — the host rejected the authorization
  (mapped to `403 FORBIDDEN` by the integration mapping).
- The most confusing case is a bare auth failure rather than a named error, so
  compare the signed entry against the invocation before suspecting either
  contract. The full failure table for the recipient's side is in
  `../examples/gas-free-claim-walkthrough.md`, and the claim-path section of
  `stuck-ephemeral-account.md` covers the same boundary.

## Quick reference

| Observation | Likely cause | Owner of the fix |
|---|---|---|
| No hash, `txInsufficientBalance` | Relayer out of funds | Relayer |
| No hash, `txInsufficientFee` | Inclusion or resource fee short | Relayer |
| No hash, connection/HTTP error | RPC or network | Upstream or local network |
| No hash, passphrase rejection | Wrong network configuration | You |
| Hash unknown, no outcome | Submission timeout | Relayer — look up, do not rebuild |
| Bare / `Error(Auth, ...)` failure | Recipient auth entry mismatch | Recipient signer |
| `UnauthorizedDestination` (13) | Locked-mode mismatch | Integrator |
| Account-state errors (2, 7, 10, 11) | Preconditions of this account | Integrator |

## Escalation

1. Endpoint or upstream suspicion: `rpc-outage.md`, then
   `https://status.stellar.org/` for an official advisory.
2. Endpoint still healthy and the failure reproduces with a funded relayer,
   a fresh simulation, and a known-good recipient: record the transaction
   hash, ledger, contract IDs, and exact error, and report via
   `../docs/support.md`.
3. `Unauthorized` (8) from `EphemeralAccount` during a claim means the stored
   controller is not the submitting `SweepController` — stop and verify the
   deployment wiring per `../registry/verify-contract-live.md` before filing
   anything else.

## Prevention checklist

For relayer operators and SDK implementers:

- [ ] Relayer identity funded to the working target, checked before a demo or
      batch, not after the first failure.
- [ ] Envelope fee taken from a simulation of the exact submitted arguments.
- [ ] Relayer, recipient, and investigator identities kept separate
      (`../config/soroban-cli-identity-setup.md`).
- [ ] Transaction hash recorded at construction, before submission.
- [ ] Unknown outcomes resolved by hash lookup, never by blind resubmission.
- [ ] Endpoint health checked once before a session of claim submissions.
- [ ] Claim failures classified by the first-split table before anyone
      debugs the contracts.

## Related Documentation

- `../examples/gas-free-claim-walkthrough.md` — the claim flow end to end, and
  the recipient-side failure table
- `../config/gas-free-claim-testing.md` — testing the two-party interaction
- `failed-sweep-signature.md` — the signed-path analogue
  (`execute_sweep`), where signature and nonce failures live
- `insufficient-reserve-errors.md` — network-level fee and rent rejections
- `rpc-outage.md` — endpoint outage handling and recovery
- `wrong-network-passphrase.md` — submission rejected for the wrong network
- `../examples/idempotency-considerations.md` — resolving unknown outcomes by
  stored hash
- `../integration/error-code-to-http-status-mapping.md` — error to HTTP
  mapping
- `../monitoring/balance-thresholds.md` — proposed funding guardrails for
  relayer identities
- `../docs/fee-model.md` — who pays which fee, per call type

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial runbook for relayer-side claim submission failures |
