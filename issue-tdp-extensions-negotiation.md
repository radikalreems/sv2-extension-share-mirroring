# Draft issue: Support extensions on Template Distribution (`0x0001` on TDP)

**Repos:** `sv2-tp`, `sv2-apps` (jd-client; pool `Sv2Tp` same client gap)

---

## Motivation

We need to support extensions within TDP.

[Spec §3.4.2](https://github.com/stratum-mining/sv2-spec/blob/main/03-Protocol-Overview.md): any implementation that uses extensions MUST first implement [`0x0001`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md) and negotiate each later extension before sending its messages. Identity is `(extension_type, msg_type)` ([§3.4.1](https://github.com/stratum-mining/sv2-spec/blob/main/03-Protocol-Overview.md)).

Mining already negotiates ([`0x0002`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0002-worker-specific-hashrate-tracking.md)); the JDC ↔ TP hop does not. Both ends must speak `0x0001` or there is no negotiated TDP session.

---

## Problem

Extension negotiation is not implemented on TDP. The hop goes `SetupConnection` → `CoinbaseOutputConstraints` and never does `RequestExtensions`.

**TP** ([`messages.h`](https://github.com/stratum-mining/sv2-tp/blob/master/src/sv2/messages.h), [`ProcessSv2Message`](https://github.com/stratum-mining/sv2-tp/blob/master/src/sv2/connman.cpp)) is stock TDP only. It does not read `extension_type` and does not handle [`0x0001` messages](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md). A `RequestExtensions` frame (`extension_type = 0x0001`, `msg_type = 0x00`) is treated as Common [`SetupConnection`](https://github.com/stratum-mining/sv2-spec/blob/main/08-Message-Types.md) (`msg_type = 0x00`) and dropped after the handshake is already confirmed. No `.Success` / `.Error` is sent. Spec assumed unsupported servers would ignore unknown `extension_type` ([§3.4.1](https://github.com/stratum-mining/sv2-spec/blob/main/03-Protocol-Overview.md)); TP misclassifies instead.

**JDC** never asks. [`Sv2Tp::setup_connection`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/template_receiver/sv2_tp/mod.rs) is SetupConnection only; [`get_negotiated_extensions_with_server`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/template_receiver/sv2_tp/message_handler.rs) returns `[]`. Mining already sends `RequestExtensions` ([Pool upstream](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/upstream/mod.rs)) and answers it ([downstream](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/downstream/extensions_message_handler.rs)). Config [`required_extensions`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/config.rs) is wired to those hops, not to `Sv2Tp`. [`ChannelManager::start`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/channel_manager/mod.rs) then sends `CoinbaseOutputConstraints`. The TP pipe is `TemplateDistributionOwned` only, so a TP `.Success` would be dropped anyway.

This issue is TDP `0x0001` on **both** sides. [`0x0002`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0002-worker-specific-hashrate-tracking.md) is Mining-only; TP does not need it.

---

## Design

Reuse [`0x0001`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md) as-is. Do not invent a TDP-specific negotiation. `stratum` already has the types ([`extensions-sv2`](https://github.com/stratum-mining/stratum/blob/main/sv2/extensions-sv2/src/lib.rs), [`parsers-sv2`](https://github.com/stratum-mining/stratum/blob/main/sv2/parsers-sv2/src/lib.rs), [`handlers-sv2`](https://github.com/stratum-mining/stratum/blob/main/sv2/handlers-sv2/src/extensions.rs)).

### Sequence

[`0x0001` §4](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md):

```text
SetupConnection / SetupConnection.Success
RequestExtensions / Success or Error          ← before any TDP message
CoinbaseOutputConstraints                     ← first TDP message
… later TDP extension messages only if Success listed them
```

If the client does not want extensions, skip `RequestExtensions` and keep today’s stock TDP.

### Dispatch

Frames are identified by `(extension_type, msg_type)` ([§3.4.1](https://github.com/stratum-mining/sv2-spec/blob/main/03-Protocol-Overview.md)). `0x0001` `msg_type`s (`0x00` / `0x01` / `0x02`) overlap Common SetupConnection bytes; the `extension_type` is what distinguishes them ([`0x0001` §3](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md)).

Unknown `extension_type` with `channel_msg` unset (TDP has no channels, [TDP §7](https://github.com/stratum-mining/sv2-spec/blob/main/07-Template-Distribution-Protocol.md)): ignore the frame ([§3.4.1](https://github.com/stratum-mining/sv2-spec/blob/main/03-Protocol-Overview.md)). Do not disconnect.

Later TDP extension messages belong under `Extensions`, not inside `TemplateDistribution`. Gate sending/handling them on the **TDP** negotiated set.

### Template Provider (server)

`sv2-tp` is the TDP server.

1. **Header.** Keep `extension_type` on recv and send. Core TDP stays `0x0000`. `0x0001` replies use `0x0001`.
2. **Dispatch** on `(extension_type, msg_type)` after `SetupConnection` is confirmed.
3. **`RequestExtensions`.** Parse `request_id`, `requested_extensions` ([`0x0001` §2](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md)). Reply with the same `request_id`.
4. **Success vs Error** ([`0x0001` §4](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md); same logic as JDC [downstream](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/downstream/extensions_message_handler.rs)):
   - `supported` = requested ∩ TP-supported
   - `missing_required` = TP-required − requested
   - **Error** if `supported` is empty **or** `missing_required` is non-empty: `unsupported_extensions` = requested − supported, `required_extensions` = `missing_required`
   - **Success** otherwise: `supported_extensions` = `supported`
   - If required were missing and the client does not retry with them, disconnect (`0x0001` example 6)
5. **Per-client state.** Store the negotiated list on the `Sv2Client`. Stock TDP (constraints, templates, `SubmitSolution`) does not depend on it. Later TDP extension handlers consult this list before accepting those messages.
6. **No `RequestExtensions`.** Leave negotiated set empty; behave as today.
7. **Config.** TP-supported and TP-required extension id lists (empty required is fine). This issue can ship with both empty except `0x0001` implied by answering the request. [`0x0002`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0002-worker-specific-hashrate-tracking.md) is not in the TP list.

Serialize/deserialize the three `0x0001` payloads in [`messages.h`](https://github.com/stratum-mining/sv2-tp/blob/master/src/sv2/messages.h) / [`messages.cpp`](https://github.com/stratum-mining/sv2-tp/blob/master/src/sv2/messages.cpp) (`U16` + `SEQ0_64K[U16]`, and Error’s two sequences). Tests: after `SetupConnection`, send `RequestExtensions` and expect `.Success` / `.Error`; unknown `extension_type` ignored; client that never asks still gets templates.

### JDC (client)

Follow the same pattern as Mining [`Upstream::send_request_extensions`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/upstream/mod.rs) + [`HandleExtensionsFromServer`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/channel_manager/extensions_message_handler.rs) on the TP hop, with TP-scoped state.

1. **Config.** Separate TP request/support lists from Mining [`required_extensions` / `supported_extensions`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/config.rs). Do not send TDP ids on the Pool socket (and vice versa).
2. **Handshake.** After TDP `SetupConnection.Success`, if the TP request list is non-empty, send `RequestExtensions`. **Wait** for `.Success` / `.Error` before [`CoinbaseOutputConstraints`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/channel_manager/mod.rs) ([`0x0001` ordering](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md)).
3. **On Success.** If every JDC-required TP extension is in `supported_extensions`, store that list as the TDP negotiated set. If any required id is missing, fail the TP connection (Mining uses fallback; TP should shutdown or reconnect per existing `Sv2Tp` retry).
4. **On Error.** Same as Mining: if the server listed `required_extensions` we support, retry `RequestExtensions` including them; otherwise proceed without extensions or fail if we required some. Implement the [`0x0001` §4.3](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md) timeout if the TP never answers (old `sv2-tp`).
5. **Pipe.** Widen [`Sv2TpIo`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/template_receiver/sv2_tp/mod.rs) so extension frames can move (e.g. `AnyMessageOwned`, or `{ TemplateDistribution, Extensions }`). [`handle_template_provider_message`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/template_receiver/sv2_tp/mod.rs) must route `MessageType::Extensions` into a `HandleExtensionsFromServer*` impl, not drop them. Channel Manager’s TP path must not assume every inbound value is `TemplateDistributionOwned` (unlike the [Pool path](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/channel_manager/mod.rs), which already has an Extensions arm).
6. **State.** TDP negotiated set is **not** Channel Manager’s Pool `negotiated_extensions` ([TDP handler](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/channel_manager/template_message_handler.rs) currently returns that list). Keep them separate so Mining TLVs and TDP extension messages are gated independently.
7. **Classifier.** [`is_extensions_message`](https://github.com/stratum-mining/sv2-apps/blob/main/stratum-apps/src/utils/protocol_message_type.rs) today matches only `0x0001`. When a TDP extension adds messages, extend it (or match `extension_type != 0` for locally handled extension frames). Out of scope for the first `0x0001` patch if no new `msg_type`s ship yet.

[`Sv2Tp`](https://github.com/stratum-mining/sv2-apps/blob/main/miner-apps/jd-client/src/lib/template_receiver/sv2_tp/mod.rs) is the natural place to send the request and wait; Channel Manager still owns `CoinbaseOutputConstraints`, so either:

- `Sv2Tp::start` completes negotiation, then Channel Manager sends constraints, or
- Channel Manager drives both, and `Sv2Tp` is a dumb frame pump once the pipe is widened.

Prefer the first: negotiation is connection setup, same as Mining `Upstream::setup_connection`.

**Pool `Sv2Tp`** ([`mod.rs`](https://github.com/stratum-mining/sv2-apps/blob/main/pool-apps/pool/src/lib/template_receiver/sv2_tp/mod.rs)) is the same client shape. Same design if Pool should request TDP extensions.

### Tests

- `sv2-tp`: `RequestExtensions` round-trip after SetupConnection; Error path; ignore unknown `extension_type`; no-request still serves templates.
- `sv2-apps`: TDP negotiation scenario (not only Mining [`extensions.rs`](https://github.com/stratum-mining/sv2-apps/blob/main/integration-tests/tests/extensions.rs)): JDC asks, TP answers, then `CoinbaseOutputConstraints` / `NewTemplate`.

---

## Acceptance (rough)

- [ ] `sv2-tp` dispatches on `(extension_type, msg_type)` and answers `RequestExtensions` per [`0x0001`](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md)
- [ ] Unknown `extension_type` + `channel_msg` unset ignored; stock TDP if the client never asks
- [ ] JDC `Sv2Tp` sends `RequestExtensions` after TDP handshake, waits, then `CoinbaseOutputConstraints`
- [ ] TP pipe + handlers carry Extensions; TDP negotiated set ≠ Pool Mining set
- [ ] TP-specific config (not Mining `required_extensions`)
- [ ] Tests as above

Mining / miners / Pool accounting unchanged.

---

## Note: TDP §7 spec verbiage

[`0x0001` §4 Ordering](https://github.com/stratum-mining/sv2-spec/blob/main/extensions/0x0001-extensions-negotiation.md): `RequestExtensions` MUST be sent immediately after `SetupConnection.Success` and before any other protocol-specific messages.

[TDP §7](https://github.com/stratum-mining/sv2-spec/blob/main/07-Template-Distribution-Protocol.md): after the initial common handshake, the client MUST immediately send `CoinbaseOutputConstraints`. That handshake is SetupConnection (same paragraph: “using the same SetupConnection handshake”). TDP does not name `0x0001`.

`RequestExtensions` is not protocol-specific (`extension_type = 0x0001`). `CoinbaseOutputConstraints` is the first TDP message. `0x0001` is allowed between the common handshake and constraints; this is not a MUST-vs-MUST contradiction.

Optional: one line in TDP §7 making that explicit.
