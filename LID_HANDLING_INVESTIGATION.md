# LID (Linked ID) Handling in Evolution API

## Investigation Summary

This document explains how LID (Linked ID) identifiers work in WhatsApp/Baileys and how Evolution API handles them in webhook payloads.

> **Status**: ✅ **FIXED** in version `2.3.7-fix-0.0.2`
> 
> The `key.participant` LID→phone conversion has been implemented. Group messages now properly resolve LID to phone numbers using `participantAlt`.

---

## 1. What is LID?

**LID (Linked ID)** is WhatsApp's internal identifier format used in their newer protocol. Instead of using phone numbers directly (`5511999999999@s.whatsapp.net`), WhatsApp now uses opaque identifiers like `123456789012345@lid`.

### Key Characteristics:
- **Format**: `<numeric_id>@lid` (e.g., `237159621595181@lid`)
- **Purpose**: Privacy enhancement - phone numbers are not directly exposed in protocol messages
- **Mapping**: WhatsApp maintains internal LID ↔ Phone Number mappings
- **Rollout**: Gradually being adopted, so you may see mixed formats

---

## 2. Where Does LID Come From?

LID comes directly from **Baileys** (the WhatsApp Web library). When WhatsApp sends a message, Baileys receives the raw protocol data which may contain:

```typescript
{
  key: {
    remoteJid: "120363123456789@g.us",     // Group JID
    fromMe: false,
    id: "3EB0ABC123...",
    participant: "237159621595181@lid",     // Sender's LID (not phone!)
    participantAlt: "5511999999999@s.whatsapp.net"  // Alternative (phone JID)
  },
  pushName: "Contact Name",  // Or may be missing!
  message: { ... }
}
```

### The Problem

When `pushName` is missing from the raw message, Evolution API's `prepareMessage()` function falls back to using the participant JID:

```typescript
// From whatsapp.baileys.service.ts line 4688-4692
pushName:
  message.pushName ||
  (message.key.fromMe
    ? 'Você'
    : message?.participant || (message.key?.participant ? message.key.participant.split('@')[0] : null)),
```

This means if `pushName` is empty, you'll see the LID number (e.g., `237159621595181`) as the `pushName` value.

---

## 3. Current LID Handling in Evolution API

### 3.1 remoteJid Conversion ✅ (Working)

For **direct chats** (1:1 messages), Evolution API converts LID to phone number:

```typescript
// whatsapp.baileys.service.ts lines 1493-1494
if (messageRaw.key.remoteJid?.includes('@lid') && messageRaw.key.remoteJidAlt) {
  messageRaw.key.remoteJid = messageRaw.key.remoteJidAlt;
}
```

### 3.2 key.participant Conversion ✅ (FIXED in 2.3.7-fix-0.0.2)

For **group messages**, the `key.participant` field now converts LID to phone number using `participantAlt`:

```typescript
// whatsapp.baileys.service.ts lines 1496-1501
// Resolve LID to phone number for group message participants
if (messageRaw.key.participant?.includes('@lid')) {
  const key = received.key as any;
  if (key.participantAlt) {
    messageRaw.key.participant = key.participantAlt;
  }
}
```

**This fix follows the same pattern as the `remoteJid` conversion and uses the `participantAlt` field that Baileys provides alongside `participant`.**

### 3.3 GROUP_PARTICIPANTS_UPDATE ✅ (Has Enhancement)

The `GROUP_PARTICIPANTS_UPDATE` event DOES include LID resolution via the `participantsData` field:

```typescript
// whatsapp.baileys.service.ts lines 1785-1826
const groupParticipants = await this.findParticipants({ groupJid: participantsUpdate.id });

const resolvedParticipants = participantsUpdate.participants.map((participantId) => {
  const participantData = groupParticipants.participants.find((p) => p.id === participantId);
  
  let phoneNumber: string;
  if (participantData?.phoneNumber) {
    phoneNumber = participantData.phoneNumber;
  } else {
    phoneNumber = normalizePhoneNumber(participantId);
  }

  return {
    jid: participantId,
    phoneNumber,
    name: participantData?.name,
    imgUrl: participantData?.imgUrl,
  };
});
```

---

## 4. Webhook Payload Structure for Group Messages

### Current MESSAGES_UPSERT Payload (Group Message)

```json
{
  "event": "messages.upsert",
  "instance": "your-instance",
  "data": {
    "key": {
      "remoteJid": "120363123456789@g.us",
      "fromMe": false,
      "id": "3EB0ABC123DEF456",
      "participant": "237159621595181@lid"  // ⚠️ LID, not phone!
    },
    "pushName": "237159621595181",  // ⚠️ Falls back to LID when name missing
    "message": {
      "conversation": "Hello world"
    },
    "messageType": "conversation",
    "messageTimestamp": 1736870400
  }
}
```

### Expected/Desired Payload

```json
{
  "event": "messages.upsert",
  "instance": "your-instance",
  "data": {
    "key": {
      "remoteJid": "120363123456789@g.us",
      "fromMe": false,
      "id": "3EB0ABC123DEF456",
      "participant": "5511999999999@s.whatsapp.net"  // ✅ Phone number
    },
    "pushName": "John Doe",  // ✅ Real name
    "message": {
      "conversation": "Hello world"
    },
    "messageType": "conversation",
    "messageTimestamp": 1736870400
  }
}
```

---

## 5. The `/group/participants` Endpoint

### Endpoint: `GET /group/participants/{instanceName}`

This endpoint uses `findParticipants()` which calls `groupMetadata()`:

