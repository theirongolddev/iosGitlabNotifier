---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - prd.md
  - product-brief-iosGitlabNotifier-2026-01-06.md
workflowType: 'architecture'
project_name: 'iosGitlabNotifier'
user_name: 'Ubuntu'
date: '2026-01-07'
lastStep: 8
status: 'complete'
completedAt: '2026-01-08'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
45 requirements spanning 12 functional areas. The core flow is: authenticate with GitLab → discover items → manage watch list → receive notifications → triage via deep links.

Key FR clusters:
- **Authentication & Connection** (6 FRs): PAT and OAuth support, connection validation, re-auth handling
- **Item Discovery** (7 FRs): List issues/MRs by involvement type, filtering, metadata display
- **Watch List** (5 FRs): Watch/unwatch toggle, view watch list, cross-device sync, mute/snooze (FR44)
- **GitLab Notification Alignment** (2 FRs): Import GitLab subscriptions, mirror subscription state
- **Device & Server Registration** (2 FRs): APNs device registration, device management
- **Push Notifications** (5 FRs): Rich notifications with actor/action/item context, new-tag detection
- **Notification Actions** (3 FRs): Watch/Dismiss quick actions, deep linking to GitLab
- **Notification History** (1 FR): View notification history for watched items
- **System Status & Feedback** (3 FRs): Connection status display, operation failure feedback, actionable errors
- **Offline & Sync** (5 FRs): Cached data display, offline indicator, queued changes, manual sync
- **macOS Menu Bar** (5 FRs): Icon, badge, dropdown, Focus mode integration
- **Diagnostics & Trust** (1 FR): View polling/sync status details (FR45)

**Non-Functional Requirements:**
- Performance: < 2 minute notification latency, 1-2 minute server polling
- Reliability: Near-zero missed notifications on watched items (> 99% - CRITICAL for /todos + per-resource polling, /events as fallback)
- Security: Platform Keychain for credentials, TLS for all communication, no plaintext logging
- Integration: GitLab API rate limit handling, APNs token refresh and error handling
- Resource: iOS/macOS battery best practices, minimal background network activity

**Scale & Complexity:**
- Primary domain: Mobile (iOS/macOS) + Backend (Node.js notification server)
- Complexity level: Low-Medium
- Estimated architectural components: 5 (iOS app, macOS app, Shared Swift package, Notification server, APNs integration)

### Technical Constraints & Dependencies

- **Self-hosted GitLab support**: Architecture must handle arbitrary instance URLs with user-provided credentials
- **APNs requirement**: Production push notifications require Apple Developer Program enrollment
- **Server deployment**: User must host the notification server - single point of failure requiring resilience strategy (health checks, restart policies, monitoring)
- **Platform minimums**: iOS 17+ / macOS 14+ (enables @Observable and modern SwiftUI features)
- **GitLab API strategy**:
  - `GET /todos` for items needing direct attention (mentions, review requests, assignments)
  - **Primary for watched items:** per-resource polling of notes/discussions (comments + system notes when available)
  - **Secondary for watched items:** per-resource "activity" endpoint(s) for state transitions (approval/merge/close/push)
  - **Last resort fallback:** global `/events?after=...` only when per-resource polling is degraded or unavailable

### Architectural Decisions (Pre-resolved)

| Decision | Resolution | Rationale |
|----------|------------|-----------|
| Watch list sync conflicts | Last-write-wins (timestamp) | Simple, predictable, sufficient for single-user |
| Error handling boundary | Shared layer handles common cases | Retry logic, auth refresh in shared code; apps handle UI for fatal errors only |
| Observability stack | Prometheus + Grafana + alerting | Required to verify reliability NFR (> 99%); server exposes metrics endpoint |

### Cross-Cutting Concerns Identified

1. **Authentication flow**: Token storage (Keychain), refresh, expiry detection - affects both clients and server
2. **Watch list synchronization**: Last-write-wins conflict resolution, server-side storage
3. **Error handling & user feedback**: Shared layer handles retries/auth; consistent fatal error presentation across platforms
4. **Offline behavior**: All clients need cached data, queued actions, clear offline indicators
5. **Push notification reliability**: Delivery confirmation via receipt flow, retry logic, Prometheus metrics for monitoring
6. **Observability & Monitoring**: Server health metrics, polling success rate, APNs delivery rate, alerting on degradation
7. **Shared code boundaries**: Explicit definition of what lives in Swift package vs platform-specific targets

---

## Starter Template Evaluation

### Primary Technology Domains

| Domain | Technology | Starter Approach |
|--------|------------|------------------|
| Notification Server | Fastify + TypeScript | DriftOS/fastify-starter template |
| iOS/macOS Apps | Swift + SwiftUI | Xcode Multiplatform template + Swift Package |

---

### Server: DriftOS/fastify-starter

