# Testnet Config Changelog

Changes to files under `testnet/config/` only. For the wider testnet docs see `testnet/docs/changelog.md`.

Add the newest entry at the top. Call out renamed or removed fields explicitly, since integrators copy these templates.

## Entry format

```
## YYYY-MM-DD
- <file>: <what changed> (<breaking | non-breaking>)
```

## Log

## 2026-10-16
- `CHANGELOG-config.md`: added this changelog (non-breaking)

## 2026-09-24
- `nonce-tracking.md`: added guide to reading the SweepController nonce (non-breaking)
- `gas-free-claim-testing.md`: added guide to testing the gas-free `claim()` path (non-breaking)
- `expiry-ledger-testing.md`: added guide to computing a near-future `expiry_ledger` (non-breaking)
