# Evolution API v2.3.7 — How `findMessages` Works

## The Problem: Your Filters Are Being Silently Ignored

Your current request body uses **dot-notation keys**:

```json
{
  "where": {
    "key.remoteJid": "120363426504089458@g.us",
    "key.fromMe": true
  },
  "limit": 50,
  "page": 1
}
```

This has **three issues** that cause all filters to be silently ignored:

1. `"key.remoteJid"` should be a **nested object**: `"key": { "remoteJid": "..." }`
2. `"limit"` is accepted by validation but **never read** by the service — the actual page-size field is `"offset"`
3. Without a valid `key` object, the Prisma `AND` clause evaluates to `[{}, {}, {}, {}]` — which matches **everything**

---

## 1. The Correct Request Format

### Filtering by group + outgoing messages

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

### All available filter options

```json
{
  "where": {
    "key": {
      "remoteJid": "120363426504089458@g.us",
      "fromMe": true,
      "id": "3EB0ABC123DEF456"
    },
    "messageType": "conversation",
    "source": "android",
    "messageTimestamp": {
      "gte": "2026-01-01T00:00:00.000Z",
      "lte": "2026-02-06T23:59:59.000Z"
    }
  },
  "offset": 100,
  "page": 1
}
```

---

## 2. How It Works Internally

### Code Path

```
POST /chat/findMessages/{instanceName}
  → chat.router.ts:164-172 (validation via messageValidateSchema)
  → chat.controller.ts:61 → fetchMessages(query)
  → channel.service.ts:600-687 → Prisma query
```

### The `where.key` Extraction

The service extracts `key` from the `where` clause and uses Prisma JSON path filtering:

