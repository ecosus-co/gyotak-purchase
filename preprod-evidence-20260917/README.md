# Preprod evidence — gyotak-purchase

These are the v2 records, superseded by
[`preprod-evidence-20260918/`](../preprod-evidence-20260918/). v2 remains deployed
and frozen at `b6f0b4d275cdc96547042f0c38226aaa95beba95e325a831e1722f1313b15179`.

Records produced on Midnight Preprod on 2026-09-17, during the rehearsal that
precedes this contract's Mainnet authorization request.

The contract was subsequently deployed to Mainnet at
`b6f0b4d275cdc96547042f0c38226aaa95beba95e325a831e1722f1313b15179` on 2026-09-18
(block 2,629,712). The records below are from the Preprod rehearsal that preceded it.

```
Contract  fe71367b28596c91a490e7e900e3d521d0597ec5456020b3e8e19cd5e76d7043
Owner     20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be
Explorer  https://preprod.midnightexplorer.com/
Indexer   https://indexer.preprod.midnight.network/api/v4/graphql
```

At the time of writing both `purchases.size` and `bindings.size` are 2: one pair
of records built by hand to measure the commitment identity (§ 1), and one pair
produced by a real customer order travelling the whole path (§ 2).

## Files

| File | What it shows |
|---|---|
| `deploy.log` | Deploying the contract |
| `test-record.log` | One purchase and one binding, built by hand from test values |
| `mirror-record.log` | One purchase and one binding from a real customer order, written by the mirror process |
| `readback.log` | Both records read back from the indexer, with no wallet involved |
| `deploy-failed-170.log` | A first deploy attempt rejected with `Custom error: 170`, kept for the reason in § 5 |

The logs are the raw stdout of the commands, with three edits: progress lines from
wallet sync were removed, terminal colour escapes were stripped, and one lot
identifier was redacted (see § 4). Nothing else was changed.

## 1. Commitment identity, measured

The first pair exists to test one claim: that the circuit's
`persistentCommit<Bytes<32>>(buyerId32, nonce)` equals a plain
`SHA-256(nonce ‖ buyerId32)` computed outside Midnight. Test values were generated
in memory, the digest was computed locally in Node **before** submission, and the
transaction was then sent. Reading the record back from the indexer gave:

```
record  PB-20260917-fa72e7fd
local   SHA-256(nonce ‖ buyerId32) = b4677b65f1219aa532c9efee4af7283c2a84565e5495433ca022e748b4fe602a
chain   purchaseCommitment         = b4677b65f1219aa532c9efee4af7283c2a84565e5495433ca022e748b4fe602a
→ MATCH
```

The `lotId` matched the independently computed
`SHA-256("gyotak:lot:" ‖ catch_report_id)` as well. A binding was then recorded
against the same purchase with handle `gyotak_test`, and read back matching.

The test nonce was not retained, so this particular record cannot be re-opened.
That is deliberate: the measurement it was made for is complete, and the property
it established — that the commitment is a plain SHA-256 — is reproducible by
anyone with their own nonce.

## 2. A real customer order, end to end

The second pair came from a real customer order and travelled the entire path
without manual intervention at any step a customer would not take themselves:

```
1. Order      placed through an AI chat connected to the operator's MCP server,
              over a personal token URL that identifies the customer
2. Payment    confirmed
3. Picking    three packs of Kuro-Kanpachi from a lot landed 2026-09-06;
              the picked lot resolved to exactly one catch record
4. Issuance   the operator issued a purchase commitment (nonce + buyerId32)
5. On-chain   the mirror recorded it: recordPurchase
6. Claim      the buyer asked for their proof and gave the handle gyotaku_proto
7. On-chain   the mirror recorded the link: recordXBinding
```

Steps 1–5 happen for every paid order whose lot resolves cleanly. Steps 6–7 happen
only when the buyer asks; a buyer who stays silent has a purchase record and no
binding.

The mirror re-verifies `SHA-256(nonce ‖ buyerId32)` against the stored commitment
before submitting and submits only on a match. It also refuses to write a binding
for a purchase not confirmed on this contract, and checks the indexer first so
that a binding already present with the same handle costs no transaction.

## 3. Transactions

All five settled successfully. Full-length values for indexer lookup:

### Deploy (ContractDeploy, block 2,588,244, 2026-09-17T10:48:42Z)

```
identifier  004008a12e1f71c081f9bd39d1f3b654b30b6d60c8436413d7f7a90bdab79ef75a
hash        46dc75f2369c71986f09cca1fb8ec749a83340ab59e1630574e231756f593274
```

### recordPurchase / recordXBinding

| Step | purchaseId | identifier | hash | block |
|---|---|---|---|---|
| Test purchase | `PB-20260917-fa72e7fd` | `00eed986423e2813f5f608bc80e8370f72a788faecec20b28dd774f130fda05792` | `a5a901e652e0102ed45dd3ee24f1166dd718613bee6e753ada8e4e363185d290` | 2,588,347 |
| Test binding | `PB-20260917-fa72e7fd` | `00c634665891aac749cfdbadba727ab53780b235fbc3337c2594664f12265ed16a` | `7f6e2f1c12f4dba5ec52ad3ab17c357538f246b2561e714602ade1a5da7f18e1` | 2,588,389 |
| Real purchase | `PB-20260917-a97953d3` | `000557906b0d030836f919da6856134539b4057b251f54b2f475566ebfff166eac` | `79b36e0c620760cb0b75da14e4e7e68bcbca02cd9a4f09fb2db789c089286c9c` | 2,589,935 |
| Real binding | `PB-20260917-a97953d3` | `0027045120a65b01ebb4120e11363846c4b560309fd5f791b4bebbb81aeae0a9b5` | `30e4147b7b59fdf17075d6f1d8e6bd530eb89ca596745a29f1fb2f13c637f66a` | 2,589,950 |