```typescript
// whatsapp.baileys.service.ts lines 4571-4595
public async findParticipants(id: GroupJid) {
  const participants = (await this.client.groupMetadata(id.groupJid)).participants;
  const contacts = await this.prismaRepository.contact.findMany({
    where: { instanceId: this.instanceId, remoteJid: { in: participants.map((p) => p.id) } },
  });
  
  const parsedParticipants = participants.map((participant) => {
    const contact = contacts.find((c) => c.remoteJid === participant.id);
    return {
      ...participant,
      name: participant.name ?? contact?.pushName,
      imgUrl: participant.imgUrl ?? contact?.profilePicUrl,
    };
  });

  return { participants: parsedParticipants };
}
```

### Response Example

```json
{
  "participants": [
    {
      "id": "237159621595181@lid",
      "admin": null,
      "name": "John Doe",
      "imgUrl": "https://..."
    }
  ]
}
```

**Note**: The `id` field may still contain LID. The `phoneNumber` field is only added in `GROUP_PARTICIPANTS_UPDATE` webhook enhancement.

---

## 6. Options to Resolve LIDs to Phone Numbers

### Option A: Use `onWhatsappCache` Lookup

Evolution API maintains a cache that maps LIDs to phone numbers in the `IsOnWhatsapp` table:

```typescript
// From onWhatsappCache.ts
// The cache stores: remoteJid, jidOptions (comma-separated alternatives), lid flag
```

You could query the database:
```sql
SELECT * FROM IsOnWhatsapp 
WHERE jidOptions LIKE '%237159621595181@lid%';
```

### Option B: Use Baileys' Internal Mapping

Baileys has an internal `signalRepository.lidMapping.getPNForLID()` method, but it's not currently exposed in Evolution API for webhook processing.

### Option C: Enhance MESSAGES_UPSERT Handler (Recommended)

Add LID→phone conversion for `key.participant` similar to how `GROUP_PARTICIPANTS_UPDATE` handles it:

```typescript
// Proposed enhancement in messages.upsert handler
if (messageRaw.key.participant?.includes('@lid')) {
  // Option 1: Use participantAlt if available
  const key = received.key as any;
  if (key.participantAlt) {
    messageRaw.key.participant = key.participantAlt;
  }
  
  // Option 2: Lookup from cache
  // const cached = await lookupFromOnWhatsappCache(messageRaw.key.participant);
  // if (cached?.phoneJid) messageRaw.key.participant = cached.phoneJid;
}
```

### Option D: Add `participantData` Field to MESSAGES_UPSERT

Similar to GROUP_PARTICIPANTS_UPDATE, add resolved participant info:

```typescript
// In webhook payload
{
  "key": {
    "participant": "237159621595181@lid"  // Keep original
  },
  "participantData": {
    "jid": "237159621595181@lid",
    "phoneNumber": "5511999999999",
    "name": "John Doe"
  }
}
```

---

## 7. Why `pushName` Shows LID Number

When Baileys receives a message without `pushName` (common for contacts not in your address book), the `prepareMessage()` function falls back:

```typescript
pushName:
  message.pushName ||  // 1. Try actual pushName
  (message.key.fromMe
    ? 'Você'           // 2. If sent by you
    : message?.participant ||  // 3. Try participant field
      (message.key?.participant 
        ? message.key.participant.split('@')[0]  // 4. Extract number from participant JID
        : null))
```

If `participant` is `237159621595181@lid`, then `split('@')[0]` returns `237159621595181`.

---

## 8. Cache Data Structure

The `IsOnWhatsapp` table stores LID mappings:

| Column | Description |
|--------|-------------|
| `remoteJid` | Primary JID (phone or LID) |
| `jidOptions` | Comma-separated list of all known JIDs for this contact |
| `lid` | Flag indicating if this is an LID-based entry |

Example:
```
remoteJid: "5511999999999@s.whatsapp.net"
jidOptions: "5511999999999@s.whatsapp.net,237159621595181@lid"
lid: null
```

---

## 9. Recommendations

### Short-term Workarounds

1. **Parse `key.participant`**: If it ends with `@lid`, query `IsOnWhatsapp` table for the phone number
2. **Ignore `pushName` when it looks like LID**: If `pushName` matches pattern `^\d+$` (only digits), treat as unknown

### Long-term Solutions

1. **Enhance MESSAGES_UPSERT handler**: Add `participantAlt` resolution for group messages
2. **Add `senderData` field**: Include resolved sender info in webhook payload
3. **Expose LID lookup API**: Add endpoint to resolve LID → phone number

---

## 10. Code References

| File | Lines | Description |
|------|-------|-------------|
| [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1493-L1494) | 1493-1494 | remoteJid LID→phone conversion |
| [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L1785-L1830) | 1785-1830 | GROUP_PARTICIPANTS_UPDATE enhancement |
| [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4682-L4733) | 4682-4733 | prepareMessage() function |
| [whatsapp.baileys.service.ts](src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts#L4571-L4595) | 4571-4595 | findParticipants() method |
| [onWhatsappCache.ts](src/utils/onWhatsappCache.ts) | 1-150 | LID caching logic |

---

## 11. Summary

| Scenario | LID Handled? | Notes |
|----------|--------------|-------|
| Direct chat `remoteJid` | ✅ Yes | Converted via `remoteJidAlt` |
| Group `key.participant` | ✅ Yes | **FIXED** - Converted via `participantAlt` (v2.3.7-fix-0.0.2) |
| `pushName` fallback | ❌ No | Shows LID number when name missing |
| GROUP_PARTICIPANTS_UPDATE | ✅ Yes | Has `participantsData` with phone numbers |
| `/group/participants` API | ⚠️ Partial | Returns LID in `id`, name from contacts |

~~The main gap is **group message sender identification** - the `key.participant` field contains raw LID without conversion.~~

**UPDATE**: The group message sender identification gap has been resolved in version `2.3.7-fix-0.0.2`.
