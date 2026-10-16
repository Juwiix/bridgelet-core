# Known Testnet Ephemeral Accounts

## Overview

This document maintains a small set of long-lived, intentionally non-swept testnet `EphemeralAccount` instances for demo/documentation purposes. These accounts can be referenced in documentation, screenshots, and integration tests without disappearing when someone else sweeps or expires them.

## Account Registry

| Account ID | Purpose | Created | Expiry Ledger | Status | Notes |
|---|---|---|---|---|---|
| TBD | Demo account for screenshots | TBD | TBD | Pending deployment | Will be created after EphemeralAccount deployed |
| TBD | Integration test fixture | TBD | TBD | Pending deployment | Reserved for SDK test suite |
| TBD | Documentation walkthrough | TBD | TBD | Pending deployment | Used in CLI examples |

> **Note**: Accounts will be added once `EphemeralAccount` is deployed to testnet (see `testnet/registry/contract-status.md`).

## Creation Procedure

When EphemeralAccount is deployed, create known accounts using:

```bash
# 1. Create a dedicated identity for known accounts
stellar keys generate --global testnet-known-accounts
stellar keys fund testnet-known-accounts --network testnet

# 2. For each known account, deploy new EphemeralAccount instance
# Using AccountFactory (once deployed) or direct deploy:

# Direct deploy (requires WASM hash):
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/ephemeral_account.wasm \
  --network testnet \
  --source testnet-known-accounts

# 3. Initialize with far-future expiry (e.g., 10 years = ~63M ledgers)
CURRENT_LEDGER=$(stellar network current-ledger --rpc-url https://soroban-testnet.stellar.org)
EXPIRY=$((CURRENT_LEDGER + 63000000))

stellar contract invoke \
  --id <NEW_ACCOUNT_ID> \
  --network testnet \
  --source testnet-known-accounts \
  -- \
  initialize \
  --creator <KNOWN_ACCOUNTS_PUBLIC_KEY> \
  --expiry_ledger $EXPIRY \
  --recovery_address <KNOWN_ACCOUNTS_PUBLIC_KEY> \
  --authorized_controller <SWEEP_CONTROLLER_ID> \
  --admin <KNOWN_ACCOUNTS_PUBLIC_KEY>

# 4. Record a small test payment (optional, for PaymentReceived status)
stellar contract invoke \
  --id <NEW_ACCOUNT_ID> \
  --network testnet \
  --source testnet-known-accounts \
  -- \
  record_payment \
  --amount 1000000 \
  --asset <TEST_TOKEN_ID>
```

## Maintenance Rules

1. **Never sweep these accounts** - They are for documentation only
2. **Use far-future expiry** - Minimum 1 year, preferably 10+ years
3. **Document in this file** - Update table immediately after creation
4. **Monitor status** - Periodically verify they remain Active/PaymentReceived
5. **Rotate if needed** - If an account is accidentally swept, create replacement

## Usage in Documentation

Reference these accounts in:
- CLI examples in `docs/api-reference.md`
- Screenshots in README and docs
- Integration test fixtures in `bridgelet-sdk`
- Walkthrough guides

Example reference:
> "See known account `GABC...` (created 2026-09-24, expires 2036) for a live `PaymentReceived` state example."

## Related Documentation

- `testnet/registry/contract-status.md` - Contract deployment status
- `testnet/registry/verify-contract-live.md` - Health check procedures
- `docs/api-reference.md` - CLI examples using known accounts

---

## Quick Reference (added 2026-10-16)

Before adding a row to the table:

1. Initialize the account with a far-future `expiry_ledger` (about 63 million ledgers is roughly 10 years at 5 seconds per ledger).
2. Never call `sweep`, `sweep_claim`, `expire` or `recover` on it.
3. Fund the creator identity separately, so the account's own balance is never used for other tests.
4. Add the row with the contract ID, creation date and expiry ledger.

A testnet reset deletes these accounts. After a reset, mark every row as gone and re-create them. See `testnet/docs/network-config.md` for reset details.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-24 | Initial known accounts registry template |
| 1.1 | 2026-10-16 | Added quick reference |
