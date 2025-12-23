# MySQL Webhook Fix for Evolution API v2.3.7

## Summary

Fixed critical bugs where `MESSAGES_UPSERT` and `MESSAGES_UPDATE` webhooks were not being triggered when using MySQL as the database provider. The issues were caused by PostgreSQL-specific SQL syntax in multiple methods that query JSON fields.

---

## Problem Description

### Symptoms
- `messages.upsert` webhooks were not being received by external applications
- Other webhooks (e.g., `connection.update`) worked correctly
- Messages were being saved to the database successfully
- No error logs were generated (silent failure)

### Affected Versions
- Evolution API v2.3.7
- Database: MySQL (PostgreSQL was unaffected)

---

## Root Cause

The methods `updateMessagesReadedByTimestamp` and `updateChatUnreadMessages` in `whatsapp.baileys.service.ts` used **PostgreSQL-specific SQL syntax** for JSON field access:

```sql
-- PostgreSQL syntax (doesn't work in MySQL)
"key"->>'remoteJid'
("key"->>'fromMe')::boolean
COUNT(*)::int
```

When using MySQL, these queries failed silently, causing the message handler to stop execution **before** reaching the `sendDataWebhook()` call.

### Execution Flow (Before Fix)
```
1. Message received from Baileys ✓
2. Message saved to database ✓
3. updateMessagesReadedByTimestamp() called ✗ (FAILS HERE - silent exception)
4. sendDataWebhook(MESSAGES_UPSERT) ✗ (NEVER REACHED)
```

---

## Solution

Modified both methods to detect the database provider via `DATABASE.PROVIDER` configuration and use the appropriate SQL syntax:

### MySQL Syntax
```sql
JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.remoteJid'))
JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.fromMe')) = 'false'
```

### PostgreSQL Syntax (unchanged)
```sql
"key"->>'remoteJid'
("key"->>'fromMe')::boolean = false
```

---

## Files Modified

| File | Changes |
|------|---------|
| `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts` | Added MySQL support to `updateMessagesReadedByTimestamp()`, `updateChatUnreadMessages()`, `getMessage()`, and `messages.update` handler |

---

## Code Changes

### Fix 1: MESSAGES_UPSERT Webhook (December 22, 2025)

#### updateMessagesReadedByTimestamp (Line ~4734)

```typescript
private async updateMessagesReadedByTimestamp(remoteJid: string, timestamp?: number): Promise<number> {
  if (timestamp === undefined || timestamp === null) return 0;

  const dbProvider = this.configService.get<Database>('DATABASE').PROVIDER;

  let result: number;

  if (dbProvider === 'mysql') {
    // MySQL syntax with JSON_UNQUOTE and JSON_EXTRACT
    result = await this.prismaRepository.$executeRawUnsafe(
      `UPDATE Message
       SET status = ?
       WHERE instanceId = ?
       AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.remoteJid')) = ?
       AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.fromMe')) = 'false'
       AND messageTimestamp <= ?
       AND (status IS NULL OR status = ?)`,
      status[4],
      this.instanceId,
      remoteJid,
      timestamp,
      status[3],
    );
  } else {
    // PostgreSQL syntax
    result = await this.prismaRepository.$executeRaw`
      UPDATE "Message"
      SET "status" = ${status[4]}
      WHERE "instanceId" = ${this.instanceId}
      AND "key"->>'remoteJid' = ${remoteJid}
      AND ("key"->>'fromMe')::boolean = false
      AND "messageTimestamp" <= ${timestamp}
      AND ("status" IS NULL OR "status" = ${status[3]})
    `;
  }
  // ... rest of method
}
```

### updateChatUnreadMessages (Line ~4770)

```typescript
private async updateChatUnreadMessages(remoteJid: string): Promise<number> {
  const dbProvider = this.configService.get<Database>('DATABASE').PROVIDER;

  let unreadMessagesPromise: Promise<number>;

  if (dbProvider === 'mysql') {
    // MySQL syntax
    unreadMessagesPromise = this.prismaRepository
      .$queryRawUnsafe<{ count: number }[]>(
        `SELECT COUNT(*) as count FROM Message
         WHERE instanceId = ?
         AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.remoteJid')) = ?
         AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.fromMe')) = 'false'
         AND status = ?`,
        this.instanceId,
        remoteJid,
        status[3],
      )
      .then((result) => Number(result[0]?.count) || 0);
  } else {
    // PostgreSQL syntax
    unreadMessagesPromise = this.prismaRepository.$queryRaw<{ count: number }[]>`
      SELECT COUNT(*)::int as count FROM "Message"
      WHERE "instanceId" = ${this.instanceId}
      AND "key"->>'remoteJid' = ${remoteJid}
      AND ("key"->>'fromMe')::boolean = false
      AND "status" = ${status[3]}
    `.then((result) => result[0]?.count || 0);
  }
  // ... rest of method
}
```

---

### Fix 2: MESSAGES_UPDATE Webhook (December 23, 2025)

#### getMessage Method (Line ~526)

```typescript
private async getMessage(key: proto.IMessageKey, full = false) {
  try {
    const dbProvider = this.configService.get<Database>('DATABASE').PROVIDER;

    let webMessageInfo: proto.IWebMessageInfo[];

    if (dbProvider === 'mysql') {
      // MySQL syntax with JSON_UNQUOTE and JSON_EXTRACT
      webMessageInfo = await this.prismaRepository.$queryRawUnsafe<proto.IWebMessageInfo[]>(
        `SELECT * FROM Message
         WHERE instanceId = ?
         AND JSON_UNQUOTE(JSON_EXTRACT(\`key\`, '$.id')) = ?`,
        this.instanceId,
        key.id,
      );
    } else {
      // PostgreSQL syntax
      webMessageInfo = await this.prismaRepository.$queryRaw<proto.IWebMessageInfo[]>`
        SELECT * FROM "Message"
        WHERE "instanceId" = ${this.instanceId}
        AND "key"->>'id' = ${key.id}
      `;
    }
    // ... rest of method
  } catch (error) {
    // ...
  }
}
```

#### messages.update Handler Query (Line ~1640)

```typescript
const configDatabaseData = this.configService.get<Database>('DATABASE');

let messages: any[];

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
} else {
  // PostgreSQL syntax
  messages = await this.prismaRepository.$queryRaw<any[]>`
    SELECT * FROM "Message"
    WHERE "instanceId" = ${this.instanceId}
    AND "key"->>'id' = ${searchId}
    LIMIT 1
  `;
}
findMessage = messages[0] || null;
```

---

## Testing

After applying the fix:

1. Rebuild the Docker image
2. Restart the Evolution API container
3. Send a WhatsApp message
4. Verify webhook is received:
   - Check logs for `sendDataWebhook` calls
   - Confirm external application receives `messages.upsert` event

---

## Additional Notes

- This fix is backward compatible with PostgreSQL installations
- The database provider is automatically detected from the `DATABASE_PROVIDER` environment variable
- No configuration changes are required after applying the fix

---

## Related Issues

- Similar PostgreSQL-specific syntax exists in `addLabel()` and `removeLabel()` methods, which may need similar fixes if label functionality is used with MySQL.

---

## Fix History

| Date | Issue | Methods Fixed |
|------|-------|---------------|
| December 22, 2025 | `MESSAGES_UPSERT` webhook not firing | `updateMessagesReadedByTimestamp()`, `updateChatUnreadMessages()` |
| December 23, 2025 | `MESSAGES_UPDATE` webhook not firing | `getMessage()`, `messages.update` handler query |

---

*Last updated: December 23, 2025*
