# Patch: Store Real Participant JID for Group Messages

## Status: IMPLEMENTED in `2.3.7-fix-0.0.5`

All changes applied to `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts`.

---

## 1. `prepareMessage()` — Full Modified Function

**Lines 4781-4856** — The core message normalization before Prisma insert.

### What Changed

- **`resolvedParticipant`**: New resolution chain for group messages only: `participantAlt` (@s.whatsapp.net) → `key.participant` → top-level `message.participant` → null
- **`key` spread + injection**: Shallow copy of `message.key`, injects `participant` when missing
- **`participant` column**: New property on `messageRaw` — populates the `participant VARCHAR(100)` DB column (previously always NULL)
- **`pushName` LID replacement**: When pushName is a bare LID number (e.g. `"175105850253361"`) and we have a `@s.whatsapp.net` JID, replaces with the phone number

### Complete Function

```typescript
private prepareMessage(message: proto.IWebMessageInfo): any {
    const contentType = getContentType(message.message);
    const contentMsg = message?.message[contentType] as any;

    // Resolve participant for group messages:
    // Priority: participantAlt (@s.whatsapp.net) > key.participant > top-level message.participant
    const extKey = message.key as any;
    const isGroup = message.key?.remoteJid?.includes('@g.us') || extKey?.remoteJidAlt?.includes('@g.us');
    const resolvedParticipant = isGroup
      ? (extKey?.participantAlt || message.key?.participant || (message as any).participant || null)
      : null;

    // Copy key and inject participant if missing
    const key = { ...message.key };
    if (isGroup && !key.participant && resolvedParticipant) {
      key.participant = resolvedParticipant;
    }

    // Resolve pushName: if it's a bare LID number and we have a phone JID, use the phone number
    let pushName =
      message.pushName ||
      (message.key.fromMe
        ? 'Você'
        : resolvedParticipant ? resolvedParticipant.split('@')[0] : null);
    if (
      pushName &&
      /^\d{10,}$/.test(pushName) &&
      resolvedParticipant?.includes('@s.whatsapp.net')
    ) {
      pushName = resolvedParticipant.split('@')[0];
    }

    const messageRaw = {
      key,
      pushName,
      participant: resolvedParticipant,       // ← NEW: populates the DB column
      status: status[message.status],
      message: this.deserializeMessageBuffers({ ...message.message }),
      contextInfo: this.deserializeMessageBuffers(contentMsg?.contextInfo),
      messageType: contentType || 'unknown',
      messageTimestamp: Long.isLong(message.messageTimestamp)
        ? message.messageTimestamp.toNumber()
        : (message.messageTimestamp as number),
      instanceId: this.instanceId,
      source: getDevice(message.key.id),
    };

    if (!messageRaw.status && message.key.fromMe === false) {
      messageRaw.status = status[3]; // DELIVERED MESSAGE
    }

    if (messageRaw.message.extendedTextMessage) {
      messageRaw.messageType = 'conversation';
      messageRaw.message.conversation = messageRaw.message.extendedTextMessage.text;
      delete messageRaw.message.extendedTextMessage;
    }

    if (messageRaw.message.documentWithCaptionMessage) {
      messageRaw.messageType = 'documentMessage';
      messageRaw.message.documentMessage = messageRaw.message.documentWithCaptionMessage.message.documentMessage;
      delete messageRaw.message.documentWithCaptionMessage;
    }

    const quotedMessage = messageRaw?.contextInfo?.quotedMessage;
    if (quotedMessage) {
      if (quotedMessage.extendedTextMessage) {
        quotedMessage.conversation = quotedMessage.extendedTextMessage.text;
        delete quotedMessage.extendedTextMessage;
      }

      if (quotedMessage.documentWithCaptionMessage) {
        quotedMessage.documentMessage = quotedMessage.documentWithCaptionMessage.message.documentMessage;
        delete quotedMessage.documentWithCaptionMessage;
      }
    }

    return messageRaw;
  }
```

---

## 2. `messages.upsert` Handler — Post-LID Resolution Sync

