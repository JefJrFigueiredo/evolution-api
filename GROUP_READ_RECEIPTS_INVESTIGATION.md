# Evolution API: Group Read Receipts Not Being Stored in MessageUpdate Table

## Quick Answer

Group read receipts are **not stored in the `MessageUpdate` table** because:

1. **`message-receipt.update`** (Baileys event for per-participant group read receipts) only updates the `Message.status` column directly — it **never creates `MessageUpdate` records**
2. **`messages.update`** (Baileys event that DOES create `MessageUpdate` records) has a **`groupsIgnore` filter** at line 1586 that skips all `@g.us` JIDs when `groupsIgnore=true` in instance settings
3. Even with `groupsIgnore=false`, Baileys may **not fire `messages.update` for group read receipts** — it uses `message-receipt.update` instead for per-participant group receipts

This means **there is no code path** that creates `MessageUpdate` records with `READ` status for group messages.

---

## 1. The Two Distinct Baileys Events

Baileys fires two separate events for message status changes. They have different purposes and Evolution API handles them very differently:

| Event | When It Fires | What Evolution API Does |
|-------|--------------|------------------------|
| `messages.update` | Private chat status changes (sent → delivered → read), edits, deletes, polls | Creates `MessageUpdate` records, sends `MESSAGES_UPDATE` webhook |
| `message-receipt.update` | Per-participant read receipts in **groups** | Updates `Message.status` column via raw SQL, does **NOT** create `MessageUpdate` records, does **NOT** send webhooks |

### Key Insight

These are **not the same event**. Group read receipts arrive through `message-receipt.update`, which follows a completely different code path than `messages.update`.

---

## 2. `message-receipt.update` Handler (Groups) — Lines 1964-1979

**File:** [whatsapp.baileys.service.ts:1964-1979](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1964-L1979)

```typescript
if (events['message-receipt.update']) {
  const payload = events['message-receipt.update'] as MessageUserReceiptUpdate[];
  const remotesJidMap: Record<string, number> = {};

  for (const event of payload) {
    if (typeof event.key.remoteJid === 'string' && typeof event.receipt.readTimestamp === 'number') {
      remotesJidMap[event.key.remoteJid] = event.receipt.readTimestamp;
    }
  }

  await Promise.all(
    Object.keys(remotesJidMap).map(async (remoteJid) =>
      this.updateMessagesReadedByTimestamp(remoteJid, remotesJidMap[remoteJid]),
    ),
  );
}
```

### What This Does

1. Receives per-participant read receipts (e.g., "User X read messages in group Y up to timestamp Z")
2. Builds a map of `{ remoteJid → readTimestamp }`
3. Calls `updateMessagesReadedByTimestamp()` for each group

### What This Does NOT Do

- **Does NOT create `MessageUpdate` records** — no `prismaRepository.messageUpdate.create()` call
- **Does NOT send any webhook** — no `sendDataWebhook()` call
- **Does NOT check `groupsIgnore`** — it processes all receipts regardless of settings
- **Does NOT check `DATABASE_SAVE_MESSAGE_UPDATE`** — irrelevant since it never touches `MessageUpdate`

---

## 3. `updateMessagesReadedByTimestamp()` — Lines 4796-4846

**File:** [whatsapp.baileys.service.ts:4796-4846](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4796-L4846)

This is what `message-receipt.update` calls. It does a **bulk UPDATE** on the `Message` table directly:

```sql
-- MySQL version (lines 4806-4818):
UPDATE Message
SET status = 'READ'
WHERE instanceId = ?
AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.remoteJid')) = ?
AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.fromMe')) = 'false'
AND messageTimestamp <= ?
AND (status IS NULL OR status = 'DELIVERY_ACK')
```

### What This Updates

- Sets `Message.status = 'READ'` for all **incoming** messages (`fromMe = false`) in that group
- Only updates messages that are currently `DELIVERY_ACK` or `NULL` (not already read)
- Uses `messageTimestamp <= readTimestamp` to mark all messages up to the read point

### Critical Detail

This updates the `Message.status` column — **NOT** the `MessageUpdate` table. So when you query `findMessages`, the `Message` records will have `status: 'READ'` on the Message row itself, but the `MessageUpdate` relation array won't contain a `READ` entry.

