# Wallet Compatibility for Manual Testnet QA

This page tracks wallets used to sign `authorized_controller` or recipient auth entries by hand. Only add a wallet as **Confirmed** after you have run the test flow below with it.

| Wallet | Testnet support | Status for Bridgelet | Notes |
|---|---|---|---|
| Freighter (browser extension) | Has a network selector with Testnet | Not yet confirmed | Switch the network to Testnet before signing |
| Others (LOBSTR, xBull, etc.) | Check the wallet's own docs | Not yet confirmed | Do not assume testnet support |

## Confirmation test

1. Set the wallet to Testnet and fund its account with Friendbot.
2. Build a `claim()` or auth-entry transaction (see `testnet/config/gas-free-claim-testing.md`).
3. Sign with the wallet and submit.
4. Record: wallet name and version, result, and any error text.

## Common false alarms

- Wallet is on the wrong network: transaction fails with `tx_bad_auth` (`testnet/runbooks/wrong-network-passphrase.md`).
- Account is not funded: the source account is not found on testnet.
- Contract ID is stale after a reset (`testnet/runbooks/contract-not-found-error.md`).

## Results

| Wallet | Version | Date | Result | Notes |
|---|---|---|---|---|
| (none recorded yet) | | | | |
