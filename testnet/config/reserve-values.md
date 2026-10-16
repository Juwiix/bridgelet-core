# Which Base Reserve Value Is In Effect on Testnet

There are two independent reserve values. Only one affects `EphemeralAccount`.

| Source | Value | Used by `EphemeralAccount`? |
|---|---|---|
| `BASE_RESERVE_STROOPS` in `contracts/ephemeral_account/src/lib.rs` | `1_000_000_000` stroops (100 XLM) | **Yes**. Passed to `init_reserve_tracking` at `initialize()` |
| `ReserveContract` (admin-set via `set_base_reserve`) | Whatever the admin last stored | **No**. Nothing in `EphemeralAccount` reads it |

1 XLM = 10,000,000 stroops, so `1_000_000_000` stroops is 100 XLM.

## What this means for testing

- Calling `ReserveContract::set_base_reserve` does **not** change reserve-reclaim behavior of any `EphemeralAccount`.
- To check the value a given account is using, call `get_reserve_remaining` or `get_reserve_available` on it. A fresh account reports `1_000_000_000`.
- Changing the effective value requires a code change to `EphemeralAccount` and an upgrade; it is not a testnet config option.

## Related

- `testnet/registry/reserve-contract-status.md`
- `testnet/runbooks/insufficient-reserve-errors.md`
