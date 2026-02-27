# Review: Webhook-Only Per-Participant Receipts

> **STATUS: IMPLEMENTED** in Docker image `ghcr.io/jefjrfigueiredo/evolution-api:2.3.7-fix-0.0.5`

---

## What Was Implemented

Three files modified, following the exact Evolution API event pattern:

| File | Change |
|------|--------|
| `src/api/types/wa.types.ts:37` | Added `MESSAGE_RECEIPT = 'message.receipt'` to Events enum |
| `src/api/integrations/event/event.controller.ts:165` | Added `'MESSAGE_RECEIPT'` to per-instance events whitelist |
| `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts:1998-2030` | Added webhook emission loop in `message-receipt.update` handler |

---

## How It Works

1. Baileys fires `message-receipt.update` when a group member reads or receives a message
2. The **existing aggregate logic is preserved** — `remotesJidMap` + `updateMessagesReadedByTimestamp` still updates `Message.status` as before
3. **After** the aggregate update, a new loop iterates over each receipt event and emits a `message.receipt` webhook per participant
4. **Self-receipts are filtered out** — receipts where `receipt.userJid` matches the instance's own JID (`this.instance.wuid`) are skipped

## Webhook Payload

```json
{
  "event": "message.receipt",
  "instance": "ws_1_phone_2",
  "data": {
    "keyId": "AC0F6EB728EC0B558C34164B3ECCA455",
    "remoteJid": "120363422540832006@g.us",
    "fromMe": true,
    "participantJid": "5511999991111@s.whatsapp.net",
    "type": "read",
    "timestamp": 1738800120,
    "readTimestamp": 1738800120,
    "deliveredTimestamp": null
  }
}
```

| Field | Description |
|-------|-------------|
| `keyId` | WhatsApp message ID |
| `remoteJid` | Group JID |
| `fromMe` | Whether the message was sent by this instance |
| `participantJid` | Who read/received the message |
| `type` | `"read"` or `"delivered"` |
| `timestamp` | The relevant timestamp (read or delivered) |
| `readTimestamp` | When participant read it (null if delivery-only) |
| `deliveredTimestamp` | When delivered to participant (null if read-only) |

## Per-Instance Control

The `MESSAGE_RECEIPT` event uses the **standard per-instance webhook event subscription** — no custom config toggle needed.

- **Instances with `events: []` (all events)**: `MESSAGE_RECEIPT` is included automatically
- **Instances with explicit event lists**: Need to add `'MESSAGE_RECEIPT'` to their events array to receive it
- **Instances with webhook disabled**: No receipts emitted

To enable for an existing instance with explicit events, update its webhook settings to include `MESSAGE_RECEIPT` in the events array.

## Decisions Made

| Decision | Choice | Reason |
|----------|--------|--------|
| Self-receipt filtering | **Filter out** | Sender's own phone generates immediate self-receipts — noise for the consuming app |
| Storage | **Webhook-only** (no Prisma model) | Consuming app (Laravel) stores the data — avoids migration and DB volume in Evolution API |
| Config toggle | **None** (use per-instance event subscription) | Standard mechanism already supports enable/disable per instance |
| Existing aggregate update | **Preserved** | `Message.status` bulk update still runs for Evolution API's internal consistency |

## Volume Consideration

For a 100-member group, each message can generate up to ~100 delivery + ~100 read webhook events. Payloads are ~200 bytes each. The consuming app should batch/throttle as needed. Instances not subscribed to `MESSAGE_RECEIPT` generate zero overhead.

## Verification Steps

1. Deploy `2.3.7-fix-0.0.5` image
2. Ensure the instance's webhook events include `MESSAGE_RECEIPT` (or use `events: []` for all)
3. Send a message to a group
4. Check webhook logs for `message.receipt` events as participants read the message
5. Confirm each event contains the correct `participantJid` and `type` (`read` or `delivered`)
6. Confirm the sender's own JID does NOT appear in the webhook events (self-receipt filtered)
7. Confirm `Message.status` still updates to `READ` as before (aggregate logic preserved)
