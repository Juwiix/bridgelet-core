# Authorized-Controller Confusion

## What this document corrects

A specific mistaken belief about `EphemeralAccount::sweep()`:

> "The function calls `require_auth()` on the `authorized_controller`, so a
> successful sweep means the sweep was properly authorized."

The gate is real, and it does something. But what it does is restrict **who may
call the function**. It does not validate the **content** of the off-chain
signed authorization at all. Conflating those two things — an identity check on
the caller with an authorization check on the operation — is a misreading this
document exists to prevent.

[`unverified-signature-path-warning.md`](unverified-signature-path-warning.md)
documents the related fact that the `auth_signature` parameter is ignored. This
document is about the mental model: what the gate *actually buys*, and why it is
not equivalent to full sweep-authorization security. The two should be read
together.

## The gate, stated exactly

From `contracts/ephemeral_account/src/lib.rs`, `sweep()` delegates its entire
"verification" to this function:

```rust
fn verify_sweep_authorization(
    env: &Env,
    _destination: &Address,
    _signature: &BytesN<64>,
) -> Result<(), Error> {
    let controller = storage::get_authorized_controller(env).ok_or(Error::Unauthorized)?;
    controller.require_auth();
    Ok(())
}
```

Both parameters are underscore-prefixed and never read. The body fetches the
address stored as `authorized_controller` and calls `require_auth()` on it.

`require_auth()` in Soroban is **invoker authentication**: the host checks that
the named address was part of the authorizing invoker set for this invocation.
It answers one question — *"is the caller the controller?"* — and nothing else.
It does not read a message, does not compute a digest, does not compare bytes,
and does not look at `auth_signature`.

The stored `authorized_controller` is set once, in `initialize()`, and has no
getter or setter in the public interface. In a normal deployment it is the
`SweepController` contract ID.

## What the gate checks versus what sweep authorization checks

| Question | `require_auth()` gate | Real sweep authorization (`SweepController`) |
|---|---|---|
| Who may call? | Only the stored controller | A signer holding the authorized Ed25519 key |
| What is validated? | The invoker's identity | A signature over `SHA256(destination ‖ nonce ‖ contract_id)` |
| Is `auth_signature` read? | **No** | Yes — `ed25519_verify` |
| Is the destination bound to the authorization? | **No** | Yes — the destination is in the signed message |
| Is there replay protection? | **No** | Yes — the on-chain nonce |
| What does a rejection prove? | The caller was not the controller | The signature or nonce was invalid |
| What does a success prove? | The caller was the controller | A valid, destination-bound authorization existed |

Read the two columns as answering different questions: the gate answers *who
called*, the authorization scheme answers *what was authorized*. Neither is a
substitute for the other.

## The reasoning chain that produces the mistake

The confusion is not one error; it is a short chain of plausible steps that
each seem reasonable in isolation.

1. **"`sweep()` takes a signature, so the signature must be checked."** The
   parameter is named `_signature` and is discarded. The shape of the interface
   suggests a guarantee the implementation does not provide.
2. **"A direct call was rejected, so the bad signature was caught."** The
   rejection is `require_auth()` on the caller — `Error::Unauthorized` (8) —
   not a signature verdict. The function never reached the signature.
3. **"`require_auth()` is an authorization mechanism."** It is an identity gate.
   Calling it "authorization" is where the two concepts get merged.
4. **"The controller is trusted, so the gate is equivalent to authorization."**
   Trusting the controller says the routing is safe; it says nothing about
   whether the operation's *content* — above all, the destination — was
   authorized. The gate would constrain nothing at all if the stored controller
   were ever a contract that forwards calls without its own checks.
5. **"The error codes distinguish a bad signature from a bad caller."** They do
   not, here. `Error::Unauthorized` (8) is the gate. There is no signature
   error on this path.

Each step is individually defensible. Together they produce the false belief
that `EphemeralAccount` enforces sweep authorization when it enforces a caller
identity and nothing more.

## Why this is consequential rather than academic

On the current code, the gap does not move funds — `EphemeralAccount` executes
no SEP-41 transfer, and a non-controller caller is rejected. The consequence is
that someone *designs around a security property that does not exist*, and that
design is what gets carried forward:

- **Believing the account enforces authorization.** Once `authorized_controller`
  is set, it can look like `EphemeralAccount` is a self-authorizing component
  that validates sweeps. It is not; all authorization content lives in
  `SweepController`.
- **Designing a custom controller that relies on the gate.** A controller that
  simply calls `sweep()` satisfies `require_auth()` without performing any
  content validation of its own. The destination is then whatever the caller
  passed — the gate does not bind it.
- **Omitting the gap from an audit or threat model.** A reviewer who reads
  `require_auth()` as authorization reports a control that is not there.
- **Assuming the account is safe to call directly.** The correct rule is the
  opposite: sweeps go through `SweepController::execute_sweep()` or `::claim()`,
  which *do* validate authorization content.

The forward-looking risk is the one the sibling warning already flags: if token
transfers are ever added to `EphemeralAccount::sweep()` before real
verification is, the same call becomes a signature-bypass drain. The
misreading here — that the gate already provides that verification — is
exactly what would make that change look harmless.

## The correct model

- `EphemeralAccount`'s controller gate is a **routing constraint**, not an
  authorization scheme. It constrains the caller, not the operation.
- **All authorization content lives in `SweepController`** — the Ed25519
  message, the destination binding, and the nonce. There is no equivalent in
  `EphemeralAccount`.
- **Never treat setting a controller as making the account self-authorizing.**
- **Route every sweep through `SweepController`** (`execute_sweep` or `claim`).
  A direct `sweep()` call does no content validation and must not be presented
  as one.
- **A rejection from a direct call is not a signature check.** Read the error:
  `Unauthorized` (8) is the caller gate.

For contrast within the same contract: `sweep_claim()` also relies purely on
the caller being the controller, but there the identity-only model is the
documented intent — it takes no signature parameter, so there is nothing to
misread. The confusion is specific to `sweep()`, where a signature-shaped
parameter sits in front of a gate that never reads it.

## Related documentation

- [`unverified-signature-path-warning.md`](unverified-signature-path-warning.md)
  — the sibling fact: `auth_signature` is not verified
- [`README.md`](../../README.md) — the **Current Stub Inventory**: the
  authoritative statement of what is and is not implemented
- [`docs/security.md`](../../docs/security.md) — design-level security model,
  including the `EphemeralAccount` signature-verification gap
- [`docs/SIGNATURE_FORMAT.md`](../../docs/SIGNATURE_FORMAT.md) — the message
  format that *is* verified
- [`../examples/signed-sweep-walkthrough.md`](../examples/signed-sweep-walkthrough.md)
  — the sweep path with real verification
- [`../examples/gas-free-claim-walkthrough.md`](../examples/gas-free-claim-walkthrough.md)
  — the other correct path
- [`README.md`](README.md) — index for this directory

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-09-26 | Initial documentation of the authorized-controller mental-model confusion |