---

## 4. `messages.update` Handler (Private Chats) — Lines 1580-1767

**File:** [whatsapp.baileys.service.ts:1580-1767](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1580-L1767)

This is the handler that **DOES** create `MessageUpdate` records. But it has a group filter:

```typescript
'messages.update': async (args, settings) => {
  for await (const { key, update } of args) {
    // ⚠️ LINE 1586: Groups are skipped if groupsIgnore is true
    if (settings?.groupsIgnore && key.remoteJid?.includes('@g.us')) {
      continue;
    }
    // ... rest of handler
```

### The `MessageUpdate` creation happens at line 1741-1744:

```typescript
if (this.configService.get<Database>('DATABASE').SAVE_DATA.MESSAGE_UPDATE) {
  const { message: _msg, ...messageData } = message;
  await this.prismaRepository.messageUpdate.create({ data: messageData });
}
```

### The Problem: Even Without `groupsIgnore`, Groups Are Still Unlikely to Hit This Path

Even if `groupsIgnore=false`, Baileys typically sends group read receipts through `message-receipt.update` (per-participant), NOT through `messages.update`. The `messages.update` event fires for:

- **Private chat status changes** (sent → server_ack → delivery_ack → read)
- **Message edits** (the `update.message` field is set)
- **Message deletes** (`update.message === null && update.status === undefined`)
- **Poll updates** (`update.pollUpdates` is set)

For groups, Baileys uses `message-receipt.update` because each participant reads at a different time — there's no single "read" status for the whole group.

---

## 5. Flow Diagram: Where Each Event Goes

```
Baileys receives status change
        │
        ├─── Private chat read receipt
        │         │
        │         ▼
        │    events['messages.update']  (line 1959)
        │         │
        │         ▼
        │    messageHandle['messages.update']  (line 1580)
        │         │
        │         ├── groupsIgnore check (line 1586) → skip if group
        │         ├── Dedup cache check (line 1592-1603)
        │         ├── Find original message in DB (line 1651-1690)
        │         ├── Update Message.status (line 1727-1730)
        │         ├── Send MESSAGES_UPDATE webhook (line 1739) ✅
        │         └── Create MessageUpdate record (line 1741-1744) ✅
        │
        └─── Group per-participant read receipt
                  │
                  ▼
             events['message-receipt.update']  (line 1964)
                  │
                  ▼
             Build { remoteJid → readTimestamp } map (line 1966-1972)
                  │
                  ▼
             updateMessagesReadedByTimestamp() (line 4796)
                  │
                  ├── Bulk UPDATE Message.status = 'READ' (line 4806-4818) ✅
                  ├── Update Chat.unreadMessages (line 4848-4893) ✅
                  ├── Create MessageUpdate record ❌ NEVER HAPPENS
                  └── Send webhook ❌ NEVER HAPPENS
```

---

## 6. Why It "Worked Before" (Container Restart Theory)

You mentioned that before container restarts, `findMessages` DID show some messages with `MessageUpdate: [{ "status": "READ" }]`. There are a few explanations:

### Theory A: Those Were Private Chat Messages

If you were looking at private chat messages (not group), the `messages.update` handler would have created `MessageUpdate` records normally. After container restart, the instance reconnects and re-processes history, but read receipts from before the restart aren't replayed.

### Theory B: Baileys Occasionally Fires `messages.update` for Groups

In some edge cases, Baileys may fire `messages.update` instead of `message-receipt.update` for group status changes — for example:
- When **you** (the sender) get an aggregate status update (all participants have read)
- During history sync after reconnection
- For `SERVER_ACK` status (message received by WhatsApp server)

These would create `MessageUpdate` records, but they'd be `SERVER_ACK` or `DELIVERY_ACK`, not `READ`.

### Theory C: Different `groupsIgnore` Setting

If `groupsIgnore` was `false` at the time AND Baileys happened to fire `messages.update` for groups, the handler would have processed them. After a restart, the setting may have changed, or the reconnection behavior is different.

---

## 7. Configuration Requirements

For `MessageUpdate` records to be created for **any** message type:

