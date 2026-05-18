# Evolution API — Duplicate Connection & Zombie-Instance Teardown — Findings

> Source tree analyzed: **Evolution API v2.3.7** (`package.json`), integration `WHATSAPP-BAILEYS`,
> Baileys **`7.0.0-rc.9`** (`package.json` dependency).
> All `file:line` references are into this repository.

## TL;DR

| Question | Answer |
|---|---|
| Why does `/instance/logout` 500? | `logoutInstance()` calls `client.logout()`, which tries to send a `remove-companion-device` IQ node over a **dead socket**. Baileys throws `Boom('Connection Closed', statusCode 428)`. Nothing catches it → `InternalServerErrorException`. |
| Does `/instance/delete` also fail? | **Yes.** `deleteInstance()` calls `logout()` internally whenever state ≠ `close`. The throw short-circuits → instance is **never removed**. It surfaces as HTTP **400 `[object Object]`**. |
| Is there a force-delete path? | Yes internally (`waMonitor.deleteInstance()` → `remove.instance` event) but **not exposed by any HTTP route**. |
| Is `connectionState` trustworthy? | **No.** It returns the in-memory `stateConnection.state`, mutated **only** when Baileys emits a `connection.update` event. A silent socket death leaves it stuck at `open`. |
| Can you detect "same account, two instances"? | Yes — compare **`ownerJid`** from `GET /instance/fetchInstances`. Evolution has **zero** built-in duplicate guard. |
| Does Evolution surface `connectionReplaced` (440)? | **No.** 440 is treated as "reconnect" → no `connection.update` close webhook, no DB write, silent reconnect with the same creds. The duplicate is invisible. |

---

## A. Why `/instance/logout` returns 500 "Connection Closed"

### A1. The route → handler → Baileys call chain

