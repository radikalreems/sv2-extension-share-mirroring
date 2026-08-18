# Stratum V2 Extension (Draft): Share Mirroring to the Template Provider

**Status:** Draft / not assigned an official extension type  
**Provisional extension type:** `0x0003` (placeholder until registered in the SV2 extension registry)  
**Protocols involved:** Template Distribution Protocol (TDP). This extension **defines new messages** that travel on the TDP connection; it does not reuse Mining Protocol message types.  
**Depends on:** [Extensions Negotiation (`0x0001`)](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md). TDP does not implement this today; see [issue-tdp-extensions-negotiation.md](./issue-tdp-extensions-negotiation.md).

Terms like "MUST," "MUST NOT," "REQUIRED," "SHOULD," and "MAY" follow [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 0. Abstract

This document describes a Stratum V2 **extension** that lets a **Template Provider (TP)** audit that their templates are actually being mined, without changing core SV2 Mining or Template Distribution message layouts for devices that do not opt in.

This extension defines new messages that travel on the **Template Distribution Protocol (TDP)** connection between the TP and a **Job Declarator Client (JDC)**. They are **not** Mining Protocol messages and MUST NOT be sent on a Mining connection.

On that TDP connection the extension does three things:

1. **Associate jobs with templates** — When the JDC begins mining a job derived from a TP template, it sends a `NewAssociatedMiningJob` message on TDP. That message carries the original `template_id` plus the job parameters needed to interpret **and verify** later shares: coinbase prefix/suffix, merkle path, the pool-bound extranonce prefix and size, and the share target.
2. **Retain pool-bound shares** — The JDC keeps a rolling window of the last 10,000 shares it submitted toward the Pool, keyed by each share's **header hash** (the same double-SHA256 the JDC already computes to validate the share).
3. **Let the TP retrieve those shares** — The TP periodically pulls the hashes of the most recent shares in the window, checks that list against the shares it already holds, and requests any it lacks by hash. In **full** audit mode the JDC also mirrors each pool-bound share to the TP as it is submitted.

Mining Devices and Pools continue to receive ordinary Mining Protocol job notifications (`NewExtendedMiningJob`, `SubmitSharesExtended`, etc.). Core SV2 behavior for those devices is unchanged.

---

## 1. Motivation

A party that purchases hashpower today typically has to participate in the **whole SV2 flow**: Template Distribution toward a Template Provider, Job Declaration / Mining toward a Pool, and the Job Declarator Client that sits between them. The buyer cannot simply just handle providing templates.

This extension assists that the buyer be **delegated to the Template Provider**. The purchaser runs or controls the TP; the JDC still talks to the Pool and to Mining Devices as usual. What the TP needs is **proof** that its templates are actually being mined.

Stock TDP only returns **`SubmitSolution`** (network-valid blocks). That is not only too rare to use as proof, especially for solo-mining, but the entity that sources the hashrate may also source different pools dynamically tailored to the client's needs. This would make it non-trivial for the buyer to retrieve proof of work on their templates. Shares already exist at the JDC so this extension routes those over to the TP via TDP.

Share **resolution** at the TP does not need to match the Pool. The Pool wants a high share rate for accurate hashrate and payouts. The TP only needs enough shares to be confident that a template is being worked. Because a live 1:1 mirror of every pool-bound share onto TDP costs **bandwidth**, this extension has two modes (§3.3): **lazy** (batch pull only) and **full** (live mirror plus the same pull as a check). A coarser attestation target is left open (§10.1).

---

## 2. Design

### 2.1 Associating templates with mining jobs

Under stock SV2, the Template Provider identifies work by `template_id` on Template Distribution (`NewTemplate`). The Pool, JDC, and Mining Devices identify work by `job_id` on Mining (`NewMiningJob` / `NewExtendedMiningJob`). Those identifiers live in different namespaces and are not linked by the base protocols. Stock TDP only returns `SubmitSolution` (network-valid blocks). A Template Provider therefore cannot, from Mining share traffic alone, prove which of **their** templates produced a given share.

**`NewAssociatedMiningJob`** (JDC → TP) exists to close that gap. When the JDC derives pool-bound work from a TP template, it sends this message on the TDP connection, tying the original `template_id` to the pool-bound `job_id` and carrying the job parameters that shares will later be checked against: merkle path, coinbase prefix and suffix, the pool-bound `extranonce_prefix` / `extranonce_size`, and the pool-bound `share_target`. The TP stores this state keyed by `template_id`.

With the association in place, any share the JDC later hands over is a self-contained proof-of-work: the TP reconstructs the coinbase, derives the merkle root, assembles the header, and checks its hash against the target (§5.4). The same computed hash is the share's identifier (§6.2), so shares need no id field of their own.

The association is the foundation for everything that follows — both delivery modes only move share records around; interpreting and verifying those records always goes through the stored association.

### 2.2 Delivering shares: two modes

The JDC always keeps a rolling window of the last 10,000 pool-bound shares, keyed by share hash (§6). What varies is how share records reach the TP. **`SetShareMirroringMode`** (TP → JDC) exists so the TP — the party paying the audit's bandwidth and doing its verification — can choose between two delivery modes after extension negotiation succeeds:

**Lazy mode** — no unsolicited traffic; the TP pulls on its own schedule. Three message pairs exist for this:

- **`RequestRecentShareHashes`** (TP → JDC) asks for the hashes of the most recent shares in the window; **`ShareHashes`** (JDC → TP) returns that list. This is deliberately hashes-only: it tells the TP *what exists* without resending records it may already hold.
- **`RequestShares`** (TP → JDC) names the hashes the TP lacks after diffing the list against its own store; **`Shares`** (JDC → TP) returns those records — and reports any requested hash that has already aged out of the window, so the TP knows the difference between "not yet fetched" and "gone."

**Full mode** — everything lazy mode has, plus a live stream. **`SubmitSharesAssociated`** (JDC → TP) exists for the TP that wants shares as they happen: the JDC sends one for every pool-bound share at submission time, carrying `template_id` and the proof fields (`nonce`, `ntime`, `version`, `extranonce`). The periodic hash pull remains and becomes a safety net: the TP diffs the pulled list against what arrived live and uses `RequestShares` to recover anything dropped in transit or missed before it connected.

In both modes the TP detects missing shares the same way — a share is missing if and only if its hash appears in a `ShareHashes` reply and the TP holds no verified record for it. The modes differ only in whether records also arrive unsolicited.

---

## 3. Extension overview

When this extension is successfully negotiated on the **JDC ↔ Template Provider TDP connection**:

1. The JDC MUST, for every TP `NewTemplate` from which it is mining work that it also submits toward a Pool, send the TP a `NewAssociatedMiningJob` (§5) on that TDP connection.
2. The JDC MUST retain the last 10,000 pool-bound shares, keyed by share hash (§6).
3. The Template Provider MUST send `SetShareMirroringMode` (§4.1) selecting **lazy** or **full**.
4. In **full** mode, the JDC MUST, for every `SubmitSharesExtended` that it submits (or accepts for submission) to the Pool for that work, send the TP a `SubmitSharesAssociated` (§7) on that TDP connection.
5. In **lazy** mode, the JDC MUST NOT send unsolicited `SubmitSharesAssociated`. The TP obtains shares only via §8.
6. In **both** modes, the TP SHOULD periodically request the hashes of the last 5,000 shares (§8), check that list against the shares it holds, and request any it lacks by hash.

`template_id` is a native field on the share and job messages. A TLV is not required.

### 3.1 Provisional identifiers

| Item | Value | Notes |
|------|-------|-------|
| Extension type | `0x0003` | **Placeholder** — replace when registered |
| `NewAssociatedMiningJob` `msg_type` | `0x00` | Local to `extension_type = 0x0003` |
| `SubmitSharesAssociated` `msg_type` | `0x01` | Local to `extension_type = 0x0003` |
| `SetShareMirroringMode` `msg_type` | `0x02` | Local to `extension_type = 0x0003` |
| `RequestRecentShareHashes` `msg_type` | `0x03` | Local to `extension_type = 0x0003` |
| `ShareHashes` `msg_type` | `0x04` | Local to `extension_type = 0x0003` |
| `RequestShares` `msg_type` | `0x05` | Local to `extension_type = 0x0003` |
| `Shares` `msg_type` | `0x06` | Local to `extension_type = 0x0003` |

### 3.2 Message types

All messages defined by this extension MUST have `extension_type = 0x0003` in the frame header (the identifier of the extension that defined their non-TLV structure). The `channel_msg` bit MUST be unset. These messages are sent only on a TDP connection that has successfully negotiated this extension.

| Message Type (8-bit) | channel_msg bit | Message Name | Direction |
|----------------------|-----------------|--------------|-----------|
| 0x00 | 0 | NewAssociatedMiningJob | JDC → Template Provider |
| 0x01 | 0 | SubmitSharesAssociated | JDC → Template Provider |
| 0x02 | 0 | SetShareMirroringMode | Template Provider → JDC |
| 0x03 | 0 | RequestRecentShareHashes | Template Provider → JDC |
| 0x04 | 0 | ShareHashes | JDC → Template Provider |
| 0x05 | 0 | RequestShares | Template Provider → JDC |
| 0x06 | 0 | Shares | JDC → Template Provider |

The 8-bit `msg_type` values are local to this extension. They MAY overlap core `msg_type` bytes; the pair `(extension_type, msg_type)` identifies the message. See SV2 spec §3.4.1 example 4.

Peers that did not negotiate this extension MUST NOT send these messages. A peer that receives an unknown `extension_type` with `channel_msg` unset MUST ignore the frame (SV2 spec §3.4.1).

There is no Success/Error on `NewAssociatedMiningJob` or `SubmitSharesAssociated`. Retrieval pairs requests and replies by `request_id` (`RequestRecentShareHashes` → `ShareHashes`, `RequestShares` → `Shares`).

### 3.3 Audit modes

`0x0001` only negotiates extension ids, not parameters. After `RequestExtensions.Success`, the Template Provider selects the mode with `SetShareMirroringMode` (§4.1). A JDC that implements this extension MUST implement both modes.

| `mode` | Name | Live `SubmitSharesAssociated` | Batch retrieval (§8) |
|--------|------|-------------------------------|----------------------|
| `0x00` | **Lazy** | MUST NOT send | MUST: TP pulls the last 5,000 hashes, then fetches every record it lacks |
| `0x01` | **Full** | MUST send every pool-bound share | MUST: same hash pull as a reconciliation check on the live stream |

Lazy is for a TP that only needs periodic proof. Full is for a TP that wants every pool-bound share as it happens, with the pull as a safety net for drops or a late join.

---

## 4. Negotiation

Negotiation uses extension `0x0001` (Extensions Negotiation) on the **TDP** connection between the JDC (client) and the Template Provider (server).

1. `SetupConnection` / `SetupConnection.Success` for Template Distribution.
2. Client sends `RequestExtensions` including provisional `0x0003`.
3. Server responds `RequestExtensions.Success` (supported) or `RequestExtensions.Error`.
4. Client sends TDP `CoinbaseOutputConstraints` (stock TDP; after negotiation so that `RequestExtensions` remains the first protocol-specific message, per `0x0001`).
5. Server sends `SetShareMirroringMode` (§4.1).

The JDC MUST NOT send `SubmitSharesAssociated` until it has received `SetShareMirroringMode`. It MAY send `NewAssociatedMiningJob` after Success and before the mode message. It MUST start retaining the share window (§6) as soon as it is submitting toward the Pool, even if the mode message has not arrived yet.

Mining Devices and the Pool connection do **not** need this extension for the TP audit path to work.

### 4.1 `SetShareMirroringMode` (Server → Client)

Selects lazy or full for the rest of this TDP session. The Template Provider MUST send this after `RequestExtensions.Success` and before it relies on live shares or issues a retrieval request.

The Template Provider MAY send it again later. The JDC MUST apply the new mode immediately: stop unsolicited `SubmitSharesAssociated` when switching to lazy; start them for subsequent pool-bound submits when switching to full. Switching to full MUST NOT dump the window; the TP uses §8 if it wants history.

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| mode | U8 | `0x00` = lazy, `0x01` = full. All other values are invalid; the JDC MUST ignore the message and keep the previous mode (or, if none, MUST NOT stream). |

---

## 5. `NewAssociatedMiningJob` (Client → Server)

Sent on the TDP connection. Informs the Template Provider of the mining-job parameters the JDC is using for work derived from one of the TP’s templates, so later share proofs can be interpreted.

The JDC SHOULD send one `NewAssociatedMiningJob` per template it is mining, not one copy per downstream channel.

### 5.1 Fields

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| template_id | U64 | The `template_id` from the TP’s `NewTemplate` that this job was built from. MUST refer to a `NewTemplate` previously sent on this TDP session (reconnect rules are TBD; see §10). |
| job_id | U32 | The `job_id` the JDC uses on pool-bound `SubmitSharesExtended` for this work. Informational; TP MUST key state by `template_id`, not by `job_id` alone. |
| min_ntime | OPTION[U32] | Same meaning as `NewExtendedMiningJob.min_ntime`: empty means a future job awaiting TDP `SetNewPrevHash` for this `template_id`. |
| version | U32 | Header version used for this job (BIP323 bits as in Mining Protocol). |
| version_rolling_allowed | BOOL | Same meaning as `NewExtendedMiningJob.version_rolling_allowed`. |
| merkle_path | SEQ0_255[U256] | Merkle path hashes ordered from deepest. MUST match `NewTemplate.merkle_path` for `template_id`. |
| coinbase_tx_prefix | B0_64K | Prefix of the coinbase transaction used for this job (BIP141 fields stripped, same rule as `NewExtendedMiningJob`). |
| coinbase_tx_suffix | B0_64K | Suffix of the coinbase transaction used for this job. |
| extranonce_prefix | B0_32 | The extranonce prefix assigned by the Pool on the JDC's pool-bound (upstream) extended channel (`OpenExtendedMiningChannel.Success.extranonce_prefix`, as later replaced by any Pool `SetExtranoncePrefix` on that channel). These bytes sit between `coinbase_tx_prefix` and the per-share `extranonce`. |
| extranonce_size | U16 | The negotiated extranonce size on that pool-bound channel (`OpenExtendedMiningChannel.Success.extranonce_size`). The `extranonce` on every share for this association MUST be exactly this many bytes. |
| share_target | U256 | The share target in effect on the pool-bound channel for this job (initially `OpenExtendedMiningChannel.Success.target`, updated by Pool `SetTarget`). The TP validates each share's header hash against this value (§5.4). |

Coinbase reconstruction for a later share is:

```text
coinbase = coinbase_tx_prefix + extranonce_prefix + extranonce + coinbase_tx_suffix
```

All values are those of the **pool-bound** (upstream) channel and job: `extranonce_prefix` and `extranonce_size` are the pool channel's, and the `extranonce` on each share record is the pool-bound `SubmitSharesExtended.extranonce` after the JDC's downstream→upstream rewrite. Downstream (miner-side) prefixes and sizes never appear on TDP.

### 5.2 When to send

The JDC MUST send `NewAssociatedMiningJob` after it has derived pool-bound work from a `NewTemplate` and before (or together with) the first share for that `template_id` that it sends to the TP (live `SubmitSharesAssociated` or a `Shares` batch).

If any field of the association changes for the same `template_id`, the JDC MUST send a replacement `NewAssociatedMiningJob` for that `template_id` before any share produced under the new values. The latest message supersedes earlier ones. Concretely, a replacement is triggered by:

- a change to `coinbase_tx_prefix` / `coinbase_tx_suffix` or `merkle_path` (e.g. a new declared job for the same template);
- a Pool `SetExtranoncePrefix` on the pool-bound channel (changes `extranonce_prefix` even though the coinbase prefix/suffix bytes do not change);
- a Pool `SetTarget` on the pool-bound channel (changes `share_target`).

Shares are verified against the association in effect when the JDC submitted them; the TP SHOULD tolerate a small reordering window around a replacement (a share validated under the previous target/prefix may arrive just after the replacement message). This applies to **identity** as well as validity: the share hash (§6.2) is derived from the record's fields together with the association's `coinbase_tx_prefix` / `extranonce_prefix` / `coinbase_tx_suffix`, so a TP that fails to match or verify a share against the latest association SHOULD retry against the immediately preceding one before treating it as invalid.

### 5.3 Example

Template Provider previously sent `NewTemplate { template_id = 42, ... }`.  
JDC’s pool-bound job id is `7`.

```text
JDC → Template Provider (TDP):
  frame: extension_type=0x0003, msg_type=0x00, channel_msg=0
  NewAssociatedMiningJob {
    template_id=42,
    job_id=7,
    merkle_path, coinbase_tx_prefix, coinbase_tx_suffix,
    extranonce_prefix, extranonce_size, share_target, ...
  }
```

Template Provider stores job parameters under `template_id 42`.

### 5.4 Share verification (Template Provider side)

Every share the TP receives for a `template_id` — live (§7) or pulled (§8) — is checked against the **latest** `NewAssociatedMiningJob` for that template:

1. **Length check.** `extranonce` MUST be exactly `extranonce_size` bytes. Reject otherwise.
2. **Coinbase.** `coinbase = coinbase_tx_prefix + extranonce_prefix + extranonce + coinbase_tx_suffix`. Because prefix/suffix are serialized with BIP141 fields stripped (§5.1), double-SHA256 of this byte string is the coinbase `txid`.
3. **Merkle root.** Fold the `txid` through `merkle_path` (ordered from deepest, double-SHA256 of the concatenation at each step), exactly as in the Mining Protocol.
4. **Header.** Assemble the 80-byte block header from: `version` (share), `prev_hash` and `nBits` from the latest TDP `SetNewPrevHash` for this `template_id`, the computed merkle root, `ntime` (share), `nonce` (share). If `version_rolling_allowed`, the share's `version` MAY differ from the association's within the BIP320 rolling mask, same rule as Mining.
5. **Target check.** The share is valid iff the double-SHA256 header hash, interpreted as in the Mining Protocol target comparison, is ≤ `share_target`.

The header hash computed in step 5 is also the share's **identifier** (§6.2): verification and identification are the same computation.

Shares failing any step MUST NOT be counted as evidence of work. If the header hash is also ≤ the block target from `SetNewPrevHash`, the TP should additionally expect a stock TDP `SubmitSolution` for this template.

**Why a self-reported target is sound.** `share_target` comes from the JDC, but the JDC cannot inflate the work it proves by misreporting it. Fabricating a record that passes step 5 costs on the order of `2^256 / share_target` hash operations, so each verified share attests the difficulty implied by the reported target — regardless of what the pool's actual target was. Reporting an *easier* target than the pool's only lowers the work per share the JDC can claim. The TP's attested-hashrate floor is the sum of `difficulty(share_target)` over verified shares per unit of time, and honest share rates at a given target are also externally sanity-checkable (a pool-tuned target yields on the order of a few shares per minute per upstream channel).

---

## 6. JDC share window

The JDC MUST keep a rolling window of the last **10,000** shares it submitted (or accepted for submission) toward the Pool on the pool-bound Mining channel used for this TDP work, keyed by each share's **share hash** (§6.2).

This window is **pool-bound only**. Downstream miner shares that do not meet the pool target, and are not forwarded, MUST NOT be stored. Mining-protocol `sequence_number`s (downstream or pool-bound) are pool accounting and are not used by this extension.

This draft assumes a single pool-bound Mining channel for that JDC (the usual JDC shape). Multiple upstream channels are TBD (§10.4).

### 6.1 What is stored

For each retained share the JDC MUST be able to produce its share hash and a `ShareRecord` (§8.5): `template_id`, `job_id`, `nonce`, `ntime`, `version`, `extranonce`.

The JDC SHOULD append to the window when (or immediately after) it forwards the share to the Pool, not on the miner-submit hot path. The share hash requires no extra computation: the JDC already produces it when validating the share against the pool target. Evict the oldest entry when the window exceeds 10,000.

At typical pool vardiff (~6 shares/minute on the upstream channel) 10,000 shares is on the order of **28 hours** and about **0.8 MB** including the 32-byte hash keys.

### 6.2 Identifier: the share hash

The share id is the **share hash**: the double-SHA256 of the share's assembled 80-byte block header (the value computed in §5.4 steps 2–5), encoded as U256 with the same convention as `SetNewPrevHash.prev_hash`.

Properties this extension relies on:

- **Unique by construction.** Two shares with the same hash are the same share. Duplicate submissions are rejected by share validation before reaching the Pool, so the window cannot contain two entries with the same key. Uniqueness holds across JDC restarts, pool fallbacks, and pool changes with no counters, epochs, or session state.
- **Self-authenticating.** The TP never takes an id on faith: for every record it receives, it recomputes the hash during verification (§5.4) and uses that computed value as the key. A record and its id cannot disagree without the record failing verification.
- **Opaque.** The TP MUST NOT infer anything from hash values (ordering, recency, adjacency). Missing shares are detected only by reconciling against the hash lists the JDC reports (§8): a share is missing if and only if its hash appears in a `ShareHashes` reply and the TP does not hold a verified record for it.

---

## 7. `SubmitSharesAssociated` (Client → Server)

Live, unsolicited share mirror. **Full mode only.**

### 7.1 Rule

When `mode = 0x01` (full), for every `SubmitSharesExtended` the JDC submits to the Pool on work that was associated under §5, the JDC MUST also send a `SubmitSharesAssociated` to the TP on the TDP connection, in real time (or with bounded delay agreed operationally).

When `mode = 0x00` (lazy), the JDC MUST NOT send this message. The TP uses §8.

`SubmitSharesAssociated` omits Mining `channel_id` and `sequence_number` (pool accounting) and carries no explicit share id: the TP derives the share hash while verifying the record (§5.4) and stores it under that key, which is how live records later match the hash lists it pulls (§8). It carries `template_id` so the TP does not have to join through Mining `job_id`.

### 7.2 Fields

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| template_id | U64 | MUST equal the `template_id` of the `NewAssociatedMiningJob` this share is for. |
| job_id | U32 | MUST equal `NewAssociatedMiningJob.job_id` for that `template_id` (the pool-bound job id). Informational. |
| nonce | U32 | Header nonce. |
| ntime | U32 | Header nTime. Same bounds as Mining `SubmitSharesExtended.ntime` relative to the latest TDP `SetNewPrevHash` for this template. |
| version | U32 | Full nVersion field. |
| extranonce | B0_32 | The pool-bound `SubmitSharesExtended.extranonce`. MUST be exactly `NewAssociatedMiningJob.extranonce_size` bytes for this `template_id`. |

### 7.3 What “every” means (full mode)

“Every share submitted to the Pool” means every share the JDC treats as a pool submission for that work—including shares that the Pool later rejects—**unless** both parties explicitly negotiate a filter (see §10 Open issues).

Default in full mode: **full pool-bound resolution** mirrored live to the Template Provider.

### 7.4 Example

```text
Mining Device → JDC:           SubmitSharesExtended { ... }
JDC → Pool:                    SubmitSharesExtended { ... }
JDC stores the share in the window under its share hash H
JDC → Template Provider:       SubmitSharesAssociated { template_id=42, job_id=7, nonce, ntime, version, extranonce }
                               frame: extension_type=0x0003, msg_type=0x01, channel_msg=0
```

Template Provider: `template_id=42` → association from §5 → verify per §5.4, computing share hash `H` → store the record under `H`.

---

## 8. Share retrieval (both modes)

The Template Provider reconciles against the JDC window in two steps:

1. **Pull the hash list.** `RequestRecentShareHashes` asks for the share hashes of the most recent shares still in the window. The reply (`ShareHashes`) is the authoritative statement of what the window holds.
2. **Set-difference, then fetch.** The TP compares that list against the hashes of shares it already holds (previous fetches and, in full mode, the live stream) and requests every hash it lacks with `RequestShares`.

In lazy mode this is the **only** share path — on each cycle the TP typically fetches all listed records it does not yet have. In full mode it is the **reconciliation** path — the live stream already delivered most records, and the fetch covers only drops.

A share is missing if and only if its hash appears in a `ShareHashes` reply and the TP does not hold a verified record for it (§6.2).

There is no Success/Error. Replies pair with requests by `request_id`.

### 8.1 Recommended pull

| Knob | Value | Why |
|------|-------|-----|
| JDC window | 10,000 shares | Slack if a pull is late |
| TP request size | 5,000 hashes (half the window) | Leaves ~5,000 shares of buffer in the ring |
| Pull interval | About **half the expected time to fill the 10,000-share window** | At ~6 pool-bound shares/minute that window is ~28 hours, so pull about every **~14 hours** (the time to accumulate 5,000 new shares) |

The interval is operational, not a protocol timer. The Template Provider SHOULD use the pool channel’s expected share rate when known; **6 shares/minute** is the usual SRI / pool default if it is not.

If the TP waits longer than the remaining 5,000-share slack, the oldest shares fall out of the window before ever appearing in a hash list, and can then only be reported as unavailable.

### 8.2 `RequestRecentShareHashes` (Server → Client)

Ask for the hashes of the most recent shares still in the window.

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | Echoed in the `ShareHashes` reply. Unique among in-flight requests on this TDP session. |
| count | U32 | Maximum number of hashes to return. MUST be ≥ 1. MUST NOT exceed 10,000. The Template Provider SHOULD send `5000`. |

The JDC MUST reply with `ShareHashes` for that `request_id`, containing the hashes of the `count` most recently appended shares (all of them if the window holds fewer), in any order. "Most recently appended" is window insertion order — nothing about the hash values themselves.

### 8.3 `ShareHashes` (Client → Server)

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | MUST equal the `request_id` of the `RequestRecentShareHashes` this answers. |
| share_hashes | SEQ0_64K[U256] | Share hashes (§6.2) of the most recently appended shares, in any order. MAY be empty (empty window). |

At 32 bytes per hash, 5,000 hashes are about **160 KB**.

### 8.4 `RequestShares` (Server → Client)

Fetch specific shares by hash.

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | Echoed in the `Shares` reply. Unique among in-flight requests on this TDP session. |
| share_hashes | SEQ0_64K[U256] | Share hashes to return. MUST NOT be empty. MUST NOT contain more than 10,000 entries. |

The JDC MUST reply with `Shares` for that `request_id`. Each requested hash still in the window MUST appear as a record in `shares`. Each requested hash not in the window MUST appear in `unavailable_share_hashes`. The JDC MUST NOT invent shares.

### 8.5 `Shares` (Client → Server)

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | MUST equal the `request_id` of the `RequestShares` this answers. |
| shares | SEQ0_64K[ShareRecord] | Records still available. MAY be empty. |
| unavailable_share_hashes | SEQ0_64K[U256] | Requested hashes that have aged out or were never stored. |

`ShareRecord` is the same payload as `SubmitSharesAssociated` — it carries no explicit id:

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| template_id | U64 | Template this share was mined on. |
| job_id | U32 | Pool-bound job id. Informational. |
| nonce | U32 | Header nonce. |
| ntime | U32 | Header nTime. |
| version | U32 | Full nVersion field. |
| extranonce | B0_32 | The pool-bound `SubmitSharesExtended.extranonce`. MUST be exactly `NewAssociatedMiningJob.extranonce_size` bytes for this `template_id`. |

The TP identifies each returned record by recomputing its share hash during verification (§5.4). Every record returned for a `RequestShares` MUST hash to one of the requested values; the TP MUST discard (and MUST NOT count) any record that does not.

At ~36–46 bytes per record, a full 5,000-record fetch is about **0.2 MB**.

### 8.6 Example (lazy)

```text
TP  → JDC:  RequestRecentShareHashes { request_id=1, count=5000 }
JDC → TP:   ShareHashes { request_id=1, share_hashes=[… 5000 hashes …] }

TP diffs the list against its store: 4,970 hashes match records already fetched
    on a previous cycle; 30 are new.

TP  → JDC:  RequestShares { request_id=2, share_hashes=[… the 30 hashes …] }
JDC → TP:   Shares { request_id=2, shares=[… 30 ShareRecords …], unavailable=[] }

TP verifies each record (§5.4); each computed hash matches a requested one.
```

If one of the requested hashes had already fallen out of the 10,000 window by fetch time:

```text
JDC → TP:   Shares { request_id=2, shares=[… 29 ShareRecords …], unavailable=[H_1901] }
```

### 8.7 Example (full)

Same two steps, used as a check on the live stream:

```text
TP  → JDC:  RequestRecentShareHashes { request_id=7, count=5000 }
JDC → TP:   ShareHashes { request_id=7, share_hashes=[… 5000 hashes …] }

TP diffs against the hashes of records received live via SubmitSharesAssociated:
    two hashes appear in the list but never arrived (dropped frames).

TP  → JDC:  RequestShares { request_id=8, share_hashes=[H_a, H_b] }
JDC → TP:   Shares { request_id=8, shares=[ShareRecord a, ShareRecord b], unavailable=[] }
```

---

## 9. End-to-end flow

```text
1. JDC ↔ Template Provider: TDP SetupConnection
   JDC → TP: RequestExtensions[0x0003]
   TP → JDC: RequestExtensions.Success
   JDC → TP: CoinbaseOutputConstraints
   TP → JDC: SetShareMirroringMode { mode = lazy | full }

2. Template Provider → JDC:  NewTemplate(template_id=T)
   Template Provider → JDC:  SetNewPrevHash(...)

3. JDC builds work from template T
   JDC → Mining Device:           NewExtendedMiningJob(...)          // stock Mining
   JDC → Template Provider:       NewAssociatedMiningJob(template_id=T, ...)

4. Mining Device hashes; on a share that meets the pool target:
   Mining Device → JDC:           SubmitSharesExtended(...)          // stock Mining
   JDC → Pool:                    SubmitSharesExtended(...)
   JDC stores the ShareRecord in the 10,000-share window under its share hash H
   if full:
     JDC → Template Provider:     SubmitSharesAssociated(template_id=T, ...)

5. Periodically (~half the 10k-window time):
   TP  → JDC:  RequestRecentShareHashes { count=5000 }
   JDC → TP:   ShareHashes { share_hashes=[…] }
   TP diffs the hash list against the shares it holds;
   for every hash it lacks:
     TP  → JDC:  RequestShares { share_hashes=[…] }
     JDC → TP:   Shares { shares and/or unavailable_share_hashes }

6. On network-valid block (unchanged base TDP):
   JDC → Template Provider:       SubmitSolution(template_id=T, ...)
```

`SubmitSolution` remains the block path. This extension does **not** replace it; it adds a share-rate audit path.

---

## 10. Open issues / future knobs

These are intentionally **not** normative in this draft; they are the natural next design choices.

### 10.1 Dual resolution (pool fine / TP coarse)

Desire from related discussion: Pool keeps an easy `SetTarget` (high share rate); Template Provider only needs a rough hashrate meter.

Verification (§5.4) uses the pool's own `share_target`, so this draft is **full pool resolution**. A future knob could let the TP declare a harder **attestation target** (e.g. alongside `SetShareMirroringMode`) and have the JDC store / mirror / return only shares meeting it — fewer, higher-difficulty proofs for the same attested hashrate.

Because reconciliation is a set difference against the JDC-reported hash list (§8), such a filter composes cleanly: filtered-out shares simply never appear in `ShareHashes`, and loss detection over the attested subset still works. The remaining design work is the target-advertisement message and its replacement rules. Alternatively, the TP can already downsample locally at zero protocol cost: verify a random subset of fetched records instead of all of them.

Lazy vs full is about **delivery**, not difficulty. Dual resolution would apply on top of either mode.

### 10.2 Downstream vs pool job ids

JDC typically uses different `job_id` / `channel_id` values toward miners vs toward the Pool. This draft keys TP state by `template_id` and defines `job_id` on these messages as the **pool-bound** id. Downstream ids never appear on TDP. The share id is the share hash (§6.2); Mining `sequence_number`s never appear on TDP.

### 10.3 Window persistence and reconnect

Share-hash ids are unique across restarts and fallbacks by construction (§6.2), so no epoch or counter rules are needed. What remains TBD: whether the JDC persists the window across a process restart (if not, up to 10,000 shares of auditable history are lost with it — including across a pool fallback if the window is owned by connection-scoped state); and TDP reconnect (association state dies with the TDP session, so after reconnect the JDC MUST re-send `NewAssociatedMiningJob` for live templates before further shares, and a reconnecting TP re-pulls whatever is still in the window).

### 10.4 Several pool-bound channels

The window is specified for one pool-bound Mining channel. If a JDC submits to more than one, the TP needs a channel discriminator or one window per channel.
