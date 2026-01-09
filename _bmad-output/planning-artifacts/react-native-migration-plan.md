# PWA + ntfy Migration Plan

**Date:** 2026-01-09
**Status:** FINAL - Ready for Execution
**Previous Plan:** React Native Migration (superseded)

---

## Executive Summary

This document defines the migration from native Swift/SwiftUI to **PWA + ntfy** for the iosGitlabNotifier project.

### Why This Plan Supersedes React Native Migration

The original React Native plan was blocked by:
1. **No macOS access** - Cannot run iOS Simulator or Expo Go
2. **No Apple Developer account** - Cannot use APNs for push notifications, cannot distribute iOS app

**Solution:** PWA (Progressive Web App) for watch list management + ntfy.sh for push notifications.

### Final Technology Decisions

| Decision | Choice |
|----------|--------|
| **Mobile App** | PWA (React + Vite) |
| **Push Notifications** | ntfy.sh (free, open source) |
| **Project Structure** | Monorepo (shared TypeScript) |
| **State Management** | Zustand + React Query |
| **UI Framework** | React + Tailwind CSS |
| **Bundler** | Vite |
| **macOS App** | Not applicable (browser works on desktop) |
| **Apple Developer** | **NOT REQUIRED** |

**Result:** Full-stack TypeScript. NO native code. NO Apple dependencies.

### Key Benefits
1. **Linux-only development** - All development and testing on Linux VPS
2. **No Apple costs** - No $99/year developer account
3. **Code sharing with server** - TypeScript types, Zod schemas via monorepo
4. **Single language** - TypeScript across entire stack (server + web)
5. **Desktop included** - PWA works on any device with a browser
6. **Push notifications work** - Via ntfy iOS app (free from App Store)

### Trade-offs Accepted
- No native iOS app feel (PWA in Safari)
- Less control over notification UI (ntfy handles display)
- Two apps on phone (ntfy + PWA) instead of one
- No rich notification actions (Watch/Dismiss buttons)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER'S iPhone                             │
│                                                                  │
│  ┌────────────────────────┐    ┌────────────────────────┐       │
│  │      PWA (Safari)       │    │     ntfy App           │       │
│  │  ┌──────────────────┐  │    │  (App Store - free)     │       │
│  │  │ Watch List Mgmt  │  │    │                         │       │
│  │  │ Item Browser     │  │    │  ← Push Notifications   │       │
│  │  │ Notification Log │  │    │                         │       │
│  │  │ Settings         │  │    │  Tap → Opens Safari     │       │
│  │  └──────────────────┘  │    │        to PWA           │       │
│  └────────────────────────┘    └────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
                │                           ▲
                │ API calls (HTTPS)         │ ntfy push
                ▼                           │
┌─────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION SERVER (Fastify)                 │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │
│  │   Static   │  │  REST API  │  │  GitLab    │  │   ntfy     │ │
│  │   Files    │  │  Endpoints │  │  Poller    │  │   Sender   │ │
│  │   (PWA)    │  │            │  │            │  │            │ │
│  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │
│                          │              │              │         │
│                          └──────┬───────┴──────────────┘         │
│                                 ▼                                │
│                          ┌────────────┐                          │
│                          │   SQLite   │                          │
│                          │  (Prisma)  │                          │
│                          └────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
        │                                           │
        │ GitLab API                                │ HTTPS POST
        ▼                                           ▼
┌───────────────────┐                    ┌───────────────────┐
│ GitLab Instance   │                    │    ntfy.sh        │
│ (self-hosted)     │                    │  (or self-hosted) │
└───────────────────┘                    └───────────────────┘
```

---

## Monorepo Structure

```
iosGitlabNotifier/
├── packages/
│   ├── server/                  # Fastify notification server
│   │   ├── src/
│   │   │   ├── routes/          # API endpoints
│   │   │   ├── services/        # GitLab poller, ntfy sender
│   │   │   ├── jobs/            # Background polling jobs
│   │   │   └── index.ts
│   │   ├── prisma/
│   │   │   └── schema.prisma
│   │   └── package.json
│   │
│   ├── web/                     # React PWA
│   │   ├── src/
│   │   │   ├── components/      # React components
│   │   │   ├── pages/           # Page components (routes)
│   │   │   ├── hooks/           # Custom React hooks
│   │   │   ├── stores/          # Zustand stores
│   │   │   ├── api/             # React Query hooks + API calls
│   │   │   └── main.tsx
│   │   ├── public/
│   │   │   └── manifest.json    # PWA manifest
│   │   ├── index.html
│   │   └── package.json
│   │
│   └── shared/                  # Shared TypeScript code
│       ├── src/
│       │   ├── types/           # API types, models
│       │   ├── validation/      # Zod schemas
│       │   └── constants.ts
│       └── package.json
│
├── package.json                 # Workspace root (pnpm)
├── pnpm-workspace.yaml
└── turbo.json                   # Build orchestration (optional)
```

---

## ntfy Integration

### How ntfy Works

1. **Topic-based:** Each user gets a unique topic (e.g., `gitlab-notifier-{userId}`)
2. **Simple API:** Server sends HTTP POST to `https://ntfy.sh/{topic}`
3. **iOS app:** User installs ntfy from App Store, subscribes to their topic
4. **Push delivery:** ntfy.sh handles APNs delivery (they have Apple Developer account)

