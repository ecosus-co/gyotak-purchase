# Preprod evidence — gyotak-purchase v3

Records produced on Midnight Preprod on 2026-09-18, during the rehearsal that precedes
this contract's Mainnet authorization request.

```
Contract  e413ff91958079d1ee2e4c792fe19a9c14a5d4ed7eb7966d3526308a3ea2f846
Owner     20fc1d0d5c405e95c669158a3db32217e2be65247dbea06e243745832af2e1be
Explorer  https://preprod.midnightexplorer.com/
Indexer   https://indexer.preprod.midnight.network/api/v4/graphql
```

The contract was deployed to Mainnet the same day at
`d11d52bd5875ecc2e89e97149e0237db20a91c989f30268950e892655a2a2a57` (block 2,636,444). Its
ledger is empty; everything below is from the Preprod rehearsal.

At the time of writing `purchases.size` and `bindings.size` on Preprod are both 2: one pair
built by hand to measure the encodings (§ 1), and one pair produced by a real customer
order travelling the whole path (§ 2).

## What v3 changes

v2 recorded a handle in each binding — an account name the buyer told us, which we never
verified. v3 records the buyer's referral id alongside it.

The two are different in kind. A handle can be claimed by anyone. A referral id is issued
by the operator when a payment is confirmed, and reaches the customer only inside their
personal connection URL. Recording both means a fabricated post has to match a binding on
*both* fields, and only someone holding that buyer's connection URL can produce the pair.

Neither alone is enough. A referral id appears in the post itself, so anyone reading a
genuine post can copy it into a fake one; the handle recorded alongside is what makes that
fail.

## Files

| File | What it shows |
|---|---|
| `deploy-preprod.log` | Deploying the contract on Preprod |
| `test-record.log` | One purchase and one binding, built by hand from test values |
| `mirror-record.log` | One purchase and one binding from a real customer order, written by the mirror process |
| `deploy-mainnet.log` | Deploying the same contract on Mainnet |

The logs are the raw stdout of the commands, with three edits: progress lines from wallet
sync were removed, terminal colour escapes were stripped, and two values were redacted
(§ 4). Nothing else was changed.

## 1. The encodings, measured

The first pair exists to test two claims.

**That the commitment is a plain SHA-256.** Test values were generated in memory, the
digest was computed locally in Node **before** submission, and the transaction was then
sent. Reading the record back from the indexer gave:

```
record  PB-20260918-b7f3c55d
local   SHA-256(nonce ‖ buyerId32) = bdeb7b88c067d51437ccf20222b9032faff29920dfdbff1139be2ec3bbc4a4a3
chain   purchaseCommitment         = bdeb7b88c067d51437ccf20222b9032faff29920dfdbff1139be2ec3bbc4a4a3
→ MATCH
```

The `lotId` matched the independently computed `SHA-256("gyotak:lot:" ‖ catch_report_id)`
as well.

**That the referral id survives the round trip.** A referral id is `aff_` followed by 32
hex characters — 36 characters, too long for `Bytes<32>` as ASCII. The hex decodes to 16
raw bytes, stored right-padded with zeros; `aff_` is a constant, so the value is
recoverable. A binding was recorded with handle `gyotak_test` and a test referral id, and
both read back matching.

The test nonce was not retained, so this particular record cannot be re-opened. That is
deliberate: the measurement it was made for is complete, and the property it established
is reproducible by anyone with their own nonce.

## 2. A real customer order, end to end

The second pair came from a real customer order and travelled the entire path without
manual intervention at any step a customer would not take themselves:

```
1. Order      placed through an AI chat connected to the operator's MCP server,
              over a personal token URL that identifies the customer
2. Payment    confirmed. The referral id is issued at this moment, automatically
3. Picking    three packs of Kuro-Kanpachi from a lot landed 2026-09-05;
              the picked lot resolved to exactly one catch record
4. Issuance   the operator issued a purchase commitment (nonce + buyerId32)
5. On-chain   the mirror recorded it: recordPurchase
6. Claim      the buyer asked for their proof and gave the handle gyotaku_proto.
              The claim carries no customer identifier as an argument — the
              connection token is the only accepted credential — and the operator
              looked up that customer's referral id server-side
7. On-chain   the mirror recorded the link: recordXBinding, with handle and referral id
```

Steps 1–5 happen for every paid order whose lot resolves cleanly. Steps 6–7 happen only
when the buyer asks; a buyer who stays silent has a purchase record and no binding.

The mirror re-verifies `SHA-256(nonce ‖ buyerId32)` against the stored commitment before
submitting and submits only on a match. It also refuses to write a binding for a purchase
not confirmed on this contract, and checks the indexer first so that a binding already
present costs no transaction.

The referral id read back from this binding was
`aff_f2638c39b45f6518f1806bb16f78db87`, matching the value the operator issued to that
customer when their payment was confirmed.

## 3. Transactions

All six settled successfully. Full-length values for indexer lookup:

### Preprod deploy (ContractDeploy, block 2,600,073, 2026-09-18T06:31:36Z)

```
identifier  001ff97659d3205bce449e2c79aa2a0325782aa4f60eb81e57087f9a7b211f8d01
hash        6ca2cdac4a836903df9696e10d41b7a8c094cca789d20962a0916a9f16ce900c
```

### Mainnet deploy (ContractDeploy, block 2,636,444, 2026-09-18T13:22:54Z)

```
identifier  00d4ff9f46f867966e1fcdd52d1da3e01a51311fe514dd63bae76034d2a1aa8db9
hash        6da22391846e225bed39622da6d3c0ff18bc325fba05c3c74fa65cacf14cfcce
```

### recordPurchase / recordXBinding (Preprod)

