# Per-Participant Read Receipts: Feasibility Analysis

> **VERDICT: YES, fully feasible.** Baileys already delivers per-participant receipt data — Evolution API currently discards it.

---

## 1. What WhatsApp / Baileys Already Provides

Each `message-receipt.update` event contains an array of `MessageUserReceiptUpdate` objects. Each object includes **per-participant, per-message** receipt data:

```typescript
type MessageUserReceiptUpdate = {
  key: WAMessageKey;       // Which message
  receipt: IUserReceipt;   // Who read/received it, and when
};
```

### Available Fields Per Receipt

| Field | Path | Type | Description |
|-------|------|------|-------------|
| Message ID | `key.id` | `string` | Unique message identifier |
| Group JID | `key.remoteJid` | `string` | Group chat JID (e.g., `120363xxx@g.us`) |
| Message sender | `key.participant` | `string` | Who **sent** the original message |
| Sent by me? | `key.fromMe` | `boolean` | Whether message was from the connected account |
| **Receipting participant** | `receipt.userJid` | `string` | **Who read/received the message** |
| Delivered timestamp | `receipt.receiptTimestamp` | `number` | Unix epoch — when delivered to this participant |
| Read timestamp | `receipt.readTimestamp` | `number` | Unix epoch — when read by this participant |

### How WhatsApp Emits These Events

1. You send a message to a group with 50 members
2. As **each member's device receives** the message → a **delivery receipt** arrives with `receipt.receiptTimestamp`
3. As **each member opens the chat** and views the message → a **read receipt** arrives with `receipt.readTimestamp`
4. One receipt node can batch **multiple message IDs** (if a member reads several unread messages at once)
5. **Group read receipts cannot be disabled** by members — WhatsApp always sends them in groups

So for a group with N members, you receive up to N separate receipt events per message, each identifying the specific participant.

---

## 2. What Evolution API Currently Does (Discards the Data)

Current handler in [whatsapp.baileys.service.ts:1964-1997](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1964-L1997):

```typescript
if (events['message-receipt.update']) {
  const payload = events['message-receipt.update'] as MessageUserReceiptUpdate[];
  const remotesJidMap: Record<string, { timestamp: number; hasExternalReceipt: boolean }> = {};

  for (const event of payload) {
    // ❌ Only uses: event.key.remoteJid, event.receipt.readTimestamp
    // ❌ DISCARDS: event.receipt.userJid (who read it)
    // ❌ DISCARDS: event.key.id (which specific message)
    // ❌ DISCARDS: event.receipt.receiptTimestamp (delivery time)
    // ❌ Only checks readTimestamp, ignoring delivery receipts entirely
  }

  // Bulk-updates ALL messages in the group up to a timestamp → READ
  // No per-message, no per-participant granularity preserved
}
```

**Three things are lost:**
1. **Who** read/received it (`receipt.userJid`)
2. **Which specific message** they read (`key.id`)
3. **Delivery vs Read distinction** per participant (`receiptTimestamp` vs `readTimestamp`)

---

## 3. What's Needed to Capture Per-Participant Data

### 3.1 New Prisma Model: `MessageReceipt`

```prisma
model MessageReceipt {
  id            String   @id @default(cuid())
  instanceId    String
  keyId         String      // Message key ID (key.id)
  remoteJid     String      // Group JID
  participantJid String     // Who read/received (receipt.userJid)
  type          String      // 'delivered' | 'read' | 'played'
  timestamp     Int         // Unix epoch of receipt
  createdAt     DateTime @default(now())

  Instance      Instance @relation(fields: [instanceId], references: [id], onDelete: Cascade)
  Message       Message  @relation(fields: [messageId], references: [id], onDelete: Cascade)
  messageId     String

  @@unique([instanceId, keyId, participantJid, type])
  @@index([instanceId, keyId])
  @@index([instanceId, remoteJid])
}
```

The `@@unique` constraint ensures upsert behavior — a participant's delivery receipt gets upgraded to read without creating duplicates.

### 3.2 Modified `message-receipt.update` Handler

```typescript
if (events['message-receipt.update']) {
  const payload = events['message-receipt.update'] as MessageUserReceiptUpdate[];

  for (const event of payload) {
    const { key, receipt } = event;
    const participantJid = receipt.userJid;
    const messageKeyId = key.id;
    const remoteJid = key.remoteJid;

    // Determine receipt type and timestamp
    let type: string;
    let timestamp: number;

    if (typeof receipt.readTimestamp === 'number') {
      type = 'read';
      timestamp = receipt.readTimestamp;
    } else if (typeof receipt.receiptTimestamp === 'number') {
      type = 'delivered';
      timestamp = receipt.receiptTimestamp;
    } else {
      continue; // No useful receipt data
    }

    // Upsert per-participant receipt
    await this.prismaRepository.messageReceipt.upsert({
      where: {
        instanceId_keyId_participantJid_type: {
          instanceId: this.instanceId,
          keyId: messageKeyId,
          participantJid,
          type,
        },
      },
      create: {
        instanceId: this.instanceId,
        keyId: messageKeyId,
        remoteJid,
        participantJid,
        type,
        timestamp,
        messageId: /* look up by keyId */,
      },
      update: {
        timestamp, // Update to latest timestamp
      },
    });
  }

  // Keep existing bulk status update for Message.status field
  // (the aggregate "highest status" for display purposes)
}
```

### 3.3 API Endpoint: Get Receipts for a Message

```
GET /chat/messageReceipts/{instanceName}?messageKeyId=3EB0XXXX&remoteJid=120363xxx@g.us
```

