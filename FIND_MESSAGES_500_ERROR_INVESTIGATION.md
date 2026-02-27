# Evolution API: `findMessages` 500 Error with JSON Path Filters on MySQL

> **STATUS: FIXED** in Docker image `ghcr.io/jefjrfigueiredo/evolution-api:2.3.7-fix-0.0.4`
>
> The `fetchMessages` method in [channel.service.ts](src/api/services/channel.service.ts) has been updated with a MySQL-specific raw SQL branch that uses `JSON_UNQUOTE(JSON_EXTRACT(...))`, following the same pattern already used elsewhere in the codebase. All `key` sub-filters (`remoteJid`, `id`, `fromMe`, `participants`) now work correctly on MySQL.
>
> **You can now use:**
> ```json
> POST /chat/findMessages/{instanceName}
> {
>   "where": {
>     "key": {
>       "remoteJid": "120363XXXXXXXXX@g.us"
>     }
>   },
>   "offset": 50,
>   "page": 1
> }
> ```
>
> The workarounds in Section 4 are no longer necessary unless you're running a version prior to this fix.

---

## Root Cause: Prisma JSON Path Filtering Doesn't Work on MySQL

The `fetchMessages` method uses Prisma's built-in JSON path filtering syntax:

```typescript
{ key: { path: ['remoteJid'], equals: keyFilters?.remoteJid } }
```

This generates SQL that works on **PostgreSQL** but **fails on MySQL**. The error occurs at the `count()` call (line 618) because it runs before `findMany()` — but both queries would fail.

**This is a known issue in the codebase.** The rest of the code already works around it by using raw SQL with `JSON_EXTRACT()` for MySQL.

---

## 1. The Failing Code

**File:** [channel.service.ts:618-632](src/api/services/channel.service.ts#L618-L632)

```typescript
// This count() call fails on MySQL when keyFilters have values
const count = await this.prismaRepository.message.count({
  where: {
    instanceId: this.instanceId,
    id: query?.where?.id,
    source: query?.where?.source,
    messageType: query?.where?.messageType,
    ...timestampFilter,
    AND: [
      keyFilters?.id        ? { key: { path: ['id'], equals: keyFilters?.id } }        : {},  // ← Fails on MySQL
      keyFilters?.fromMe    ? { key: { path: ['fromMe'], equals: keyFilters?.fromMe } }    : {},
      keyFilters?.remoteJid ? { key: { path: ['remoteJid'], equals: keyFilters?.remoteJid } } : {},
      keyFilters?.participants ? { key: { path: ['participants'], equals: keyFilters?.participants } } : {},
    ],
  },
});
```

### What Prisma Generates (PostgreSQL vs MySQL)

**PostgreSQL** (works):
```sql
SELECT COUNT(*) FROM "Message"
WHERE "instanceId" = $1
AND "key"->>'remoteJid' = $2
```

**MySQL** (fails):
Prisma attempts to use JSON path syntax that either generates invalid SQL for MySQL's `count()` or uses an unsupported operator combination. MySQL requires explicit `JSON_EXTRACT()` / `JSON_UNQUOTE()` calls.

---

## 2. Evidence: The Codebase Already Knows About This

There are **four other places** in the same file that use raw SQL specifically to work around this exact MySQL JSON path issue:

### Example 1: `getMessage` ([line 529-537](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L529-L537))
```typescript
if (dbProvider === 'mysql') {
  // MySQL syntax with JSON_UNQUOTE and JSON_EXTRACT
  webMessageInfo = await this.prismaRepository.$queryRawUnsafe<any[]>(
    `SELECT * FROM Message
     WHERE instanceId = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.id')) = ?`,
    this.instanceId,
    key.id,
  );
}
```

### Example 2: `messages.update` handler ([line 1664-1672](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1664-L1672))
```typescript
if (configDatabaseData.PROVIDER === 'mysql') {
  // MySQL syntax with JSON_UNQUOTE and JSON_EXTRACT
  messages = await this.prismaRepository.$queryRawUnsafe<any[]>(
    `SELECT * FROM Message
     WHERE instanceId = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.id')) = ?
     LIMIT 1`,
    this.instanceId,
    searchId,
  );
}
```

### Example 3: `updateMessagesReadedByTimestamp` ([line 4804-4816](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4804-L4816))
```typescript
if (dbProvider === 'mysql') {
  result = await this.prismaRepository.$executeRawUnsafe(
    `UPDATE Message
     SET status = ?
     WHERE instanceId = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.remoteJid')) = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.fromMe')) = 'false'
     AND messageTimestamp <= ?`,
    ...
  );
}
```

### Example 4: Unread messages count ([line 4855-4865](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4855-L4865))
```typescript
// MySQL syntax
unreadMessagesPromise = this.prismaRepository
  .$queryRawUnsafe<{ count: number }[]>(
    `SELECT COUNT(*) as count FROM Message
     WHERE instanceId = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.remoteJid')) = ?
     AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.fromMe')) = 'false'
     AND status = ?`,
    ...
  )