1. **Route** — `DELETE /instance/logout/:instanceName`
   [`src/api/routes/instance.router.ts:79-88`](src/api/routes/instance.router.ts#L79-L88) → `instanceController.logout(instance)`.

2. **Controller** — [`InstanceController.logout()` `src/api/controllers/instance.controller.ts:436-450`](src/api/controllers/instance.controller.ts#L436-L450):

   ```ts
   public async logout({ instanceName }: InstanceDto) {
     const { instance } = await this.connectionState({ instanceName });

     if (instance.state === 'close') {                                  // line 439
       throw new BadRequestException('The "' + instanceName + '" instance is not connected');
     }
     try {
       await this.waMonitor.waInstances[instanceName]?.logoutInstance(); // line 444
       return { status: 'SUCCESS', error: false, response: { message: 'Instance logged out' } };
     } catch (error) {
       throw new InternalServerErrorException(error.toString());         // line 448
     }
   }
   ```

3. **Service** — [`logoutInstance()` `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts:267-299`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L267-L299):

   ```ts
   public async logoutInstance() {
     this.messageProcessor.onDestroy();
     await this.client?.logout('Log out instance: ' + this.instanceName);  // line 269  ← THROWS HERE
     this.client?.ws?.close();                                             // line 271  ← never reached
     ...
   }
   ```

   The Baileys call is **`client.logout()`** at **line 269**.

### A2. Where "Connection Closed" originates

`client.logout()` (Baileys `Socket`) sends an `iq` node containing `remove-companion-device`
to `s.whatsapp.net`. That send goes through the Noise-encoded WebSocket. When the underlying
WebSocket is **not `OPEN`**, Baileys rejects the in-flight query with:

```js
new Boom('Connection Closed', { statusCode: DisconnectReason.connectionClosed }) // 428
```

- It **is** a Baileys `Boom` error, `statusCode = 428` (`DisconnectReason.connectionClosed`).
- It is thrown by the **WebSocket send / query path**, not by Evolution.
- `error.toString()` on that Boom yields the literal string **`"Error: Connection Closed"`**.

### A3. No state inspection, no try/catch around the Baileys call

- `logoutInstance()` (line 269) has **no try/catch** around `client.logout()`. Any rejection bubbles.
- The controller (line 439) **does** check `instance.state === 'close'` first — but in the zombie
  scenario `connectionState` reports `open` (see Section C), so the guard passes and execution proceeds.
- The controller's `catch` (line 448) re-throws everything as `InternalServerErrorException` — there is
  **no branch that returns a clean 200 or 409** when the socket is already dead.

### A4. Exact 500 payload

`InternalServerErrorException` ([`src/exceptions/500.exception.ts`](src/exceptions/500.exception.ts))
throws `{ status: 500, error: 'Internal Server Error', message: ["Error: Connection Closed"] }`.
The global error middleware ([`src/main.ts:76-118`](src/main.ts#L76-L118)) wraps `message` under `response`:

```json
{
  "status": 500,
  "error": "Internal Server Error",
  "response": { "message": ["Error: Connection Closed"] }
}
```

This matches the production symptom exactly.

> **Edge case:** if `this.client` is **`null`** (socket fully torn down), `this.client?.logout()` is
> a no-op (`await undefined`) and `logoutInstance()` **succeeds**. The 500 happens specifically when
> `this.client` still exists but its WebSocket is dead — the typical zombie state.

---

## B. Robustly tearing down a zombie instance

### B4. `DELETE /instance/delete` — it also fails on a zombie

- **Route** — `DELETE /instance/delete/:instanceName`
  [`src/api/routes/instance.router.ts:89-98`](src/api/routes/instance.router.ts#L89-L98).
- **Handler** — [`InstanceController.deleteInstance()` `src/api/controllers/instance.controller.ts:452-476`](src/api/controllers/instance.controller.ts#L452-L476):

  ```ts
  public async deleteInstance({ instanceName }: InstanceDto) {
    const { instance } = await this.connectionState({ instanceName });
    try {
      const waInstances = this.waMonitor.waInstances[instanceName];
      ...
      if (instance.state === 'connecting' || instance.state === 'open') {
        await this.logout({ instanceName });            // line 459  ← calls A2/A3, THROWS
      }
      ...
      this.eventEmitter.emit('remove.instance', instanceName, 'inner');  // line 471 ← never reached
      return { status: 'SUCCESS', ... };
    } catch (error) {
      throw new BadRequestException(error.toString());   // line 474
    }
  }
  ```

  **`delete` requires logout to succeed.** Because a zombie reports `state: 'open'`, line 458 is true,
  line 459 runs `logout()`, which throws the 500-object from Section A. The `catch` at line 474 wraps it
  in `BadRequestException`. Since the thrown value is a plain object, `error.toString()` produces the
  string `"[object Object]"`, so the client sees:

  ```json
  { "status": 400, "error": "Bad Request", "response": { "message": ["[object Object]"] } }
  ```

  The `remove.instance` emit at line 471 **never executes** — the instance stays in memory and in the DB.
  `delete` does **not** force-remove regardless of connection state; it is gated on `logout()` succeeding.

### B5. Is there a force-delete path?

Yes, but **not reachable over HTTP**:

- [`WAMonitoringService.deleteInstance()` `src/api/services/monitor.service.ts:265-271`](src/api/services/monitor.service.ts#L265-L271)
  emits `remove.instance` **directly**, with no logout and no socket check.
- The `remove.instance` listener
  [`src/api/services/monitor.service.ts:392-410`](src/api/services/monitor.service.ts#L392-L410) runs
  `cleaningUp()` + `cleaningStoreData()` and `delete this.waInstances[instanceName]`.
  - [`cleaningUp()` `monitor.service.ts:159-189`](src/api/services/monitor.service.ts#L159-L189) — sets
    DB `connectionStatus: 'close'`, `rmSync` the instance dir, deletes `session` rows, clears Redis.
  - [`cleaningStoreData()` `monitor.service.ts:191-225`](src/api/services/monitor.service.ts#L191-L225) —
    deletes the `Instance` row and all related rows (`chat`, `contact`, `message`, `webhook`, …).
  - **None of this touches the WhatsApp socket.**

But `WAMonitoringService.deleteInstance()` is only called internally — from `createInstance()`'s rollback
path ([`instance.controller.ts:303`](src/api/controllers/instance.controller.ts#L303)). **No router maps
to it.** The only public delete is `InstanceController.deleteInstance()`, which always routes through
`logout()` when `state ≠ close`.

> **Net result:** out of the box there is **no HTTP endpoint** that tears down a zombie instance whose
> socket is dead but whose `state` reads `open`. Both `logout` and `delete` fail. See Recommendation (1).

### B6. WhatsApp-side effect of `logout` vs `delete`

- **`logout`** (`client.logout()`, `whatsapp.baileys.service.ts:269`) sends the
  `remove-companion-device` IQ → **this unlinks the companion device on WhatsApp's servers**
  (the entry disappears from *WhatsApp → Linked devices*). It only does so if the socket is alive;
  on a dead socket nothing reaches WhatsApp.
- **`delete`** (`cleaningUp` + `cleaningStoreData`) only removes **Evolution-local** state
  (DB rows, session files, Redis keys). It does **not** send anything to WhatsApp and does **not**
  unlink the device. If you `delete` without a successful `logout`, the device may remain linked on
  the phone until the user removes it manually or WhatsApp times it out.

---

## C. Is `connectionState` trustworthy? — No.

### C7. Where the `state` comes from

- **Endpoint** — [`InstanceController.connectionState()` `src/api/controllers/instance.controller.ts:393-400`](src/api/controllers/instance.controller.ts#L393-L400):

  ```ts
  public async connectionState({ instanceName }: InstanceDto) {
    return { instance: { instanceName,
      state: this.waMonitor.waInstances[instanceName]?.connectionStatus?.state } };
  }
  ```

- `connectionStatus` is a getter ([`whatsapp.baileys.service.ts:263-265`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L263-L265))
  returning the **in-memory field** `stateConnection`
  ([`whatsapp.baileys.service.ts:259`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L259)),
  initialized to `{ state: 'close' }`.

- `stateConnection` is mutated in exactly **one** place — `connectionUpdate()`
  ([`whatsapp.baileys.service.ts:419-424`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L419-L424)):

  ```ts
  if (connection) {
    this.stateConnection = {
      state: connection,
      statusReason: (lastDisconnect?.error as Boom)?.output?.statusCode ?? 200,
    };
  }
  ```

So the returned `state` is **Baileys' last-emitted `connection.update` value held in process memory** —
**not** a DB column, **not** a live socket probe. (The DB column `Instance.connectionStatus` exists but is
**not** read by this endpoint — see C9.)

### C8. Why it can say `open` when the link is dead

`connectionState` is purely **event-driven and passive**:

- It only changes when Baileys fires a `connection.update` with a non-null `connection`
  (wired at [`whatsapp.baileys.service.ts:1953-1955`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1953-L1955)).
- If the companion link dies **without** Baileys translating it into a `connection: 'close'` event
  (silent TCP drop, server-side unlink not yet noticed, keep-alive not yet failed), `stateConnection`
  is **never updated** and stays at the last value — `open`.
- Even when a `close` event *does* fire with a *reconnectable* code (428/440/515/500/411 — see Section E),
  Evolution immediately calls `connectToWhatsapp()` and the next `connecting`/`open` event overwrites the
  brief `close`. The window where `state === 'close'` is milliseconds, and no webhook announces it.

There is **no independent health signal** consulted. More reliable signals that *exist* but are unused
by this endpoint:

- `client.ws` WebSocket readiness (Baileys exposes the socket).
- `client.user` presence (set only while authenticated).
- Baileys `lastDisconnect` (available inside `connectionUpdate`, not exposed).
- DB columns `disconnectionAt` / `disconnectionReasonCode` / `disconnectionObject`
  (`prisma/postgresql-schema.prisma`, model `Instance`) — but these are written **only** on a
  non-reconnect close (see E14).
- An **active probe** (any endpoint that performs an IQ query over the socket) is the only thing that
  reliably distinguishes a live socket from a zombie — see Recommendation (2).

### C9. After a process restart

- On boot, [`loadInstancesFromDatabasePostgres()` `monitor.service.ts:340-365`](src/api/services/monitor.service.ts#L340-L365)
  reads every `Instance` row and calls `setInstance()`.
- [`setInstance()` `monitor.service.ts:273-308`](src/api/services/monitor.service.ts#L273-L308): if the DB
  `connectionStatus` is `open` or `connecting`, it **auto-calls `connectToWhatsapp()`** (lines 296-300).
- The in-memory `stateConnection` still starts at `{ state: 'close' }` and only becomes `open` once the
  socket genuinely reconnects. So `connectionState` is **not served stale-from-DB** — it reflects the
  live in-memory value.
- **However:** the reconnect branch of `connectionUpdate` (E14) **never writes `close` to the DB**. So a
  zombie's DB `connectionStatus` stays `open`, and every process restart **auto-revives the zombie**,
  re-establishing a socket with the still-valid creds. This is a structural reason zombies persist.

---

## D. Detecting that two instances are the same WhatsApp account

### D10. `GET /instance/fetchInstances` — fields returned

- **Route** — [`instance.router.ts:57-68`](src/api/routes/instance.router.ts#L57-L68) →
  [`InstanceController.fetchInstances()` `instance.controller.ts:402-430`](src/api/controllers/instance.controller.ts#L402-L430).
- It returns the raw Prisma `Instance` rows from
  [`WAMonitoringService.instanceInfo()` `monitor.service.ts:86-130`](src/api/services/monitor.service.ts#L86-L130),
  with relations `Chatwoot, Proxy, Rabbitmq, Nats, Sqs, Websocket, Setting` and a `_count` of
  `Message / Contact / Chat`.

Per-instance fields (Prisma model `Instance`, `prisma/postgresql-schema.prisma`):

| Field | Meaning / reliability for account identity |
|---|---|
| `id` | Evolution instance UUID/cuid — per-instance, **not** the account. |
| `name` | Instance name (`@unique`). Per-instance. |
| `connectionStatus` | DB enum `open\|close\|connecting`. **Stale** (see C9). |
| **`ownerJid`** | **The WhatsApp account JID** (e.g. `5511999998888@s.whatsapp.net`). Written on `connection: 'open'` (see below). **This is the reliable account identifier.** |
| `profileName` | WhatsApp profile display name. Human-readable, **not unique**, can change. |
| `profilePicUrl` | Profile picture URL. Not an identifier. |
| `number` | The number **supplied at create time** (pairing-code flow). Optional, may be `null`/blank for QR-code instances. |
| `integration` | `WHATSAPP-BAILEYS` etc. |
| `businessId`, `token`, `clientName` | Config / auth metadata. |
| `disconnectionReasonCode`, `disconnectionObject`, `disconnectionAt` | Last non-reconnect disconnect (often `null`). |
| `createdAt`, `updatedAt` | Timestamps. |

There is **no** `owner`, no `wuid`, no `profilePictureUrl` field on the row — those names appear only in
webhook payloads. The persisted account-identity column is **`ownerJid`**.

**How `ownerJid` is populated** — in `connectionUpdate()` on `connection === 'open'`
([`whatsapp.baileys.service.ts:467-498`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L467-L498)):

```ts
this.instance.wuid = this.client.user.id.replace(/:\d+/, '');   // line 468 — strip device suffix
...
await this.prismaRepository.instance.update({
  where: { id: this.instanceId },
  data: { ownerJid: this.instance.wuid, profileName: ..., profilePicUrl: ..., connectionStatus: 'open' },
});                                                              // lines 490-498
```

Reliability: `ownerJid` is **reliably populated once the instance has reached `open` at least once**.
Before the first successful connection it is `null`. It is **not cleared** on disconnect, so it remains a
stable historical identifier even for a zombie.

### D11. Other ways to expose the account JID/number

- The **`connection.update` webhook** for `state: 'open'` includes `wuid`, `profileName`,
  `profilePictureUrl` ([`whatsapp.baileys.service.ts:509-515`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L509-L515)).
  `wuid` == `ownerJid`.
- `GET /instance/fetchInstances?number=<n>` resolves an instance by the `number` column
  ([`fetchInstances` → `instanceInfoById` `monitor.service.ts:132-157`](src/api/services/monitor.service.ts#L132-L157)) —
  but `number` is the *create-time* value, not the connected account, and is often empty.
- Bottom line: **`ownerJid` (DB) / `wuid` (webhook) is the single reliable cross-instance match key.**
  Two instances with the same non-null `ownerJid` are the same WhatsApp account.

### D12. Does Evolution guard against the same number on multiple instances?

**No.** There is no such guard anywhere:

- `Instance.number` and `Instance.ownerJid` are **not** `@unique` in `prisma/postgresql-schema.prisma`
  (only `Instance.name` is `@unique`).
- `createInstance()` ([`instance.controller.ts:37-307`](src/api/controllers/instance.controller.ts#L37-L307))
  never checks for an existing instance with the same `number`/`ownerJid`.
- `connectToWhatsapp()` ([`instance.controller.ts:309-344`](src/api/controllers/instance.controller.ts#L309-L344))
  only checks the *current* instance's own `state` — not whether the account is connected elsewhere.
- No config flag exists to enforce one-instance-per-account.

Preventing duplicates is **entirely the calling application's responsibility**.

---

## E. Disconnect-reason signalling (`connection.update` webhook)

### E13. What Baileys emits on companion-device invalidation

Baileys (`7.0.0-rc.9`) emits `connection.update` with `connection: 'close'` and
`lastDisconnect.error` = a `Boom` whose `output.statusCode` is one of `DisconnectReason`:

| Reason | statusCode |
|---|---|
| `connectionClosed` | 428 |
| `connectionLost` / `timedOut` | 408 |
| `connectionReplaced` | 440 |
| `loggedOut` | 401 |
| `badSession` | 500 |
| `restartRequired` | 515 |
| `multideviceMismatch` | 411 |
| `forbidden` | 403 |

- User unlinks the device → typically `loggedOut` (401).
- Another session takes over the same registration → `connectionReplaced` (440).
- Initial handshake needs a socket restart → `restartRequired` (515).

### E14. How Evolution handles each — and what it forwards

The decision lives in `connectionUpdate()`, `connection === 'close'` branch
([`whatsapp.baileys.service.ts:426-465`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L426-L465)):

```ts
if (connection === 'close') {
  const statusCode = (lastDisconnect?.error as Boom)?.output?.statusCode;
  const codesToNotReconnect = [DisconnectReason.loggedOut, DisconnectReason.forbidden, 402, 406]; // 401, 403, 402, 406
  const shouldReconnect = !codesToNotReconnect.includes(statusCode);
  if (shouldReconnect) {
    await this.connectToWhatsapp(this.phoneNumber);            // silent reconnect, SAME creds
  } else {
    this.sendDataWebhook(Events.STATUS_INSTANCE, { ... });     // status.instance webhook
    await this.prismaRepository.instance.update({ ... connectionStatus: 'close' ... });
    this.eventEmitter.emit('logout.instance', ...);
    this.client?.ws?.close();
    this.client.end(new Error('Close connection'));
    this.sendDataWebhook(Events.CONNECTION_UPDATE, { instance, ...this.stateConnection }); // connection.update
  }
}
```

| statusCode | reason | `shouldReconnect` | webhook emitted? | DB `connectionStatus` |
|---|---|---|---|---|
| 401 | `loggedOut` | **false** | ✅ `status.instance` **+** `connection.update` (`state:'close'`) | written `close` |
| 403 | `forbidden` | **false** | ✅ `status.instance` **+** `connection.update` (`state:'close'`) | written `close` |
| 402 | (banned/payment) | **false** | ✅ `status.instance` **+** `connection.update` (`state:'close'`) | written `close` |
| 406 | (not acceptable) | **false** | ✅ `status.instance` **+** `connection.update` (`state:'close'`) | written `close` |
| 428 | `connectionClosed` | **true** | ❌ **none** — silent reconnect | unchanged |
| 408 | `connectionLost`/`timedOut` | **true** | ❌ **none** — silent reconnect | unchanged |
| **440** | **`connectionReplaced`** | **true** | ❌ **none** — silent reconnect | unchanged |
| 500 | `badSession` | **true** | ❌ **none** — silent reconnect | unchanged |
| 515 | `restartRequired` | **true** | ❌ **none** — silent reconnect | unchanged |
| 411 | `multideviceMismatch` | **true** | ❌ **none** — silent reconnect | unchanged |

So Evolution **only** forwards a disconnect for `401/403/402/406`. For every reconnectable code —
**including `connectionReplaced` (440)** — there is **no `connection.update` close webhook, no
`status.instance` webhook, and no DB write**. The app just sees `connecting` then `open` again later
(or nothing, if the reconnect hangs).

#### Exact webhook JSON shapes

All webhook deliveries share the envelope built in
[`webhook.controller.ts:93-103`](src/api/integrations/event/webhook/webhook.controller.ts#L93-L103):
`{ event, instance, data, destination, date_time, sender, server_url, apikey }`.

**`connection.update` — `state: 'open'`** (`whatsapp.baileys.service.ts:509-515`):

```json
{
  "event": "connection.update",
  "instance": "ws_1_phone_2",
  "data": {
    "instance": "ws_1_phone_2",
    "wuid": "5511999998888@s.whatsapp.net",
    "profileName": "My Business",
    "profilePictureUrl": "https://.../pp.jpg",
    "state": "open",
    "statusReason": 200
  },
  "destination": "https://app.example/webhook",
  "date_time": "2026-05-18T12:00:00.000Z",
  "sender": "5511999998888@s.whatsapp.net",
  "server_url": "https://evo.example",
  "apikey": "..."
}
```

**`connection.update` — `state: 'connecting'`** (`whatsapp.baileys.service.ts:519`):

```json
{ "event": "connection.update", "instance": "ws_1_phone_2",
  "data": { "instance": "ws_1_phone_2", "state": "connecting", "statusReason": 200 },
  "destination": "...", "date_time": "...", "sender": "...", "server_url": "...", "apikey": "..." }
```

**`connection.update` — `state: 'close'` — ONLY for 401 / 403 / 402 / 406**
(`whatsapp.baileys.service.ts:463`; `statusReason` is `...this.stateConnection`, set at lines 420-423):

```json
{ "event": "connection.update", "instance": "ws_1_phone_2",
  "data": { "instance": "ws_1_phone_2", "state": "close", "statusReason": 401 },
  "destination": "...", "date_time": "...", "sender": "...", "server_url": "...", "apikey": "..." }
```

**`status.instance` — paired with the above 401/403/402/406 close**
(`whatsapp.baileys.service.ts:433-439`):

```json
{ "event": "status.instance", "instance": "ws_1_phone_2",
  "data": {
    "instance": "ws_1_phone_2",
    "status": "closed",
    "disconnectionAt": "2026-05-18T12:00:00.000Z",
    "disconnectionReasonCode": 401,
    "disconnectionObject": "{\"error\":{\"output\":{\"statusCode\":401}}}"
  },
  "destination": "...", "date_time": "...", "sender": "...", "server_url": "...", "apikey": "..." }
```

**For 428 / 408 / 440 / 500 / 515 / 411 — NO webhook is emitted.** The only thing the app may observe
is a later `connection.update` with `state: 'connecting'` and then `state: 'open'`.

**QR-limit refusal** (not a disconnect, but related — `whatsapp.baileys.service.ts:350-357`):

```json
{ "event": "connection.update", "instance": "ws_1_phone_2",
  "data": { "instance": "ws_1_phone_2", "state": "refused", "statusReason": 428,
            "wuid": "...", "profileName": "...", "profilePictureUrl": "..." }, ... }
```

### E15. Same number on two Baileys sessions → `connectionReplaced`?

Yes. When the same WhatsApp registration is used by a second socket, WhatsApp sends
`connectionReplaced` (**440**) to the older session. What Evolution does with it:

- 440 is **not** in `codesToNotReconnect` → `shouldReconnect = true`
  ([`whatsapp.baileys.service.ts:428-431`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L428-L431)).
- Evolution **does not surface it**: no `connection.update`, no `status.instance`, no DB update.
- It immediately calls `connectToWhatsapp(this.phoneNumber)` → builds a **fresh socket reusing the same
  still-valid creds** ([`createClient()` `whatsapp.baileys.service.ts:591-736`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L591-L736)).
- That new socket connects → WhatsApp again sees a conflicting session → sends 440 to one of them again.
  The two instances enter a **reconnect ping-pong**. Whichever socket most recently completed `open`
  flips its `stateConnection` to `open`; `connectionState` therefore frequently reads `open` for **both**.

This is precisely the production symptom: both instances report `open`, the phone shows only one linked
device, and the loser is a **zombie** — Evolution leaves `connectionState` at `open` (last event seen)
and never tells the app anything went wrong. `logout` on it then 500s (Section A) because by the time you
call it the socket has settled into a closed/dead state.

---

## Recommendations

### (1) An idempotent "disconnect" the app can rely on

**There is no such call in v2.3.7 today.** On a zombie, `logout` → 500 and `delete` → 400 `[object Object]`,
and no public route force-removes. Two fixes — apply the first, optionally both:

**Fix A — make `logoutInstance()` tolerate a dead socket** (the minimal, highest-value patch).
[`src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts:267-271`](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L267-L271):

```ts
public async logoutInstance() {
  this.messageProcessor.onDestroy();
  try {
    await this.client?.logout('Log out instance: ' + this.instanceName);
  } catch (error) {
    // Socket already dead — the companion device cannot be unlinked remotely,
    // but local teardown must still succeed. Treat as idempotent success.
    this.logger.warn(`logout: socket already closed for ${this.instanceName} — ${error?.toString()}`);
  } finally {
    try { this.client?.ws?.close(); } catch {}
    try { this.client?.end?.(new Error('logout')); } catch {}
  }
  // ... existing creds/session cleanup unchanged ...
}
```

With Fix A, `DELETE /instance/logout` returns a clean **200** even on a zombie, and `DELETE /instance/delete`
(which calls `logout` internally at `instance.controller.ts:459`) then completes and removes the instance.

**Fix B — decouple `delete` from `logout` success** (defense in depth).
[`src/api/controllers/instance.controller.ts:458-460`](src/api/controllers/instance.controller.ts#L458-L460):

```ts
if (instance.state === 'connecting' || instance.state === 'open') {
  try {
    await this.logout({ instanceName });
  } catch (error) {
    this.logger.warn(`delete: logout failed for ${instanceName}, forcing removal — ${error}`);
  }
}
```

so `remove.instance` always fires. Optionally also expose a dedicated force endpoint
(`DELETE /instance/delete/:instanceName?force=true`) that calls
`waMonitor.deleteInstance(instanceName)` directly (Section B5) and skips logout entirely.

**App-side behavior after the fix:** call `DELETE /instance/logout/{instance}`; treat **200** and
**"Connection Closed"** identically as "disconnected"; then call `DELETE /instance/delete/{instance}`.
Until the image is patched, the app cannot reliably tear a zombie down over HTTP — flag it for manual
DB/restart cleanup.

### (2) Most reliable signal for duplicate detection and zombie liveness

**Duplicate-account detection** — do **not** trust `connectionStatus`. Instead:

1. Poll `GET /instance/fetchInstances` periodically.
2. Group instances by **`ownerJid`** (ignore `null`).
3. Any `ownerJid` shared by ≥ 2 instances ⇒ **same WhatsApp account connected twice.**
4. `ownerJid` is stable and survives disconnects, so it catches duplicates even after a zombie's socket dies.

**Knowing whether an instance is actually alive** (since `connectionState` lies):

- `connectionState` / DB `connectionStatus` = **necessary-not-sufficient**. If it says `close`, it's
  genuinely down; if it says `open`, verify.
- **Active liveness probe:** call any endpoint that performs a real IQ query over the socket — e.g.
  `POST /chat/whatsappNumbers/{instance}` (checks numbers) or a profile fetch. On a zombie the socket
  is dead, so the call fails with **`"Connection Closed"`** (statusCode 428). A successful probe ⇒
  the socket is genuinely live. This is the only client-side test that distinguishes `open` from zombie-`open`.
- **Decide the winner of a duplicate pair:** probe both instances sharing an `ownerJid`. The one whose
  probe succeeds holds the live companion link; the other is the zombie — tear it down via
  Recommendation (1).
- For richer signals server-side, consider patching `connectionUpdate` to also emit a webhook (and write
  the DB) on the reconnectable codes (especially `connectionReplaced` 440), and to expose the underlying
  `client.ws` readyState / `lastDisconnect` through `connectionState`.

---

## Key file:line index

| What | Location |
|---|---|
| `logout` / `delete` / `connectionState` / `fetchInstances` routes | `src/api/routes/instance.router.ts:37-98` |
| `InstanceController.logout()` | `src/api/controllers/instance.controller.ts:436-450` |
| `InstanceController.deleteInstance()` | `src/api/controllers/instance.controller.ts:452-476` |
| `InstanceController.connectionState()` | `src/api/controllers/instance.controller.ts:393-400` |
| `InstanceController.fetchInstances()` | `src/api/controllers/instance.controller.ts:402-430` |
| `logoutInstance()` (calls `client.logout()`) | `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts:267-299` |
| `connectionStatus` getter / `stateConnection` field | `whatsapp.baileys.service.ts:259-265` |
| `connectionUpdate()` — close / reconnect / webhook logic | `whatsapp.baileys.service.ts:334-521` |
| `codesToNotReconnect` / `shouldReconnect` | `whatsapp.baileys.service.ts:426-431` |
| `instanceInfo()` (fetchInstances data source) | `src/api/services/monitor.service.ts:86-130` |
| `cleaningUp()` / `cleaningStoreData()` | `src/api/services/monitor.service.ts:159-225` |
| `WAMonitoringService.deleteInstance()` (internal force-delete) | `src/api/services/monitor.service.ts:265-271` |
| `remove.instance` / `logout.instance` listeners | `src/api/services/monitor.service.ts:392-426` |
| `setInstance()` auto-reconnect on boot | `src/api/services/monitor.service.ts:273-308` |
| Webhook envelope shape | `src/api/integrations/event/webhook/webhook.controller.ts:93-103` |
| `InternalServerErrorException` / `BadRequestException` | `src/exceptions/500.exception.ts`, `src/exceptions/400.exception.ts` |
| Global error middleware (500 JSON shape) | `src/main.ts:76-118` |
| `Instance` Prisma model (`ownerJid`, `number`, no unique on either) | `prisma/postgresql-schema.prisma` |