In both pairs the binding settled after its purchase — 4 min 12 s later for the
test pair, 90 s for the real one — and the contract's own `committedAt` and
`boundAt` preserve the same order.

**Identifier and hash are different fields.** The Midnight SDK returns the
66-character identifier; the indexer accepts it as
`transactions(offset: { identifier: ... })`. The 64-character cryptographic hash
is a separate field. Passing an identifier to the hash field returns
`invalid transaction hash: cannot convert to ...ByteArray<32>` — we verified this.
Block explorers expect the hash.

## 4. What was redacted

One value was removed from `mirror-record.log`: the raw `catch_report_id` of the
lot, a UUID internal to the operator's database. The `lotId` derived from it —
`158eee82…`, the value actually written to chain — is left intact, so the log
still shows what was recorded and what it resolves to.

Nothing else was removed. The logs contain the preprod wallet addresses and local
filesystem paths, as do the published logs of `gyotak-catch` and
`gyotak-temp-log`. They contain no seed, no nonce, no buyer identifier, no API
token, and no customer name, address, phone number or account identifier; we
checked the published files against the operator's database values directly.

## 5. The failed attempt

`deploy-failed-170.log` records a first deploy attempt rejected with
`1010: Invalid Transaction: Custom error: 170` (`InvalidDustSpendProof`). No DUST
was spent; the transaction never entered the pool.

We include it because the cause is not obvious from the error, and the fix may
save another operator the same afternoon. The wallet state had just been copied
from another project directory and had caught up by a few hundred events. Its view
of the DUST tree roots disagreed with the node's at the moment the spend was
declared. The ledger's own message says as much: the failure is *"just as likely a
disagreement on dust state on the declared time as the proof being invalid."*

The fix was to wait. The second attempt copied the wallet state again — the
rejected coin is left marked as pending locally, so the balance reads zero until
it expires — then, after the wallet reported `isSynced`, waited 45 seconds and
required one further synced update before declaring the spend. That attempt
succeeded, and every transaction in § 3 was submitted the same way.

## 6. How to verify these records yourself

### Reading contract state

The Preprod indexer exposes contract state over GraphQL. Query
`contractAction(address: "fe71367b…7043")` and read the `state` field. Records in
`purchases` are keyed by `purchaseId` and carry `purchaseCommitment`, `lotId`,
`committedAt` and `schema`; records in `bindings` use the same key and carry
`handle`, `boundAt` and `schema`. No GYOTAK service is involved in this path.

**The indexer accepts POST only.** A `GET` returns HTTP 405. If your tooling can
fetch URLs but cannot issue a GraphQL POST, you will receive no data — this is not
evidence that a record is absent. The correct conclusion in that case is "could not
verify", not "no such record exists". We have observed AI agents make exactly this
error.

### Opening a commitment

Given a nonce and a buyer identifier, both 32 bytes:

```
sha256(nonce_bytes || buyer_id32_bytes)
```

Compare to the `purchaseCommitment` in the record. No Midnight library is needed;
any SHA-256 implementation will do. This is the property measured in § 1.

### Checking a claim

Take the purchase id from a post, read both records, and compare the on-chain
`handle` to the account that published the post. The handle is stored as ASCII
right-padded with zero bytes, so it reads directly out of the state without
decoding.

Note what this does and does not establish. The binding shows that the operator
recorded this claim, at a block later than the purchase it refers to, and has not
altered it since. It does not show that the account belongs to the buyer: the
handle is self-reported and the operator does not verify it. An account that
publishes its own purchase id, in its own profile or a pinned post, is doing
something the operator cannot do on its behalf.

### Checking a lot

`lotId` is `SHA-256("gyotak:lot:" ‖ catch_report_id)` where the tag is ASCII with
no separator. The resulting value resolves to a landing recorded by
`gyotak-catch` on Mainnet, which carries species, region and landing date, and
which links onward to the hourly storage temperatures published by
`gyotak-temp-log`.

## 7. Build reproducibility

The contract source is 7,123 bytes,
`SHA-256 0740779ca34a8eb5b75c7fee9d2b02c25881e392c2b2d803888a0f65422ad9c8`,
compiled with compactc 0.30.0. A full rebuild from a clean directory, including
proving-key generation, reproduced all twenty artefacts under
`contracts/managed/keys` and `contracts/managed/zkir` byte for byte. Their
SHA-256 values are listed in `deployments/gyotak-purchase.md` § 5.4.

The verifier keys stored in the deployed contract's state are byte-for-byte
identical to the rebuilt `keys/*.verifier` files, which is what establishes that
the source published here is the source deployed.

## 8. Cost

Each transaction settled for approximately 0.30 DUST, read from the
`DustSpendProcessed` events on chain rather than from the indexer's fee field,
which reports 1 speck and does not reflect what was actually spent. A purchase and
its binding therefore cost about 0.60 DUST in total. The contract holds no assets.