| Environment Variable | Required Value | What It Controls |
|---------------------|---------------|-----------------|
| `DATABASE_SAVE_MESSAGE_UPDATE=true` | ✅ Required | Gate for `messageUpdate.create()` at line 1741 |
| `DATABASE_SAVE_DATA_NEW_MESSAGE=true` OR `DATABASE_SAVE_DATA_HISTORIC=true` | ✅ At least one | Gate for the `findMessage` lookup at line 1651 — without this, `findMessage` is null and the handler skips (line 1685-1688) |
| `groupsIgnore=false` (instance setting) | ✅ Required for groups | Gate at line 1586 — skips all `@g.us` JIDs if true |

**But even with all of these set correctly**, group read receipts still won't create `MessageUpdate` records because they arrive through `message-receipt.update`, not `messages.update`.

---

## 8. What Gets Stored vs. What Doesn't

| Scenario | `Message.status` Updated? | `MessageUpdate` Record Created? | Webhook Sent? |
|----------|--------------------------|-------------------------------|---------------|
| **Private chat:** sent → server_ack | ✅ Yes (via `messages.update`) | ✅ Yes | ✅ `MESSAGES_UPDATE` |
| **Private chat:** delivered → read | ✅ Yes (via `messages.update`) | ✅ Yes | ✅ `MESSAGES_UPDATE` |
| **Group:** per-participant read receipt | ✅ Yes (via `message-receipt.update` → `updateMessagesReadedByTimestamp`) | ❌ **No** | ❌ **No** |
| **Group:** server_ack/delivery_ack (if `messages.update` fires) | ✅ Yes (if `groupsIgnore=false`) | ✅ Yes (if `groupsIgnore=false`) | ✅ (if `groupsIgnore=false`) |
| **Group:** message edit | ✅ Yes (if `groupsIgnore=false`) | ✅ Yes (if `groupsIgnore=false`) | ✅ (if `groupsIgnore=false`) |

---

## 9. How to Get Group Read Receipt Data

### Option A: Query `Message.status` Directly (Works Now)

Since `message-receipt.update` DOES update `Message.status`, you can get read status from the `Message` table itself:

```json
POST /chat/findMessages/{instanceName}

{
  "where": {
    "key": {
      "remoteJid": "120363426504089458@g.us",
      "fromMe": true
    }
  },
  "offset": 50,
  "page": 1
}
```

In the response, check each message's top-level `status` field (NOT the `MessageUpdate` relation):

```php
// Laravel
$messages = $response['messages']['records'];

foreach ($messages as $msg) {
    $status = $msg['status'];  // ← This IS updated by message-receipt.update
    // Values: 'SERVER_ACK', 'DELIVERY_ACK', 'READ', 'PLAYED', etc.

    $messageUpdates = $msg['MessageUpdate'];  // ← This will likely be empty for groups
}
```

**Important:** This requires the `findMessages` MySQL fix from `2.3.7-fix-0.0.4` to filter by group JID.

### Option B: Code Modification (For `MessageUpdate` + Webhook)

If you need `MessageUpdate` records AND webhooks for group read receipts, the `message-receipt.update` handler (lines 1964-1979) would need to be modified to:

1. Create `MessageUpdate` records for each receipt
2. Send `MESSAGES_UPDATE` (or a new `MESSAGE_RECEIPT_UPDATE`) webhook

This would look something like:

```typescript
// Conceptual modification to message-receipt.update handler
if (events['message-receipt.update']) {
  const payload = events['message-receipt.update'] as MessageUserReceiptUpdate[];
  const remotesJidMap: Record<string, number> = {};

  for (const event of payload) {
    if (typeof event.key.remoteJid === 'string' && typeof event.receipt.readTimestamp === 'number') {
      remotesJidMap[event.key.remoteJid] = event.receipt.readTimestamp;

      // NEW: Create MessageUpdate record
      if (this.configService.get<Database>('DATABASE').SAVE_DATA.MESSAGE_UPDATE) {
        await this.prismaRepository.messageUpdate.create({
          data: {
            keyId: event.key.id,
            remoteJid: event.key.remoteJid,
            fromMe: event.key.fromMe,
            participant: event.receipt.userJid,  // Who read it
            status: 'READ',
            instanceId: this.instanceId,
          },
        });
      }

      // NEW: Send webhook
      this.sendDataWebhook(Events.MESSAGES_UPDATE, {
        keyId: event.key.id,
        remoteJid: event.key.remoteJid,
        fromMe: event.key.fromMe,
        participant: event.receipt.userJid,
        status: 'READ',
        readTimestamp: event.receipt.readTimestamp,
      });
    }
  }

  // Existing: bulk update Message.status
  await Promise.all(
    Object.keys(remotesJidMap).map(async (remoteJid) =>
      this.updateMessagesReadedByTimestamp(remoteJid, remotesJidMap[remoteJid]),
    ),
  );
}
```