```

**Pattern is clear:** Every other MySQL JSON query in the codebase uses raw SQL with `JSON_UNQUOTE(JSON_EXTRACT(...))`. The `fetchMessages` method was simply never updated for MySQL compatibility.

---

## 3. Why It Fails Only With Filters (and Works Without)

When you send an empty `where` or dot-notation keys:

```json
{ "where": { "key.remoteJid": "..." } }
```

- `keyFilters` is `undefined`
- All `AND` conditions become `{}` (empty objects)
- `AND: [{}, {}, {}, {}]` is equivalent to no filter
- Prisma generates a simple `SELECT COUNT(*) FROM Message WHERE instanceId = ?`
- No JSON path filtering → no MySQL incompatibility → works fine

When you send the correct nested `key` format:

```json
{ "where": { "key": { "remoteJid": "..." } } }
```

- `keyFilters.remoteJid` has a value
- The `AND` clause includes `{ key: { path: ['remoteJid'], equals: '...' } }`
- Prisma tries to generate MySQL-compatible JSON path SQL
- `count()` fails with the 500 error

---

## 4. Available Workarounds

### Workaround A: Query Without Key Filters and Filter Client-Side

Until this is fixed, you can skip the `key` filter and filter in your Laravel application:

```json
POST /chat/findMessages/{instanceName}

{
  "where": {},
  "offset": 200,
  "page": 1
}
```

Then in Laravel:

```php
$records = collect($response['messages']['records'])
    ->filter(function ($msg) use ($groupJid) {
        return ($msg['key']['remoteJid'] ?? '') === $groupJid
            && ($msg['key']['fromMe'] ?? false) === true;
    });
```

**Cons:** Fetches all messages, wasteful for instances with many chats. Pagination is unreliable (page counts won't match filtered results).

### Workaround B: Use `findStatusMessage` Instead

For read receipt tracking, the `findStatusMessage` endpoint queries the `MessageUpdate` table which has `remoteJid` as a **proper column** (not JSON), so it works fine on MySQL:

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

Returns flat `MessageUpdate` records with `status`, `keyId`, `remoteJid`, `fromMe` — all as real columns. No JSON path filtering needed.

**Cons:** Doesn't include the message content/type, only status updates. You'd need to correlate `keyId` with messages separately.

### Workaround C: Use `messageType` Filter Only

Non-JSON filters like `messageType` and `source` work fine because they're real columns:

```json
POST /chat/findMessages/{instanceName}

{
  "where": {
    "messageType": "conversation"
  },
  "offset": 50,
  "page": 1
}
```

**Cons:** Can't filter by group or fromMe.

### Workaround D: Use Time Range Filter

The `messageTimestamp` filter also uses real columns:

```json
POST /chat/findMessages/{instanceName}

{
  "where": {
    "messageTimestamp": {
      "gte": "2026-02-06T00:00:00.000Z",
      "lte": "2026-02-06T23:59:59.000Z"
    }
  },
  "offset": 100,
  "page": 1
}
```

**Cons:** Same as A — can't filter by group or fromMe server-side.

---

## 5. The Fix (Implemented)

The `fetchMessages` method has been updated in [channel.service.ts](src/api/services/channel.service.ts) to use raw SQL for MySQL, following the exact pattern already used elsewhere in the codebase.

### How It Works

When **MySQL** is the database provider **AND** key filters are present, the method:

1. **Builds a dynamic WHERE clause** using `JSON_UNQUOTE(JSON_EXTRACT(...))` for all key sub-fields
2. **Runs a raw SQL `COUNT(*)`** for pagination totals
3. **Runs a raw SQL `SELECT id`** with `LIMIT`/`OFFSET` for paginated message IDs
4. **Uses Prisma `findMany({ where: { id: { in: messageIds } } })`** to fetch full records with `MessageUpdate` relations

This two-step approach (raw SQL for filtering + Prisma for relations) ensures MySQL compatibility while preserving the `MessageUpdate` join that Prisma handles well.

When **PostgreSQL** is the provider (or no key filters are present), the original Prisma JSON path filtering is used unchanged.

### Supported Key Filters on MySQL

| Filter | SQL Generated |
|--------|--------------|
| `key.remoteJid` | `JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.remoteJid')) = ?` |
| `key.id` | `JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.id')) = ?` |
| `key.fromMe` | `JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.fromMe')) = 'true'` |
| `key.participants` | `JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.participant')) = ?` |

Non-JSON filters (`source`, `messageType`, `messageTimestamp`, `id`) are also supported in the MySQL branch and use standard parameterized SQL.

### Example: Filter Group Messages

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