Response:
```json
{
  "messageKeyId": "3EB0XXXX",
  "remoteJid": "120363426504089458@g.us",
  "readBy": [
    { "participantJid": "5511999991111@s.whatsapp.net", "timestamp": 1738800120 },
    { "participantJid": "5511999992222@s.whatsapp.net", "timestamp": 1738800180 }
  ],
  "deliveredTo": [
    { "participantJid": "5511999993333@s.whatsapp.net", "timestamp": 1738800060 },
    { "participantJid": "5511999994444@s.whatsapp.net", "timestamp": 1738800065 }
  ],
  "pending": [
    "5511999995555@s.whatsapp.net"
  ]
}
```

The `pending` list would be computed by comparing group participants against stored receipts.

### 3.4 Optional: Webhook Emission

Emit a webhook event whenever a new receipt is stored:

```json
{
  "event": "message.receipt",
  "instance": "myInstance",
  "data": {
    "keyId": "3EB0XXXX",
    "remoteJid": "120363426504089458@g.us",
    "participantJid": "5511999991111@s.whatsapp.net",
    "type": "read",
    "timestamp": 1738800120
  }
}
```

This would allow your Laravel app to receive real-time per-participant read notifications without polling.

---

## 4. Implementation Considerations

### 4.1 Database Volume

For a group with 100 members and 50 messages/day:
- Up to **100 delivery receipts + 100 read receipts** per message = 200 rows/message
- 50 messages × 200 = **10,000 rows/day per group**
- With 10 active groups: **100,000 rows/day**

Mitigation strategies:
- **TTL/cleanup**: Auto-delete receipts older than X days (most useful data is recent)
- **Config toggle**: Make per-participant receipt storage optional via env var (e.g., `DATABASE_SAVE_PER_PARTICIPANT_RECEIPTS=true`)
- **Proper indexing**: The `@@index([instanceId, keyId])` index is critical for query performance

### 4.2 Race Conditions

Multiple receipt events can arrive simultaneously for the same message from different participants. The `@@unique` constraint with `upsert` handles this gracefully — no duplicates, and concurrent writes for different participants don't conflict.

### 4.3 Delivery → Read Upgrade

When a participant first delivers then reads a message, you get two separate events. The schema stores them as two rows (`type: 'delivered'` and `type: 'read'`). An alternative design would be a single row per participant with nullable `deliveredAt` and `readAt` columns:

```prisma
model MessageReceipt {
  // ...
  participantJid  String
  deliveredAt     Int?     // receiptTimestamp
  readAt          Int?     // readTimestamp
  @@unique([instanceId, keyId, participantJid])
}
```

This halves the row count but requires conditional upsert logic.

### 4.4 Message ID Lookup

The `key.id` in the receipt is the WhatsApp message ID (e.g., `3EB0XXXX`), stored inside the JSON `key` column of the `Message` table. To link `MessageReceipt.messageId` to `Message.id` (the Prisma UUID), you need a lookup:

```typescript
// MySQL
const msg = await this.prismaRepository.$queryRawUnsafe(
  `SELECT id FROM Message WHERE instanceId = ? AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.id')) = ? LIMIT 1`,
  this.instanceId, messageKeyId
);
```

Or store `keyId` (the WhatsApp message ID string) directly on `MessageReceipt` without a foreign key to `Message`, and join at query time. This avoids the lookup overhead during receipt ingestion.

### 4.5 Current Code Already Has the Data

The `receipt.userJid` field is **already present** in every `message-receipt.update` event. The data arrives at your server — it just gets thrown away. No additional WhatsApp API calls, no protocol changes, no extra connections needed. It's purely a matter of storing what's already there.

---

## 5. Scope of Changes

| Component | Change Required |
|-----------|----------------|
| Prisma schema (mysql + postgresql) | Add `MessageReceipt` model |
| Migration | Generate and deploy new migration |
| `message-receipt.update` handler | Store per-participant receipts instead of (or in addition to) bulk update |
| New API endpoint | `/chat/messageReceipts/{instanceName}` |
| Webhook (optional) | New `message.receipt` event type |
| Config (optional) | `DATABASE_SAVE_PER_PARTICIPANT_RECEIPTS` toggle |

### Files to Modify

| File | Change |
|------|--------|
| `prisma/mysql-schema.prisma` | Add `MessageReceipt` model |
| `prisma/postgresql-schema.prisma` | Add `MessageReceipt` model |
| `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts` | Modify `message-receipt.update` handler |
| `src/api/services/channel.service.ts` | Add `fetchMessageReceipts` method |
| `src/api/controllers/chat.controller.ts` | Add `messageReceipts` route handler |
| `src/api/routes/chat.router.ts` | Add route |
| `src/api/dto/chat.dto.ts` | Add DTO |
| `src/validate/chat.schema.ts` | Add validation schema |

---

## 6. Summary

| Question | Answer |
|----------|--------|
| Is the data available from WhatsApp? | **Yes** — Baileys provides per-participant, per-message receipts |
| Does it include "Read by" with timestamps? | **Yes** — `receipt.userJid` + `receipt.readTimestamp` |
| Does it include "Delivered to" with timestamps? | **Yes** — `receipt.userJid` + `receipt.receiptTimestamp` |
| Can we compute "Pending" (not yet delivered)? | **Yes** — by comparing group members against stored receipts |
| Do we need protocol changes? | **No** — data already arrives, just needs to be stored |
| Is it high volume? | **Moderate** — ~200 rows per message per 100-member group |
| Can it be made optional? | **Yes** — config toggle recommended |

The WhatsApp "Message Info" screen (showing "Read by", "Delivered to", "Pending") can be fully replicated with the data Baileys already provides. The only missing piece is **storage and API exposure** in Evolution API.
