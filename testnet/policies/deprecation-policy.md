# Superseded Testnet Contract ID Deprecation Policy

> **The short version, stated up front:** a superseded testnet contract ID stays
> documented as "previously live" for at least **90 days** after it is
> superseded, then it is deprecated and removed from active documentation.
> The 90 days is a floor, not a ceiling — an ID still referenced by an open
> issue or incident keeps its full entry until that reference closes.

## Purpose

Testnet contract IDs are superseded constantly: a reset wipes every ID at once,
and a targeted redeploy replaces a subset. Each supersession leaves two
questions the registry has never answered:

1. **How long should the old ID still be recognisably "the contract that was
   live"?** Someone reading a three-week-old issue, a stale transaction link, or
   a recorded demo needs to map an old ID to the deployment it belonged to.
2. **When does keeping it stop helping and start being cruft?** A registry that
   accumulates every ID ever used becomes harder to read, and a long list of
   "previously live" IDs is an invitation to mistake one of them for a live one.

This policy answers both with one number, so the decision is made once rather
than case-by-case by whoever happens to be curating the registry.

## Status and verification

The policy is being written ahead of its first real use. As of 2026-09-26 no
Bridgelet contracts are deployed to testnet, and every "current ID" in the
repository is an unverified record from an earlier deployment
(`reset-policy.md`). The archive file this policy depends on,
`testnet/reset/historical-deployments.md`, does not exist yet; it is planned as
the archive of superseded ID generations. Until it exists, a superseded ID has
no active home, and this policy is prospective.

The first real supersession will be the first test of the 90-day window. Treat
the number as a calibrated default, not a validated one — the same standing as
the thresholds proposed in `../monitoring/alert-thresholds.md`.

## Scope

**In scope:** testnet contract IDs (the `C...` Strkeys of deployed contracts)
that have been superseded, and the lifecycle that moves them from "current" to
"deprecated".

**Explicitly out of scope:**

| Not covered | Governed by |
|---|---|
| Mainnet contract IDs | Nothing here — this is a testnet policy |
| Contract code versioning and WASM-hash history | `versioning-scheme.md`, `../registry/upgrade-history.md` |
| Notice periods for breaking redeploys | `breaking-change-notice.md` |
| The reset procedure itself | `reset-policy.md` |
| Retention of data written to testnet | `data-retention.md` (permanence, not duration) |
| Which specific IDs are live | `../registry/contract-status.md` |

## Why a duration is needed at all

`data-retention.md` establishes that anything written to testnet is public and
permanent — on-chain data generally cannot be deleted. This policy is not about
deleting anything from the network. It is about what the **documentation**
continues to present as meaningful.

`../reset/state-export-before-reset.md` keeps exports because they map old IDs
mentioned in issues to a deployment, and notes that a maintainer may prune old
ones. That "may prune, eventually" is the gap this policy closes: without a
stated window, pruning is either never done (cruft) or done inconsistently
(inconsistency).

## Lifecycle stages

| Stage | Where the ID lives | What it means |
|---|---|---|
| **Current** | `../registry/contract-status.md` current table | Presented as the candidate live ID, verified or not |
| **Superseded — previously live** | `testnet/reset/historical-deployments.md`, full entry | Known to have been live; not current; retained for reference |
| **Deprecated — archived** | One-line index entry, or removed from the active file | No longer presented as relevant; permanent record is git history, the changelog, and state exports |

A transition to **Superseded** happens the moment an ID is replaced — by a
redeploy or a reset. A transition to **Deprecated** happens when the retention
window expires with no reason to extend it.

## The retention window

> **Default: 90 calendar days as "previously live", measured from the date the
> ID was superseded.**

The date recorded in `historical-deployments.md` is the supersession date, not
the date the entry was written, so a late-recorded entry does not restart its
own window.

### Why 90 days

The window has to be long enough that the common reasons to look up an old ID
are still covered, and short enough that the registry does not become a
graveyard.

- **It covers the working lifetime of a report.** An issue filed against a live
  ID, a transaction link in a demo, or a postmortem is typically resolved or
  abandoned within a quarter. 90 days covers a full cycle.
