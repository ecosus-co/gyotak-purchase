# gyotak-purchase

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Midnight](https://img.shields.io/badge/Midnight-Mainnet-purple)](https://midnight.network)

A Midnight Compact contract that records anonymous purchase commitments for
flash-frozen seafood, operated by ECOSUS CO., LTD. (Pranburi, Thailand). This is
the third contract in the GYOTAK traceability system;
[`gyotak-catch`](https://github.com/ecosus-co/gyotak-catch) and
[`gyotak-temp-log`](https://github.com/ecosus-co/gyotak-temp-log) are
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
bindings    the subset a buyer has claimed, naming where they post and who they are
```

**`purchases`** holds one record per purchase: `purchaseCommitment` =
`persistentCommit<Bytes<32>>(buyerId32, nonce)`, plus `lotId`, `committedAt` and
`schema`. The buyer's identifier and the opening nonce enter the circuit as
witnesses and never reach the chain. For a `Bytes<32>` value the commitment is the
plain `SHA-256(nonce ‖ buyerId32)`, so a buyer can open their own record with one
hash on any machine — no Midnight SDK, no cooperation from GYOTAK.

**`bindings`** holds the claims. A buyer who chooses to speak publicly attaches two
values to one of their purchases: the account handle they will post from, and their
own referral id. Both are stored in plaintext, with `boundAt` recording when the
claim was made. A binding cannot exist without its purchase, and neither map allows
updates or deletes.

Reading both maps together yields "@handle, whose referral link is this one, bought
lot X at time T". Reading only `purchases` yields "someone bought lot X at time T".
A buyer who stays silent leaves no public trace at all.

`schema` says how to read `lotId`. Records written before 2026-10-04 carry
`schema = 1`, described next; records written from that day on carry `schema = 2`,
described in [Record format versions](#record-format-versions-schema).

For `schema = 1`, `lotId` is `SHA-256("gyotak:lot:" ‖ catch_report_id)`, which
resolves to the landing recorded by `gyotak-catch` and from there to the storage
temperatures published by `gyotak-temp-log`.

When a purchase cannot be tied to exactly one landing — the catch photo was
missed, the landing's species list was left empty, or the only match was filled in
later by inference — the purchase is still recorded, once per order, with the fixed
value `lotId = SHA-256("gyotak:lot:no-lot")`
(`e6a193f127a97a4f600f904800f48a1d41aa733f12ce384951ace8255c2921f8`). This value
means "no catch record": it resolves to no landing in `gyotak-catch` and no
temperature log, and the proof page says so. It is the same for every such purchase
and contains no order number, so it links nothing to anything. `catch_report_id`
values are UUIDs and are never `no-lot`.

### Record format versions (`schema`)

| `schema` | Written | One record per | `lotId` |
|---|---|---|---|
| 1 | before 2026-10-04 | lot of an order | `SHA-256("gyotak:lot:" ‖ catch_report_id)`, or the fixed no-lot value above |
| 2 | from 2026-10-04 | whole order | `SHA-256("gyotak:order-lots:v1:" ‖ manifest)` |

The date is Thailand time (UTC+7). The first `schema = 2` record is purchase
`PB-20261004-74418a35`, committed at 2026-10-03 23:43 UTC (Mainnet v3, block
2,858,510); it comes from a GYOTAK test-customer order.

Records written before the switch keep `schema = 1`. They are not rewritten, and
they are checked exactly as described above.

**`schema = 2`: one record per order.** The manifest is a text list of the order's
items, built as follows:

1. For each item of the order, take its `catch_report_id` in lowercase if the item
   is tied to exactly one catch record, and the literal `no-lot` otherwise.
2. Remove duplicates.
3. Sort in ASCII order.
4. Join with commas, with no spaces.

Then `lotId = SHA-256("gyotak:order-lots:v1:" ‖ manifest)`, with the text taken as
UTF-8 and the digest written in lowercase hex. For an order in which no item is
tied to exactly one catch record, the manifest is just `no-lot`:

```bash
printf '%s' 'gyotak:order-lots:v1:no-lot' | sha256sum
# a93ba14f9607b2b775e966e55a1febae2280263406589a7922abd19cb6b7dd9d
```

That is the `lotId` on the first record above. On macOS use `shasum -a 256`.

What the record holds, and what it does not:

- The chain holds only the hash. The manifest lists catch record IDs and `no-lot`
  only — no fish names and no weights — and it is not on the chain. GYOTAK
  publishes it on the purchase's confirmation page
  (`https://line-harness.gyotak.workers.dev/verify/purchase/?id=<purchase ID>`) and
  in the `lot.manifest` field of the JSON at
  `https://line-harness.gyotak.workers.dev/gyotak/verify-purchase?id=<purchase ID>`.
  To check a record, hash the published manifest as above and compare it with the
  `lotId` stored on the chain.
- `no-lot` in a manifest stands for an item that is not tied to exactly one catch
  record. A manifest with `no-lot` next to catch record IDs means some of the fish
  in that order have no catch record; a manifest of `no-lot` alone means no item of
  the order is tied to one.
- The hash of a manifest of `no-lot` alone is a fixed value, the same for every such
  order, so it contains no order number. A manifest that includes catch record IDs
  gives a `lotId` specific to that set of catches.
- A value other than 1 or 2 in `schema` has no reading defined here.

### Why two identifiers

A handle is self-reported. Anyone can claim one that is not theirs, and the
operator cannot detect it — the earlier version of this contract named that as the
sharpest limit of its design.

A referral id is different. The operator issues it when a payment is confirmed, and
it reaches the customer only inside their personal connection URL. Recording both
means a fabricated post must match a binding on *both* fields, and only someone
holding that buyer's connection URL can produce the pair. A referral id appears in
the post itself, so anyone reading a genuine post can copy it into a fake one; the
handle recorded alongside it is what makes that fail.

This is evidence, not proof. A connection URL can be shared or leaked, and the
operator would not detect it. `deployments/gyotak-purchase.md` § 6 states the
boundary in full.

## Mainnet deployment

- **Application document**: [`deployments/gyotak-purchase.md`](deployments/gyotak-purchase.md)
- **Empirical evidence**: [`preprod-evidence-20260918/`](preprod-evidence-20260918/)

### Contract addresses

Mainnet: `d11d52bd5875ecc2e89e97149e0237db20a91c989f30268950e892655a2a2a57`
(deployed 2026-09-18, block 2,636,444)

Preprod: `e413ff91958079d1ee2e4c792fe19a9c14a5d4ed7eb7966d3526308a3ea2f846`

### Owner public key

`20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be`

### Earlier versions

The contract has been deployed three times. Each generation is a separate address;
records written to one stay there.

| | Mainnet | Preprod | Status |
|---|---|---|---|
| v3 (current) | `d11d52bd…2a2a57` | `e413ff91…3ea2f846` | active |
| v2 | `b6f0b4d2…3b15179` | `fe71367b…5e76d7043` | frozen |
| v1 | — | `a853e32e…fdfc7f1c7` | frozen |

v2 was authorized via
[PR #316](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/316)
and its address recorded in
[PR #317](https://github.com/midnightntwrk/midnight-improvement-proposals/pull/317).
It holds one purchase and one binding from the rehearsal described there. v3 adds
the referral id to each binding; the submitting process now targets v3 only, and
the switch was made when no record was pending, so no purchase is stranded without
its binding.

### Technical reports

Two defensive publications cover this contract. Both are CC BY 4.0.
The full series is listed at https://gyotak-tr.pages.dev/gyotaku-protocol/

**TR-2026-015 — Anonymous Purchase Commitment, and the Claim That Names It**
https://gyotak-tr.pages.dev/tr-2026-015/
Archived: https://perma.cc/HS7L-YZZD
TDCommons: https://www.tdcommons.org/dpubs_series/11785

The construction itself: what the chain records, what a buyer can verify
unaided, and where the guarantee stops.

**TR-2026-016 — Referral-Bound Purchase Claims**
https://gyotak-tr.pages.dev/tr-2026-016/
Archived: https://perma.cc/8K6N-8F6V
TDCommons: https://www.tdcommons.org/dpubs_series/11796

Why a binding carries two identifiers rather than one, and the referral
cycle around it. Published 2026-09-18 as a disclosure of a design that did
not yet exist; v3 was deployed to Mainnet later the same day, at block
2,636,444.

## Repository structure

```
contracts/                          Compact source (gyotak-purchase.compact)
deployments/gyotak-purchase.md      Mainnet authorization application
preprod-evidence-20260918/          v3 records and verification procedures
preprod-evidence-20260917/          v2 records (kept for reference)
```

## Build

```bash
compact compile contracts/gyotak-purchase.compact contracts/managed
```

Built with compactc 0.30.0 (language 0.22.0, runtime 0.15.0). A full rebuild from a
clean directory, including proving-key generation, reproduces all twenty artefacts
under `contracts/managed/keys` and `contracts/managed/zkir` byte for byte; their
SHA-256 values are listed in the application document § 5.4. The verifier keys
stored in both deployed contracts' state are byte-for-byte identical to the rebuilt
`keys/*.verifier` files.

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