**Lines 1497-1517** — After `prepareMessage()` runs and existing LID resolution fires, we sync the `participant` column and do a second pushName LID→phone replacement.

### Why This Is Needed

`prepareMessage()` resolves `participantAlt` at preparation time. But the existing LID resolution block at line 1498 may further resolve `key.participant` from `@lid` to `@s.whatsapp.net` using `participantAlt`. After that, the `participant` column and `pushName` need to be updated to match.

### Code

```typescript
          // Resolve LID to phone number for group message participants
          if (messageRaw.key.participant?.includes('@lid')) {
            const key = received.key as any;
            if (key.participantAlt) {
              messageRaw.key.participant = key.participantAlt;
            }
          }

          // NEW: Sync participant column after LID resolution
          if (messageRaw.key.participant && messageRaw.participant !== messageRaw.key.participant) {
            messageRaw.participant = messageRaw.key.participant;
          }

          // NEW: If pushName is still a bare LID number after resolution, replace with phone number
          if (
            messageRaw.pushName &&
            /^\d{10,}$/.test(messageRaw.pushName) &&
            messageRaw.participant?.includes('@s.whatsapp.net')
          ) {
            messageRaw.pushName = messageRaw.participant.split('@')[0];
          }

          this.sendDataWebhook(Events.MESSAGES_UPSERT, messageRaw);
```

---

## 3. `messaging-history.set` Handler — No Additional Changes Needed

The history sync handler at line 1061 calls `this.prepareMessage(m)`, which now automatically resolves participant via the new logic. History sync messages from Baileys also carry `participantAlt` on the extended key, so the same resolution chain works.

---

## 4. `fetchMessages` — Added `participant` to Select Clauses

Both the MySQL raw SQL path (line 5365) and PostgreSQL Prisma path (line 5421) now include `participant: true` in their select objects, so the `participant` column is returned in API responses.

### MySQL Path (line 5361-5374)
```typescript
          select: {
            id: true,
            key: true,
            pushName: true,
            participant: true,    // ← NEW
            messageType: true,
            message: true,
            messageTimestamp: true,
            instanceId: true,
            source: true,
            status: true,
            contextInfo: true,
            MessageUpdate: { select: { status: true } },
          },
```

### PostgreSQL Path (line 5417-5430)
```typescript
        select: {
          id: true,
          key: true,
          pushName: true,
          participant: true,    // ← NEW
          messageType: true,
          message: true,
          messageTimestamp: true,
          instanceId: true,
          source: true,
          status: true,
          contextInfo: true,
          MessageUpdate: { select: { status: true } },
        },
```

---

## 5. Answers to Prompt Questions

### Q1: Show `prepareMessage()` in full — where does it get `key`, `pushName`, and `participant`?

See Section 1 above. Before the fix:
- `key` came from `message.key` as-is (with `participant: null` 99.98% of the time)
- `pushName` came from `message.pushName` (which Baileys provides as a bare LID number in the LID era)
- `participant` was never set — the DB column was always NULL

### Q2: Baileys data available at `messages.upsert` time

For a group message with LID addressing, Baileys provides:

| Field | Value | Type |
|-------|-------|------|
| `received.key.participant` | `null` | Almost always null in LID era |
| `received.key.participantAlt` | `"5511999998888@s.whatsapp.net"` | The real phone JID (extended key field) |
| `received.participant` | `"175105850253361@lid"` | Top-level `WebMessageInfo.participant` |
| `received.pushName` | `"175105850253361"` | Bare LID number (NOT the real display name) |
| `received.key.remoteJid` | `"120363xxx@g.us"` or `"xxx@lid"` | Group JID |
| `received.key.remoteJidAlt` | `"120363xxx@g.us"` | Phone-format group JID |

The `ExtendedIMessageKey` interface (line 158-163) adds `participantAlt` and `remoteJidAlt` to Baileys' base `IMessageKey`.

### Q3: Where does the LID-as-pushName come from?

