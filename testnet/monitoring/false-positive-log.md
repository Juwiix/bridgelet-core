# Testnet Alert False-Positive Log

## Status

> **Log format only.** No collector, evaluator, dashboard, or alert destination
> is active in this repository. There are no alerts, therefore no entries, and
> that is the correct state of this document.

This document defines how false-positive alerts will be recorded once the
thresholds proposed in [`alert-thresholds.md`](alert-thresholds.md) are
implemented. It is written ahead of the implementation so the log exists on the
day the first alert fires, not three months later when nobody remembers why the
threshold was set the way it was.

## Why a log rather than one-time tuning

A threshold is a guess about what "normal" looks like, made before any
data about normal exists. [`alert-thresholds.md`](alert-thresholds.md) is
explicit that its numbers are "an initial operating threshold, not a
statistically validated SLO" and that they must be calibrated "after at least
seven days of representative data."

Nothing in that calibration happens automatically. Without a written record of
which alerts fired and why, the only input to threshold tuning is memory, and
the predictable outcome is a threshold that is set once and never revisited.
The log exists to make tuning evidence-driven: every false positive is a data
point about where the real boundary sits.

A false-positive log is also the honest counterpart to the alert policy. An
alerting setup with no false-positive record is indistinguishable from one that
has never fired — or from one whose noise is silently being ignored. Ignoring
an alert because "it always does that" is a threshold failure that the log
makes visible.

## Append-only discipline

| Rule | Rationale |
|---|---|
| Entries are never edited and never deleted | The history is the dataset |
| A wrong conclusion is corrected by a new entry that supersedes the old one | The wrong conclusion is itself evidence about how the alert misleads |
| False positives are entries, not omissions | They set the base rate; an untuned threshold is only as good as its false-positive record |
| Closed classification vocabulary | Free-text causes cannot be counted or compared |
| One entry per firing event | Bursts are counted, not merged away |

The log records what happened. The policy change that follows is a separate,
reviewable change to the threshold — linked from the entry, not folded into it.

## Entry schema

| Field | Required | Notes |
|---|---|---|
| Entry ID | yes | `FP-0001`, monotonically increasing, never reused; also the target of `supersedes` |
| Observed at (UTC) | yes | `YYYY-MM-DDTHH:MM:SSZ` |
| Alert name | yes | The signal, e.g. `expiry_to_sweep_breach`, `sweep_completion_silence` |
| Window | yes | `window_start` and `window_end` ledger and time from the alert payload |
| Signal values | yes | The counts and ratio that tripped it (`E`, `S`, ratio), or `unknown` |
| Why it fired | yes | The observation, in one or two sentences. **Not the conclusion.** |
| Classification | yes | One value from the vocabulary below. **No free text.** |
| Root-cause evidence | yes | What the investigation found, or `inconclusive` with what was ruled out |
| Tuning action | yes | `none`, or a link to the threshold change (PR/issue) |
| Threshold version | yes | Commit or date of the threshold in force when it fired |
| Supersedes | no | Entry ID this entry corrects or reclassifies |
| Logged by | yes | GitHub handle or `unassigned` |

## Classification vocabulary

| Classification | Meaning | Typical tuning signal |
|---|---|---|
| `threshold-too-tight` | Real signal, but the threshold sits inside normal variation | Raise the ratio or extend the sustain window |
| `window-too-short` | Burst activity broke the lookback or sustain assumption | Lengthen trailing lookback or sustain duration |
| `low-sample-noise` | Fired on too few events; the minimum-sample guard was wrong or missing | Raise the minimum sample |
| `data-quality-fault` | RPC error, cursor lag, skipped ledger, or decode failure misread as a condition | Fix the data-quality distinction, not the ratio |
| `low-traffic-period` | Zero or near-zero events during an idle period | Confirm idle-vs-outage handling; consider suppression |
| `testnet-reset` | A reset invalidated the baseline the threshold assumed | Add reset-epoch handling to the evaluator |
| `external-abuse` | The pattern was produced by known griefing, not a contract problem | Record as expected; tune only if it is frequent |
| `metric-definition-error` | The signal counted the wrong thing (e.g. `claim()` treated as funds received) | Fix the metric definition |
| `maintenance-or-demo` | Planned demo or test activity distorted the window | Mark known activity; consider a quiet period |
| `inconclusive` | Investigated and the cause could not be established | Real answer, not a failure; keep the evidence |
| `not-a-false-positive` | The alert was correct; reclassifies an earlier entry via `supersedes` | No tuning; the threshold worked |

