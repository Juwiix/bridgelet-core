# Third-Party Dependency Status

## Status

> **Reference document only.** Nothing here polls, checks, or alerts
> automatically. The probes below are manual commands an operator runs while
> triaging. No dependency monitor exists in this repository.

## Purpose

The shared testnet deployment depends on a handful of external services that
this project does not control. When something breaks, the first question is
always *"is this us or is it upstream?"* — and answering it by guesswork is how
an upstream outage gets misattributed to a contract bug, or a contract bug gets
blamed on upstream and goes uninvestigated.

This page is the single reference point for that question: what the
dependencies are, how to check each one directly, and what a failure of each
one does and does not imply. It is the companion to
[`../runbooks/rpc-outage.md`](../runbooks/rpc-outage.md), which covers the full
handling procedure once an upstream problem is confirmed.

## Dependencies

| Dependency | Endpoint | What we use it for | Our control | Status reference |
|---|---|---|---|---|
| Soroban RPC | `soroban-testnet.stellar.org` | Simulating and submitting every contract call; reading contract storage and events | None | [Stellar status page](https://status.stellar.org/) + direct probe |
| Horizon | `horizon-testnet.stellar.org` | Classic account data: balances, payment history, trustlines | None | [Stellar status page](https://status.stellar.org/) + direct probe |
| Friendbot | `friendbot.stellar.org` | Funding fresh testnet accounts with test XLM | None | No separate status page documented; probe by request |
| Stellar Expert | `stellar.expert/explorer/testnet` | Human drill-down and corroboration of transactions and contracts | None | No separate status page documented; probe by page fetch |

Endpoints and passphrase are defined in
[`../docs/network-config.md`](../docs/network-config.md). Explorer link
construction for recorded and dynamic IDs is in
[`explorer-links.md`](explorer-links.md).

> The Stellar status page is advisory. Record both its state **and** your direct
> endpoint evidence: one network location failing is a suspected upstream
> incident until broader reachability is established (`../runbooks/rpc-outage.md`).

## Direct probes

### Soroban RPC

```bash
curl --fail-with-body --silent --show-error \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"getLatestLedger"}' \
  "$STELLAR_SOROBAN_RPC_URL"
```

Compare the returned sequence with an earlier successful check. A response
proves the RPC answered, not that transaction submission or event history is
healthy.

### Horizon

```bash
curl --fail-with-body --silent --show-error \
  "$STELLAR_HORIZON_URL/payments?order=desc&limit=1"
```

Record the HTTP result and any paging or ledger information. An empty result
set is not an outage.

### Friendbot

```bash
curl --fail-with-body --silent --show-error \
  "https://friendbot.stellar.org/?addr=<YOUR_TESTNET_G...>"
```

Use one of your own throwaway testnet addresses. Success returns the funded
transaction; a rate-limit response is expected behaviour, not an outage —
Friendbot is rate-limited per address
(`../examples/integration-test-harness-notes.md`). Probing it funds a real
testnet address, so do not point it at an address you are watching.

### Stellar Expert

```bash
curl --silent --show-error -o /dev/null -w '%{http_code}\n' \
  "https://stellar.expert/explorer/testnet"
```

An HTTP 200 proves only that a page was served. A missing page can indicate a
stale ID, a reset, indexing delay, or an explorer limitation — it is not a
contract-health verdict (`explorer-links.md`). Never treat the explorer as
authoritative telemetry: it is a drill-down surface.

## Reading the results

The value of checking all four is the combination, not any single probe:

| Observation | Interpretation | Where to go |
|---|---|---|
| All four fail | Upstream-wide problem, or the local network | Stellar status page, then `../runbooks/rpc-outage.md` |
| RPC down, Horizon up | Contract operations blocked; classic payments fine | Pause contract automation; follow `../runbooks/rpc-outage.md` |
| Horizon down, RPC up | Classic balance/payment reads degraded; contract calls unaffected | Note for payment-watching only |
| Status page reports an incident | Treat as upstream until your own probe says otherwise | `../runbooks/rpc-outage.md`, step 3 |
| All probes pass, but our call fails | Likely **us** — configuration, contract, or passphrase | `wrong-network-passphrase.md`, `contract-not-found-error.md` |
| Explorer stale, RPC healthy | Indexing lag, not an outage | Do not page anyone; retry the explorer later |
| Friendbot failing only | New account funding blocked; existing operations unaffected | Fund from an existing account, retry later |

The last two rows matter as much as the first: both are common false
attributions. An explorer that has not indexed yet is not a deployment that
broke, and a faucet at its rate limit is not a network incident.

## What this page does not cover

- **The Soroban/SDK toolchain.** Version-pairing and advisory triage live in
  [`../security/dependency-audit-notes.md`](../security/dependency-audit-notes.md).
- **Secondary explorers.** Lumenscan appears in `explorer-links.md` as a
  secondary view whose availability is not guaranteed by this repository. The
  same caveat as Stellar Expert applies.
- **Community channels** (chat, forums). They are not operational
  dependencies of the deployment and are not tracked here.

## Maintaining this page

- Update only after recording the evidence that made the change — a probe
  result, a status-page advisory, and the UTC time and ledger it was observed
  at.
- If a dependency adds a dedicated status page, record the exact URL; otherwise
  leave "no separate status page documented" in place rather than guessing one.
- If an endpoint from `../docs/network-config.md` changes, update both files in
  the same change.

## Related documents

- [`../runbooks/rpc-outage.md`](../runbooks/rpc-outage.md) — handling an
  upstream outage, and the pause/resume procedure
- [`explorer-links.md`](explorer-links.md) — explorer link construction and
  their limits
- [`../docs/network-config.md`](../docs/network-config.md) — endpoints and
  network passphrase
- [`../registry/verify-contract-live.md`](../registry/verify-contract-live.md)
  — distinguishing a stale ID from an outage
- [`../security/dependency-audit-notes.md`](../security/dependency-audit-notes.md)
  — SDK/CLI version pairing and advisory triage
- [`README.md`](README.md) — monitoring status and inventory