### Server-Side ntfy Configuration

```typescript
// services/ntfy-sender.ts

interface NtfyPayload {
  topic: string;
  title: string;
  message: string;
  priority?: 1 | 2 | 3 | 4 | 5;  // 1=min, 3=default, 5=urgent
  tags?: string[];               // Emoji tags like "gitlab", "comment"
  click?: string;                // URL to open when notification tapped
  actions?: NtfyAction[];        // Button actions (limited on iOS)
}

async function sendNotification(userId: string, notification: NotificationPayload) {
  const user = await prisma.user.findUnique({ where: { id: userId } });

  const payload: NtfyPayload = {
    topic: user.ntfyTopic,  // e.g., "gitlab-notifier-abc123"
    title: notification.title,
    message: notification.body,
    priority: notification.importance === 'urgent' ? 5 : 3,
    tags: ['gitlab', notification.actionType],
    click: notification.webUrl,  // Deep link to GitLab
  };

  await fetch(`https://ntfy.sh/${payload.topic}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
}
```

### Prisma Schema Updates

```prisma
model User {
  id                 String   @id @default(uuid())
  gitlabUrl          String
  gitlabUserId       Int
  gitlabPatEncrypted String
  ntfyTopic          String   @unique  // NEW: unique topic per user
  ntfyServer         String   @default("https://ntfy.sh")  // Allow self-hosted
  createdAt          DateTime @default(now())
  // ... rest unchanged
}
```

### User Setup Flow

1. User opens PWA, enters GitLab credentials
2. Server generates unique ntfy topic: `gitlab-notifier-{uuid}`
3. PWA shows instructions: "Install ntfy app, subscribe to topic: `{topic}`"
4. User installs ntfy from App Store, adds topic subscription
5. Test notification sent to verify setup
6. Setup complete - notifications flow through ntfy

---

## PWA Implementation

### PWA Manifest

```json
// packages/web/public/manifest.json
{
  "name": "GitLab Notifier",
  "short_name": "GL Notifier",
  "description": "Watch list management for GitLab notifications",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1f2937",
  "theme_color": "#f97316",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### Key PWA Features

| Feature | Implementation |
|---------|---------------|
| Offline indicator | React Query offline detection + UI banner |
| Add to Home Screen | PWA manifest + Safari prompt |
| Full-screen mode | `display: standalone` in manifest |
| App icon | Custom icons in manifest |
| Splash screen | Theme color + icons |

### Pages / Routes

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | `HomePage` | Main items list with watch toggles |
| `/setup` | `SetupPage` | GitLab connection + ntfy topic setup |
| `/watch` | `WatchListPage` | Current watch list management |
| `/history` | `HistoryPage` | Recent notification history |
| `/settings` | `SettingsPage` | Quiet hours, noise filter, diagnostics |
| `/test` | `TestPage` | Test notification trigger |

---

## Beads Migration Summary

### CLOSE (No longer applicable)

**All native iOS/macOS beads - replace with web equivalents:**

| Epic | Reason |
|------|--------|
| Epic 5: Swift Package (`iosGitlabNotifier-tcn`) | Replaced by `packages/shared` |
| Epic 6: iOS App (`iosGitlabNotifier-bnk`) | Replaced by `packages/web` |
| Epic 7: macOS App (`iosGitlabNotifier-03u`) | Browser works on desktop |
| All APNs-related tasks | Replaced by ntfy |

**Total to close:** ~38+ beads

### KEEP (Server-side, unaffected)

| Epic | Notes |
|------|-------|
| Epic 1: Server Foundation (`iosGitlabNotifier-14j`) | No changes |
| Epic 2: GitLab Polling Service (`iosGitlabNotifier-rsy`) | No changes |
| Epic 4: Server API Endpoints (`iosGitlabNotifier-6ab`) | Minor updates for ntfy |
| Epic 8: Monitoring & Observability (`iosGitlabNotifier-ulx`) | Update metrics for ntfy |

### MODIFY

| Epic | Changes |
|------|---------|
| Epic 3: APNs Integration (`iosGitlabNotifier-hfu`) | **Rename to ntfy Integration**, rewrite tasks |
| Epic 9: Integration Testing (`iosGitlabNotifier-vuo`) | Update for web testing |

---

## New Epic Structure

### Epic: PWA Foundation
**ID:** `pwa-foundation`
**Description:** Set up React web app, monorepo structure, and shared packages

| Task | Title | Priority |
|------|-------|----------|
| pwa-foundation.1 | Initialize Vite + React + TypeScript project | P0 |
| pwa-foundation.2 | Configure pnpm workspaces monorepo | P0 |
| pwa-foundation.3 | Set up shared types package | P0 |
| pwa-foundation.4 | Set up shared validation (Zod) package | P0 |
| pwa-foundation.5 | Configure Tailwind CSS | P0 |
| pwa-foundation.6 | Configure PWA manifest | P0 |
| pwa-foundation.7 | Configure ESLint and Prettier | P0 |
| pwa-foundation.8 | Set up Vitest testing infrastructure | P0 |

### Epic: PWA Core Services
**ID:** `pwa-services`
**Description:** Implement core services and utilities for the PWA

| Task | Title | Priority |
|------|-------|----------|
| pwa-services.1 | Implement Zustand auth store | P0 |
| pwa-services.2 | Implement React Query API client | P0 |
| pwa-services.3 | Implement watch list store | P0 |
| pwa-services.4 | Implement offline detection hook | P0 |
| pwa-services.5 | Implement error message catalogue | P0 |
| pwa-services.6 | Implement local storage persistence | P1 |

### Epic: PWA Views
**ID:** `pwa-views`
**Description:** Implement PWA pages and components

| Task | Title | Priority |
|------|-------|----------|
| pwa-views.1 | Implement routing with React Router | P0 |
| pwa-views.2 | Implement SetupPage (GitLab + ntfy) | P0 |
| pwa-views.3 | Implement HomePage (item list) | P0 |
| pwa-views.4 | Implement WatchListPage | P0 |
| pwa-views.5 | Implement HistoryPage | P0 |
| pwa-views.6 | Implement SettingsPage | P0 |
| pwa-views.7 | Implement quiet hours config UI | P0 |
| pwa-views.8 | Implement noise filter UI | P1 |
| pwa-views.9 | Implement test notification UI | P0 |
| pwa-views.10 | Implement offline mode banner | P1 |
| pwa-views.11 | Implement item filtering | P1 |

### Epic: ntfy Integration
**ID:** `ntfy-integration`
**Description:** Implement server-side ntfy notification delivery

| Task | Title | Priority |
|------|-------|----------|
| ntfy-integration.1 | Add ntfy fields to Prisma schema | P0 |
| ntfy-integration.2 | Implement ntfy sender service | P0 |
| ntfy-integration.3 | Implement topic generation | P0 |
| ntfy-integration.4 | Update polling to use ntfy sender | P0 |
| ntfy-integration.5 | Implement test notification endpoint | P0 |
| ntfy-integration.6 | Add ntfy delivery metrics | P1 |
| ntfy-integration.7 | Support self-hosted ntfy server option | P2 |

### Epic: PWA Testing
**ID:** `pwa-testing`
**Description:** Implement comprehensive test coverage

| Task | Title | Priority |
|------|-------|----------|
| pwa-testing.1 | Implement Vitest unit tests for shared package | P1 |
| pwa-testing.2 | Implement Vitest unit tests for stores | P1 |
| pwa-testing.3 | Implement Vitest unit tests for hooks | P1 |
| pwa-testing.4 | Implement component tests with Testing Library | P1 |
| pwa-testing.5 | Implement E2E tests with Playwright | P2 |

---

## Development Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                        Linux VPS (Development)                   │
├─────────────────────────────────────────────────────────────────┤
│  ✓ Write TypeScript/React code                                   │
│  ✓ Run Vite dev server (npm run dev)                            │
│  ✓ Run Vitest unit tests (npm test)                             │
│  ✓ Run TypeScript type checking (npm run typecheck)             │
│  ✓ Run ESLint (npm run lint)                                    │
│  ✓ Run Playwright E2E tests (npm run test:e2e)                  │
│  ✓ Build production bundle (npm run build)                      │
│  ✓ Deploy to server (same machine or remote)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Your iPhone (Testing)                         │
├─────────────────────────────────────────────────────────────────┤
│  • Open Safari, navigate to https://your-server/                │
│  • Add to Home Screen for PWA experience                        │
│  • Install ntfy app from App Store                              │
│  • Subscribe to your notification topic                         │
│  • Receive push notifications via ntfy                          │
└─────────────────────────────────────────────────────────────────┘
```

**No Mac required at any point.** Everything runs on Linux.

---

## Key Dependencies

```json
// packages/web/package.json
{
  "dependencies": {
    "react": "^18.x",
    "react-dom": "^18.x",
    "react-router-dom": "^6.x",
    "zustand": "^5.x",
    "@tanstack/react-query": "^5.x",
    "tailwindcss": "^4.x"
  },
  "devDependencies": {
    "vite": "^6.x",
    "vitest": "^3.x",
    "@testing-library/react": "^16.x",
    "@playwright/test": "^1.x",
    "typescript": "^5.x"
  }
}

// packages/shared/package.json
{
  "dependencies": {
    "zod": "^3.x"
  }
}

// packages/server/package.json (additions)
{
  "dependencies": {
    // ... existing deps
    "@fastify/static": "^8.x"  // Serve PWA static files
  }
}
```

---

## PRD Updates Needed

| Section | Update |
|---------|--------|
| Executive Summary | Remove "macOS menu bar", add "PWA + ntfy" |
| MVP Features | Remove #8 (Dual Platform), update for PWA |
| User Journeys 5-6 | REMOVE (macOS specific) |
| Platform Requirements | PWA requirements instead of iOS 17+ |
| Push Notification Strategy | ntfy instead of APNs |
| Mobile App Specific Requirements | Rename to "PWA Requirements" |
| FR37-FR41 (macOS Menu Bar) | REMOVE |
| NFR5-6 (Keychain) | Server-side JWT only |

---

## Architecture Document Updates Needed

| Section | Update |
|---------|--------|
| Requirements Overview | Update component list (PWA instead of iOS/macOS apps) |
| Starter Template | Add Vite + React template info |
| Client Architecture | Complete rewrite for PWA |
| Project Structure | Monorepo with `packages/web` |
| APNs Configuration | REMOVE, replace with ntfy config |
| APNs Payload | REMOVE, replace with ntfy payload format |
| Swift Package | REMOVE entirely |
| macOS MenuBarExtra | REMOVE entirely |
| Notification Receipt Flow | Simplified (ntfy handles delivery) |
| Testing | Vitest + Playwright instead of XCTest |
| Component patterns | React/TypeScript patterns |

---

## Execution Checklist

- [ ] Close all Swift/iOS/macOS beads (~38+ beads)
- [ ] Close Epic 3: APNs Integration (will be recreated as ntfy)
- [ ] Create new epic: `pwa-foundation`
- [ ] Create new epic: `pwa-services`
- [ ] Create new epic: `pwa-views`
- [ ] Create new epic: `ntfy-integration`
- [ ] Create new epic: `pwa-testing`
- [ ] Update Epic 8: Monitoring for ntfy metrics
- [ ] Update Epic 9: Integration Testing for web
- [ ] Update PRD document
- [ ] Update Architecture document
- [ ] Update CLAUDE.md with PWA development workflow

---

## User Setup Instructions (for end result)

### One-time Setup

1. **Install ntfy app** on your iPhone (free from App Store)
2. **Open the PWA** at `https://your-server.com`
3. **Connect GitLab** - Enter your GitLab URL and Personal Access Token
4. **Subscribe to notifications** - ntfy app will show your unique topic
5. **Add to Home Screen** - Safari menu → "Add to Home Screen"
6. **Test notifications** - Use the test button in Settings

### Daily Usage

- **View items:** Open PWA from home screen
- **Watch/unwatch:** Toggle items in the list
- **Receive notifications:** They arrive via ntfy app
- **Tap notification:** Opens Safari to GitLab item

---

**Document Status:** FINAL - Ready for bead migration and document updates

**Next Steps:**
1. Execute bead closures for Swift/iOS/macOS tasks
2. Create new PWA + ntfy epics in beads
3. Update PRD and Architecture documents
4. Begin implementation