- **It is three times the registry's own staleness threshold.** The registry
  treats a `last-verified` timestamp older than 30 days as stale
  (`../registry/contract-status.md`). An old ID is meaningfully "recent" for
  roughly three such cycles.
- **It is long relative to redeploy cadence.** Redeploys are infrequent and
  resets are periodic but not weekly, so 90 days spans more than one generation
  without accumulating many.
- **It is short enough to matter.** A number in the hundreds of days would let
  the "previously live" list grow without bound between prunes, which is the
  cruft problem this policy exists to solve.

### Why not shorter or longer

A 30-day window would expire IDs while an unresolved issue still referenced
them. A window of a year or more would make "previously live" a second current
list. 90 days is a deliberate middle that can be revised with evidence after
the first few supersessions.

### The window is a minimum, not a maximum

The 90 days is the **floor** for retention, mirroring the way
`breaking-change-notice.md` treats its notice periods. An ID is retained longer
when there is a reason:

- **Open references.** While an open issue, incident, or postmortem names the
  ID, it stays in the full "previously live" form regardless of age. Removal is
  a documentation decision, not a way to hide a live reference.
- **Security significance.** An ID involved in an incident, a postmortem, or a
  catalogued abuse pattern is exempt from removal and kept permanently with a
  note saying why.

## What "removed from active documentation" means

Deprecation does **not** delete the record. It means the ID is no longer
presented as a candidate live ID:

- It is removed from `../registry/contract-status.md` (it should already have
  left the current table at supersession) and from any active ID list.
- In `testnet/reset/historical-deployments.md`, its full entry is either
  compressed to a single index line — ID, live window, replacement ID — or
  removed from the active file.
- The permanent record remains reachable without it: git history, the
  redeploy's `../docs/changelog.md` entry, and the state export under
  `testnet/reset/exports/<UTC date>/`.

Anyone investigating an old hash should search git history and the changelog,
not the live registry.

## Interaction with a reset

A testnet reset supersedes **every** ID in a generation at once. The whole
generation shares one supersession date and therefore one 90-day window, and is
recorded as a single generation in `testnet/reset/historical-deployments.md`.
The reset's changelog entry links the state export, as
`../reset/state-export-before-reset.md` already requires.

## What is not decided

1. **Whether 90 days is the right number.** It is a calibrated default; the
   first real supersessions may argue for a different figure.
2. **Whether deprecated entries are compressed or fully removed.** Both are
   permitted until a first use shows which reads better.
3. **Whether the archive keeps a permanent one-line index** or relies entirely
   on git history and the changelog for the deprecated tail.
4. **Who performs the archival.** No maintainer is named; this is the same
   limitation `reset-policy.md` records for resets generally.

## What would count as fixing this

- A stated window that the registry curator actually applies, not a default
  that is ignored in practice.
- Evidence from at least one real supersession that 90 days was long enough —
  or a revised number if it was not.
- A named owner for the archival step, so deprecation happens on a schedule
  rather than opportunistically.

## Related Documentation

- [`reset-policy.md`](reset-policy.md) — when and how the shared deployment is
  wiped and redeployed
- [`breaking-change-notice.md`](breaking-change-notice.md) — notice periods for
  breaking redeploys; the pattern this policy's floor-and-ladder follows
- [`versioning-scheme.md`](versioning-scheme.md) — contract versioning, which is
  about code, not IDs
- [`data-retention.md`](data-retention.md) — why on-chain data is permanent,
  and what "cleanup" can and cannot mean
- [`../reset/state-export-before-reset.md`](../reset/state-export-before-reset.md)
  — the export that maps old IDs to a deployment
- [`../registry/contract-status.md`](../registry/contract-status.md) — the
  current-ID registry this policy keeps clean
- [`../registry/upgrade-history.md`](../registry/upgrade-history.md) — in-place
  upgrades, which keep the same ID and so do not supersede one
- [`../registry/verify-contract-live.md`](../registry/verify-contract-live.md)
  — confirming whether an ID is live before treating it as current
- [`../docs/changelog.md`](../docs/changelog.md) — the redeploy history that
  survives deprecation

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial deprecation policy; proposes a 90-day "previously live" floor with reference-based extensions |
