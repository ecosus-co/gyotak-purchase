# gyotak-purchase

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Midnight](https://img.shields.io/badge/Midnight-Preprod-purple)](https://midnight.network)

A Midnight Compact contract that records anonymous purchase commitments for
flash-frozen seafood, operated by ECOSUS CO., LTD. (Pranburi, Thailand). This is
the third contract in the GYOTAK traceability system; the first two,
[`gyotak-catch`](https://github.com/ecosus-co/gyotak-catch) and
[`gyotak-temp-log`](https://github.com/ecosus-co/gyotak-temp-log), are
Mainnet-approved via
[PR #96](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/96)
and
[PR #224](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/224).

## Overview

ECOSUS operates **GYOTAK**, a sashimi-grade flash-frozen fish brand serving B2B
and B2C channels across Thailand. `gyotak-catch` records where a fish was landed;
`gyotak-temp-log` records how it was kept. This contract closes the chain at the
buyer's end, so that a customer's account of a meal can be checked against
something that existed before the account was written.

The contract keeps two ledger maps:

```
purchases   every purchase; the buyer is hidden inside a commitment
bindings    the subset a buyer has attached a public handle to
```

**`purchases`** holds one record per purchase:
`purchaseCommitment` = `persistentCommit<Bytes<32>>(buyerId32, nonce)`, plus
`lotId`, `committedAt` and `schema`. The buyer's identifier and the opening nonce
enter the circuit as witnesses and never reach the chain. For a `Bytes<32>` value
the commitment is the plain `SHA-256(nonce ‖ buyerId32)`, so a buyer can open
their own record with one hash on any machine — no Midnight SDK, no cooperation
from GYOTAK.

**`bindings`** holds the claims. A buyer who chooses to speak publicly can attach
their account handle to one of their purchases; the handle is stored in plaintext,
with `boundAt` recording when the claim was made. A binding cannot exist without
its purchase, and neither map allows updates or deletes.

Reading both maps together yields "@handle bought lot X at time T". Reading only
`purchases` yields "someone bought lot X at time T". A buyer who stays silent
leaves no public trace at all.

`lotId` is `SHA-256("gyotak:lot:" ‖ catch_report_id)`, which resolves to the
landing recorded by `gyotak-catch` and from there to the storage temperatures
published by `gyotak-temp-log`.

### What this does not prove

The handle in a binding is **self-reported**. The claim arrives over the buyer's
own authenticated connection, so the operator knows which customer is speaking —
but the operator does not verify that the account is theirs. The chain attests
that GYOTAK recorded a commitment and a claim at given blocks and has not altered
them since; it does not attest that a purchase physically occurred, and opening a
commitment demonstrates knowledge of the nonce rather than identity.
TR-2026-015 § 5 states the boundary in full.

Full technical report (defensive publication):
[TR-2026-015](https://gyotak-tr.pages.dev/tr-2026-015/) (CC BY 4.0), permanently
archived at [perma.cc/5TXA-G7BP](https://perma.cc/5TXA-G7BP).

Follow-up report (design disclosure):
[TR-2026-016 — Referral-Bound Purchase Claims](https://gyotak-tr.pages.dev/tr-2026-016/)
(CC BY 4.0), published 2026-09-18, permanently archived at
[perma.cc/5B7R-F7FC](https://perma.cc/5B7R-F7FC). It extends the binding described
above with the buyer's own referral ID (v3, not yet implemented), so that a referral
link and the purchase it cites can be tied together on chain alone.
SHA-256 of the published HTML:
`a7007f66dcd118ad3bd7ac3f36dbbbbec0cbe62739724f45bea6bcb83d3239b2`

## Mainnet deployment authorization application

This repository accompanies a Mainnet deployment authorization application
submitted to the Midnight Foundation:

- **Application document**: [`deployments/gyotak-purchase.md`](deployments/gyotak-purchase.md)
- **Empirical evidence**: [`preprod-evidence-20260917/`](preprod-evidence-20260917/)

### Contract addresses

Mainnet: `b6f0b4d275cdc96547042f0c38226aaa95beba95e325a831e1722f1313b15179`
(deployed 2026-09-18, block 2,629,712)

Preprod: `fe71367b28596c91a490e7e900e3d521d0597ec5456020b3e8e19cd5e76d7043`

### Owner public key

`20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be`

## Repository structure

```
contracts/                          Compact source (gyotak-purchase.compact)
deployments/gyotak-purchase.md      Mainnet authorization application
preprod-evidence-20260917/          Preprod records and verification procedures
```

## Build

```bash
compact compile contracts/gyotak-purchase.compact contracts/managed
```

Built with compactc 0.30.0 (language 0.22.0, runtime 0.15.0). A full rebuild from
a clean directory, including proving-key generation, reproduces all twenty
artefacts under `contracts/managed/keys` and `contracts/managed/zkir` byte for
byte; their SHA-256 values are listed in the application document § 5.4. The
verifier keys stored in the deployed contract's state are byte-for-byte identical
to the rebuilt `keys/*.verifier` files.

## Related repositories

- [`ecosus-co/gyotak-catch`](https://github.com/ecosus-co/gyotak-catch) — catch provenance contract (Mainnet authorized)
- [`ecosus-co/gyotak-temp-log`](https://github.com/ecosus-co/gyotak-temp-log) — cold-chain contract (Mainnet authorized)
- [`ecosus-co/gyotak-compact-teaching`](https://github.com/ecosus-co/gyotak-compact-teaching) — teaching extract of GYOTAK Compact contracts

## License

Apache License 2.0. See [LICENSE](LICENSE) for the full text.

## Contact

Takuya Ogura, Chairman
ECOSUS CO., LTD.
138/30 Moo 5, Pak Nam Pran Subdistrict, Pranburi, Prachuap Khiri Khan, Thailand
Email: ecosus2023@gmail.com
