# Testnet Pre-Flight Checklist

Walk through this before running integration tests. Most "contract bugs" found at this stage are environment problems.

- [ ] **Identity funded.** `stellar keys fund <identity> --network testnet`, then confirm the balance on Horizon.
- [ ] **Passphrase exact.** `Test SDF Network ; September 2015` (spaces around `;`).
- [ ] **Contract IDs current.** Copy from `deployments/testnet.json`. Testnet resets wipe contracts.
- [ ] **RPC reachable.** The command below returns `"status":"healthy"`.
- [ ] **Horizon reachable.** `curl -s https://horizon-testnet.stellar.org | head -c 200`
- [ ] **Authorized signer matches.** Your signing key is the Ed25519 key the `SweepController` was initialized with.

```bash
curl -s -X POST https://soroban-testnet.stellar.org \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}'
```

## If a check fails

| Symptom | Runbook |
|---|---|
| `tx_bad_auth` | `testnet/runbooks/wrong-network-passphrase.md` |
| Contract not found | `testnet/runbooks/contract-not-found-error.md` |
| RPC errors or timeouts | `testnet/runbooks/rpc-outage.md` |
