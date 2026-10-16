# Weekly Testnet Health Summary Template

## Purpose

This is a template for a short, recurring manual health summary of the shared
testnet deployment: sweep success rate, incidents, and registry freshness.

It exists because **no automated monitoring is implemented in this
repository** (see [`README.md`](README.md)). Everything proposed elsewhere in
`testnet/monitoring/` — the collector, alert evaluator, dashboard, and uptime
check — is a plan, not a running service. Until one of those exists, a manual
summary on a fixed cadence is the only way a regression gets noticed between
demos.

The cadence is a suggestion, not a commitment. Weekly is frequent enough that a
quietly broken deployment is caught within a week, and cheap enough that it
survives a busy period. The point is the recurrence, not the period.

Do not present a completed summary as evidence that monitoring exists. The
status in `README.md` remains `testnet_monitoring: not_configured` until the
activation checklist there is complete.

## How to use this template

- Copy the sections below into one document per period. Do not edit this
  template in place.
- Record all times in UTC.
- Use `unknown` instead of guessing. An explicit gap is more useful than a
  plausible number that nobody can trust.
- Keep the first pass short. If a section turns up something significant, that
  becomes an incident record
  ([`../runbooks/incident-postmortem-template.md`](../runbooks/incident-postmortem-template.md))
  rather than a paragraph here.
- The summary is an operational record. This repository has no committed
  location for them; keep the completed copy with the operator's own records,
  and open an issue for anything that needs to be visible in the repository.

## Summary record

- **Period:** `{{YYYY-MM-DD}} to {{YYYY-MM-DD}}`
- **Prepared by:** `{{GitHub handle or unassigned}}`
- **Prepared at:** `{{YYYY-MM-DDTHH:MM:SSZ}}`
- **Deployment state:** `{{deployed | partially deployed | nothing deployed | unknown}}`
- **Testnet reset observed this period?:** `{{yes | no | unknown}}`

## Deployment and registry freshness

_Is the registry still an accurate description of what is live?_

- Active contract IDs and WASM hashes last verified:
  `{{YYYY-MM-DD or never}}` (`../registry/contract-status.md`)
- Any entry older than the 30-day staleness threshold?:
  `{{yes — which | no | unknown}}`
- `last-verified.json` age: `{{days}}`
- Contract upgrade observed this period?:
  `{{yes — see ../registry/upgrade-history.md | no | unknown}}`
- Read-only verification run this period?
  ([`../registry/verify-contract-live.md`](../registry/verify-contract-live.md)):
  `{{yes | no}}`
- Notes: `{{anything that makes the registry suspect}}`

## Sweep health

_The manual baseline described in [`README.md`](README.md), bounded to the
period's finalized ledgers. If attempts cannot be enumerated, report the event
counts and mark the rate `unknown` rather than estimating._

- Ledger range queried: `{{start}}–{{end}}` (finalized)
- Successful sweeps (`SweepCompleted` / topic `sweep`): `{{count}}`
- Expirations (`AccountExpired` / topic `expired`): `{{count}}`
- Account creations (`created`): `{{count}}`
- Observed success rate: `{{count/count, or unknown}}`
- Accounts past `expiry_ledger` with no `expired` event: `{{count or unknown}}`
- Notes on anomalies: `{{free text}}`

> A `sweep` event proves a controller completion event, not by itself a token
> transfer, and failures are transaction failures that emit no event. The
> counts above are event counts, not a definitive success rate. Say so in the
> summary rather than presenting the rate as authoritative.

## Incidents

_One row per anything that needed attention. A significant incident gets its
own postmortem; link it from here._

| Date (UTC) | What happened | User impact | Postmortem / issue |
|---|---|---|---|
| `{{YYYY-MM-DD}}` | `{{short description}}` | `{{none / degraded / single-integration}}` | `{{link or none}}` |

If there were none: `No incidents recorded this period.`

## Upstream dependencies

_Did anything upstream degrade? Check [`third-party-dependency-status.md`](third-party-dependency-status.md)
and the [Stellar status page](https://status.stellar.org/)._

- Upstream issues observed: `{{yes — which | no | unknown}}`
- Did they affect this deployment?: `{{yes | no}}`
- Notes: `{{short}}`

## Alerts and false positives

- Alerting active this period?: `{{no — not implemented | yes}}`
- False positives filed ([`false-positive-log.md`](false-positive-log.md)):
  `{{entry IDs or none}}`

If alerting is not active, record `not implemented` — do not leave this blank,
because a blank field reads like "no alerts fired" rather than "no alerting
exists."

## Data quality and gaps

_What could not be observed this period, and why. This is the section that
keeps the summary honest._

- Readings that could not be taken: `{{list or none}}`
- Suspected data-quality problems (gaps, cursor lag, decode errors):
  `{{list or none}}`
- Confirmed versus inferred: `{{note any number above that is inferred}}`

## Follow-ups

| Action | Owner | Target date | Status |
|---|---|---|---|
| `{{specific action}}` | `{{owner or unassigned}}` | `{{date or none}}` | `open` |

An explicitly unassigned action is better than inventing an owner.

## Related documents

- [`README.md`](README.md) — monitoring status, inventory, and the manual
  baseline procedure
- [`alert-thresholds.md`](alert-thresholds.md) — the proposed alerting this
  summary is an interim substitute for
- [`third-party-dependency-status.md`](third-party-dependency-status.md) —
  upstream triage reference
- [`false-positive-log.md`](false-positive-log.md) — false-positive records
- [`../runbooks/incident-postmortem-template.md`](../runbooks/incident-postmortem-template.md)
  — for incidents worth recording in full
- [`../registry/verify-contract-live.md`](../registry/verify-contract-live.md)
  — the read-only check behind registry freshness
- [`../registry/event-topics.md`](../registry/event-topics.md) — the topics
  counted in the sweep-health section
