# Signal Session Silent Failure — Investigation

**Scope:** Evolution API v2.3.7 (fork tag `2.3.7-fix-0.0.5`) wrapping Baileys.
**Symptom:** Instance reports `state: open`; every incoming message fails to decrypt (`Bad MAC`, `No matching sessions found`, `Invalid PreKey ID`, group `skmsg` `No session found to decrypt message`); `messages.upsert` stops firing; `PUT /instance/restart` returns 200 but changes nothing.

---

## 1. TL;DR

**`PUT /instance/restart/{instance}` cannot recover from Signal session corruption.** It closes the WebSocket and re-opens a new socket **on top of the same in-memory auth/key store** — the corrupt `session-*`, `sender-key-*`, `pre-key-*`, and `app-state-sync-*` records are preserved verbatim, so the same decrypt failures reproduce the instant a message arrives.

**`DELETE /instance/logout` is currently the only reliable recovery** and forces a new QR scan.

The silent-drop behaviour is not a side effect — Evolution explicitly catches decrypt-failure message stubs and `continue`s out of the loop before they ever reach the `messages.upsert` webhook ([whatsapp.baileys.service.ts:1103-1119](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1103-L1119)). From outside, there is no webhook to observe — only log lines and the absence of traffic.

This is a **known, upstream-acknowledged cluster**: Baileys umbrella issue [#1769](https://github.com/WhiskeySockets/Baileys/issues/1769) and the still-open PR [#2372](https://github.com/WhiskeySockets/Baileys/pull/2372). As of 2026-04, no merged fix exists and "re-pair the device" remains the official remedy across the ecosystem.

---

## 2. What `/instance/restart` Actually Does

**Route →** [instance.router.ts:27](src/api/routes/instance.router.ts#L27)
**Handler →** [instance.controller.ts:346-391](src/api/controllers/instance.controller.ts#L346-L391)

The controller first checks for a `restart()` method on the channel service ([instance.controller.ts:360-361](src/api/controllers/instance.controller.ts#L360-L361)) — **that method does not exist on `BaileysStartupService`**, so the code always falls through to:

```ts
instance.client?.ws?.close();                       // L376
instance.client?.end(new Error('restart'));          // L377
return connectToWhatsapp({ instanceName });          // L378
```

`connectToWhatsapp()` then re-invokes `makeWASocket()` ([whatsapp.baileys.service.ts:656](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L656)) **passing the same in-memory `this.instance.authState.state.creds` / `keys` object** ([whatsapp.baileys.service.ts:659-660](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L659-L660)). No reload from Prisma, Redis, or disk occurs.

| Signal bucket | Cleared on restart? |
|---|---|
| `session-*` | ❌ preserved |
| `sender-key-*` (group) | ❌ preserved |
| `pre-key-*` | ❌ preserved |
| `app-state-sync-*` | ❌ preserved |
| WebSocket | ✅ reconnected |
| EventEmitter subjects | ✅ re-bound (see Evolution [PR #2186](https://github.com/EvolutionAPI/evolution-api/pull/2186)) |

**Conclusion:** `restart` fixes transport-layer issues (dead WebSocket, stale `messages.upsert` subjects). It cannot fix Signal state.

---

## 3. Baileys Auto-Recovery — Per Error Type

Baileys sends a retry receipt (`sendRetryRequest`) on decrypt failures, gated by `msgRetryCounterCache` (NodeCache, 1 h TTL) and capped by `maxMsgRetryCount`. Once the cap is hit the message is **silently dropped**. The retry receipt includes fresh pre-key material only on attempts > 1, and only if `enableAutoSessionRecreation` is set.

| Error | `retryReceipt`? | Fresh pre-keys attached? | Group SKDM re-request? | Effective recovery? |
|---|---|---|---|---|
| **Whisper `pn` — Bad MAC** | yes | only if retry > 1 + flag set | n/a | Only if sender is online and re-encrypts against a fresh session — in practice flaky when the ratchet has diverged. |
| **PreKey `pkmsg` — Invalid PreKey ID** | yes | yes (calls internal `uploadPreKeys()`) | n/a | Race-prone: upload must reach WA servers before sender re-sends. Frequently fails in the wild — upstream issue [#2317](https://github.com/WhiskeySockets/Baileys/issues/2317). |
| **Group `skmsg` — No session found** | yes (cosmetic) | — | **no primitive to request a fresh `SenderKeyDistributionMessage`** | None. Entire group stays dark until the sender volunteers a new SKDM. |
| **Self-JID `pn` — No matching sessions** | yes | — | — | None. Requires re-pair — upstream [#2378](https://github.com/WhiskeySockets/Baileys/issues/2378). |

**Key finding:** Baileys has no public `resetSession(jid)` primitive and no group-SKDM re-request RPC. It exposes `assertSessions(jids[], force?)` (rebuild pairwise sessions) and an internal `uploadPreKeys()` — both are usable only from inside Baileys or by patching the wrapper.

---

## 4. Non-Destructive Recovery Options We Are **Not** Using Today

Ranked by likely effectiveness. None are exposed as Evolution REST endpoints — they would require a fork patch.

1. **Call `sock.assertSessions([peerJid], true)` on the stuck peer.** Rebuilds the pairwise Signal session without touching creds or pairing. Highest leverage for `Bad MAC` / `No matching sessions` on pairwise chats. Baileys publicly exports this on `makeWASocket`.
2. **Delete just the offending `session-<peer>` rows from the key store, then call `assertSessions`.** Forces a fresh X3DH handshake on next inbound. Requires a writable handle to the `SignalKeyStore` that Evolution already holds (`this.instance.authState.state.keys`).
3. **Force `uploadPreKeys()` via the Baileys socket.** Addresses `Invalid PreKey ID` by pushing a fresh batch of pre-keys to WA servers so new `pkmsg` can be decrypted. Internal in Baileys — callable only from a patched service.
4. **Delete the `sender-key-<group>` record for a dark group and wait for the next sender message.** The sender ships the SKDM inline with their next message; deleting the stale record forces Baileys to treat the incoming SKDM as new. Low user-visible impact (group messages sent *before* the next sender message remain lost).
5. **`client.readMessages([key])`** / **`client.sendPresenceUpdate()`** — exposed as `POST /chat/markMessageAsRead` ([chat.router.ts:63](src/api/routes/chat.router.ts#L63)) and `POST /chat/sendPresence` ([chat.router.ts:134](src/api/routes/chat.router.ts#L134)). They emit receipts/presence to the peer but do **not** rebuild any local Signal state. Plausibly harmless, not a recovery mechanism. Do not rely on them.

**Not available anywhere:** any REST endpoint for `uploadPreKeys`, `resyncAppState`, or per-peer session reset. Searches for those names in the Evolution source return nothing.

---

## 5. Recommended Recovery Sequence (with existing REST surface only)

Given the surface Evolution exposes today, this is the best we can do without patching:

1. **Verify the state.** `GET /instance/connectionState` → `open`. Tail `docker logs` for the error strings listed above. If the decrypt-error rate is > 0 sustained for > 5 minutes and `messages.upsert` is silent, the instance is in silent-dead.
2. **Try `PUT /instance/restart`.** Expected outcome: **no change in decrypt-error rate within 60 s**. Do this anyway — it rules out transport-level issues and costs nothing.
3. **If group `skmsg` errors dominate: ask the group members to send a new message.** Their Baileys ships a fresh SKDM; nothing to do on our side.
4. **If symptoms persist: `DELETE /instance/logout` + `/instance/connect` (QR re-scan).** Currently the only reliable remediation. Wipes *all* Signal state and the paired-device record ([whatsapp.baileys.service.ts:267-299](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L267-L299)).

**Feature candidates (fork patches, in priority order):**
- Add `POST /instance/:name/reset-session` taking `{ remoteJid }` → wraps `sock.assertSessions([jid], true)` + optional session-row delete.
- Add `POST /instance/:name/refresh-prekeys` → wraps `sock.uploadPreKeys()`.
- Add a "soft-logout" that clears `session-*` and `sender-key-*` but preserves `creds.json`, avoiding QR re-scan.

---

## 6. Detection Heuristic (External Consumer)

**Cheapest reliable signature — stderr log grep.** Evolution silently swallows decrypt-failure stubs at [whatsapp.baileys.service.ts:1103-1119](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1103-L1119), but the underlying Baileys pino logger still writes the error to stderr **only if `LOG_BAILEYS` is at least `error`** (default; set in [env.config.ts:738](src/config/env.config.ts#L738)). Confirm this env is not silenced in your deployment.

Recommended external rule for the Laravel monitor:

```
silent_dead := 
    GET /instance/connectionState == "open"
  AND no messages.upsert webhook in > 15 min
  AND docker_logs matches /(Bad MAC|No matching sessions|Invalid PreKey|No session found to decrypt)/ ≥ 2/min in the last 5 min
```

Thresholds justified from code: Baileys' `keepAliveIntervalMs` is 30 s ([whatsapp.baileys.service.ts:671](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L671)) — outbound only, no inbound heartbeat. `msgRetryCounterCache` has a 1 h TTL, so the log rate stays elevated for as long as senders keep trying; this makes a 5-minute sampling window generous.

**Secondary corroboration.** `POST /chat/findMessages` ([chat.router.ts:164-173](src/api/routes/chat.router.ts#L164-L173)) is safe to poll. In silent-dead the `Message` table stays dry because Evolution `continue`s before persisting the failed stub — so both the webhook and the DB go quiet simultaneously. Useful as a second axis when the log stream isn't available.

**What won't work.** The `/metrics` Prometheus endpoint ([index.router.ts:91-161](src/api/routes/index.router.ts#L91-L161)) only exposes `evolution_instance_up` (1 if `state == open`) — silent-dead appears as `up == 1`. The Sentry integration ([main.ts:151-158](src/main.ts#L151-L158)) wraps Express only; it does not capture Baileys decrypt errors.

---

## 7. Open Upstream Issues / PRs To Track

**Baileys**
- [#1769](https://github.com/WhiskeySockets/Baileys/issues/1769) — umbrella for the exact four-error cluster. Open.
- [#2372](https://github.com/WhiskeySockets/Baileys/pull/2372) — **the PR most aligned with this cluster.** Open, mergeable, last touched 2026-04-19. Fixes (a) LID/PN transaction mutex races that corrupt the ratchet → Bad MAC; (b) PN session retention across LID migration → "No matching sessions"; (c) deferred pre-key deletion → "Invalid PreKey ID". **Pin and track.**
- [#2340](https://github.com/WhiskeySockets/Baileys/issues/2340) — prekey-store race instrumentation (gates #2372).
- [#2271](https://github.com/WhiskeySockets/Baileys/issues/2271), [#2331](https://github.com/WhiskeySockets/Baileys/issues/2331), [#2363](https://github.com/WhiskeySockets/Baileys/issues/2363) — duplicate reports of the exact silent-dead shape.
- [#2321](https://github.com/WhiskeySockets/Baileys/issues/2321) — regression after rc9 relative to rc4. Consider pinning to an earlier rc if feasible.
- [#2378](https://github.com/WhiskeySockets/Baileys/issues/2378) — self-JID @lid `fromMe: true` decrypt failure.
- Merged but **did not solve it**: [#2307](https://github.com/WhiskeySockets/Baileys/pull/2307), [#1663](https://github.com/WhiskeySockets/Baileys/pull/1663).

**Evolution API**
- [#1640](https://github.com/EvolutionAPI/evolution-api/issues/1640), [#245](https://github.com/EvolutionAPI/evolution-api/issues/245), [#1512](https://github.com/EvolutionAPI/evolution-api/issues/1512) — closed; workaround was re-pair.
- [PR #2186](https://github.com/EvolutionAPI/evolution-api/pull/2186) — merged 2025-11-07; fixes the RxJS subject-close issue after `logout` → `connect`. Confirms `restart` is the right call *for transport issues*. Does not touch Signal state.

No Evolution issue proposes per-peer reset, pre-key re-upload, or any non-destructive Signal recovery API.

---

## 8. Risks and Side Effects of Each Recovery Step

| Step | Visible to peer? | Touches linked device? | Emits `connection.update`? | Other |
|---|---|---|---|---|
| `PUT /instance/restart` | No | No | Yes — `close` → `connecting` → `open`. Your downstream must tolerate a brief flap. | No-op for Signal state. |
| `POST /chat/markMessageAsRead` | Yes — read receipt. | No | No | Cannot undo read state. Only useful for hiding sender-side "not delivered" state; not a recovery. |
| `POST /chat/sendPresence` | Yes — typing/paused indicator appears on peer's screen. | No | No | Harmless but user-visible. |
| `DELETE /instance/logout` | Yes — peers see you as offline / paired device lost. The WA server is notified via `sock.logout()` ([whatsapp.baileys.service.ts:269](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L269)). | **Yes — paired-device record is revoked.** | Yes — `close` | Wipes `creds.json`, `Session` table rows, Redis session keys, and all Signal state. User must re-scan QR. |
| (proposed) `POST /chat/resetSession` | Next message from peer triggers fresh X3DH — peer sees nothing user-visible. | No | No | Requires fork patch. |

---

## Not In Scope

- No patches to Evolution proposed in this document beyond the feature-candidate list in §5.
- Webhook contract unchanged.
- Presence / `presenceSubscribe` not analysed here.