`inconclusive` and `not-a-false-positive` exist so the log never forces a
false certainty. The first is an honest dead end; the second is the
reclassification path when an investigation later proves the alert was right.

## What counts as a false positive

An entry is warranted when an alert fired and the investigation established
that the firing did **not** correspond to the condition the threshold exists to
detect — or that it corresponded to a condition that required no action.

Log it when:

- Someone investigated an alert and found nothing to act on.
- An alert fired during a known idle period, reset, or demo.
- The alert revealed a data-quality problem rather than a health problem.
- An alert was correct but caused by someone else's griefing, not by the
  contracts or the deployment.

Do **not** log it here when:

- The alert never fired. Absence of alerts is not a false positive; with no
  implementation there is nothing to log.
- The alert correctly identified a real condition. That belongs in an incident
  record (`../runbooks/incident-postmortem-template.md`), not the tuning log.
- The cause was a genuine contract regression. Also an incident, not a
  false positive.

## Entries

| Entry ID | Observed at (UTC) | Alert name | Classification | Supersedes |
|---|---|---|---|---|
| — | — | — | — | — |

**No entries.** No alert evaluator or notifier exists in this repository, so no
alert has fired. The table above is the format, not a result.

## Entry template

Copy this block into the table, or use it as the checklist. Every `required`
field from the schema must be filled; `unknown` is an acceptable value, silence
is not.

```markdown
| FP-0000 | {{YYYY-MM-DDTHH:MM:SSZ}} | {{alert name}} | {{window start–end ledger/time}} | {{E, S, ratio or unknown}} | {{the observation that tripped it, no conclusion}} | {{one value from the vocabulary}} | {{what the investigation found}} | {{none or link}} | {{threshold commit/date}} | {{entry ID or none}} | {{GitHub handle or unassigned}} |
```

## Worked example

> ### Example only — not a record
>
> Everything below is **synthetic**, invented to show the format. It is not a
> claim that any alert fired. It is marked as an example so it is never counted
> as evidence about the deployment.

```markdown
| FP-0001 | 2026-10-02T09:15:00Z | expiry_to_sweep_breach | ledgers 1000000–1000600 | E=24, S=8, ratio=3.0 | Ratio exceeded 2.0 for 20 minutes | maintenance-or-demo | Matching `created`/`payment` events came from a scheduled demo batch; no real expiry backlog | none | 2026-09-26 | none | example-operator |
```

The entry records the observation in the signal fields and the conclusion in
the root-cause field, in that order — the same discipline the reentrancy
observation log uses. A reader can check the counts independently of the
explanation.

## How to file an entry

1. Confirm the alert is a false positive by the definition above, not simply
   inconvenient.
2. Gather the alert payload fields listed in
   [`alert-thresholds.md`](alert-thresholds.md) (`window_start`, collector
   cursor, `E`, `S`, contract IDs) and the raw evidence.
3. Assign the next `FP-` ID and append one row. Do not renumber.
4. Pick exactly one classification. If none fits, that is a gap in the
   vocabulary worth raising, not a reason to invent free text.
5. If this entry overturns an earlier one, set `supersedes` to that entry ID
   and leave the earlier row untouched.

## From entries to threshold changes

Tuning is a reviewable change to [`alert-thresholds.md`](alert-thresholds.md),
not an edit to the log:

1. Re-read the entries for a classification and check whether it is a pattern.
2. Propose the smallest change that removes the pattern (a higher ratio, a
   longer sustain window, a higher minimum sample, or a data-quality fix).
3. Record the change and the entry IDs that motivated it in the threshold's
   own history, and link the change from each affected entry's tuning action.
4. Keep the previous threshold version in the entries that predate the change.
   They were accurate against the threshold in force at the time.

The baseline-and-activation checklist in `alert-thresholds.md` is the gate for
first activation; this log is the loop that runs after it.

## Related documents

- [`alert-thresholds.md`](alert-thresholds.md) — the thresholds this log tunes
- [`README.md`](README.md) — monitoring status and inventory
- [`../runbooks/incident-postmortem-template.md`](../runbooks/incident-postmortem-template.md)
  — for alerts that were real
- [`../security/reentrancy-observation-log.md`](../security/reentrancy-observation-log.md)
  — the append-only log pattern this document follows
- [`../registry/event-topics.md`](../registry/event-topics.md) — the events the
  signals are built from
