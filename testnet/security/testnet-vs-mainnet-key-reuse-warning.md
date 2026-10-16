# ⚠️ Warning: Never Reuse a Mainnet-Intended Keypair on Testnet

> **The rule, stated once and without qualification:** never point a keypair
> that is intended for mainnet — or that holds, or may ever hold, real value —
> at testnet. Not "just to check", not "it's the same person", not once. Use a
> dedicated testnet key, always.
>
> Testnet XLM is worthless. The key is not.

## Why this document exists

`data-retention.md` already states the prohibition in its wallet-key rule, and
`admin-key-hygiene.md` already requires a dedicated testnet key. This document
is the focused warning for the specific, common, real-world mistake those two
cover only in passing: taking a keypair that was generated for mainnet and
running it against testnet for a quick experiment.

It is a common mistake because nothing technical stops it. A Stellar keypair is
the same format on both networks. The only difference is the network passphrase,
and a compatible tool will happily sign and submit with a mainnet key against
testnet. There is no guardrail to trigger — only a decision not to do it.

## What is actually at risk

Being precise about this matters, because overstating it trains people to
dismiss the warning.

| Thing | At risk? | Why |
|---|---|---|
| Testnet XLM on the key | **No** | It is worthless; losing it is not an incident |
| Funds held on a *different* mainnet key | **No** | The exposure is to the reused key, not to unrelated keys |
| **The reused key's secrecy** | **Yes** | Its exposure surface is now the testnet workflow's, not mainnet custody's |
| **Any value that key ever holds later** | **Yes** | Every past testnet exposure becomes a retroactive compromise |
| **Your habits and your team's habits** | **Yes** | Testnet tolerance for sloppy key handling is what leaks into mainnet work |

The first two rows are why the mistake feels harmless. The last three are why
it is not.

## The mechanics: how a mainnet key's secrecy degrades on testnet

Key handling on testnet is, correctly, more casual than on mainnet. That is
precisely the problem: pointing a mainnet key at testnet lets the mainnet key
inherit the testnet workflow's exposure surfaces, none of which were designed
to protect it.

- **Shell history and terminal logs** — a "quick test" with an exported `S...`
  secret or a signing seed on the command line records it on disk.
- **CI and deploy logs** — testnet tooling often prints or echoes environment
  values that mainnet tooling would never log.
- **Committed configuration** — secret scanners exist because this happens.
  A key committed "because it's only testnet" becomes a real incident the
  moment the same key is used on mainnet
  (`../examples/integration-test-harness-notes.md`).
- **Screen shares and demo recordings** — a public demo is an exposure surface.
- **Issue and chat pastes** — a testnet footprint is public, permanent, and
  outside the project's control (`../policies/data-retention.md`).

A key reused this way is no longer protected to mainnet standards, no matter
how carefully it is stored afterwards. The degradation has already happened.

## Why "it's only testnet" is backwards here

"It's only testnet" is a valid reason to be relaxed about *testnet* keys, whose
whole purpose is to be thrown away. It is exactly the wrong reason when the key
in question is a *mainnet* key, because the statement is about the environment
and the risk is about the key.

The asymmetry is the whole point:

- **Testnet key on mainnet** — a throwaway key holding value. Bad, and the
  usual mistake: the key should never have been given anything to lose.
- **Mainnet key on testnet** — a protected key placed into an unprotected
  workflow. Worse, because the damage is invisible until the key is used for
  its real purpose, at which point the exposure is already historical.

Both directions are wrong. This document is about the second, because it is the
one people talk themselves into.

## It applies to every key type, not just account keypairs

- **Stellar account keys** (`G...` public / `S...` secret) — the obvious case.
- **The `SweepController` sweep-signing key** — the Ed25519 seed used to sign
  sweep authorizations. It is signing-only and does not need to be a funded
  account, which makes it feel lower-stakes; it is not, if it is the seed
  intended to protect a mainnet deployment. Keep it dedicated and separate
  (`../tooling/sweep-signer-integration-notes.md`).
- **The deployer/admin key** — an `S...` secret that pays fees and can upgrade
  contracts. Covered in depth by `admin-key-hygiene.md`; the reuse prohibition
  applies to it too, in both directions.

## If you think you have already done this

1. Decide what the key is actually for.
2. **If it has never held value and never will** — retire it now, before it is
   used for anything, and generate a replacement. A key with a testnet history
   is not worth carrying forward as a mainnet key.
3. **If it holds value, or is intended to soon** — treat it as compromised,
   not as "needs care later". Move the value to a freshly generated key and
   stop using the exposed one entirely. Rotating *before* mainnet use is cheap;
   rotating *after* a leak is an incident.
4. Remove the key from any testnet configuration, CI variable, or local file it
   was placed in, and check git history rather than only the working tree.

## Rules

- [ ] Every environment has its own key. Testnet keys are generated for
      testnet and never promoted.
- [ ] A mainnet-intended key is never used, funded, or configured against
      testnet — not by tooling, not by hand, not "temporarily".
- [ ] Testnet tooling is never configured with a production secret, even for a
      read-only test.
- [ ] If a key's purpose changes, it is regenerated; keys are not repurposed
      across the mainnet/testnet boundary.
- [ ] The sweep-signer seed and the deployer/admin key are dedicated, and never
      the same key as each other or as any wallet.

## What this document is not

- It is not a mainnet key-management policy. It is a warning about one specific
  cross-environment mistake.
- It is not a claim that any testnet key has been compromised. It describes a
  practice to avoid, not an observed incident.
- It does not replace `admin-key-hygiene.md` (testnet admin key) or
  `data-retention.md` (the wallet-key rule); it points at the same boundary
  from the mainnet-keyholder's direction.

## Related documentation

- [`admin-key-hygiene.md`](admin-key-hygiene.md) — handling the testnet
  deployer/admin key
- [`../policies/data-retention.md`](../policies/data-retention.md) — the
  wallet-key rule: never use a mainnet secret key against testnet
- [`../examples/integration-test-harness-notes.md`](../examples/integration-test-harness-notes.md)
  — why committed keys are a habit that becomes an incident
- [`../tooling/sweep-signer-integration-notes.md`](../tooling/sweep-signer-integration-notes.md)
  — the signing-only key's intended scope
- [`testnet-threat-model.md`](testnet-threat-model.md) — what is and is not at
  risk on testnet
- [`../runbooks/wrong-network-passphrase.md`](../runbooks/wrong-network-passphrase.md)
  — the one technical difference between the two networks
- [`README.md`](README.md) — index for this directory

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial warning against reusing a mainnet-intended keypair on testnet |