**Caveat:** This would create many more `MessageUpdate` records (one per participant per message), and the webhook volume could be high for large groups.

---

## 10. The `findMessage` Gate: Another Silent Failure Point

Even in the `messages.update` handler, there's a gate that can silently skip records:

**Lines 1651-1688:**
```typescript
if (configDatabaseData.SAVE_DATA.HISTORIC || configDatabaseData.SAVE_DATA.NEW_MESSAGE) {
  // ... find the original message in DB ...
  findMessage = messages[0] || null;

  if (!findMessage?.id) {
    this.logger.warn(`Original message not found for update. Skipping. Key: ${JSON.stringify(key)}`);
    continue;  // ← Silently skips if original message not in DB
  }
  message.messageId = findMessage.id;
}
```

If either:
- `DATABASE_SAVE_DATA_HISTORIC` and `DATABASE_SAVE_DATA_NEW_MESSAGE` are both `false`, OR
- The original message wasn't saved to the DB (e.g., arrived before `DATABASE_SAVE_DATA_NEW_MESSAGE` was enabled)

Then `findMessage` will be null/undefined, and the handler **silently skips** — no `MessageUpdate` record, no webhook, no error. Just a warning log.

---

## 11. Summary

| Question | Answer |
|----------|--------|
| **Where does `message-receipt.update` get handled?** | Lines [1964-1979](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1964-L1979) — updates `Message.status` via raw SQL, no `MessageUpdate` created |
| **Separate events for group vs private?** | Yes: groups use `message-receipt.update` (per-participant), private chats use `messages.update` |
| **Code path for `MessageUpdate` with `READ`?** | Only through `messages.update` handler at [line 1741](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1741) — groups typically don't hit this path |
| **Difference between the two events?** | `messages.update` → creates `MessageUpdate` + sends webhook; `message-receipt.update` → only bulk-updates `Message.status` |
| **Config/settings that control storage?** | `DATABASE_SAVE_MESSAGE_UPDATE`, `DATABASE_SAVE_DATA_NEW_MESSAGE`/`HISTORIC`, and `groupsIgnore` — but none help because the group receipt event bypasses all of them |

### The Root Cause

The `message-receipt.update` handler was designed as a lightweight bulk-update mechanism. It efficiently marks messages as read in the `Message` table but was never wired to create `MessageUpdate` records or send webhooks. This is a **design gap**, not a configuration issue.

### Recommended Approach

1. **For now:** Query `Message.status` via `findMessages` (with the 0.0.4 MySQL fix) — the `READ` status IS there on the Message record itself
2. **For `MessageUpdate` records:** Requires a code modification to the `message-receipt.update` handler
3. **For webhooks:** Same code modification needed — add `sendDataWebhook()` to the handler

---

## 12. Code References

| Feature | File | Lines |
|---------|------|-------|
| `message-receipt.update` handler | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1964-1979](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1964-L1979) |
| `messages.update` handler | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1580-1767](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1580-L1767) |
| `groupsIgnore` filter in `messages.update` | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1586](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1586) |
| `MessageUpdate` creation (private chats) | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1741-1744](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1741-L1744) |
| `updateMessagesReadedByTimestamp` | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [4796-4846](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4796-L4846) |
| `updateChatUnreadMessages` | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [4848-4893](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4848-L4893) |
| `findMessage` gate (HISTORIC/NEW_MESSAGE) | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1651-1690](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1651-L1690) |
| Status mapping | [renderStatus.ts](src/utils/renderStatus.ts) | [1-10](src/utils/renderStatus.ts#L1-L10) |
| `DATABASE_SAVE_MESSAGE_UPDATE` config | [env.config.ts](src/config/env.config.ts) | [492](src/config/env.config.ts#L492) |
