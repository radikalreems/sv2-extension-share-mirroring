# Stratum V2 Extension (Draft): Share Mirroring to the Template Provider

**Status:** Draft / not assigned an official extension type  
**Provisional extension type:** `0x0003` (placeholder until registered in the SV2 extension registry)  
**Protocols involved:** Template Distribution Protocol (TDP). This extension **defines new messages** that travel on the TDP connection; it does not reuse Mining Protocol message types.  
**Depends on:** [Extensions Negotiation (`0x0001`)](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md). TDP does not implement this today; see [issue-tdp-extensions-negotiation.md](./issue-tdp-extensions-negotiation.md).

Terms like "MUST," "MUST NOT," "REQUIRED," "SHOULD," and "MAY" follow [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

**Roles in this document**

| Role | SV2 / SRI name | In this extension |
|------|----------------|-------------------|
| Template supplier (TDP server) | **Template Provider (TP)** | Sends `NewTemplate`; receives job association + mirrored shares |
| Mid-tier (TDP client, MP aggregator) | **Job Declarator Client (JDC)** | Builds jobs from TP templates, serves Mining Devices, submits shares to the Pool, mirrors audit traffic to the TP |
| Upstream payout endpoint | **Pool** | Receives `SubmitSharesExtended` for accounting |
| Downstream hashers | **Mining Device** | Receive jobs / submit shares on Mining Protocol |

---

## 0. Abstract

This document describes a Stratum V2 **extension** that lets a **Template Provider (TP)** audit that their templates are actually being mined, in real time, without changing core SV2 Mining or Template Distribution message layouts for devices that do not opt in.

This extension **defines two new messages**, `NewAssociatedMiningJob` and `SubmitSharesAssociated`. They travel on the **Template Distribution Protocol (TDP)** connection between the TP and a **Job Declarator Client (JDC)**. They are **not** Mining Protocol messages and MUST NOT be sent on a Mining connection.

On that TDP connection the extension does two things:

1. **Associate jobs with templates** — When the JDC begins mining a job derived from a TP template, it sends a `NewAssociatedMiningJob` message on TDP. That message carries the original `template_id` plus the job parameters needed to interpret later shares.
2. **Mirror share proofs** — For every `SubmitSharesExtended` the JDC submits toward a Pool, it also sends a `SubmitSharesAssociated` message to the TP on the same TDP connection. That message carries `template_id` and the proof fields.

Mining Devices and Pools continue to receive ordinary Mining Protocol job notifications (`NewExtendedMiningJob`, `SubmitSharesExtended`, etc.). Core SV2 behavior for those devices is unchanged.

---

## 1. Motivation

A party that purchases hashpower today typically has to participate in the **whole SV2 flow**: Template Distribution toward a Template Provider, Job Declaration / Mining toward a Pool, and the Job Declarator Client that sits between them. The buyer cannot simply just handle providing templates.

This extension assists that the buyer be **delegated to the Template Provider**. The purchaser runs or controls the TP; the JDC still talks to the Pool and to Mining Devices as usual. What the TP needs is **proof** that its templates are actually being mined.

Stock TDP only returns **`SubmitSolution`** (network-valid blocks). That is not only too rare to use as proof, especially for solo-mining, but the entity that sources the hashrate may also source different pools dynamically tailored to the client's needs. This would make it non-trivial for the buyer to retrieve proof of work on their templates. Shares already exist at the JDC so this extension routes those over to the TP via TDP.

Share **resolution** at the TP does not need to match the Pool. The Pool wants a high share rate for accurate hashrate and payouts. The TP only needs enough shares to be confident that a template is being worked. Because mirroring every Pool-bound share onto TDP costs **bandwidth**, that TP-facing stream MAY be **reduced** (a coarser share target, or a sampled subset) relative to what the JDC submits to the Pool. This draft still specifies a 1:1 mirror as the default (§6.3); a dual-resolution / sampled mode is left open (§8.1).

---

## 2. Design

Under stock SV2, the Template Provider identifies work by `template_id` on Template Distribution (`NewTemplate`). The Pool, JDC, and Mining Devices identify work by `job_id` on Mining (`NewMiningJob` / `NewExtendedMiningJob`). Those identifiers live in different namespaces and are not linked by the base protocols. Stock TDP only returns `SubmitSolution` (network-valid blocks). A Template Provider therefore cannot, from Mining share traffic alone, prove which of **their** templates produced a given share.

**`NewAssociatedMiningJob`** closes that gap. When the JDC begins mining a job derived from a TP template, it sends this message on the TDP connection. The message carries the original `template_id` plus the job parameters the TP needs to interpret later shares: the pool-bound `job_id`, merkle path, coinbase prefix and suffix, and related header fields. The TP stores that state keyed by `template_id`.

**`SubmitSharesAssociated`** is the share-mirroring path. For every `SubmitSharesExtended` the JDC submits toward a Pool on that work, it also sends `SubmitSharesAssociated` to the TP on the same TDP connection. That message carries `template_id` and the proof fields (`nonce`, `ntime`, `version`, `extranonce`). The TP maps each share to a template via `template_id`.

---

## 3. Extension overview

When this extension is successfully negotiated on the **JDC ↔ Template Provider TDP connection**:

1. The JDC MUST, for every TP `NewTemplate` from which it is mining work that it also submits toward a Pool, send the TP a `NewAssociatedMiningJob` (§5) on that TDP connection.
2. The JDC MUST, for every `SubmitSharesExtended` that it submits (or accepts for submission) to the Pool for that work, send the TP a `SubmitSharesAssociated` (§6) on that TDP connection. The TP maps each share to a template via `template_id`.

`template_id` is a native field on these messages. A TLV is not required.

### 3.1 Provisional identifiers

| Item | Value | Notes |
|------|-------|-------|
| Extension type | `0x0003` | **Placeholder** — replace when registered |
| `NewAssociatedMiningJob` `msg_type` | `0x00` | Local to `extension_type = 0x0003` |
| `SubmitSharesAssociated` `msg_type` | `0x01` | Local to `extension_type = 0x0003` |

### 3.2 Message types

All messages defined by this extension MUST have `extension_type = 0x0003` in the frame header (the identifier of the extension that defined their non-TLV structure). The `channel_msg` bit MUST be unset. These messages are sent only on a TDP connection that has successfully negotiated this extension.

| Message Type (8-bit) | channel_msg bit | Message Name | Direction |
|----------------------|-----------------|--------------|-----------|
| 0x00 | 0 | NewAssociatedMiningJob | JDC → Template Provider |
| 0x01 | 0 | SubmitSharesAssociated | JDC → Template Provider |

The 8-bit `msg_type` values are local to this extension. They MAY overlap core `msg_type` bytes; the pair `(extension_type, msg_type)` identifies the message. See SV2 spec §3.4.1 example 4.

Peers that did not negotiate this extension MUST NOT send these messages. A peer that receives an unknown `extension_type` with `channel_msg` unset MUST ignore the frame (SV2 spec §3.4.1).

### 3.3 Relation to Mining Protocol messages

`NewAssociatedMiningJob` and `SubmitSharesAssociated` are **not** `NewExtendedMiningJob` and `SubmitSharesExtended`. Those are core **Mining Protocol** messages: `extension_type = 0x0000`, `channel_msg = 1`, defined Server→Client and Client→Server respectively on a Mining connection.

TDP has no channels, and the spec requires the `channel_msg` bit to be unset on that protocol. Core protocols may only grow via TLV on *existing* messages of that protocol. Reusing Mining frames on TDP would therefore be illegal even if the 8-bit `msg_type` values do not collide with TDP’s `0x70–0x76` range.

This extension instead follows the same pattern as [Extensions Negotiation (`0x0001`)](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md): it **introduces new messages**. Downstream Mining Devices still receive ordinary `NewExtendedMiningJob` / `NewMiningJob` on the Mining Protocol. The Pool still receives ordinary `SubmitSharesExtended` on the Mining Protocol.

---

## 4. Negotiation

Negotiation uses extension `0x0001` (Extensions Negotiation) on the **TDP** connection between the JDC (client) and the Template Provider (server).

1. `SetupConnection` / `SetupConnection.Success` for Template Distribution.
2. Client sends `RequestExtensions` including provisional `0x0003`.
3. Server responds `RequestExtensions.Success` (supported) or `RequestExtensions.Error`.
4. Client sends TDP `CoinbaseOutputConstraints` (stock TDP; after negotiation so that `RequestExtensions` remains the first protocol-specific message, per `0x0001`).

Only after Success may the JDC send `NewAssociatedMiningJob` or `SubmitSharesAssociated` on that TDP link.

Mining Devices and the Pool connection do **not** need this extension for the TP audit path to work.

---

## 5. `NewAssociatedMiningJob` (Client → Server)

Sent on the TDP connection. Informs the Template Provider of the mining-job parameters the JDC is using for work derived from one of the TP’s templates, so later `SubmitSharesAssociated` proofs can be interpreted.

The JDC SHOULD send one `NewAssociatedMiningJob` per template it is mining, not one copy per downstream channel.

### 5.1 Fields

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| template_id | U64 | The `template_id` from the TP’s `NewTemplate` that this job was built from. MUST refer to a `NewTemplate` previously sent on this TDP session (reconnect rules are TBD; see §8). |
| job_id | U32 | The `job_id` the JDC uses on pool-bound `SubmitSharesExtended` for this work. Informational; TP MUST key state by `template_id`, not by `job_id` alone. |
| min_ntime | OPTION[U32] | Same meaning as `NewExtendedMiningJob.min_ntime`: empty means a future job awaiting TDP `SetNewPrevHash` for this `template_id`. |
| version | U32 | Header version used for this job (BIP323 bits as in Mining Protocol). |
| version_rolling_allowed | BOOL | Same meaning as `NewExtendedMiningJob.version_rolling_allowed`. |
| merkle_path | SEQ0_255[U256] | Merkle path hashes ordered from deepest. MUST match `NewTemplate.merkle_path` for `template_id`. |
| coinbase_tx_prefix | B0_64K | Prefix of the coinbase transaction used for this job (BIP141 fields stripped, same rule as `NewExtendedMiningJob`). |
| coinbase_tx_suffix | B0_64K | Suffix of the coinbase transaction used for this job. |

Coinbase reconstruction for a later share is:

```text
coinbase = coinbase_tx_prefix + extranonce_prefix + extranonce + coinbase_tx_suffix
```

`extranonce_prefix` is **not** in this message (it is a Mining-channel property). How the TP obtains it is an open issue (§8.3). Implementations MUST NOT assume prefix length is zero.

### 5.2 When to send

The JDC MUST send `NewAssociatedMiningJob` after it has derived pool-bound work from a `NewTemplate` and before (or together with) the first `SubmitSharesAssociated` for that `template_id`.

If the JDC later changes coinbase prefix/suffix or merkle path for the same `template_id` (for example after `SetExtranoncePrefix` toward miners, or a new declared job), it MUST send a replacement `NewAssociatedMiningJob` for that `template_id`. The latest message supersedes earlier ones.

### 5.3 Example

Template Provider previously sent `NewTemplate { template_id = 42, ... }`.  
JDC’s pool-bound job id is `7`.

```text
JDC → Template Provider (TDP):
  frame: extension_type=0x0003, msg_type=0x00, channel_msg=0
  NewAssociatedMiningJob {
    template_id=42,
    job_id=7,
    merkle_path, coinbase_tx_prefix, coinbase_tx_suffix, ...
  }
```

Template Provider stores job parameters under `template_id 42`.

---

## 6. `SubmitSharesAssociated` (Client → Server)

### 6.1 Rule

For every `SubmitSharesExtended` the JDC submits to the Pool on work that was associated under §5, the JDC MUST also send a `SubmitSharesAssociated` to the TP on the TDP connection, in real time (or with bounded delay agreed operationally).

`SubmitSharesAssociated` omits Mining `channel_id` and `sequence_number` (those are pool accounting). It carries `template_id` so the TP does not have to join through Mining `job_id`.

### 6.2 Fields

| Field Name | Data Type | Description |
|------------|-----------|-------------|
| template_id | U64 | MUST equal the `template_id` of the `NewAssociatedMiningJob` this share is for. |
| job_id | U32 | MUST equal `NewAssociatedMiningJob.job_id` for that `template_id` (the pool-bound job id). Informational. |
| nonce | U32 | Header nonce. |
| ntime | U32 | Header nTime. Same bounds as Mining `SubmitSharesExtended.ntime` relative to the latest TDP `SetNewPrevHash` for this template. |
| version | U32 | Full nVersion field. |
| extranonce | B0_32 | Bytes inserted between `extranonce_prefix` and `coinbase_tx_suffix` (same role as `SubmitSharesExtended.extranonce`). |

### 6.3 What “every” means

“Every share submitted to the Pool” means every share the JDC treats as a pool submission for that work—including shares that the Pool later rejects—**unless** both parties explicitly negotiate a filter (see §8 Open issues).

Default in this draft: **full pool-bound resolution** mirrored to the Template Provider (maximizes auditability; higher bandwidth).

### 6.4 Example

```text
Mining Device → JDC:           SubmitSharesExtended { job_id=…, nonce, ntime, version, extranonce, ... }
JDC → Pool:                    SubmitSharesExtended { ... pool channel_id / job_id / sequence_number ... }
JDC → Template Provider:       SubmitSharesAssociated { template_id=42, job_id=7, nonce, ntime, version, extranonce }
                               frame: extension_type=0x0003, msg_type=0x01, channel_msg=0
```

Template Provider: `template_id=42` → job parameters from §5 → verify PoW (verification completeness is §8.3).

---

## 7. End-to-end flow

```text
1. JDC ↔ Template Provider: TDP SetupConnection
   JDC → TP: RequestExtensions[0x0003]
   TP → JDC: RequestExtensions.Success
   JDC → TP: CoinbaseOutputConstraints

2. Template Provider → JDC:  NewTemplate(template_id=T)
   Template Provider → JDC:  SetNewPrevHash(...)

3. JDC builds work from template T
   JDC → Mining Device:           NewExtendedMiningJob(...)          // stock Mining
   JDC → Template Provider:       NewAssociatedMiningJob(template_id=T, ...)  // this extension

4. Mining Device hashes; on share:
   Mining Device → JDC:           SubmitSharesExtended(...)          // stock Mining
   JDC → Pool:                    SubmitSharesExtended(...)          // payout resolution
   JDC → Template Provider:       SubmitSharesAssociated(template_id=T, ...)  // this extension

5. On network-valid block (unchanged base TDP):
   JDC → Template Provider:       SubmitSolution(template_id=T, ...) // still required for block propagation
```

`SubmitSolution` remains the block path. This extension does **not** replace it; it adds a share-rate audit path.

---

## 8. Open issues / future knobs

These are intentionally **not** normative in this draft; they are the natural next design choices.

### 8.1 Dual resolution (pool fine / TP coarse)

Desire from related discussion: Pool keeps an easy `SetTarget` (high share rate); Template Provider only needs a rough hashrate meter.

Possible additive rules (future):

- Template Provider advertises an **attestation target** (harder than pool target), e.g. as a field on `NewAssociatedMiningJob` or a TP→JDC message.
- JDC only sends `SubmitSharesAssociated` for shares that also meet the attestation target; or
- JDC mirrors all shares but the TP samples locally.

### 8.2 Downstream vs pool job ids

JDC typically uses different `job_id` / `channel_id` values toward miners vs toward the Pool. This draft keys TP state by `template_id` and defines `job_id` on these messages as the **pool-bound** id. Downstream ids never appear on TDP.

### 8.3 Share verification completeness

`NewAssociatedMiningJob` does not carry `extranonce_prefix` / `extranonce_size`. Without those (and without an attestation target, §8.1), the TP cannot reconstruct the coinbase or check share difficulty. Likely next fields on `NewAssociatedMiningJob`: `extranonce_prefix` (B0_32) and `extranonce_size` (U16), plus a `U256` attestation target either there or in a TP→JDC message.
