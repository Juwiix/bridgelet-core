# Testnet Contract Status Registry

## Overview

This is the canonical quick-reference table for Bridgelet contract deployment status on Stellar Testnet. Updated whenever someone verifies testnet state. Pair with `testnet/registry/last-verified.json` for automated staleness checks.

---

## Contract Deployment Status Table

| Contract | Deployed | Contract ID | WASM Hash | Last Verified | Caveats |
|---|---|---|---|---|---|
| EphemeralAccount | ❌ No | TBD | TBD | Never | Not yet deployed; see deploy scripts |
| SweepController | ❌ No | TBD | TBD | Never | Not yet deployed; see deploy scripts |
| ReserveContract | ❌ No | N/A | N/A | Never | No testnet presence or wiring |
| AccountFactory | ❌ No | N/A | N/A | Never | Real but entirely undocumented/undeployed |

> **Legend**: ✅ = Deployed and verified live; ❌ = Not deployed; ⚠️ = Deployed but issues; TBD = To be determined

---

## Detailed Status by Contract

### EphemeralAccount
- **Status**: Not deployed
- **Blocker**: `scripts/deploy-testnet.sh` needs fixes (unbound `RESERVE_CONTRACT_ID` variable)
- **WASM**: Built by `scripts/build.sh` (includes `ephemeral_account`, `sweep_controller`, `reserve_contract`)
- **Dependencies**: None (standalone)
- **Next Steps**: Fix deploy script → Deploy → Verify with `verify-contract-live.md` → Update this table

### SweepController
- **Status**: Not deployed
- **Blocker**: Same as EphemeralAccount (deploy script issues)
- **WASM**: Built by `scripts/build.sh`
- **Dependencies**: Requires `authorized_signer` (Ed25519 pubkey) and optional `authorized_destination`
- **Next Steps**: Deploy after EphemeralAccount → Configure signer → Verify

### ReserveContract
- **Status**: Not deployed
- **Blocker**: Not included in `scripts/deploy-testnet.sh` at all
- **WASM**: Built by `scripts/build.sh`
- **Integration**: **Not wired** - EphemeralAccount uses hardcoded `BASE_RESERVE_STROOPS` (1 XLM), does not read from ReserveContract
- **Gap**: Two separate issues: (1) not deployed, (2) no cross-contract wiring
- **Next Steps**: Add to deploy script → Deploy → Decide on wiring → Update EphemeralAccount if needed

### AccountFactory
- **Status**: Not deployed
- **Blocker**: Not built by `scripts/build.sh`; not in CI; not in deploy script
- **WASM**: Must build manually: `cd contracts/account_factory && cargo build --target wasm32-unknown-unknown --release`
- **Dependencies**: Requires `ephemeral_account.wasm` hash at `initialize()`
- **Known Issue**: `batch_initialize` swallows per-account error detail (`error: None`)
- **Next Steps**: Add to build script → Add to CI → Add to deploy script → Deploy → Document

---

## Deployment Checklist

When deploying contracts to testnet, update this table and the corresponding status files:

- [ ] **EphemeralAccount deployed**
  - Contract ID recorded
  - WASM hash recorded
  - Verified with `verify-contract-live.md` procedure
  - `last-verified.json` updated

- [ ] **SweepController deployed**
  - Contract ID recorded
  - WASM hash recorded
  - `authorized_signer` configured
  - Verified with `verify-contract-live.md`
  - `last-verified.json` updated

- [ ] **ReserveContract deployed** (optional)
  - Contract ID recorded
  - WASM hash recorded
  - `admin` set
  - `base_reserve` configured
  - Verified with `verify-contract-live.md`
  - `last-verified.json` updated

- [ ] **AccountFactory deployed** (optional)
  - Contract ID recorded
  - WASM hash recorded
  - Initialized with `ephemeral_account` WASM hash
  - Verified with `verify-contract-live.md`
  - `last-verified.json` updated

---

## Cross-References

| Document | Purpose |
|---|---|
| `testnet/registry/last-verified.json` | Machine-readable timestamps for staleness checks |
| `testnet/registry/wasm-hash-reference.md` | WASM hash verification |
| `testnet/registry/verify-contract-live.md` | Health check procedures |
| `testnet/registry/event-topics.md` | Event topics for indexers |
| `testnet/registry/admin-addresses.md` | Admin/controller addresses on live contracts |
| `testnet/registry/upgrade-history.md` | Contract upgrade log |
| `testnet/registry/reserve-contract-status.md` | ReserveContract detailed status |
| `testnet/registry/account-factory-status.md` | AccountFactory detailed status |
| `testnet/registry/dependency-graph.md` | Contract call graph |
| `testnet/registry/known-test-accounts.md` | Long-lived test accounts for demos |

---

## Update Process

1. **Verify**: Run health checks per `verify-contract-live.md`
2. **Record**: Update this table with Contract ID, WASM Hash, Date, Caveats
3. **Sync**: Update `last-verified.json` with ISO timestamps
4. **Notify**: Commit changes; CI/bot can detect staleness (>30 days)

---

## Quick Reference (added 2026-10-16)

The table above reflects 2026-09-24. Newer sources:

- `scripts/deploy-testnet.sh` now deploys and initializes all four contracts.
- `deployments/testnet.json` (`deployedAt` 2026-10-10T04:38:14Z, updated in commit `e7efd55`) lists an ID for each.

| Contract | ID in `deployments/testnet.json` |
|---|---|
| EphemeralAccount | `CB7Z22TXR6ZKG7MDXIGJL6QVQQ3L3DQOFR6JV325NZK4CTYWFPVZBBP3` |
| SweepController | `CAJB4RRM5JRPXLITC5JUA7Q3LU2WQJDRPXVKMORXWHJJBPZRYWDJ5V4S` |
| ReserveContract | `CBG3Y4GWFTL4D37JGV2XWAWC7EUQ7XGH4XM3SFLQYYSKPB2IYMAS54HY` |
| AccountFactory | `CDBY4L3NTLW4OBZZTY73VL3NNWO2AGFFGBRJVDL3IMKDDLFF2W57WK6H` |

`testnet/docs/network-config.md` lists different IDs for the last three, from an earlier deploy. Prefer the JSON file. Mark a contract as deployed in the table only after `verify-contract-live.md` passes for it.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-24 | Initial testnet contract status registry |
| 1.1 | 2026-10-16 | Added quick reference |