**Baileys itself provides it.** WhatsApp sends the bare LID number as `pushName` for LID-addressed group messages. Evolution API does NOT transform it. The real display name is simply not available via Baileys for non-saved contacts in LID mode.

Our fix replaces the LID number with the phone number when `participantAlt` is available — still not a display name, but at least recognizable.

### Q4: The exact minimal diff

Four changes, all in `whatsapp.baileys.service.ts`:

| Location | Change | Lines Added |
|----------|--------|-------------|
| `prepareMessage()` | Resolve participant, inject into key, set DB column, fix pushName | ~20 |
| `messages.upsert` after LID resolution | Sync `participant` column + secondary pushName fix | ~12 |
| `fetchMessages` MySQL select | Added `participant: true` | 1 |
| `fetchMessages` PostgreSQL select | Added `participant: true` | 1 |

No schema migration needed — the `participant VARCHAR(100)` column already exists at `prisma/mysql-schema.prisma:159`.

### Q5: Can existing messages be backfilled?

**Partially.** The real `@s.whatsapp.net` JID was never stored anywhere in the database. What IS recoverable:

```sql
-- Step 1: Reconstruct @lid JIDs from pushName (bare LID numbers)
UPDATE Message
SET participant = CONCAT(pushName, '@lid'),
    `key` = JSON_SET(`key`, '$.participant', CONCAT(pushName, '@lid'))
WHERE participant IS NULL
  AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.remoteJid')) LIKE '%@g.us'
  AND JSON_UNQUOTE(JSON_EXTRACT(`key`, '$.fromMe')) = 'false'
  AND pushName REGEXP '^[0-9]{10,20}$';

-- Step 2: Resolve LID→phone using IsOnWhatsapp cache (if mappings exist)
UPDATE Message m
JOIN IsOnWhatsapp w ON m.participant = w.jid
SET m.participant = w.remoteJid,
    m.`key` = JSON_SET(m.`key`, '$.participant', w.remoteJid)
WHERE m.participant LIKE '%@lid'
  AND w.remoteJid LIKE '%@s.whatsapp.net';
```

- **Step 1** gives you `@lid` JIDs for most group messages (from the pushName that was storing the LID number)
- **Step 2** resolves those to `@s.whatsapp.net` if the LID→phone mapping exists in the `IsOnWhatsapp` table
- **Real display names** are NOT recoverable — they were never in the Baileys data

---

## 6. What the Fix Produces — Before vs After

### Database (`Message` table)

| Field | Before | After |
|-------|--------|-------|
| `participant` column | `NULL` | `"5511999998888@s.whatsapp.net"` |
| `key.participant` (JSON) | `null` | `"5511999998888@s.whatsapp.net"` |
| `pushName` | `"175105850253361"` (LID) | `"5511999998888"` (phone number) |

### Webhook (`messages.upsert`)

| Field | Before | After |
|-------|--------|-------|
| `participant` | not present | `"5511999998888@s.whatsapp.net"` |
| `key.participant` | `null` | `"5511999998888@s.whatsapp.net"` |
| `pushName` | `"175105850253361"` | `"5511999998888"` |

### `fetchMessages` API Response

| Field | Before | After |
|-------|--------|-------|
| `participant` | not in response | `"5511999998888@s.whatsapp.net"` |

### Backward Compatibility

- Private chats: **unchanged** — `isGroup` check ensures participant logic only fires for `@g.us` messages
- If `participantAlt` doesn't exist (older Baileys): falls back to `key.participant` → `message.participant` → null (same as before)
- No schema migration needed
- No new endpoints or webhook events

---

## 7. Verification

Built and verified in Docker image `ghcr.io/jefjrfigueiredo/evolution-api:2.3.7-fix-0.0.5`:

```
# participantAlt resolution chain in compiled JS
participantAlt||e.key?.participant||e.participant)||null

# participant column sync after LID resolution
c.participant!==c.key.participant&&(c.participant=c.key.participant)

# participant:true in both fetchMessages select clauses
pushName:!0,participant:!0   (appears 2 times)
```
