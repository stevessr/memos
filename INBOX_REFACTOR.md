# Inbox Service Refactor Summary

## Overview

The `inbox_service.proto` has been completely redesigned to follow **GitHub's Notifications API** as the industry standard reference. This ensures consistency, familiarity, and best practices.

**Reference**: [GitHub Notifications API](https://docs.github.com/en/rest/activity/notifications?apiVersion=2022-11-28)

---

## Key Changes

### 1. **Data Model Redesign**

#### Before (Old Design)
```protobuf
message Inbox {
  string name = 1;
  string sender = 2;
  string receiver = 3;
  Status status = 4;  // enum: UNREAD, ARCHIVED
  Timestamp create_time = 5;
  Type type = 6;  // enum: MEMO_COMMENT, VERSION_UPDATE
  optional int32 activity_id = 7;
}
```

#### After (GitHub-aligned Design)
```protobuf
message Inbox {
  string name = 1;
  string sender = 2;              // With resource_reference
  string receiver = 3;            // With resource_reference
  bool unread = 4;                // ✨ GitHub: "unread" field
  Timestamp create_time = 5;
  Timestamp update_time = 6;      // ✨ GitHub: "updated_at"
  optional Timestamp last_read_at = 7;  // ✨ GitHub: "last_read_at"
  string reason = 8;              // ✨ GitHub: "reason" (comment, mention, etc.)
  Subject subject = 9;            // ✨ GitHub: "subject" object
  string activity = 10;           // Resource reference instead of int32 ID
}

message Subject {
  string title = 1;               // ✨ GitHub: "subject.title"
  string type = 2;                // ✨ GitHub: "subject.type"
  string url = 3;                 // ✨ GitHub: "subject.url"
  optional string latest_comment_url = 4;  // ✨ GitHub: "subject.latest_comment_url"
}
```

### 2. **API Operations**

| Operation | GitHub Equivalent | Status |
|-----------|------------------|--------|
| `ListInboxes` | List notifications | ✅ Enhanced |
| `GetInbox` | Get a thread | ✅ **NEW** |
| `UpdateInbox` | Mark a thread as read/done | ✅ Updated |
| `DeleteInbox` | Delete a thread subscription | ✅ Updated |
| `MarkInboxAsRead` | Mark a thread as read | ✅ **NEW** |
| `MarkAllInboxesAsRead` | Mark notifications as read | ✅ **NEW** |

### 3. **List API Enhancements**

#### Request Parameters
```protobuf
message ListInboxesRequest {
  string parent = 1;              // Required: users/{user}
  bool unread_only = 2;           // ✨ GitHub: "all" parameter
  int32 page_size = 3;            // Max 100 (GitHub standard)
  string page_token = 4;
  string reason = 5;              // ✨ Filter by reason
  optional Timestamp since = 6;   // ✨ GitHub: "since" parameter
  optional Timestamp before = 7;  // ✨ GitHub: "before" parameter
}
```

#### Response
```protobuf
message ListInboxesResponse {
  repeated Inbox inboxes = 1;
  string next_page_token = 2;
}
```

---

## Breaking Changes

### Field Changes

| Old Field | New Field | Change Type | Migration |
|-----------|-----------|-------------|-----------|
| `status` (enum) | `unread` (bool) | Type change | `UNREAD` → `true`, `ARCHIVED/READ` → `false` |
| `type` (enum) | `reason` (string) | Type change | `MEMO_COMMENT` → `"comment"`, etc. |
| `activity_id` (int32) | `activity` (string) | Type + format | `123` → `"activities/123"` |
| N/A | `update_time` | **NEW** | Set to current time for existing records |
| N/A | `last_read_at` | **NEW** | Set to `null` for unread, current time for read |
| N/A | `subject` | **NEW** | Build from existing data |

### Enum Removal

**Old Status Enum** (REMOVED):
```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;
  UNREAD = 1;
  READ = 2;        // Not used before
  ARCHIVED = 3;
}
```

**New Approach**: Simple `bool unread` field

**Old Type Enum** (REMOVED):
```protobuf
enum Type {
  TYPE_UNSPECIFIED = 0;
  MEMO_COMMENT = 1;
  VERSION_UPDATE = 2;
  MEMO_MENTION = 3;
}
```

**New Approach**: String `reason` field with values like:
- `"comment"` (was `MEMO_COMMENT`)
- `"mention"` (was `MEMO_MENTION`)
- `"version_update"` (was `VERSION_UPDATE`)
- Future: `"assign"`, `"review_requested"`, etc.

---

## Database Migration Required

### Schema Changes

```sql
-- Add new columns
ALTER TABLE inbox ADD COLUMN update_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
ALTER TABLE inbox ADD COLUMN last_read_at TIMESTAMP NULL;
ALTER TABLE inbox ADD COLUMN reason TEXT NOT NULL DEFAULT '';
ALTER TABLE inbox ADD COLUMN subject_title TEXT NOT NULL DEFAULT '';
ALTER TABLE inbox ADD COLUMN subject_type TEXT NOT NULL DEFAULT '';
ALTER TABLE inbox ADD COLUMN subject_url TEXT NOT NULL DEFAULT '';
ALTER TABLE inbox ADD COLUMN subject_latest_comment_url TEXT NULL;

-- Migrate existing data
UPDATE inbox SET 
  unread = (status = 'UNREAD'),
  reason = CASE type
    WHEN 'MEMO_COMMENT' THEN 'comment'
    WHEN 'VERSION_UPDATE' THEN 'version_update'
    WHEN 'MEMO_MENTION' THEN 'mention'
    ELSE ''
  END,
  last_read_at = CASE 
    WHEN status != 'UNREAD' THEN update_time 
    ELSE NULL 
  END;

-- Convert activity_id to activity resource name
-- This needs to be handled in application code during migration

-- Remove old columns (after migration is complete)
-- ALTER TABLE inbox DROP COLUMN status;
-- ALTER TABLE inbox DROP COLUMN type;
-- ALTER TABLE inbox DROP COLUMN activity_id;
```

---

## Backend Implementation Tasks

### 1. Update Store Layer (`store/inbox.go`)

```go
type Inbox struct {
    ID               int32
    Name             string
    Sender           string
    Receiver         string
    Unread           bool                // Changed from Status
    CreateTime       time.Time
    UpdateTime       time.Time           // NEW
    LastReadAt       *time.Time          // NEW
    Reason           string              // Changed from Type enum
    SubjectTitle     string              // NEW
    SubjectType      string              // NEW
    SubjectURL       string              // NEW
    SubjectLatestCommentURL *string      // NEW
    Activity         string              // Changed from ActivityID int32
}
```

### 2. Update Service Layer (`server/router/api/v1/inbox_service.go`)

**Implement New RPCs**:
- ✅ `GetInbox` - Fetch single notification
- ✅ `MarkInboxAsRead` - Mark single as read
- ✅ `MarkAllInboxesAsRead` - Batch mark as read

**Update Existing RPCs**:
- ✅ `ListInboxes` - Add new filters (unread_only, reason, since, before)
- ✅ `UpdateInbox` - Handle `unread` field instead of `status`
- ✅ `DeleteInbox` - No changes needed

### 3. Update Converters

```go
func convertInboxFromStore(inbox *store.Inbox) *apiv1.Inbox {
    return &apiv1.Inbox{
        Name:       fmt.Sprintf("inboxes/%d", inbox.ID),
        Sender:     fmt.Sprintf("users/%d", inbox.SenderID),
        Receiver:   fmt.Sprintf("users/%d", inbox.ReceiverID),
        Unread:     inbox.Unread,
        CreateTime: timestamppb.New(inbox.CreateTime),
        UpdateTime: timestamppb.New(inbox.UpdateTime),
        LastReadAt: convertOptionalTimestamp(inbox.LastReadAt),
        Reason:     inbox.Reason,
        Subject: &apiv1.Inbox_Subject{
            Title:              inbox.SubjectTitle,
            Type:               inbox.SubjectType,
            Url:                inbox.SubjectURL,
            LatestCommentUrl:   inbox.SubjectLatestCommentURL,
        },
        Activity:   fmt.Sprintf("activities/%d", inbox.ActivityID),
    }
}
```

---

## Frontend Updates Required

### 1. Update Type Definitions (`web/src/types/proto/api/v1/inbox_service.ts`)

The generated TypeScript types will change significantly:

```typescript
// OLD
interface Inbox {
  name: string;
  sender: string;
  receiver: string;
  status: Inbox_Status;  // UNREAD, ARCHIVED
  createTime: Date;
  type: Inbox_Type;      // MEMO_COMMENT, VERSION_UPDATE
  activityId?: number;
}

// NEW
interface Inbox {
  name: string;
  sender: string;
  receiver: string;
  unread: boolean;       // ✨ Changed
  createTime: Date;
  updateTime: Date;      // ✨ NEW
  lastReadAt?: Date;     // ✨ NEW
  reason: string;        // ✨ Changed
  subject: Inbox_Subject; // ✨ NEW
  activity: string;      // ✨ Changed
}

interface Inbox_Subject {
  title: string;
  type: string;
  url: string;
  latestCommentUrl?: string;
}
```

### 2. Update UI Components

**Inbox List Component**:
```typescript
// OLD
{inbox.status === Inbox_Status.UNREAD && <UnreadBadge />}

// NEW
{inbox.unread && <UnreadBadge />}
```

**Mark as Read**:
```typescript
// OLD
await updateInbox({
  inbox: { ...inbox, status: Inbox_Status.ARCHIVED }
});

// NEW
await markInboxAsRead({ name: inbox.name });
```

**Display Subject**:
```typescript
// NEW
<div>
  <h3>{inbox.subject.title}</h3>
  <span>{inbox.subject.type}</span>
  <a href={inbox.subject.url}>View</a>
</div>
```

---

## Comparison with GitHub's API

### GitHub Notification Object
```json
{
  "id": "1",
  "unread": true,
  "reason": "mention",
  "updated_at": "2023-01-01T12:00:00Z",
  "last_read_at": null,
  "subject": {
    "title": "Issue title",
    "url": "https://api.github.com/repos/...",
    "latest_comment_url": "https://api.github.com/repos/.../comments/1",
    "type": "Issue"
  },
  "repository": {...},
  "url": "https://api.github.com/notifications/threads/1"
}
```

### Memos Inbox Object
```json
{
  "name": "inboxes/1",
  "sender": "users/2",
  "receiver": "users/1",
  "unread": true,
  "reason": "comment",
  "createTime": "2023-01-01T12:00:00Z",
  "updateTime": "2023-01-01T12:05:00Z",
  "lastReadAt": null,
  "subject": {
    "title": "New comment on your memo",
    "url": "/m/123",
    "latestCommentUrl": "/m/123#comment-456",
    "type": "Memo"
  },
  "activity": "activities/789"
}
```

**Similarities**:
- ✅ `unread` boolean field
- ✅ `reason` string field
- ✅ `subject` nested object with title, url, type
- ✅ Timestamp fields (updated_at → update_time, last_read_at)
- ✅ List/Get/Mark as read operations

**Differences**:
- Memos uses resource names (`users/1`) vs GitHub uses objects
- Memos links to `activity` resource vs GitHub links to `repository`
- Memos uses `create_time` in addition to `update_time`

---

## Benefits

### 1. **Industry Standard Alignment**
- Developers familiar with GitHub's API will understand this immediately
- Reduces cognitive load for new contributors

### 2. **Better Developer Experience**
- Simpler boolean `unread` vs enum `Status`
- Flexible string `reason` vs limited enum `Type`
- Rich `subject` object with all notification context

### 3. **Future Extensibility**
- Easy to add new `reason` values without proto changes
- `subject` can include more fields as needed
- Matches patterns used in other popular APIs (GitLab, Bitbucket, etc.)

### 4. **Consistency**
- Aligns with Activity service (similar patterns)
- Follows Google API design guidelines
- Resource-oriented design

---

## Migration Checklist

### Phase 1: Proto & Code Generation ✅
- [x] Update `inbox_service.proto`
- [x] Run `buf format` and `buf generate`
- [x] Commit proto changes

### Phase 2: Backend Implementation 🚧
- [ ] Create database migration scripts
- [ ] Update `store/inbox.go` model
- [ ] Update store CRUD operations
- [ ] Implement `GetInbox` RPC
- [ ] Implement `MarkInboxAsRead` RPC
- [ ] Implement `MarkAllInboxesAsRead` RPC
- [ ] Update `ListInboxes` with new filters
- [ ] Update converters
- [ ] Add/update tests

### Phase 3: Frontend Updates 🚧
- [ ] Regenerate TypeScript types
- [ ] Update inbox list component
- [ ] Update notification badge logic
- [ ] Update mark-as-read handlers
- [ ] Update UI to display `subject` fields
- [ ] Handle `reason` field display
- [ ] Add/update tests

### Phase 4: Data Migration 🚧
- [ ] Create migration script
- [ ] Test migration on staging data
- [ ] Run migration on production
- [ ] Verify data integrity

### Phase 5: Cleanup 🚧
- [ ] Remove old enum references
- [ ] Remove deprecated fields
- [ ] Update documentation
- [ ] Announce breaking changes

---

## Timeline Estimate

- **Phase 1**: ✅ Complete (Proto changes)
- **Phase 2**: 2-3 days (Backend implementation)
- **Phase 3**: 2-3 days (Frontend updates)
- **Phase 4**: 1 day (Migration + testing)
- **Phase 5**: 1 day (Cleanup + docs)

**Total**: ~1 week for complete refactor

---

## Questions & Considerations

### Q: Why remove Status enum in favor of boolean?
**A**: GitHub's approach is simpler. A notification is either unread or not. The "archived" concept can be handled through deletion or a separate flag if needed later.

### Q: Why string `reason` instead of enum?
**A**: Flexibility. We can add new reasons without changing the proto definition. GitHub does this too.

### Q: What about backward compatibility?
**A**: This is a breaking change. We need:
1. Version the API (v1 → v2) OR
2. Coordinated deployment with migration OR
3. Support both formats during transition

### Q: Do we need `total_size` and `unread_count` in ListResponse?
**A**: GitHub doesn't include these. They can be expensive to compute. We can add them back if needed, or provide separate summary endpoints.

---

## References

- [GitHub Notifications API](https://docs.github.com/en/rest/activity/notifications)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [GitLab Notifications API](https://docs.gitlab.com/ee/api/notifications.html)
- [Memos Activity Service](./proto/api/v1/activity_service.proto)
