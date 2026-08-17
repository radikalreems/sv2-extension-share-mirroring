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
2. **Retain pool-bound shares** — The JDC keeps a rolling window of the last 10,000 shares it submitted toward the Pool, keyed by the pool-bound `sequence_number`.
3. **Let the TP retrieve those shares** — The TP pulls recent shares in batch and, if any `sequence_number`s are missing, asks for them by id. In **full** audit mode the JDC also mirrors each pool-bound share to the TP as it is submitted.

Mining Devices and Pools continue to receive ordinary Mining Protocol job notifications (`NewExtendedMiningJob`, `SubmitSharesExtended`, etc.). Core SV2 behavior for those devices is unchanged.

---

## 1. Motivation

A party that purchases hashpower today typically has to participate in the **whole SV2 flow**: Template Distribution toward a Template Provider, Job Declaration / Mining toward a Pool, and the Job Declarator Client that sits between them. The buyer cannot simply just handle providing templates.

This extension assists that the buyer be **delegated to the Template Provider**. The purchaser runs or controls the TP; the JDC still talks to the Pool and to Mining Devices as usual. What the TP needs is **proof** that its templates are actually being mined.

Stock TDP only returns **`SubmitSolution`** (network-valid blocks). That is not only too rare to use as proof, especially for solo-mining, but the entity that sources the hashrate may also source different pools dynamically tailored to the client's needs. This would make it non-trivial for the buyer to retrieve proof of work on their templates. Shares already exist at the JDC so this extension routes those over to the TP via TDP.

Share **resolution** at the TP does not need to match the Pool. The Pool wants a high share rate for accurate hashrate and payouts. The TP only needs enough shares to be confident that a template is being worked. Because a live 1:1 mirror of every pool-bound share onto TDP costs **bandwidth**, this extension has two modes (§3.3): **lazy** (batch pull only) and **full** (live mirror plus the same pull as a check). A coarser attestation target is left open (§10.1).

---

## 2. Design

Under stock SV2, the Template Provider identifies work by `template_id` on Template Distribution (`NewTemplate`). The Pool, JDC, and Mining Devices identify work by `job_id` on Mining (`NewMiningJob` / `NewExtendedMiningJob`). Those identifiers live in different namespaces and are not linked by the base protocols. Stock TDP only returns `SubmitSolution` (network-valid blocks). A Template Provider therefore cannot, from Mining share traffic alone, prove which of **their** templates produced a given share.

**`NewAssociatedMiningJob`** closes that gap. When the JDC begins mining a job derived from a TP template, it sends this message on the TDP connection. The message carries the original `template_id` plus everything the TP needs to interpret **and independently verify** later shares: the pool-bound `job_id`, merkle path, coinbase prefix and suffix, the pool-bound `extranonce_prefix` / `extranonce_size`, and the pool-bound `share_target`. The TP stores that state keyed by `template_id`. With it, every share record is a self-contained proof-of-work: the TP reconstructs the coinbase, derives the merkle root, assembles the header, and checks the hash against the target (§5.4). No trust in the JDC's counting is required.

The JDC already assigns a pool-bound `sequence_number` on each `SubmitSharesExtended` it sends to the Pool. That id is the handle for this extension’s share window and retrieval.

**`SubmitSharesAssociated`** is the live share-mirroring path (full mode only). It carries `template_id`, the pool-bound `sequence_number`, and the proof fields (`nonce`, `ntime`, `version`, `extranonce`). The TP maps each share to a template via `template_id`.

**Retrieval** is the batch path (both modes). The TP asks for the most recent shares, compares `sequence_number`s to what it already has, and requests any gaps by id. The JDC answers from its 10,000-share window.

---

## 3. Extension overview

When this extension is successfully negotiated on the **JDC ↔ Template Provider TDP connection**:

1. The JDC MUST, for every TP `NewTemplate` from which it is mining work that it also submits toward a Pool, send the TP a `NewAssociatedMiningJob` (§5) on that TDP connection.
2. The JDC MUST retain the last 10,000 pool-bound shares and their pool `sequence_number`s (§6).
3. The Template Provider MUST send `SetShareMirroringMode` (§4.1) selecting **lazy** or **full**.
4. In **full** mode, the JDC MUST, for every `SubmitSharesExtended` that it submits (or accepts for submission) to the Pool for that work, send the TP a `SubmitSharesAssociated` (§7) on that TDP connection.
5. In **lazy** mode, the JDC MUST NOT send unsolicited `SubmitSharesAssociated`. The TP obtains shares only via §8.
6. In **both** modes, the TP SHOULD periodically request the last 5,000 shares (§8) and MUST be able to request missing `sequence_number`s by id.