**File:** [channel.service.ts:600-606](src/api/services/channel.service.ts#L600-L606)

```typescript
public async fetchMessages(query: Query<Message>) {
  const keyFilters = query?.where?.key as {
    id?: string;
    fromMe?: boolean;
    remoteJid?: string;
    participants?: string;
  };
```

This expects `query.where.key` to be an **object** with nested properties. If you pass `"key.remoteJid"` as a flat string key instead, `query.where.key` is `undefined`, and `keyFilters` becomes `undefined`.

### The Prisma Query

**File:** [channel.service.ts:642-660](src/api/services/channel.service.ts#L642-L660)

```typescript
const messages = await this.prismaRepository.message.findMany({
  where: {
    instanceId: this.instanceId,
    id: query?.where?.id,
    source: query?.where?.source,
    messageType: query?.where?.messageType,
    ...timestampFilter,
    AND: [
      keyFilters?.id        ? { key: { path: ['id'], equals: keyFilters?.id } }        : {},
      keyFilters?.fromMe    ? { key: { path: ['fromMe'], equals: keyFilters?.fromMe } }    : {},
      keyFilters?.remoteJid ? { key: { path: ['remoteJid'], equals: keyFilters?.remoteJid } } : {},
      keyFilters?.participants ? { key: { path: ['participants'], equals: keyFilters?.participants } } : {},
    ],
  },
  orderBy: { messageTimestamp: 'desc' },
  skip: query.offset * (query?.page === 1 ? 0 : (query?.page as number) - 1),
  take: query.offset,
  select: {
    id: true,
    key: true,
    pushName: true,
    messageType: true,
    message: true,
    messageTimestamp: true,
    instanceId: true,
    source: true,
    contextInfo: true,
    MessageUpdate: {        // ← Includes status updates!
      select: {
        status: true,
      },
    },
  },
});
```

**Key insights:**
- The `key` column is a **JSON field** in the database (not separate columns)
- Filtering uses Prisma's JSON path queries: `{ key: { path: ['remoteJid'], equals: value } }`
- If a key filter is `undefined`, it becomes `{}` (empty — matches everything)
- When ALL filters are empty, the `AND: [{}, {}, {}, {}]` matches every row
- **MessageUpdate records are included** in the response automatically as a nested array

### Why Dot-Notation Fails

When you send `"key.remoteJid": "..."` in the body:

```typescript
// What happens with Object.assign:
ref.where = {
  "key.remoteJid": "120363426504089458@g.us",  // A single key with a literal dot
  "key.fromMe": true
};

// The service reads:
const keyFilters = query.where.key;  // undefined! There's no "key" property
// keyFilters is undefined → all AND conditions become {} → matches everything
```

---

## 3. Pagination: `offset` vs `limit`

### The Naming Confusion

The `Query<T>` class ([repository.service.ts:5-10](src/api/repository/repository.service.ts#L5-L10)):

```typescript
export class Query<T> {
  where?: T;
  sort?: 'asc' | 'desc';
  page?: number;
  offset?: number;  // ← This is actually the PAGE SIZE (items per page)
}
```

The validation schema allows `limit` ([chat.schema.ts:240](src/validate/chat.schema.ts#L240)):

```typescript
properties: {
  where: { ... },
  limit: { type: 'integer' },  // ← Accepted by validation...
},
```

But the service only reads `offset` ([channel.service.ts:634-636](src/api/services/channel.service.ts#L634-L636)):

```typescript
if (!query?.offset) {
  query.offset = 50;  // ← Defaults to 50
}
```

**Bottom line:**

| Body Field | Accepted by Validation | Read by Service | Effect |
|------------|----------------------|-----------------|--------|
| `"limit": 100` | Yes | **No** | Silently ignored |
| `"offset": 100` | Yes (not restricted) | **Yes** | Sets page size to 100 |
| Neither | — | — | Defaults to 50 |

**There is no hardcoded maximum** — `offset` can be any integer. It defaults to 50 if omitted.

### How Pagination Works

```typescript
skip: query.offset * (query?.page === 1 ? 0 : (query?.page as number) - 1),
take: query.offset,
```

| `page` | `offset` | `skip` | `take` | Records |
|--------|----------|--------|--------|---------|
| 1 | 50 | 0 | 50 | 1-50 |
| 2 | 50 | 50 | 50 | 51-100 |
| 3 | 50 | 100 | 50 | 101-150 |
| 1 | 100 | 0 | 100 | 1-100 |

### Response Structure

```json
{
  "messages": {
    "total": 342,
    "pages": 7,
    "currentPage": 1,
    "records": [
      {
        "id": "clx1abc...",
        "key": {
          "id": "3EB0ABC123...",
          "fromMe": true,
          "remoteJid": "120363426504089458@g.us"
        },
        "pushName": "John",
        "messageType": "conversation",
        "message": { "conversation": "Hello!" },
        "messageTimestamp": 1704067200,
        "instanceId": "...",
        "source": "android",
        "contextInfo": null,
        "MessageUpdate": [
          { "status": "SERVER_ACK" },
          { "status": "DELIVERY_ACK" },
          { "status": "READ" }
        ]
      }
    ]
  }
}
```

**Notice:** Each message includes a `MessageUpdate` array with all status updates. This is exactly what you need for tracking read receipts!

---

## 4. Querying MessageUpdate Directly: `findStatusMessage`

There's a separate endpoint specifically for querying message status updates.

### Endpoint

```
POST /chat/findStatusMessage/{instanceName}
```

### Implementation

**File:** [channel.service.ts:689-707](src/api/services/channel.service.ts#L689-L707)

```typescript
public async fetchStatusMessage(query: any) {
  if (!query?.offset) { query.offset = 50; }
  if (!query?.page) { query.page = 1; }

  return await this.prismaRepository.messageUpdate.findMany({
    where: {
      instanceId: this.instanceId,
      remoteJid: query.where?.remoteJid,
      keyId: query.where?.id,
    },
    skip: query.offset * (query?.page === 1 ? 0 : (query?.page as number) - 1),
    take: query.offset,
  });
}
```

### Request Format

```json
POST /chat/findStatusMessage/{instanceName}

{
  "where": {
    "remoteJid": "120363426504089458@g.us",
    "id": "3EB0ABC123DEF456"
  },
  "offset": 100,
  "page": 1
}
```

**Note:** Here `where.remoteJid` and `where.id` are flat (not nested inside `key`), because `MessageUpdate` has these as proper columns.

### Supported Filters

| Filter | Maps To | Notes |
|--------|---------|-------|
| `where.remoteJid` | `MessageUpdate.remoteJid` | Filter by group/chat JID |
| `where.id` | `MessageUpdate.keyId` | Filter by message key ID |

**Not supported** (despite being in validation schema):

| Schema Field | Used in Query? |
|-------------|---------------|
| `where.fromMe` | No |
| `where.status` | No |
| `where.participant` | No |

### MessageUpdate Database Schema

**File:** [mysql-schema.prisma:184-199](prisma/mysql-schema.prisma#L184-L199)

```prisma
model MessageUpdate {
  id          String   @id @default(cuid())
  keyId       String   @db.VarChar(100)   // Message key ID
  remoteJid   String   @db.VarChar(100)   // Group/chat JID
  fromMe      Boolean                      // Was it from the instance
  participant String?  @db.VarChar(100)    // Participant in group
  pollUpdates Json?    @db.Json
  status      String   @db.VarChar(30)     // ERROR, PENDING, SERVER_ACK, DELIVERY_ACK, READ, PLAYED
  Message     Message  @relation(...)
  messageId   String                       // FK to Message.id
  Instance    Instance @relation(...)
  instanceId  String
}
```

---

## 5. Strategy for Polling Group Read Receipts

### Option A: Use `findMessages` (Recommended)

**Best approach** — returns messages WITH their status updates in a single query.

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

**What you get back:** Each message includes a `MessageUpdate` array:

```json
{
  "messages": {
    "total": 15,
    "pages": 1,
    "currentPage": 1,
    "records": [
      {
        "key": {
          "id": "3EB0ABC123...",
          "fromMe": true,
          "remoteJid": "120363426504089458@g.us"
        },
        "messageTimestamp": 1704067200,
        "MessageUpdate": [
          { "status": "SERVER_ACK" },
          { "status": "DELIVERY_ACK" },
          { "status": "READ" }
        ]
      }
    ]
  }
}
```

**In your Laravel app:**
```php
foreach ($response['messages']['records'] as $message) {
    $statuses = collect($message['MessageUpdate'])->pluck('status');
    $isRead = $statuses->contains('READ');
    $isDelivered = $statuses->contains('DELIVERY_ACK');
    // ...
}
```

### Option B: Use `findStatusMessage` for Specific Messages

If you already know the message key ID:

```json
POST /chat/findStatusMessage/{instanceName}

{
  "where": {
    "remoteJid": "120363426504089458@g.us",
    "id": "3EB0ABC123DEF456"
  }
}
```

Returns flat `MessageUpdate` records:

```json
[
  {
    "id": "clx...",
    "keyId": "3EB0ABC123...",
    "remoteJid": "120363426504089458@g.us",
    "fromMe": true,
    "participant": null,
    "status": "READ",
    "messageId": "clx...",
    "instanceId": "..."
  }
]
```

### Option C: Use `findStatusMessage` for All Group Updates

To get all status updates for a group:

```json
POST /chat/findStatusMessage/{instanceName}

{
  "where": {
    "remoteJid": "120363426504089458@g.us"
  },
  "offset": 200,
  "page": 1
}
```

**Caveat:** This doesn't filter by `fromMe`, `status`, or time range — you'd need to filter in your application.

### Recommended Polling Approach

```
1. Periodically call findMessages with:
   - key.remoteJid = your tracked group
   - key.fromMe = true  (only outgoing messages)
   - offset = 50
   - page = 1 (most recent first)

2. For each message, check MessageUpdate array:
   - Contains "READ"? → Mark as read in your system
   - Contains "DELIVERY_ACK" only? → Delivered but not read
   - Contains "SERVER_ACK" only? → Sent but not delivered

3. Track the last messageTimestamp you processed
   to avoid re-processing old messages
```

---

## 6. Additional Filter Options

### Filter by Time Range

```json
{
  "where": {
    "key": {
      "remoteJid": "120363426504089458@g.us",
      "fromMe": true
    },
    "messageTimestamp": {
      "gte": "2026-02-06T00:00:00.000Z",
      "lte": "2026-02-06T23:59:59.000Z"
    }
  },
  "offset": 100,
  "page": 1
}
```

**Note:** Both `gte` and `lte` must be provided together — the code checks for both before applying the filter ([channel.service.ts:610](src/api/services/channel.service.ts#L610)):

```typescript
if (query.where.messageTimestamp['gte'] && query.where.messageTimestamp['lte']) {
  // Only applies filter if BOTH are present
}
```

### Filter by Message Type

```json
{
  "where": {
    "key": { "remoteJid": "120363426504089458@g.us" },
    "messageType": "conversation"
  }
}
```

Common `messageType` values: `conversation`, `imageMessage`, `videoMessage`, `audioMessage`, `documentMessage`, `stickerMessage`, `reactionMessage`, `extendedTextMessage`

### Filter by Specific Message ID

```json
{
  "where": {
    "key": {
      "id": "3EB0ABC123DEF456"
    }
  }
}
```

---

## 7. Summary: What Was Wrong and How to Fix

### Your Original Request (Broken)

```json
{
  "where": {
    "key.remoteJid": "120363426504089458@g.us",   // ❌ Dot notation
    "key.fromMe": true                              // ❌ Dot notation
  },
  "limit": 50,                                      // ❌ Wrong field name
  "page": 1
}
```

### Correct Request

```json
{
  "where": {
    "key": {                                         // ✅ Nested object
      "remoteJid": "120363426504089458@g.us",
      "fromMe": true
    }
  },
  "offset": 50,                                     // ✅ Correct field name
  "page": 1
}
```

### Quick Reference

| What you want | Field | Example |
|--------------|-------|---------|
| Filter by group | `where.key.remoteJid` | `"120363...@g.us"` |
| Only outgoing | `where.key.fromMe` | `true` |
| Specific message | `where.key.id` | `"3EB0ABC..."` |
| Message type | `where.messageType` | `"conversation"` |
| Time range | `where.messageTimestamp` | `{ "gte": "...", "lte": "..." }` |
| Page size | `offset` (not `limit`) | `50` (default), any integer |
| Page number | `page` | `1` (default) |

---

## 8. Code References

| Feature | File | Lines |
|---------|------|-------|
| Route definition | [chat.router.ts](src/api/routes/chat.router.ts) | [164-172](src/api/routes/chat.router.ts#L164-L172) |
| Controller | [chat.controller.ts](src/api/controllers/chat.controller.ts) | [61-63](src/api/controllers/chat.controller.ts#L61-L63) |
| Service (fetchMessages) | [channel.service.ts](src/api/services/channel.service.ts) | [600-687](src/api/services/channel.service.ts#L600-L687) |
| Service (fetchStatusMessage) | [channel.service.ts](src/api/services/channel.service.ts) | [689-707](src/api/services/channel.service.ts#L689-L707) |
| Validation schema (messages) | [chat.schema.ts](src/validate/chat.schema.ts) | [205-242](src/validate/chat.schema.ts#L205-L242) |
| Validation schema (status) | [chat.schema.ts](src/validate/chat.schema.ts) | [244-264](src/validate/chat.schema.ts#L244-L264) |
| Query class | [repository.service.ts](src/api/repository/repository.service.ts) | [5-10](src/api/repository/repository.service.ts#L5-L10) |
| Message model (MySQL) | [mysql-schema.prisma](prisma/mysql-schema.prisma) | [155-182](prisma/mysql-schema.prisma#L155-L182) |
| MessageUpdate model (MySQL) | [mysql-schema.prisma](prisma/mysql-schema.prisma) | [184-199](prisma/mysql-schema.prisma#L184-L199) |
| dataValidate (body mapping) | [abstract.router.ts](src/api/abstract/abstract.router.ts) | [29-56](src/api/abstract/abstract.router.ts#L29-L56) |
