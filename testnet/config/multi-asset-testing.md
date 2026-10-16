# Multi-Asset Sweep Testing on Testnet

`EphemeralAccount` accepts up to 10 distinct assets, and `SweepController::execute_sweep` moves them atomically. Testing this needs more than native XLM.

## Assets you can use

| Asset | How to get its contract ID |
|---|---|
| Native XLM | `stellar contract id asset --asset native --network testnet` |
| Your own test token | Issue a classic asset, then wrap it (below) |

The native XLM contract ID is also listed in `testnet/docs/network-config.md`.

## Issue and wrap a test token

```bash
stellar keys generate --global testnet-issuer && stellar keys fund testnet-issuer --network testnet
ISSUER=$(stellar keys address testnet-issuer)
stellar contract asset deploy --asset TESTUSD:$ISSUER --source testnet-issuer --network testnet
stellar contract id asset --asset TESTUSD:$ISSUER --network testnet
```

The last command prints the SEP-41 contract ID to pass to `record_payment`.

## What to assert

- One payment per asset: a repeat for the same asset returns `DuplicateAsset` (code 13).
- An 11th distinct asset returns `TooManyPayments` (code 14).
- See `testnet/runbooks/multi-payment-edge-cases.md` for full cases.