`template_id` is a native field on the share and job messages. A TLV is not required.

### 3.1 Provisional identifiers

| Item | Value | Notes |
|------|-------|-------|
| Extension type | `0x0003` | **Placeholder** — replace when registered |
| `NewAssociatedMiningJob` `msg_type` | `0x00` | Local to `extension_type = 0x0003` |
| `SubmitSharesAssociated` `msg_type` | `0x01` | Local to `extension_type = 0x0003` |
| `SetShareMirroringMode` `msg_type` | `0x02` | Local to `extension_type = 0x0003` |
| `RequestRecentShares` `msg_type` | `0x03` | Local to `extension_type = 0x0003` |
| `RequestShares` `msg_type` | `0x04` | Local to `extension_type = 0x0003` |
| `Shares` `msg_type` | `0x05` | Local to `extension_type = 0x0003` |

### 3.2 Message types

All messages defined by this extension MUST have `extension_type = 0x0003` in the frame header (the identifier of the extension that defined their non-TLV structure). The `channel_msg` bit MUST be unset. These messages are sent only on a TDP connection that has successfully negotiated this extension.

| Message Type (8-bit) | channel_msg bit | Message Name | Direction |
|----------------------|-----------------|--------------|-----------|
| 0x00 | 0 | NewAssociatedMiningJob | JDC → Template Provider |
| 0x01 | 0 | SubmitSharesAssociated | JDC → Template Provider |
| 0x02 | 0 | SetShareMirroringMode | Template Provider → JDC |
| 0x03 | 0 | RequestRecentShares | Template Provider → JDC |
| 0x04 | 0 | RequestShares | Template Provider → JDC |
| 0x05 | 0 | Shares | JDC → Template Provider |

The 8-bit `msg_type` values are local to this extension. They MAY overlap core `msg_type` bytes; the pair `(extension_type, msg_type)` identifies the message. See SV2 spec §3.4.1 example 4.

Peers that did not negotiate this extension MUST NOT send these messages. A peer that receives an unknown `extension_type` with `channel_msg` unset MUST ignore the frame (SV2 spec §3.4.1).

There is no Success/Error on `NewAssociatedMiningJob` or `SubmitSharesAssociated`. Retrieval uses `request_id` on `RequestRecentShares` / `RequestShares` / `Shares` only.

### 3.3 Audit modes

`0x0001` only negotiates extension ids, not parameters. After `RequestExtensions.Success`, the Template Provider selects the mode with `SetShareMirroringMode` (§4.1). A JDC that implements this extension MUST implement both modes.

| `mode` | Name | Live `SubmitSharesAssociated` | Batch retrieval (§8) |
|--------|------|-------------------------------|----------------------|
| `0x00` | **Lazy** | MUST NOT send | MUST: TP pulls last 5,000; then asks for any missing ids |
| `0x01` | **Full** | MUST send every pool-bound share | MUST: same pull as a reconciliation check |

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

Shares are verified against the association in effect when the JDC submitted them; the TP SHOULD tolerate a small reordering window around a replacement (a share validated under the previous target/prefix may arrive just after the replacement message).

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

Shares failing any step MUST NOT be counted as evidence of work. If the header hash is also ≤ the block target from `SetNewPrevHash`, the TP should additionally expect a stock TDP `SubmitSolution` for this template.

**Why a self-reported target is sound.** `share_target` comes from the JDC, but the JDC cannot inflate the work it proves by misreporting it. Fabricating a record that passes step 5 costs on the order of `2^256 / share_target` hash operations, so each verified share attests the difficulty implied by the reported target — regardless of what the pool's actual target was. Reporting an *easier* target than the pool's only lowers the work per share the JDC can claim. The TP's attested-hashrate floor is the sum of `difficulty(share_target)` over verified shares per unit of time, and honest share rates at a given target are also externally sanity-checkable (a pool-tuned target yields on the order of a few shares per minute per upstream channel).

---

## 6. JDC share window

The JDC MUST keep a rolling window of the last **10,000** shares it submitted (or accepted for submission) toward the Pool on the pool-bound Mining channel used for this TDP work, together with each share’s pool-bound `sequence_number`.

