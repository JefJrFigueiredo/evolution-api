# Evolution API: GROUP_UPDATE Webhook Not Firing - Investigation & Fix

This document explains why the `GROUP_UPDATE` webhook is not being sent when group metadata (name, description) changes in Evolution API v2.3.7.

---

## Table of Contents

- [The Problem](#the-problem)
- [Root Cause Analysis](#root-cause-analysis)
- [Technical Details](#technical-details)
- [Solutions](#solutions)
- [Verification Steps](#verification-steps)

---

## The Problem

When a WhatsApp group's **name** or **description** is changed via the mobile app:

- ❌ No `GROUP_UPDATE` webhook is dispatched
- ✅ Other events like `GROUP_PARTICIPANTS_UPDATE` work correctly
- ✅ All other webhook events (MESSAGES_UPSERT, CONTACTS_UPDATE, etc.) work
- ✅ The change is visible in WhatsApp immediately

The webhook configuration explicitly includes `GROUP_UPDATE`:

```json
{
  "events": [
    "CONNECTION_UPDATE",
    "MESSAGES_UPSERT",
    "GROUPS_UPSERT",
    "GROUP_UPDATE",
    "GROUP_PARTICIPANTS_UPDATE"
  ]
}
```

Yet no webhook is sent.

---

## Root Cause Analysis

### The Issue: Enum vs Configuration Naming Mismatch

There's an **inconsistency between the Events enum and configuration layer**:

| Component | Name Used | Correct? |
|-----------|-----------|----------|
| **Events enum** | `GROUPS_UPDATE` (plural) | ✅ Correct |
| **Baileys handler** | `groups.update` | ✅ Correct |
| **Webhook config type** | `GROUP_UPDATE` (singular) | ❌ Wrong |
| **Validation schemas** | `GROUP_UPDATE` (singular) | ❌ Wrong |
| **Environment variable** | `WEBHOOK_EVENTS_GROUPS_UPDATE` (plural) | ✅ Correct |

### The Disconnect

```
Baileys emits: "groups.update"
  ↓
Handler receives it as: groups.update payload
  ↓
Handler emits webhook with: Events.GROUPS_UPDATE (enum value: "groups.update")
  ↓
Webhook dispatcher looks for: webhook config "GROUP_UPDATE" (doesn't match enum)
  ↓
Result: Webhook is NOT dispatched ❌
```

---

## Technical Details

### 1. Event Emission (Works Correctly ✅)

**File:** `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts`

**Lines 1764-1774** - Group metadata handler:

```typescript
'groups.update': (groupMetadataUpdate: Partial<GroupMetadata>[]) => {
  this.sendDataWebhook(Events.GROUPS_UPDATE, groupMetadataUpdate);  // ✅ Correct enum
  
  groupMetadataUpdate.forEach((group) => {
    if (isJidGroup(group.id)) {
      this.updateGroupMetadataCache(group.id);
    }
  });
}
```

**Lines 1990-1993** - Baileys event listener:

```typescript
if (events['groups.update']) {
  const payload = events['groups.update'];
  this.groupHandler['groups.update'](payload);  // ✅ Correctly routes to handler
}
```

✅ **This part works correctly!** The event IS being received and processed.

### 2. Events Enum Definition

**File:** `src/api/types/server.types.ts`

**Lines 27-29**:

```typescript
export enum Events {
  GROUPS_UPSERT = 'groups.upsert',
  GROUPS_UPDATE = 'groups.update',      // ✅ PLURAL (correct)
  GROUP_PARTICIPANTS_UPDATE = 'group-participants.update',
}
```

✅ **Enum is correct** - uses `GROUPS_UPDATE` (plural).

### 3. Configuration Type Mismatch ❌

**File:** `src/api/types/index.ts`

**Lines 89 & 221** - WebhookEvents interface:

```typescript
export interface ServerWebhookEvents {
  // ...
  GROUP_UPDATE: boolean;              // ❌ SINGULAR (WRONG!)
  GROUP_PARTICIPANTS_UPDATE: boolean; // ✅ Correct
  // ...
}
```

❌ **Problem:** Config uses `GROUP_UPDATE` (singular) but enum uses `GROUPS_UPDATE` (plural).

### 4. Validation Schema Mismatch ❌

**File:** `src/validate/webhook.schema.ts`

**Lines 81, 118, 155, 192** - Event validation:

```typescript
enum: [
  'GROUP_UPDATE',              // ❌ SINGULAR (expects singular)
  'GROUP_PARTICIPANTS_UPDATE',
  // ...
]
```

❌ **Problem:** Validation schema expects `GROUP_UPDATE` but webhook dispatcher sends `GROUPS_UPDATE`.

### 5. Working Comparison: GROUP_PARTICIPANTS_UPDATE

**File:** `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts`

**Lines 1775-1790** - This event WORKS:

```typescript
'group-participants.update': async (participantsUpdate: {...}) => {
  // ... resolve participant data ...
  this.sendDataWebhook(Events.GROUP_PARTICIPANTS_UPDATE, enhancedData);
  // ...
}
```

**Why it works:**
- ✅ Enum: `GROUP_PARTICIPANTS_UPDATE` (plural)
- ✅ Config: `GROUP_PARTICIPANTS_UPDATE` (plural)
- ✅ Validation: `GROUP_PARTICIPANTS_UPDATE` (plural)
- ✅ Environment: `WEBHOOK_EVENTS_GROUP_PARTICIPANTS_UPDATE`

**All naming is CONSISTENT!**

---

## How the Webhook Dispatcher Works

When `sendDataWebhook(Events.GROUPS_UPDATE, data)` is called:

1. Event name becomes: `"groups.update"` (the enum value)
2. Webhook config is checked for: `webhook.events` array contains this event
3. But the config was set with event name: `"GROUP_UPDATE"` (singular)
4. String comparison: `"groups.update"` ≠ `"GROUP_UPDATE"` ❌
5. Event is NOT found in webhook configuration
6. Webhook is **NOT dispatched**

---

## Solutions

### Solution 1: Fix the Configuration Type (Recommended)

**Change:** Rename `GROUP_UPDATE` → `GROUPS_UPDATE` in configuration layer

**Files to modify:**

#### 1. `src/api/types/index.ts` (Lines 89, 221)

```typescript
// Change from:
GROUP_UPDATE: boolean;

// To:
GROUPS_UPDATE: boolean;
```

Also update the environment variable mapping:
```typescript
// Change from:
GROUP_UPDATE: process.env?.WEBHOOK_EVENTS_GROUPS_UPDATE === 'true',

// To:
GROUPS_UPDATE: process.env?.WEBHOOK_EVENTS_GROUPS_UPDATE === 'true',
```

#### 2. `src/validate/webhook.schema.ts` (Lines 81, 118, 155, 192)

Change all occurrences of `'GROUP_UPDATE'` to `'GROUPS_UPDATE'`:

```typescript
enum: [
  'GROUPS_UPSERT',
  'GROUPS_UPDATE',              // Changed from GROUP_UPDATE
  'GROUP_PARTICIPANTS_UPDATE',
  // ... rest of events
]
```

#### 3. Update webhook configuration examples

Any documentation or examples using `"GROUP_UPDATE"` should use `"GROUPS_UPDATE"`:

```json
{
  "events": [
    "CONNECTION_UPDATE",
    "MESSAGES_UPSERT",
    "GROUPS_UPSERT",
    "GROUPS_UPDATE",              // Not "GROUP_UPDATE"
    "GROUP_PARTICIPANTS_UPDATE"
  ]
}
```

### Solution 2: Workaround (If You Can't Modify Code)

If you're using a managed Evolution API service and can't modify the code:

1. **Use environment variables** - The environment variable is correct:
   ```bash
   WEBHOOK_EVENTS_GROUPS_UPDATE=true
   ```

2. **Don't specify in JSON config** - Use environment variables instead:
   ```bash
   # Instead of specifying in instance webhook config, rely on:
   WEBHOOK_EVENTS_GROUPS_UPDATE=true
   ```

3. **Monitor the server logs** - Group update events ARE being emitted. You can verify in Evolution API logs that `groups.update` handler is being called.

### Solution 3: Patch the Evolution API Container

If you're running your own Evolution API Docker image:

1. Download the source code
2. Apply the fix to `src/api/types/index.ts` and `src/validate/webhook.schema.ts`
3. Rebuild the Docker image
4. Restart your container

---

## Verification Steps

### Step 1: Verify Baileys is Receiving Events

Add debug logging to `whatsapp.baileys.service.ts` line 1764:

```typescript
'groups.update': (groupMetadataUpdate: Partial<GroupMetadata>[]) => {
  console.log('🔍 Baileys groups.update received:', groupMetadataUpdate);  // Debug log
  this.sendDataWebhook(Events.GROUPS_UPDATE, groupMetadataUpdate);
  // ...
}
```

When you change a group name, you should see this log. If you do, Baileys is working correctly.

### Step 2: Verify Webhook Configuration

When creating/updating an instance, include `GROUPS_UPDATE` in the webhook events array:

```bash
curl -X POST "http://evolution-api:8080/webhook/update/myinstance" \
  -H "apikey: changeme" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "http://web/api/webhooks",
    "events": [
      "CONNECTION_UPDATE",
      "MESSAGES_UPSERT",
      "GROUPS_UPDATE"  # Use plural (if you've applied the fix)
    ]
  }'
```

### Step 3: Test with a Group Update

1. Change a group name via WhatsApp mobile app
2. Check your webhook receiver logs
3. You should receive:

```json
{
  "event": "groups.update",
  "instance": "your_instance",
  "data": [
    {
      "id": "120363123456789012@g.us",
      "subject": "New Group Name",
      "subjectTime": 1704700000,
      "subjectOwner": "5511999999999@s.whatsapp.net",
      "restrict": false,
      "announce": false,
      "desc": "Group description",
      "descTime": 1704700000,
      "descOwner": "5511999999999@s.whatsapp.net"
    }
  ]
}
```

---

## Summary Table

| Issue | Root Cause | Impact | Fix |
|-------|-----------|--------|-----|
| Naming mismatch | `GROUP_UPDATE` (config) vs `GROUPS_UPDATE` (enum) | Webhook not dispatched | Rename to `GROUPS_UPDATE` |
| Validation conflict | Schema validates `GROUP_UPDATE` but enum sends `GROUPS_UPDATE` | Instance rejects plural form | Update validation schema |
| Config type inconsistency | `WebhookEvents` interface uses singular form | Type checking fails | Update interface definition |

---

## Expected GROUP_UPDATE Payload

When a group's metadata changes, the webhook will contain:

```json
{
  "event": "groups.update",
  "instance": "ws_1_phone_3",
  "data": [
    {
      "id": "120363123456789012@g.us",
      "subject": "New Group Name",
      "subjectTime": 1704700000,
      "subjectOwner": "5511999999999@s.whatsapp.net",
      "restrict": false,
      "announce": false,
      "desc": "Updated description here",
      "descTime": 1704700000,
      "descOwner": "5511999999999@s.whatsapp.net",
      "descId": "...",
      "instanceId": "uuid-of-instance"
    }
  ],
  "date_time": "2026-01-08T10:30:00.000Z",
  "sender": "5511888888888@s.whatsapp.net",
  "server_url": "https://api.example.com"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Group JID (e.g., `120363...@g.us`) |
| `subject` | string | Group name |
| `subjectTime` | number | Timestamp of last name change |
| `subjectOwner` | string | JID of who changed the name |
| `restrict` | boolean | Is group restricted (only admins post) |
| `announce` | boolean | Is group in announcements mode |
| `desc` | string | Group description |
| `descTime` | number | Timestamp of last description change |
| `descOwner` | string | JID of who changed the description |

---

## Files Affected

| File | Lines | Change |
|------|-------|--------|
| `src/api/types/index.ts` | 89, 221 | Rename `GROUP_UPDATE` → `GROUPS_UPDATE` |
| `src/validate/webhook.schema.ts` | 81, 118, 155, 192 | Update enum from `GROUP_UPDATE` → `GROUPS_UPDATE` |

---

## Related Events

For reference, here are similar group events that SHOULD work:

| Event Name | Triggered When |
|------------|----------------|
| `GROUPS_UPSERT` | New groups received |
| `GROUPS_UPDATE` | Group metadata changes (name, description, picture) |
| `GROUP_PARTICIPANTS_UPDATE` | Members join/leave or role changes |

---

*Investigation completed: January 8, 2026*  
*Based on Evolution API v2.3.7 source code analysis*
