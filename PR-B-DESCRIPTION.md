# PR B: Ledger passthrough — DRAFT, NOT OPENABLE YET

**Status: gated on the keychain release carrying the `ledger` feature. Do not
open this as a pull request yet.**

This branch cannot be a PR until `solana-foundation/solana-keychain#301` ships a
release carrying the `ledger` feature. Everything else is done and this file is
the description to use on the day that lands.

Two edits ship it, both marked `TODO(ledger)` in
`rust/crates/kit/Cargo.toml`:

1. Uncomment `ledger = ["solana-keychain/ledger"]`.
2. Bump the `solana-keychain` version to the release carrying it.

Delete this file in the same commit.

---

## Why the feature line is commented out

Not because it is untested — because **an uncommented line breaks the default
build for every consumer**.

Cargo validates that a referenced dependency feature *exists* at resolve time,
whether or not anyone enables it. With the line live and no keychain release
carrying `ledger`, plain `cargo check --workspace` fails:

```
package `solana-pay-kit` depends on `solana-keychain` with feature `ledger`
but `solana-keychain` does not have that feature
```

I assumed, as the plan for this branch did, that an unreleased feature reference
would sit harmlessly until enabled. It does not. Verified by doing it: the
failure arrives before any `--features` flag is passed.

The alternative — a git dependency on a fork branch — makes it build today and
was the first thing rejected in review on #300, correctly: `cargo publish`
refuses a git dep, and a mutable branch on a personal fork does not belong in a
payments SDK even temporarily. Being openly blocked beats being quietly
unpublishable.

---

## The description to post

### What this does

Enables Ledger hardware-wallet signing in pay-kit. There is no pay-kit code here
and there is not meant to be: the MPP and x402 builders already take
`&dyn TransactionSigner`, which `solana_keychain::LedgerSigner` implements, so
this is a feature passthrough and nothing more. That is the payoff of the
keychain 2.x migration in #300, which this is stacked on.

### Scope

- `ledger = ["solana-keychain/ledger"]`, deliberately **outside every aggregate
  feature** including `default`. It pulls `hidapi`, which compiles a native HID
  library and needs `libudev-dev` + `pkg-config` on Debian, `systemd-devel` +
  `pkgconf-pkg-config` on Fedora, nothing on macOS. Everything else in this
  manifest builds with no system dependencies, so that requirement must never
  arrive because someone enabled `axum`.
- A worked fee-payer example in `rust/README.md` using `LedgerConfig` with
  `auto_open_app: false` and a shorter signing timeout, which is the shape a
  server wants.
- A CI note on the same build dependencies.

### Three things reviewers should know

**It is not a drop-in for a software key.** Every signature needs a physical
button press, one process serializes to one on-device confirmation at a time,
and a second signing request while the device is mid-prompt fails fast rather
than queueing. That suits a treasury or a demo, not a request path.

**`sign_message` does not sign your bytes.** A Ledger cannot raw-sign arbitrary
data; the payload is wrapped in the Solana app's off-chain-message envelope and
the device signs the envelope. Anything verifying such a signature must rebuild
the same bytes. This is exactly why pay-kit's message-only signer slots stay
`SolanaSigner` while transaction slots are `TransactionSigner`: on hardware the
two are not interchangeable, even though they are on `MemorySigner`.

**There is an upstream blocker on current devices.** A Ledger running Solana app
1.16.0 cannot be enumerated by any published `solana-remote-wallet`: the app
returns a 7-byte app-configuration vector and `ledger.rs:349` requires exactly 5.
Not this branch, not the device — see the agave issue linked from #301. Devices
on app ≤ 1.15.x work today.

### Testing

`cargo check --workspace` clean, so the parked feature costs consumers nothing.
`cargo test` 973 passed, unchanged from the base commit.
`cargo check -p solana-pay-kit --features ledger` fails with the version-select
error quoted above, which confirms the TODO describes the real blocker rather
than a guessed one.

The signing paths themselves are validated in keychain #301 against a Nano Gen5:
transaction signing, off-chain message signing, reject-then-sign-again, and 20
consecutive reconnects. Nothing in pay-kit changes those paths, which is the
point of it being a passthrough.