This window is **pool-bound only**. Downstream miner shares that do not meet the pool target, and are not forwarded, MUST NOT be stored. Downstream `sequence_number`s MUST NOT be used as keys.

This draft assumes a single pool-bound Mining channel for that JDC (the usual JDC shape). Multiple upstream channels are TBD (§10.4).

### 6.1 What is stored

For each retained share the JDC MUST be able to produce a `ShareRecord` (§8.4): `sequence_number`, `template_id`, `job_id`, `nonce`, `ntime`, `version`, `extranonce`.

The JDC SHOULD append to the window when (or immediately after) it forwards the share to the Pool, not on the miner-submit hot path. Evict the oldest entry when the window exceeds 10,000.

At typical pool vardiff (~6 shares/minute on the upstream channel) 10,000 shares is on the order of **28 hours** and about **0.4–0.5 MB** of share payloads.

### 6.2 Identifier

`sequence_number` is the U32 the JDC puts on the pool-bound `SubmitSharesExtended`. It is the only share id this extension uses. The Template Provider treats it as a monotonically increasing id on that pool channel (wrap-around is possible; see §10.3).

---

## 7. `SubmitSharesAssociated` (Client → Server)

Live, unsolicited share mirror. **Full mode only.**

### 7.1 Rule

When `mode = 0x01` (full), for every `SubmitSharesExtended` the JDC submits to the Pool on work that was associated under §5, the JDC MUST also send a `SubmitSharesAssociated` to the TP on the TDP connection, in real time (or with bounded delay agreed operationally).

When `mode = 0x00` (lazy), the JDC MUST NOT send this message. The TP uses §8.

`SubmitSharesAssociated` omits Mining `channel_id` (pool accounting). It carries the pool-bound `sequence_number` so the TP can detect gaps and request them. It carries `template_id` so the TP does not have to join through Mining `job_id`.

### 7.2 Fields

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| sequence_number | U32 | MUST equal the pool-bound `SubmitSharesExtended.sequence_number` for this share. |
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
JDC → Pool:                    SubmitSharesExtended { sequence_number=1001, ... }
JDC → Template Provider:       SubmitSharesAssociated { sequence_number=1001, template_id=42, job_id=7, nonce, ntime, version, extranonce }
                               frame: extension_type=0x0003, msg_type=0x01, channel_msg=0
```

Template Provider: `template_id=42` → association from §5 → verify per §5.4. Record `sequence_number=1001`.

---

## 8. Share retrieval (both modes)

The Template Provider pulls from the JDC window. This is the **only** share path in lazy mode and the **reconciliation** path in full mode.

There is no Success/Error. The JDC answers with `Shares` carrying the same `request_id`.

### 8.1 Recommended pull

| Knob | Value | Why |
|------|-------|-----|
| JDC window | 10,000 shares | Slack if a pull is late |
| TP request size | 5,000 shares (half the window) | Leaves ~5,000 shares of buffer in the ring |
| Pull interval | About **half the expected time to fill the 10,000-share window** | At ~6 pool-bound shares/minute that window is ~28 hours, so pull last 5,000 about every **~14 hours** (the time to accumulate 5,000 new shares) |

The interval is operational, not a protocol timer. The Template Provider SHOULD use the pool channel’s expected share rate when known; **6 shares/minute** is the usual SRI / pool default if it is not.

If the TP waits longer than the remaining 5,000-share slack, the oldest shares fall out of the window and can only be reported as unavailable.

After each `Shares` reply, the TP MUST compare the returned `sequence_number`s to the set it already has (previous pulls and, in full mode, live `SubmitSharesAssociated`). Any id in the returned range, or in the span since the last successful pull, that the TP does not have MUST be requested with `RequestShares`.

### 8.2 `RequestRecentShares` (Server → Client)

Ask for the most recent shares still in the window, newest retained share first in meaning (the JDC MAY return them in any order; the TP sorts by `sequence_number`).

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | Echoed in the `Shares` reply. Unique among in-flight requests on this TDP session. |
| count | U32 | Maximum number of shares to return. MUST be ≥ 1. MUST NOT exceed 10,000. The Template Provider SHOULD send `5000`. |

The JDC MUST reply with `Shares` for that `request_id`. If the window holds fewer than `count` shares, return all of them. `unavailable_sequence_numbers` MUST be empty (this request does not name ids).

### 8.3 `RequestShares` (Server → Client)

Ask for specific pool-bound `sequence_number`s (gap fill).

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | Echoed in the `Shares` reply. Unique among in-flight requests on this TDP session. |
| sequence_numbers | SEQ0_64K[U32] | Pool-bound ids to return. MUST NOT be empty. MUST NOT contain more than 10,000 entries. |

The JDC MUST reply with `Shares` for that `request_id`. Each requested id still in the window MUST appear in `shares`. Each requested id not in the window MUST appear in `unavailable_sequence_numbers`. The JDC MUST NOT invent shares.

### 8.4 `Shares` (Client → Server)

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| request_id | U32 | MUST equal the `request_id` of the request this answers. |
| shares | SEQ0_64K[ShareRecord] | Records still available. MAY be empty. |
| unavailable_sequence_numbers | SEQ0_64K[U32] | Requested ids that have aged out or were never stored. Empty after `RequestRecentShares`. |

`ShareRecord` is the same payload as `SubmitSharesAssociated`:

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| sequence_number | U32 | Pool-bound `sequence_number`. |
| template_id | U64 | Template this share was mined on. |
| job_id | U32 | Pool-bound job id. Informational. |
| nonce | U32 | Header nonce. |
| ntime | U32 | Header nTime. |
| version | U32 | Full nVersion field. |
| extranonce | B0_32 | The pool-bound `SubmitSharesExtended.extranonce`. MUST be exactly `NewAssociatedMiningJob.extranonce_size` bytes for this `template_id`. |

At ~40–50 bytes per record, 5,000 shares are about **0.2–0.25 MB**.

### 8.5 Example (lazy)

```text
TP  → JDC:  RequestRecentShares { request_id=1, count=5000 }
JDC → TP:   Shares { request_id=1, shares=[… 5000 ShareRecords …], unavailable=[] }