**Repository:** [github.com/DriftOS/fastify-starter](https://github.com/DriftOS/fastify-starter)

**Rationale for Selection:**
- Prometheus metrics + Grafana dashboards pre-configured
- Prisma ORM - configured for SQLite
- JWT authentication - secures device registration and API endpoints
- Docker Compose ready with monitoring stack
- Health check endpoints for resilience
- Structured logging with Pino
- Golden Orchestrator pattern - natural fit for poll → detect → notify pipeline

**Initialization:**
```bash
git clone https://github.com/DriftOS/fastify-starter.git server
cd server
npm install
```

**Day-One Modifications:**

| File | Change |
|------|--------|
| `prisma/schema.prisma` | Change `provider = "postgresql"` to `provider = "sqlite"` |
| `docker/docker-compose.yml` | Remove Postgres service, keep Prometheus + Grafana |
| `.env` | Update `DATABASE_URL` to `file:./data/notifier.db` |

**Prisma Schema:**

```prisma
model User {
  id                 String   @id @default(uuid())
  gitlabUrl          String
  gitlabUserId       Int      // Fetched from /api/v4/user during registration
  gitlabPatEncrypted String   // AES-256 encrypted at application layer
  gitlabVersion      String?  // Best-effort version string (self-hosted instances vary)
  capabilitiesJson   String?  // Stored capability discovery results for this instance/user
  createdAt          DateTime @default(now())
  devices            DeviceToken[]
  watchedItems       WatchedItem[]
  pollState          PollState?
  notifiedItems      NotifiedItem[]
}

model DeviceToken {
  id        String   @id @default(uuid())
  token     String   @unique
  platform  String   // "ios" | "macos"
  active    Boolean  @default(true)
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}

model WatchedItem {
  id               String    @id @default(uuid())
  itemIid          Int       // Project-scoped IID (Issue IID / MR IID)
  itemType         String    // "issue" | "merge_request"
  projectId        Int       // Needed for /events filtering
  title            String?
  webUrl           String?
  projectPath      String?
  mutedUntil       DateTime? // Server-side mute for focus protection
  syncSource       String    @default("app") // "app" | "gitlab" | "hybrid"
  watchSource      String    @default("manual") // "manual" | "auto" (auto = added via new-tag auto-watch)
  gitlabSubscribed Boolean   @default(false) // Last-known GitLab subscription state
  userId           String
  user             User      @relation(fields: [userId], references: [id])
  cursor           WatchedItemCursor?
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  @@unique([userId, projectId, itemIid, itemType])
}

// Tracks items user explicitly dismissed (for reversible dismiss)
model DismissedItem {
  id          String   @id @default(uuid())
  itemIid     Int
  itemType    String   // "issue" | "merge_request"
  projectId   Int
  title       String?
  webUrl      String?
  projectPath String?
  userId      String
  user        User     @relation(fields: [userId], references: [id])
  dismissedAt DateTime @default(now())

  @@unique([userId, projectId, itemIid, itemType])
  @@index([userId, dismissedAt])
}

// Actor deny-list for noise filtering (e.g., bots)
model ActorDenyList {
  id           String   @id @default(uuid())
  userId       String
  user         User     @relation(fields: [userId], references: [id])
  gitlabUserId Int      // GitLab user ID to suppress
  displayName  String?  // Cached display name for UI
  reason       String?  // Optional note (e.g., "CI bot")
  createdAt    DateTime @default(now())

  @@unique([userId, gitlabUserId])
  @@index([userId])
}

model PollState {
  id                   String   @id @default(uuid())
  userId               String   @unique
  user                 User     @relation(fields: [userId], references: [id])
  lastTodosFetchedAt   DateTime // Timestamp for /todos polling
  lastTodosFetchedId   Int?     // ID tie-breaker for timestamp collisions
  lastEventsFetchedAt  DateTime // Fallback-only cursor for global /events
  lastEventsFetchedId  Int?     // ID tie-breaker for timestamp collisions
  lastSuccessfulPollAt DateTime
  pollLockUntil        DateTime? // Prevent overlapping polls on restart / multi-instance
}

model WatchedItemCursor {
  id                    String      @id @default(uuid())
  watchedItemId         String      @unique
  watchedItem           WatchedItem @relation(fields: [watchedItemId], references: [id], onDelete: Cascade)
  lastActivityFetchedAt DateTime
  lastSeenNoteId        Int?        // For notes/discussions endpoints when available
  updatedAt             DateTime    @updatedAt
}

model NotifiedItem {
  id           String   @id @default(uuid())
  sourceType   String   // "todo" | "note" | "activity" | "event" (note/activity = per-resource, event = last resort fallback)
  sourceId     Int      // gitlabTodoId OR gitlabNoteId OR activityId OR gitlabEventId
  userId       String
  user         User     @relation(fields: [userId], references: [id])
  notifiedAt   DateTime @default(now())

  @@unique([userId, sourceType, sourceId])
  @@index([userId, notifiedAt])
}

model NotificationReceipt {
  id             String   @id @default(uuid())
  notificationId String   @unique // UUID sent in push payload
  userId         String
  sentAt         DateTime @default(now())
  receivedAt     DateTime? // Populated when app pings back

  @@index([userId, sentAt])
}

model NotificationOutbox {
  id             String    @id @default(uuid())
  notificationId String    @unique
  userId         String
  payloadJson    String    // Full APNs payload for retry
  createdAt      DateTime  @default(now())
  status         String    @default("queued") // queued|sending|sent|dead
  lastError      String?
  nextAttemptAt  DateTime?
  attempts       NotificationAttempt[]

  @@index([status, nextAttemptAt])
  @@index([userId, createdAt])
}

model NotificationAttempt {
  id             String   @id @default(uuid())
  notificationId String
  outbox         NotificationOutbox @relation(fields: [notificationId], references: [notificationId])
  deviceTokenId  String
  attemptedAt    DateTime @default(now())
  result         String   // success|retryable_error|permanent_error
  apnsId         String?  // From APNs response when available
  error          String?

  @@index([notificationId, attemptedAt])
}
```

**Security Notes:**
- GitLab PAT must be encrypted at application layer before storage (AES-256-GCM recommended)
- Never log credentials - configure Pino redaction for sensitive fields
- TLS required for all client-server communication

**Secret Management Enhancements:**
- Support encryption key rotation:
  - ENCRYPTION_KEY_CURRENT (used for new writes)
  - ENCRYPTION_KEYS_OLD (comma-separated, tried for reads)
  - Store keyVersion alongside gitlabPatEncrypted
- Validate PAT/OAuth scopes during /auth/register (fail fast with actionable error)
- Add "revoke all sessions" endpoint (invalidate JWTs + deactivate device tokens)

**Cold-Start Strategy:**
```
ON USER REGISTRATION (first poll):

1. Fetch GitLab /api/v4/user to get gitlabUserId
2. Fetch current /todos to populate baseline
   - Set PollState.lastTodosFetchedAt = now()
   - Do NOT create notifications (these are pre-existing)
3. Set PollState.lastEventsFetchedAt = now()
   - Do NOT fetch actual events (avoid notification flood)
4. Subsequent polls will only catch NEW activity after registration
```

**Testing Notes:**
- Vitest included for unit/integration tests
- Mock APNs endpoint via simple Fastify route
- HTTP-level mocking with nock/msw for GitLab API
- Integration test: Poll mock GitLab → detect change → verify APNs payload

---

### Server API Contract

#### POST /auth/register
Register new user with GitLab credentials. Fetches gitlabUserId from GitLab API.
Probes instance capabilities for self-hosted compatibility.
```json
// Request
{
  "gitlabUrl": "https://gitlab.company.com",
  "personalAccessToken": "glpat-xxxxxxxxxxxx"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "gitlabUserId": 42,
  "gitlabVersion": "16.8.1",  // Best-effort, may be null
  "capabilities": {
    "notes": true,                    // Per-resource notes endpoint available
    "activity": "partial",            // "full" | "partial" | "unavailable"
    "resourceStateEvents": true,      // MR/Issue state event endpoints
    "globalEvents": true,             // Fallback /events endpoint
    "rateLimitHeaders": true          // Instance provides rate limit headers
  },
  "expiresAt": "2026-01-14T12:00:00Z"
}
```

**Capability Discovery Logic:**
During registration, probe endpoints to determine available features:
1. `GET /api/v4/version` - Attempt to get GitLab version
2. `GET /api/v4/user` - Required, fails registration if unavailable
3. Test per-resource endpoints on a sample project (if accessible)
4. Store capabilities in `capabilitiesJson` for polling decisions

#### POST /auth/refresh
Refresh JWT token.
```json
// Response
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "expiresAt": "2026-01-14T12:00:00Z"
}
```

#### POST /pair/start
Creates a short-lived pairing code displayed on an already-authenticated device.
Enables secure multi-device setup without re-entering credentials.
```json
// Request (requires valid JWT)
{}

// Response
{
  "pairingCode": "ABC123",        // 6-char alphanumeric, expires in 5 minutes
  "expiresAt": "2026-01-07T10:35:00Z"
}
```

#### POST /pair/exchange
Exchanges pairing code for a device-scoped refresh token + JWT.
```json
// Request
{
  "pairingCode": "ABC123",
  "platform": "macos",
  "deviceName": "MacBook Pro"
}

// Response
{
  "token": "eyJhbGciOiJIUzI1NiIs...",      // Device-scoped JWT
  "refreshToken": "drt_xxxxxxxxxxxxxxxx",  // Device-scoped refresh token
  "deviceId": "550e8400-e29b-41d4-a716-446655440002",
  "expiresAt": "2026-01-14T12:00:00Z"
}
```

#### POST /devices
Register APNs device token. Requires device-scoped JWT.
```json
// Request (requires device-scoped JWT)
{
  "apnsToken": "a94b25e7f9d8c3b1a...",
  "platform": "ios",
  "appVersion": "1.0.0"
}

// Response
{
  "deviceId": "550e8400-e29b-41d4-a716-446655440001",
  "registered": true
}
```

#### DELETE /devices/:id
Unregister device. Revokes device-scoped tokens for this device.

#### GET /items
List user's GitLab items (issues/MRs).
```json
// Response
{
  "items": [
    {
      "id": 123,
      "type": "merge_request",
      "title": "Add authentication flow",
      "projectId": 42,
      "projectPath": "team/backend",
      "webUrl": "https://gitlab.company.com/team/backend/-/merge_requests/123",
      "state": "opened",
      "watched": true,
      "lastActivityAt": "2026-01-07T10:30:00Z"
    }
  ]
}
```

#### GET /watch
List watched items.

#### POST /watch
Add item to watch list.
```json
// Request
{
  "itemIid": 123,
  "itemType": "merge_request",
  "projectId": 42
}

// Response
{
  "watchedItemId": "550e8400-e29b-41d4-a716-446655440002",
  "watching": true
}
```

#### DELETE /watch/:id
Remove item from watch list.

#### POST /watch/import
Import tracked items from GitLab notification/subscription state (best-effort).
```json
// Response
{
  "imported": 5,
  "skipped": 2,
  "items": [
    { "itemIid": 123, "itemType": "merge_request", "projectId": 42, "title": "Auth refactor" }
  ]
}
```

#### POST /watch/:id/sync
Reconcile app tracked state ↔ GitLab subscription state (push/pull based on user preference).
```json
// Request
{
  "direction": "pull" // "pull" = GitLab → app, "push" = app → GitLab
}
```

#### POST /watch/:id/mute
Temporarily suppress notifications for this tracked item (server-side).
```json
// Request
{
  "until": "2026-01-07T14:00:00Z" // or relative: { "minutes": 30 }
}

// Response
{
  "mutedUntil": "2026-01-07T14:00:00Z"
}
```

#### DELETE /watch/:id/mute
Clear mute for a tracked item.

#### GET /noise-filter/actors
List actors in the deny-list (notifications from these actors are suppressed).
```json
// Response
{
  "deniedActors": [
    { "gitlabUserId": 123, "displayName": "CI Bot", "reason": "CI bot" }
  ]
}
```

#### POST /noise-filter/actors
Add an actor to the deny-list.
```json
// Request
{
  "gitlabUserId": 123,
  "displayName": "CI Bot",
  "reason": "CI bot"
}

// Response
{
  "success": true
}
```

#### DELETE /noise-filter/actors/:gitlabUserId
Remove an actor from the deny-list.
```json
// Response
{
  "success": true
}
```

#### POST /notifications/:id/received
Record notification delivery receipt.
```json
// Response
{
  "acknowledged": true,
  "receivedAt": "2026-01-07T12:00:05Z"
}
```

#### GET /notifications
List recent notifications (for in-app history + macOS dropdown).
Supports pagination and filtering by time.
```json
// Response
{
  "notifications": [
    {
      "id": "uuid",
      "sentAt": "2026-01-07T12:00:00Z",
      "source": "event",
      "sourceId": 12345,
      "itemType": "merge_request",
      "projectId": 42,
      "itemIid": 847,
      "title": "Maria commented on MR !847",
      "body": "Can we use the existing token validator here instead?",
      "webUrl": "https://gitlab.example.com/...#note_123"
    }
  ],
  "nextCursor": "opaque"
}
```

#### GET /health
Health check (no auth required).
```json
// Response
{
  "status": "healthy",
  "uptime": 3600,
  "lastPollAt": "2026-01-07T12:00:00Z"
}
```

#### GET /status
Diagnostics endpoint for trust verification (requires auth).
Surfaces last poll times, cursor positions, pending notifications, and recent delivery stats.
```json
// Response
{
  "pollState": {
    "lastTodosFetchedAt": "2026-01-08T10:00:00Z",
    "lastEventsFetchedAt": "2026-01-08T09:58:00Z",
    "lastSuccessfulPollAt": "2026-01-08T10:00:05Z"
  },
  "watchedItems": {
    "count": 5,
    "lastCursorUpdates": [
      { "itemIid": 847, "projectId": 42, "lastActivityFetchedAt": "2026-01-08T09:55:00Z" }
    ]
  },
  "notifications": {
    "pendingOutbox": 0,
    "sentLast24h": 12,
    "deliveredLast24h": 11,
    "deliveryRate": 0.917
  },
  "devices": {
    "active": 2,
    "platforms": ["ios", "macos"]
  }
}
```

#### GET /metrics
Prometheus metrics (no auth required).

#### POST /notifications/test
Enqueue a test notification through the normal outbox pipeline (requires auth).
Verifies end-to-end push delivery health. Goes through the same outbox → APNs flow as real notifications.
```json
// Request
{
  "deviceId": "optional-specific-device-or-all"  // Omit to send to all active devices
}

// Response
{
  "queued": true,
  "notificationId": "550e8400-e29b-41d4-a716-446655440099",
  "targetDevices": 2
}
```

---

### Clients: Xcode Multiplatform + Local Swift Package

**Project Structure:**
```
iosGitlabNotifier/
├── iosGitlabNotifier.xcworkspace
├── iOS/
│   ├── iosGitlabNotifierApp.swift
│   ├── AppDelegate.swift
│   └── Views/
├── iOS-NotificationExtension/           # Notification Service Extension
│   ├── NotificationService.swift        # Rich notification processing
│   └── Info.plist
├── macOS/
│   ├── iosGitlabNotifierApp.swift
│   ├── MenuBarManager.swift
│   └── Views/
├── Shared/                               # Shared SwiftUI views
│   ├── NotificationListView.swift        # Used by both iOS and macOS
│   ├── ItemRowView.swift
│   └── WatchToggleView.swift
└── GitLabNotifierKit/                    # Local Swift Package
    ├── Package.swift
    └── Sources/GitLabNotifierKit/
        ├── Models/
        │   ├── GitLabItem.swift
        │   ├── WatchedItem.swift
        │   └── NotificationPayload.swift
        ├── API/
        │   ├── GitLabClient.swift
        │   ├── GitLabEndpoints.swift
        │   ├── ServerAPIClient.swift
        │   └── AuthenticationManager.swift
        ├── Services/
        │   ├── WatchListManager.swift
        │   ├── DeviceManager.swift
        │   └── KeychainService.swift
        └── Utilities/
            ├── DateFormatting.swift
            └── DataExtensions.swift
```

**Architectural Decisions:**

| Category | Decision |
|----------|----------|
| Language | Swift 5.9+ |
| UI Framework | SwiftUI |
| Minimum iOS | iOS 17+ |
| Minimum macOS | macOS 14+ |
| Deployment Target | Latest stable OS at implementation time (keep aligned with current Xcode), with minimums above |
| Code Sharing | Local Swift Package (GitLabNotifierKit) |
| Credential Storage | Keychain Services via shared `KeychainService` |
| Local Persistence | SwiftData |
| Networking | URLSession (native) |
| State Management | @Observable + Environment |

**Platform-Specific Responsibilities:**

| Component | iOS | macOS | Shared Package |
|-----------|-----|-------|----------------|
| Push handling | AppDelegate | NSApplicationDelegate | - |
| UI chrome | TabView/NavigationStack | MenuBarExtra | - |
| GitLab API | - | - | GitLabClient |
| Server API | - | - | ServerAPIClient |
| Watch list logic | - | - | WatchListManager |
| Device registration | - | - | DeviceManager |
| Keychain access | - | - | KeychainService |
| Models | - | - | All models |

---

### Additional Infrastructure Decisions

**Deployment:**
- Server runs on home server / Raspberry Pi (always-on, self-hosted)
- Docker Compose includes Prometheus + Grafana for monitoring
- Requires static IP or dynamic DNS for client connectivity

**APNs Configuration:**
- Token-based authentication (.p8 key from Apple Developer Portal)
- Required environment variables:
  ```
  APNS_KEY_PATH=/path/to/AuthKey_XXXXXX.p8
  APNS_KEY_ID=XXXXXX
  APNS_TEAM_ID=YYYYYY
  APNS_BUNDLE_ID=com.yourname.iosGitlabNotifier
  APNS_ENVIRONMENT=development|production
  ```

**Watch List Sync:**
- Server-side storage (watch list in Prisma schema)
- Clients fetch on app launch and return to foreground
- No real-time push for watch list changes (sufficient for single-user)

---

### Optional Webhook Ingestion (Advanced)

If configured on the GitLab instance/projects:
- Server exposes POST /webhooks/gitlab with shared-secret validation
- Webhook events are normalized into the same NotificationPayload pipeline
- Polling remains enabled as a fallback + reconciliation path
- Near-instant notifications when webhooks fire (sub-second latency)
- Reduces GitLab API polling load and rate-limit risk

**Scheduled Tasks:**
```typescript
// NotificationReceipt cleanup - runs daily at 3:00 AM
async function cleanupOldReceipts() {
  const sevenDaysAgo = subDays(new Date(), 7);

  const result = await prisma.notificationReceipt.deleteMany({
    where: {
      sentAt: { lt: sevenDaysAgo }
    }
  });

  log.info({ deletedCount: result.count }, 'Cleaned up old notification receipts');
}
```

---

### Swift Package Service Details

**DeviceManager:**
```swift
class DeviceManager {
    func registerIfNeeded(token: Data) async throws {
        let tokenString = token.hexString
        let storedToken = KeychainService.getStoredDeviceToken()

        if tokenString != storedToken {
            try await serverAPIClient.registerDevice(token: tokenString)
            KeychainService.storeDeviceToken(tokenString)
        }
    }
}
```

**WatchListManager with offline queue:**
```swift
struct PendingWatchChange: Codable {
    let itemId: Int
    let itemType: String
    let projectId: Int
    let action: WatchAction  // .watch or .unwatch
    let timestamp: Date
}

class WatchListManager {
    private var pendingChanges: [PendingWatchChange] = []

    func syncPendingChanges() async {
        // Replay queue on network restore
        // Sort by timestamp, apply in order (last-write-wins)
    }

    func fetchWatchList() async throws {
        // Called on app launch and foreground
    }
}
```

---

### APNs Notification Payload

```json
{
  "aps": {
    "alert": {
      "title": "Maria commented on MR !847",
      "subtitle": "Auth refactor",
      "body": "Can we use the existing token validator here instead?"
    },
    "sound": "default",
    "badge": 1,
    "mutable-content": 1,
    "category": "GITLAB_ACTIVITY"
  },
  "data": {
    "notificationId": "550e8400-e29b-41d4-a716-446655440099",
    "source": "event",
    "sourceId": 12345,
    "actionType": "commented",
    "itemType": "merge_request",
    "itemId": 847,
    "projectId": 42,
    "projectPath": "team/backend",
    "webUrl": "https://gitlab.example.com/.../merge_requests/847#note_123",
    "actor": {
      "name": "Maria",
      "username": "maria",
      "avatarUrl": "https://gitlab.example.com/uploads/.../avatar.png"
    },
    "isNewMention": false,
    "batchCount": 1
  }
}
```

**Source-specific mappings:**

| Field | From `/todos` | From per-resource notes (PRIMARY) | From per-resource activity (SECONDARY) | From `/events` (LAST RESORT) |
|-------|---------------|-----------------------------------|----------------------------------------|------------------------------|
| `source` | `"todo"` | `"note"` | `"activity"` | `"event"` |
| `sourceId` | `todo.id` | `note.id` | `activity.id` (system note or state change) | `event.id` |
| `actionType` | `todo.action_name` | `"commented"` (notes are always comments) | Derived from state change (approved/merged/closed/pushed) | `event.action_name` |
| `itemType` | `todo.target_type` | From WatchedItem context | From WatchedItem context | `event.target_type` |
| `itemId` | `todo.target.iid` | From WatchedItem context | From WatchedItem context | `event.target_iid` |
| `projectId` | `todo.project.id` | From WatchedItem context | From WatchedItem context | `event.project_id` |
| `actor` | `todo.author` | `note.author` | `activity.author` / system | `event.author` |

**Action type normalization:**

| GitLab Action | Normalized `actionType` | Notification Title Template |
|---------------|-------------------------|----------------------------|
| `mentioned` | `mentioned` | "{actor} mentioned you on {itemType} {itemRef}" |
| `assigned` | `assigned` | "{actor} assigned you to {itemType} {itemRef}" |
| `review_requested` | `review_requested` | "{actor} requested your review on MR {itemRef}" |
| `commented` | `commented` | "{actor} commented on {itemType} {itemRef}" |
| `approved` | `approved` | "{actor} approved MR {itemRef}" |
| `merged` | `merged` | "{actor} merged MR {itemRef}" |
| `closed` | `closed` | "{actor} closed {itemType} {itemRef}" |
| `reopened` | `reopened` | "{actor} reopened {itemType} {itemRef}" |
| `pushed` | `pushed` | "{actor} pushed to MR {itemRef}" |
| _(unknown)_ | _(fallback)_ | "{actor} updated {itemType} {itemRef}" |

**Special notification types:**
- `type: "auth_expired"` - GitLab token invalid, prompt re-auth
- `batchCount > 1` - Batched notification ("3 new comments on MR !847")

**Notification categories (for action buttons):**
- `GITLAB_ACTIVITY` - Default, just "Open" action
- `GITLAB_NEW_MENTION` - Shows "Watch" and "Dismiss" buttons

---

### Error Handling Strategy

| Failure | Detection | Response | Metrics |
|---------|-----------|----------|---------|
| GitLab /todos 5xx/timeout | HTTP error | Log, skip todos this cycle, continue to per-resource polling | `gitlab_todos_failures_total++` |
| Per-resource notes 5xx/timeout | HTTP error | Log, mark item for retry, continue to next item | `gitlab_notes_failures_total++` |
| GitLab /events 5xx/timeout (fallback) | HTTP error | Log, skip events this cycle | `gitlab_events_failures_total++` |
| GitLab /todos 401 | HTTP 401 | Stop polling user. Send `auth_expired` push. | `gitlab_auth_failures_total++` |
| GitLab notes/events 401 | HTTP 401 | Stop polling user. Send `auth_expired` push. | `gitlab_auth_failures_total++` |
| APNs transient error | APNs response | Retry with exponential backoff (max 3 attempts) | `apns_retries_total++` |
| APNs BadDeviceToken | APNs response | Mark DeviceToken as `active = false` | `apns_invalid_tokens_total++` |
| 3+ consecutive failures (any endpoint) | Counter | Send alert push: "Notification service experiencing issues" | `poll_consecutive_failures` |
| Server restart | Process start | Resume from PollState/WatchedItemCursor timestamps. No duplicates due to NotifiedItem. | — |

**Graceful degradation:**
- If per-resource notes fails for an item → fall back to /events for that item (if enabled)
- If `/todos` fails → continue with per-resource polling (watched items still covered)
- If all sources fail → increment failure counter, retry next cycle
- **Note:** Per-resource polling is PRIMARY - individual item failures don't block others

**Error Message Catalogue:**

| Server/API Response | User-Facing Message |
|---------------------|---------------------|
| GitLab 401 Unauthorized | "GitLab session expired. Please reconnect your account." |
| GitLab 403 Forbidden | "Your token is missing required permissions. Ensure it has `read_api` scope." |
| GitLab 404 Not Found | "This item no longer exists or you don't have access to it." |
| GitLab 429 Rate Limited | "GitLab is rate limiting requests. Notifications may be delayed." |
| GitLab 5xx / Timeout | "GitLab is temporarily unavailable. We'll keep trying." |
| Server 401 (JWT expired) | "Session expired. Please sign in again." |
| Server 5xx | "Something went wrong on our end. Try again in a moment." |
| Network unreachable | "No internet connection. Changes will sync when you're back online." |

---

### macOS MenuBarExtra States

| State | Icon | Dropdown Content |
|-------|------|------------------|
| Connected, no notifications | Default icon | "No recent activity" empty state |
| Connected, has notifications | Icon + badge count | Notification list (most recent 20) |
| Offline | Dimmed icon | "Offline - showing cached data" banner + cached list |
| Auth expired | Warning icon | "GitLab connection expired" banner + "Reconnect" button |
| Server unreachable | Error icon | "Cannot reach notification server" + retry option |

---

### Polling Algorithm (Primary: per-resource watched items)

```
POLL CYCLE (adaptive per user, with jitter):

   ADAPTIVE INTERVAL LOGIC:
   ─────────────────────────────────────────────
   - Base interval: 60s
   - Scale with watched item count:
     • 1–20 items: 60s
     • 21–50 items: 90s
     • 51+ items: 120s
   - Backoff on 429 / repeated 5xx (exponential, capped at 10 minutes)
   - Allow /todos to run at higher cadence than deep watched-item scans if needed
   - Track effective interval via poll_interval_seconds{user_id} metric

1. FETCH TODOS (items needing your direct action)
   ─────────────────────────────────────────────
   GET /todos?state=pending&per_page=100 (paginate until exhausted)

   For each todo:
   - Check NotifiedItem for (userId, sourceType='todo', sourceId=todo.id)
   - If exists → SKIP (already notified)
   - If NOT exists AND todo.created_at > (PollState.lastTodosFetchedAt - 120s):
     → Add to pendingNotifications[]

   Cursor update rule:
   - Compute maxTodoCreatedAt from processed todos
   - Set PollState.lastTodosFetchedAt = maxTodoCreatedAt (not "now")
   - Set PollState.lastTodosFetchedId = max(todo.id) for tie-breaking when timestamps collide
   - Use overlap: when querying/processing, treat created_at > (lastTodosFetchedAt - 120s)
   - When timestamps match, use todo.id > lastTodosFetchedId as secondary filter

2. FETCH WATCHED ITEM NOTES (per-resource - PRIMARY for comments)
   ─────────────────────────────────────────────
   For each WatchedItem (with concurrency cap, e.g., 5 parallel):
     GET /projects/:projectId/merge_requests/:iid/notes?sort=asc&order_by=created_at&per_page=100&created_after=<cursor>
     GET /projects/:projectId/issues/:iid/notes?sort=asc&order_by=created_at&per_page=100&created_after=<cursor>

   For each note:
   - If note.author.id === user.gitlabUserId → SKIP (self-action)
   - Check NotifiedItem for (userId, sourceType='note', sourceId=note.id)
   - If exists → SKIP (already notified)
   - If NOT exists:
     → Add to pendingNotifications[]

   Cursor update rule (per WatchedItem):
   - Store lastActivityFetchedAt = max(note.created_at) in WatchedItemCursor
   - Store lastSeenNoteId = max(note.id) for tie-breaking
   - Only advance per-item cursor if fetch+process succeeded end-to-end for that item

3. FETCH WATCHED ITEM STATE CHANGES (per-resource - SECONDARY for approvals/merges)
   ─────────────────────────────────────────────
   For each WatchedItem (if capability available):
     For MRs: Check approval state, merge status, pipeline events via per-resource endpoints
     For Issues: Check state changes, label changes via system notes or resource events

   Sources (vary by GitLab version/config):
   - System notes (type: "system") filtered from notes endpoint
   - MR approvals: GET /projects/:projectId/merge_requests/:iid/approvals
   - MR state events: GET /projects/:projectId/merge_requests/:iid/resource_state_events
   - Issue state events: GET /projects/:projectId/issues/:iid/resource_state_events

   For each activity:
   - If activity.author === user.gitlabUserId → SKIP (self-action)
   - Check NotifiedItem for (userId, sourceType='activity', sourceId=activity.id)
   - If exists → SKIP (already notified)
   - If NOT exists:
     → Add to pendingNotifications[] with derived actionType

4. (LAST RESORT FALLBACK) FETCH EVENTS (global)
   ─────────────────────────────────────────────
   Only used if per-resource polling is disabled or degraded for specific items.
   GET /events?after=<lastEventsFetchedAt - overlap>&scope=all&per_page=100 (paginate)

   For each event:
   - If event.author.id === user.gitlabUserId → SKIP (self-action)
   - Check if event.project_id + event.target_type + event.target_iid
     matches any WatchedItem for this user
   - If NO match → SKIP (not watching this item)
   - Check NotifiedItem for (userId, sourceType='event', sourceId=event.id)
   - If exists → SKIP (already notified)
   - If NOT exists:
     → Add to pendingNotifications[]

   Cursor update rule:
   - Compute maxEventCreatedAt from processed events
   - Set PollState.lastEventsFetchedAt = maxEventCreatedAt (not "now")
   - Set PollState.lastEventsFetchedId = max(event.id) for tie-breaking when timestamps collide
   - When timestamps match, use event.id > lastEventsFetchedId as secondary filter

5. NORMALIZE & GROUP NOTIFICATIONS
   ─────────────────────────────────────────────
   For each pendingNotification:
   - Normalize to common NotificationPayload structure
   - Apply action type template (with fallback for unknown types)
   - Group by (itemType, itemIid, projectId) for batching
   - If group.count > 1 → create batched notification

6. WRITE NOTIFICATION OUTBOX (transactional - exactly-once guarantee)
   ─────────────────────────────────────────────
   In a SINGLE database transaction:
   For each notification:
   - Generate unique notificationId (UUID)
   - Insert NotificationOutbox(status="queued", payloadJson, nextAttemptAt=now())
   - Insert NotifiedItem(sourceType/sourceId) for dedupe
   - Update PollState cursors + WatchedItemCursor high-water marks
   - (Optional) Insert NotificationReceipt(sentAt=null) as "pending-delivery" record

   Set PollState.lastSuccessfulPollAt = now()

   Emit poll-phase metrics:
   - gitlab_todos_fetched_total{user_id}
   - gitlab_notes_fetched_total{user_id, item_type}
   - gitlab_activity_fetched_total{user_id} (state changes)
   - gitlab_events_fetched_total{user_id} (last resort fallback)
   - notifications_queued_total{user_id, source_type}
   - poll_duration_seconds{user_id, source}

7. DELIVER OUTBOX (async worker - separate process/loop)
   ─────────────────────────────────────────────
   Worker loop (runs independently of poll cycle):
   - Lease a batch of NotificationOutbox rows:
     WHERE status="queued" AND nextAttemptAt <= now()
     SET status="sending" (prevents concurrent delivery)
   - For each leased outbox entry:
     - Send to APNs for each active DeviceToken
     - Record NotificationAttempt per device (success/retryable_error/permanent_error)
     - If ALL devices succeeded → status="sent"
     - If retryable errors remain → status="queued", nextAttemptAt = exponential backoff
     - If retry budget exhausted (e.g., 5 attempts) → status="dead"
   - Update NotificationReceipt.sentAt when APNs accepts (delivery evidence)

   Emit delivery-phase metrics:
   - notifications_sent_total{user_id, source_type}
   - notifications_delivery_attempts_total{user_id, result}
   - notifications_dead_total{user_id} (exhausted retries)

8. METRICS & HEALTH
   ─────────────────────────────────────────────
   - Poll loop metrics reflect enqueue success (step 6)
   - Delivery metrics reflect APNs acceptance + receipt acknowledgements (step 7)
   - Outbox depth gauge for monitoring queue health
```

---

### Prometheus Metrics

```yaml
# Polling metrics (per source)
gitlab_todos_fetched_total{user_id}           # Total todos fetched
gitlab_notes_fetched_total{user_id}           # Total notes from per-resource polling (PRIMARY)
gitlab_activity_fetched_total{user_id}        # Total activity events from per-resource (SECONDARY)
gitlab_events_fetched_total{user_id}          # Total events fetched (LAST RESORT FALLBACK)
gitlab_events_matched_total{user_id}          # Events matching watched items
gitlab_poll_duration_seconds{user_id, source} # source: "todos" | "notes" | "activity" | "events"
poll_loop_heartbeat_timestamp_seconds{user_id} # Gauge, set every successful cycle start
watched_items_polled_total{user_id}           # Per-resource poll attempts

# Failure tracking
gitlab_todos_failures_total{user_id, error_type}
gitlab_notes_failures_total{user_id, error_type}     # Per-resource notes failures
gitlab_activity_failures_total{user_id, error_type}  # Per-resource activity failures
gitlab_events_failures_total{user_id, error_type}    # Fallback events failures
gitlab_auth_failures_total{user_id}
poll_consecutive_failures{user_id}            # Gauge, reset on success

# Notification metrics (outbox-based)
notifications_queued_total{user_id, source}   # source: "todo" | "note" | "activity" | "event"
notifications_sent_total{user_id, source}
notifications_batched_total{user_id}          # Batched (multiple activities)
notifications_dead_total{user_id}             # Exhausted retry budget
outbox_depth_gauge{user_id, status}           # Current outbox size by status (queued/sending/dead)
outbox_oldest_age_seconds{user_id}            # Age of oldest queued notification

# APNs metrics (delivery worker)
apns_sent_total{user_id}
apns_received_total{user_id}                  # From receipt pings
apns_delivery_rate{user_id}                   # Gauge: received/sent
apns_retries_total{user_id}
apns_invalid_tokens_total{user_id}
delivery_attempt_duration_seconds{user_id}    # Time to complete APNs send

# System health
poll_cycle_duration_seconds{user_id}          # Full poll cycle time
poll_interval_seconds{user_id}                # Effective interval chosen by adaptive scheduler
delivery_cycle_duration_seconds{user_id}      # Full delivery worker cycle time
active_users_gauge                            # Users being polled
watched_items_gauge{user_id}                  # Items per user
```

**Alerting rules:**

```yaml
# Alert if delivery rate drops
- alert: APNsDeliveryDegraded
  expr: apns_delivery_rate < 0.95
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "APNs delivery rate below 95%"

# Alert if either endpoint consistently failing
- alert: GitLabPollingFailing
  expr: poll_consecutive_failures > 3
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "GitLab polling failing for user {{ $labels.user_id }}"

# Alert if poll loop stalls (no heartbeat)
- alert: PollLoopStalled
  expr: time() - poll_loop_heartbeat_timestamp_seconds > 180
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Poll loop stalled (no heartbeat) for user {{ $labels.user_id }}"

# Alert if outbox is backing up (delivery worker unhealthy)
- alert: OutboxBacklog
  expr: outbox_depth_gauge{status="queued"} > 50
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Notification outbox backing up for user {{ $labels.user_id }}"

# Alert if notifications stuck too long
- alert: OutboxStale
  expr: outbox_oldest_age_seconds > 300
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Notifications stuck in outbox > 5 minutes for user {{ $labels.user_id }}"
```

**SQLite Ops:**
- Enable WAL mode for concurrent reads during writes
- Nightly backup of notifier.db (copy while using WAL-safe procedure)
- Ensure system time sync (NTP) to reduce cursor/ordering risk

---

### Polling Strategy Rationale

**Primary: Per-resource polling** (implemented)
- Polls notes/discussions directly on each watched Issue/MR
- Scales with number of watched items, not instance activity
- Smaller payloads, precise cursors, lower blast radius from API oddities
- Uses `WatchedItemCursor` for per-item high-water marks

**Fallback: Global `/events`** (available but not primary)
- Used only if per-resource endpoints fail or are unavailable
- Scans all user activity and filters against watch list
- Higher volume but provides backup coverage

**Why per-resource is primary:**
- Reliability: avoids global event stream ordering/pagination edge cases
- Performance: fetches only data for items you care about
- Mental model: "we poll exactly the things you're watching"

---

### Mock APNs Endpoint (Testing)

```typescript
// routes/mock-apns.ts (test environment only)
fastify.post('/mock-apns', async (request, reply) => {
  const payload = request.body;

  // Log for test verification
  fastify.log.info({ apnsPayload: payload }, 'Mock APNs received');

  // Store for assertion
  mockApnsStore.push({
    payload,
    receivedAt: new Date().toISOString()
  });

  return { success: true };
});
```

---

### Test Cases: /events Filtering

| Test Case | Input | Expected Result |
|-----------|-------|-----------------|
| Event matches watched MR | event.project_id=42, target_type=MergeRequest, target_iid=123; WatchedItem exists | → Notification queued |
| Event matches watched Issue | event.project_id=42, target_type=Issue, target_iid=456; WatchedItem exists | → Notification queued |
| Event on unwatched item (same project) | event.project_id=42, target_iid=999; No WatchedItem | → Skip, no notification |
| Event on unwatched project | event.project_id=99; No WatchedItem in project | → Skip, no notification |
| Self-action on watched item | event.author.id === user.gitlabUserId | → Skip, no notification |
| Already notified event | NotifiedItem exists for (event, sourceType) | → Skip, deduplicated |
| Unknown action type | event.action_name = "cherry_picked" | → Notification with fallback title |
| Batch: multiple events same item | 3 events for MR !847 in same poll | → 1 batched notification |

---

### Implementation Sequence

1. **Server initialization** - Clone DriftOS starter, apply SQLite modifications, add mock APNs route
2. **Swift Package setup** - Create GitLabNotifierKit with purpose-built models and API clients
3. **iOS app** - Create with SwiftData, @Observable, NavigationStack, and optimistic updates
4. **macOS menu bar app** - MenuBarExtra with shared views
5. **APNs integration** - With notification receipt flow for delivery verification
6. **Prometheus metrics** - Full observability with alerting rules

---

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Platform versions: iOS 17+ / macOS 14+ minimum (enables @Observable)
- State management: @Observable + Environment pattern
- Local persistence: SwiftData
- Server validation: Zod schemas in Fastify
- Polling strategy: Dual `/todos` + `/events` for complete activity coverage

**Important Decisions (Shape Architecture):**
- Navigation: NavigationStack (iOS)
- Database migrations: Prisma Migrate
- Rate limiting: Per-IP basic limiting
- Error UX: Inline + Toast hybrid with user-friendly messages
- Loading UX: Optimistic updates with documented revert pattern
- Observability: Notification receipt flow for delivery verification

**Deferred Decisions (Post-MVP):**
- CI/CD automation (manual builds initially)
- Fastlane match for code signing
- CloudKit sync for SwiftData

### Data Architecture

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Server Database | SQLite via Prisma | Simple, file-based, sufficient for single-user |
| Migrations | Prisma Migrate | Auto-generated from schema, version controlled |
| Validation | Server-side only (Zod) | Single source of truth, simpler client code |
| Client Persistence | SwiftData | Modern, works with @Observable, future CloudKit option |
| Client Models | Purpose-built | Only include what UI needs, decoupled from server internals |
| Caching | SwiftData local cache | GitLab items cached locally, refreshed on foreground |

#### Server-to-Client Model Mapping

| Server Entity (Prisma) | Client Model (SwiftData) | Notes |
|------------------------|--------------------------|-------|
| User | — | Not stored on client; auth token in Keychain |
| DeviceToken | — | Managed by DeviceManager, not persisted locally |
| WatchedItem | `WatchedItem` | itemIid, itemType, projectId, title, projectPath |
| NotifiedItem | — | Server-only deduplication |
| PollState | — | Server-only |
| — | `CachedGitLabItem` | Client-only cache of items from /items endpoint |
| — | `PendingWatchChange` | Client-only offline queue |

### Authentication & Security

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Server Auth | JWT tokens (7-day expiry) | Standard, stateless, refresh flow built in |
| GitLab PAT Storage | AES-256-GCM encryption | Application-layer encryption before DB storage |
| Client Credentials | Keychain Services | Platform secure storage via KeychainService |
| API Security | TLS required + per-IP rate limiting | Defense in depth for self-hosted server |
| Rate Limiting | 100 req/min per IP | Prevents accidental floods, uses Fastify rate-limit plugin |

### API & Communication Patterns

| Decision | Choice | Rationale |
|----------|--------|-----------|
| API Style | REST | Simple, well-understood, sufficient for this use case |
| Documentation | Inline code comments | Personal project, architecture.md has contract |
| Error Format | Standardized JSON errors | `{ error: string, code?: string, details?: object }` |
| Error Copy | User-friendly with hints | See Error Message Catalogue above |
| Polling Strategy | Dual: `/todos` + `/events` | Complete activity coverage for watched items |

### Client Architecture (iOS/macOS)

| Decision | Choice | Version/Notes |
|----------|--------|---------------|
| Minimum iOS | iOS 17+ | Required for @Observable |
| Minimum macOS | macOS 14+ | Required for @Observable |
| Deployment Target | Latest stable at implementation | Keep aligned with current Xcode |
| State Management | @Observable + Environment | Clean, minimal boilerplate |
| Navigation (iOS) | NavigationStack | Type-safe, simple app structure |
| Navigation (macOS) | MenuBarExtra | Native menu bar pattern |
| Local Persistence | SwiftData | Modern, integrates with @Observable |
| Networking | URLSession | Native, no dependencies |
| Error Presentation | Inline + Toast hybrid | Validation inline, network errors as toasts |
| Loading UX | Optimistic updates | See pattern below |
| Code Signing | Xcode automatic (MVP) | Revisit fastlane match if painful |

#### Optimistic Update Pattern

```swift
/// Standard pattern for optimistic UI updates with revert on failure
func toggleWatch(item: CachedGitLabItem) async {
    // 1. Optimistic update
    let previousState = item.isWatched
    item.isWatched.toggle()

    // 2. Attempt server sync with timeout
    do {
        try await withTimeout(seconds: 5) {
            try await api.setWatch(item.itemIid, item.projectId, item.isWatched)
        }
    } catch {
        // 3. Revert on failure
        item.isWatched = previousState

        // 4. Show user-friendly error
        ToastManager.shared.show(
            message: "Couldn't update watch status. Try again.",
            style: .error
        )
    }
}
```

**Apply this pattern to:**
- Watch/unwatch toggle
- Dismiss notification action
- Manual sync trigger

### Testing Architecture

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Server Test Runner | Vitest | Included in DriftOS starter |
| GitLab API Mocking | HTTP-level (nock/msw) | Intercept real requests, test actual flow |
| APNs Testing | Simple Fastify mock route | `/mock-apns` endpoint logs payloads |
| SwiftData Testing | inMemoryOnly stores | Fast, isolated, no disk cleanup |
| Test Commands | `npm test` (server), `xcodebuild test` (clients) | Documented for manual runs |

### Observability & Monitoring

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Metrics | Prometheus | Verify reliability NFR (> 99% delivery) |
| Dashboards | Grafana | Visualize polling success, delivery rates |
| Logging | Pino (structured JSON) | Built into DriftOS starter |
| Delivery Verification | Notification Service Extension + open-acks | Track best-effort device receipt + user opens |

#### Notification Receipt Flow

```
1. Server sends push with unique notification_id in payload:
   { "data": { "notificationId": "uuid-here", ... } }

2. When notification is received, Notification Service Extension pings server (best-effort):
   POST /notifications/:id/received { "type": "delivered" }

3. When user opens notification, app also pings server (optional, separate metric):
   POST /notifications/:id/received { "type": "opened" }

4. Server tracks in Prometheus:
   - apns_sent_total (counter)
   - apns_delivered_total (counter, NSE-based)
   - apns_opened_total (counter)
   - apns_delivery_rate (gauge: delivered/sent)

5. Alert if delivery_rate < 95% over 1 hour (only if NSE-delivered coverage is stable)
```

### Infrastructure & Deployment

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Server Hosting | Home server / Raspberry Pi (primary) | Always-on, self-hosted, Docker Compose |
| Alternative Hosting | fly.io / DigitalOcean App Platform | Cloud options for users without home server |
| Monitoring | Prometheus + Grafana | Verify reliability NFR |
| Logging | Pino (structured JSON) | Built into DriftOS starter |
| CI/CD | Manual builds initially | Add automation later if needed |
| APNs Auth | Token-based (.p8 key) | Modern, recommended by Apple |

**Deployment Options:**
1. **Self-hosted (Home Server/RPi):** Docker Compose with SQLite volume mount
   - Prefer private connectivity (VPN/overlay like Tailscale/WireGuard) over exposing a public endpoint
   - If public: terminate TLS with a reverse proxy (Caddy/nginx) and automated certificates (Let's Encrypt)
   - Add container restart policies (`restart: unless-stopped`) + external watchdog (health check → restart)
2. **fly.io:** Single-instance deployment, attach persistent volume for SQLite, set FLY_ALLOC_ID secret for sticky routing
3. **DigitalOcean App Platform:** Dockerfile deploy, attach block storage for SQLite persistence

**Connectivity & Security:**
- **Preferred:** Private networking via VPN overlay (Tailscale, WireGuard) - avoids exposing server publicly
- **Alternative:** Dynamic DNS + TLS termination with automated certificate renewal
- Never expose SQLite database port or internal services directly

**Backup Strategy (Required for Single Point of Failure Mitigation):**
- Enable WAL mode for concurrent reads during writes
- Daily automated backup of `notifier.db` using WAL-safe procedure:
  ```bash
  sqlite3 notifier.db "VACUUM INTO 'backup/notifier-$(date +%Y%m%d).db'"
  ```
- Retain 7 days of backups minimum
- Periodic restore test (monthly) to verify backup integrity
- Consider off-site backup replication for disaster recovery

**Health & Readiness Endpoints:**
```
GET /health         → { "status": "ok", "version": "1.0.0", "uptime": 12345 }
GET /health/ready   → { "ready": true, "db": "ok", "apns": "ok" }  # Checks DB connectivity + APNs provider init
```

**Smoke Test Endpoint (dev/staging only):**
```
POST /debug/test-notification
// Sends a test push to verify APNs pipeline end-to-end
// Request: { "deviceId": "..." }
// Response: { "sent": true, "apnsId": "..." }
```

**Alerting Rules (Prometheus):**
```yaml
# Critical: Outbox backlog growing
- alert: NotificationOutboxBacklog
  expr: notification_outbox_queued > 100
  for: 5m
  labels: { severity: critical }

# Warning: Poll failures
- alert: GitLabPollFailures
  expr: rate(gitlab_poll_errors_total[5m]) > 0.1
  for: 10m
  labels: { severity: warning }

# Warning: APNs delivery failures
- alert: ApnsDeliveryFailures
  expr: rate(apns_delivery_failures_total[5m]) / rate(apns_delivery_total[5m]) > 0.05
  for: 15m
  labels: { severity: warning }
```

### Component Completion Checklist

When implementing any architectural component, verify:

- [ ] Server endpoint exists with Zod validation schema
- [ ] HTTP-level tests (nock/msw) cover happy path + error cases
- [ ] Client integration added to GitLabNotifierKit
- [ ] SwiftData model created (if persistence needed)
- [ ] Purpose-built model mapping documented
- [ ] Error handling returns user-friendly messages from catalogue
- [ ] Optimistic update pattern applied (if applicable)
- [ ] Loading/error states render correctly (inline or toast)
- [ ] Prometheus metrics emitted for observability
- [ ] Self-review against Component Completion Checklist
- [ ] Code builds successfully on all target platforms

### Architecture Coherence Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** HIGH

**Key Strengths:**
- Per-resource polling (primary) + /todos + /events (fallback) ensures notification reliability
- Clean separation via GitLabNotifierKit enables code reuse
- 30 standardized patterns prevent AI agent conflicts
- Observability built-in from day one
- Clear Definition of Done for consistent quality

**Areas for Future Enhancement:**
- Advanced notification threading
- Automated full-flow testing (GitLab → device)
- CloudKit sync for SwiftData

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed (Low-Medium)
- [x] Technical constraints identified
- [x] Cross-cutting concerns mapped

**✅ Architectural Decisions**
- [x] Critical decisions documented with versions
- [x] Technology stack fully specified
- [x] Integration patterns defined
- [x] Performance considerations addressed

**✅ Implementation Patterns**
- [x] Naming conventions established (30 patterns)
- [x] Structure patterns defined
- [x] Communication patterns specified
- [x] Process patterns documented

**✅ Project Structure**
- [x] Complete directory structure defined
- [x] Component boundaries established
- [x] Integration points mapped
- [x] Requirements to structure mapping complete

**✅ Test Architecture**
- [x] E2E notification flow integration test defined
- [x] Swift test factory pattern documented
- [x] Definition of Done established

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** HIGH

### Implementation Handoff

**For AI Agents:**
This architecture document is your complete guide for implementing iosGitlabNotifier. Follow all decisions, patterns, and structures exactly as documented.

**First Implementation Priority:**
```bash
git clone https://github.com/DriftOS/fastify-starter.git server
cd server && npm install
# Apply SQLite modifications per architecture.md
```

**Development Sequence:**
1. Initialize project using documented starter template
2. Set up development environment per architecture
3. Implement core architectural foundations
4. Build features following established patterns
5. Maintain consistency with documented rules

### Quality Assurance Checklist

**✅ Architecture Coherence**
- [x] All decisions work together without conflicts
- [x] Technology choices are compatible
- [x] Patterns support the architectural decisions
- [x] Structure aligns with all choices

**✅ Requirements Coverage**
- [x] All functional requirements are supported
- [x] All non-functional requirements are addressed
- [x] Cross-cutting concerns are handled
- [x] Integration points are defined

**✅ Implementation Readiness**
- [x] Decisions are specific and actionable
- [x] Patterns prevent agent conflicts
- [x] Structure is complete and unambiguous
- [x] Examples are provided for clarity

### Project Success Factors

**🎯 Clear Decision Framework**
Every technology choice was made collaboratively with clear rationale, ensuring all stakeholders understand the architectural direction.

**🔧 Consistency Guarantee**
Implementation patterns and rules ensure that multiple AI agents will produce compatible, consistent code that works together seamlessly.

**📚 Complete Coverage**
All project requirements are architecturally supported, with clear mapping from business needs to technical implementation.

**🏗️ Solid Foundation**
The chosen starter template and architectural patterns provide a production-ready foundation following current best practices.

---

**Architecture Status:** READY FOR IMPLEMENTATION ✅

**Next Phase:** Begin implementation using the architectural decisions and patterns documented herein.

**Document Maintenance:** Update this architecture when major technical decisions are made during implementation.