**Response** (same format as before):
```json
{
  "messages": {
    "total": 142,
    "pages": 3,
    "currentPage": 1,
    "records": [
      {
        "id": "...",
        "key": { "remoteJid": "120363...@g.us", "fromMe": true, "id": "3EB0..." },
        "pushName": "...",
        "messageType": "conversation",
        "message": { ... },
        "messageTimestamp": 1738800000,
        "source": "android",
        "contextInfo": null,
        "MessageUpdate": [
          { "status": "DELIVERY_ACK" }
        ]
      }
    ]
  }
}
```

### MySQL Raw SQL Equivalent

The Prisma query:
```typescript
AND: [
  { key: { path: ['remoteJid'], equals: '120363...@g.us' } },
  { key: { path: ['fromMe'], equals: true } },
]
```

Translates to MySQL:
```sql
AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.remoteJid')) = '120363...@g.us'
AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.fromMe')) = 'true'
```

**Note:** In MySQL JSON, booleans are stored as `true`/`false` but `JSON_UNQUOTE(JSON_EXTRACT(...))` returns the string `"true"` or `"false"`, so comparison must be against the string representation.

---

## 6. Alternative Approach: `findStatusMessage` for Read Receipts

If you specifically need group read receipt tracking (rather than full message queries), this approach is still useful as a lighter alternative:

### Strategy: Use `findStatusMessage` for Status-Only Queries

**Step 1:** Use `findStatusMessage` to get status updates for a specific group:

```json
POST /chat/findStatusMessage/{instanceName}

{
  "where": {
    "remoteJid": "120363426504089458@g.us"
  },
  "offset": 100,
  "page": 1
}
```

This works because `MessageUpdate.remoteJid` is a real column, not JSON.

**Step 2:** In your Laravel app, group by `keyId` and check statuses:

```php
$statusUpdates = $this->evolutionApi->findStatusMessage($instance, [
    'where' => [
        'remoteJid' => $groupJid,
    ],
    'offset' => 100,
    'page' => 1,
]);

// Group by message key ID
$messageStatuses = collect($statusUpdates)
    ->groupBy('keyId')
    ->map(function ($updates, $keyId) {
        $statuses = $updates->pluck('status')->unique()->values();
        return [
            'keyId' => $keyId,
            'fromMe' => $updates->first()['fromMe'],
            'isRead' => $statuses->contains('READ'),
            'isDelivered' => $statuses->contains('DELIVERY_ACK'),
            'latestStatus' => $statuses->last(),
        ];
    })
    ->filter(fn ($msg) => $msg['fromMe'] === true); // Only outgoing
```

### Why This Can Still Be Useful (Even After the Fix)

- **Lighter queries** — `MessageUpdate` table is much smaller than `Message` (only status rows, no message content)
- **Real indexed columns** — `remoteJid`, `keyId`, `fromMe` are proper columns, faster than JSON extraction
- **Focused data** — When you only need delivery/read status, not full message content

---

## 7. Summary

| Aspect | Detail |
|--------|--------|
| **Root cause** | Prisma `count()` (and `findMany()`) with JSON `path` filter generates invalid SQL for MySQL |
| **Scope** | Only `fetchMessages` was affected — other MySQL JSON queries in the codebase already used raw SQL |
| **PostgreSQL** | Works fine — Prisma JSON path filtering is a PostgreSQL-first feature |
| **Why no filter worked** | Empty key filters produce `AND: [{}, {}, {}, {}]` which matches everything — no JSON path SQL generated |
| **Status** | **FIXED** in `2.3.7-fix-0.0.4` — MySQL branch uses raw SQL with `JSON_EXTRACT()`, matching existing codebase pattern |
| **Fixed file** | [channel.service.ts](src/api/services/channel.service.ts) — `fetchMessages` method |

---

## 8. Code References

| Feature | File | Lines |
|---------|------|-------|
| Failing count() call | [channel.service.ts](src/api/services/channel.service.ts) | [618-632](src/api/services/channel.service.ts#L618-L632) |
| Failing findMany() call | [channel.service.ts](src/api/services/channel.service.ts) | [642-677](src/api/services/channel.service.ts#L642-L677) |
| fetchStatusMessage (works) | [channel.service.ts](src/api/services/channel.service.ts) | [689-707](src/api/services/channel.service.ts#L689-L707) |
| MySQL raw SQL pattern 1 | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [529-537](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L529-L537) |
| MySQL raw SQL pattern 2 | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [1664-1672](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1664-L1672) |
| MySQL raw SQL pattern 3 | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [4804-4816](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4804-L4816) |
| MySQL raw SQL pattern 4 | [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts) | [4855-4865](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4855-L4865) |
| MessageUpdate schema | [mysql-schema.prisma](prisma/mysql-schema.prisma) | [184-199](prisma/mysql-schema.prisma#L184-L199) |
| Prisma version | [package.json](package.json) | `@prisma/client: ^6.16.2` |