TP notices sequence_number 1842 and 1901 are in the expected span but not in this batch
    and not from a previous pull.

TP  → JDC:  RequestShares { request_id=2, sequence_numbers=[1842, 1901] }
JDC → TP:   Shares { request_id=2, shares=[ShareRecord 1842, ShareRecord 1901], unavailable=[] }
```

If `1901` had already fallen out of the 10,000 window:

```text
JDC → TP:   Shares { request_id=2, shares=[ShareRecord 1842], unavailable=[1901] }
```

### 8.6 Example (full)

Same pull as §8.5, used as a check. Live `SubmitSharesAssociated` already carried most ids. `RequestShares` is only for holes in the live stream.

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
   JDC → Pool:                    SubmitSharesExtended(sequence_number=N, ...)
   JDC stores ShareRecord N in the 10,000-share window
   if full:
     JDC → Template Provider:     SubmitSharesAssociated(sequence_number=N, template_id=T, ...)

5. Periodically (~half the 10k-window time; last 5,000 shares):
   TP  → JDC:  RequestRecentShares { count=5000 }
   JDC → TP:   Shares { … }
   if any sequence_number missing:
     TP  → JDC:  RequestShares { sequence_numbers=[…] }
     JDC → TP:   Shares { shares and/or unavailable_sequence_numbers }

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

Any such filter must also define how it interacts with gap detection (§8.1): filtered-out shares still consume pool `sequence_number`s, so the TP could no longer treat every missing id as a lost share. That interaction is why this stays a future knob rather than part of this revision. Alternatively, the TP can already downsample locally at zero protocol cost: verify a random subset of pulled records instead of all of them.

Lazy vs full is about **delivery**, not difficulty. Dual resolution would apply on top of either mode.

### 10.2 Downstream vs pool job ids

JDC typically uses different `job_id` / `channel_id` values toward miners vs toward the Pool. This draft keys TP state by `template_id` and defines `job_id` on these messages as the **pool-bound** id. Downstream ids never appear on TDP. Share ids are the pool-bound `sequence_number` only.

### 10.3 `sequence_number` wrap and reconnect

U32 wrap-around, JDC restart (sequence factory reset), and TDP reconnect are TBD. A reconnecting TP can `RequestRecentShares` for whatever is still in the window; shares older than 10,000 are gone. Whether the JDC persists the window across process restart is TBD.

### 10.4 Several pool-bound channels

The window is specified for one pool-bound Mining channel. If a JDC submits to more than one, the TP needs a channel discriminator or one window per channel.