| Step | purchaseId | identifier | hash | block |
|---|---|---|---|---|
| Test purchase | `PB-20260918-b7f3c55d` | `00e63d61095951f8301c2f3bc618d939894fc1ade981f4de93624fcfbd099862c2` | `917a73d90107040c516417ee7481dabe0358f5a60fc61b9e1ddf4bdbeeb6e576` | 2,600,143 |
| Test binding | `PB-20260918-b7f3c55d` | `00ba73429b364cab10746340e2569d6ad2a751570e589ac38b3fd00ad5eb585516` | `908c17489291f2706294ac5de6dd65b21ae66e6886799caf5a947d3519e7bfdd` | 2,600,157 |
| Real purchase | `PB-20260918-168b140a` | `001fa34013f18436bcb3e0b6f04064f9188153eaaea21f1ca6d9b5c1d232d32636` | `0399eea299ce5789914b6d5971bd388fa4d624ea0e27db8a001e7fa9195de27a` | 2,604,101 |
| Real binding | `PB-20260918-168b140a` | `00a63f6443f435ce533271ac342218d34b7baf60b35f1cc122215c93f6486893e5` | `79623f6b9f4075abae9354650d400dd489f2906aa27da27294da61d83b317608` | 2,604,115 |

In both pairs the binding settled 84 seconds after its purchase — 14 blocks — and the
contract's own `committedAt` and `boundAt` preserve the same order.

**Identifier and hash are different fields.** The Midnight SDK returns the 66-character
identifier; the indexer accepts it as `transactions(offset: { identifier: ... })`. The
64-character cryptographic hash is a separate field. Passing an identifier to the hash
field returns `invalid transaction hash`. Block explorers expect the hash.

## 4. What was redacted

Two values were removed from the logs.

The raw `catch_report_id` of the lot, a UUID internal to the operator's database, was
removed from `mirror-record.log`. The `lotId` derived from it — `a25ff954…`, the value
actually written to chain — is left intact.

The API key in the Mainnet RPC endpoint was removed from `deploy-mainnet.log`. The SDK
prints the full connection URL on its WebSocket disconnect line, key included. Worth
knowing if you run this stack: a deploy log pasted into an issue would leak yours.

Nothing else was removed. The logs contain wallet addresses and local filesystem paths, as
do the published logs of `gyotak-catch` and `gyotak-temp-log`. They contain no seed, no
nonce, no buyer identifier, and no customer name, address, phone number or account
identifier; we checked the published files against the operator's database values
directly.

## 5. How to verify these records yourself

### Reading contract state

The indexer exposes contract state over GraphQL. Query `contractAction(address)` and read
the `state` field. Records in `purchases` are keyed by `purchaseId` and carry
`purchaseCommitment`, `lotId`, `committedAt` and `schema`; records in `bindings` use the
same key and carry `handle`, `referrerId`, `boundAt` and `schema`. No GYOTAK service is
involved.

**The indexer accepts POST only.** A `GET` returns HTTP 405. If your tooling can fetch URLs
but cannot issue a GraphQL POST, you will receive no data — this is not evidence that a
record is absent. The correct conclusion in that case is "could not verify", not "no such
record exists". We have observed AI agents make exactly this error.

### Checking a claim

Take the purchase id from a post, read both records, and compare **both** the handle and
the referral id against what the post shows: the account that published it, and the
referral URL it carries.

Both must match. The handle alone can be claimed by anyone. The referral id alone can be
copied out of a genuine post. Only someone holding that buyer's connection URL can produce
a binding where both are correct.

Note what this does and does not establish. It shows that the operator recorded this claim,
at a block later than the purchase it refers to, and has not altered it since — and that
producing it required the buyer's own credential. It does not prove identity: a connection
URL can be shared or leaked, and the operator would not detect it.

Note also what the referral id makes visible. It is stable across a buyer's purchases, so
reading it from one post lets you find every binding that buyer has made. That is useful —
a reader can tell someone who bought once from someone who buys repeatedly — and it is
permanent.

### The encodings

Handle: ASCII, right-padded with zero bytes to 32.

Referral id: the 16 bytes behind `aff_<hex>`, right-padded with zero bytes to 32. To
reconstruct, take the first 16 bytes, hex-encode them, and prepend `aff_`.

Commitment: given a nonce and a buyer identifier, both 32 bytes,

```
sha256(nonce_bytes || buyer_id32_bytes)
```

Compare to the `purchaseCommitment` in the record. No Midnight library is needed.

Lot: `lotId` is `SHA-256("gyotak:lot:" ‖ catch_report_id)` where the tag is ASCII with no
separator. The result resolves to a landing recorded by `gyotak-catch` on Mainnet, which
carries species, region and landing date, and which links onward to the hourly storage
temperatures published by `gyotak-temp-log`.

## 6. Build reproducibility

The contract source is 6,574 bytes,
`SHA-256 b4dc7879e5e872d9044747184f8c8bf62bb0a74b3e647b656d0810bfc901235e`, compiled with
compactc 0.30.0 (language 0.22.0, runtime 0.15.0). A full rebuild from a clean directory,
including proving-key generation, reproduced all twenty artefacts byte for byte. Their
SHA-256 values are listed in `deployments/gyotak-purchase.md` § 5.4.

The verifier keys stored in both deployed contracts' state are byte-for-byte identical to
the rebuilt `keys/*.verifier` files.

## 7. Cost

Preprod: the deploy cost 0.30 DUST and each record 0.30 DUST — 1.50 DUST across the five
Preprod transactions.

The Mainnet deploy cost 50 DUST, a deliberate fee overhead set for deploys only: a failed
deploy produces a different contract address and would invalidate published verification
links. Recording transactions use the ordinary overhead. The contract holds no assets.
