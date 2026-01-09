---
stepsCompleted: [1, 2, 3, 4, 7, 8, 9, 10, 11]
inputDocuments:
  - product-brief-iosGitlabNotifier-2026-01-06.md
documentCounts:
  briefCount: 1
  researchCount: 0
  brainstormingCount: 0
  projectDocsCount: 0
workflowType: 'prd'
lastStep: 11
skippedSteps: [5, 6]
status: complete
completedAt: '2026-01-07'
---

# Product Requirements Document - iosGitlabNotifier

**Author:** Ubuntu
**Date:** 2026-01-07

## Executive Summary

iosGitlabNotifier is a lightweight Progressive Web App (PWA) paired with ntfy push notifications that delivers alerts for GitLab activity from self-hosted instances within 1-2 minutes of events.

### The Problem

GitLab users on self-hosted instances have no reliable way to receive timely mobile notifications. Comments on merge requests, issue updates, and review requests go unseen for hours. Email notifications get buried in inbox noise, third-party apps are outdated or don't support self-hosted instances well, and there's no official GitLab mobile app for notifications.

### Target User

A developer working on a team using self-hosted GitLab as the central hub - someone who needs to protect focus while staying informed. The app enables triage at a glance without costly context-switching.

### What Makes This Special

- **Focused simplicity:** Does one thing well - no bloat, no in-app workflows
- **Self-hosted support:** Works with private GitLab instances, not just gitlab.com
- **Fast polling:** 1-2 minute notification latency vs. hours with email
- **No Apple Developer account required:** Uses ntfy.sh for push notifications, avoiding $99/year cost
- **Cross-platform by design:** PWA works on any device with a browser; ntfy app handles push on iOS
- **Personal utility:** Built to solve a real workflow problem, not to be a product

## Project Classification

**Technical Type:** web_app (PWA)
**Domain:** general (developer productivity)
**Complexity:** low
**Project Context:** Greenfield - new project

This is a straightforward notification system focused on reliability and speed. A PWA provides watch list management while ntfy.sh handles push delivery. No high-complexity domain concerns like regulatory compliance or safety-critical systems apply.

## Success Criteria

### User Success

**Primary Indicator:** Reduced anxiety about missing time-sensitive GitLab activity

**Observable Behaviors:**
- No longer compulsively checking email/GitLab "just in case"
- Trusts that if something needs attention, the app will surface it
- Can focus on deep work without background worry about missed comments
- Elimination of the "5 hours late discovery" frustration - no more finding out someone replied or asked something hours ago

**Success Moment:** When the user realizes they haven't manually checked GitLab in hours because they trust the app to notify them.

### Business Success

N/A - This is a personal utility, not a commercial product. Success is measured entirely by whether it solves the user's workflow problem.

### Technical Success

| Metric | Target | Failure Threshold |
|--------|--------|-------------------|
| Notification latency | < 2 minutes from GitLab activity | > 5 minutes is degraded experience |
| Notification reliability | > 99% of watched items | Frequent missed notifications undermine trust |
| Battery/resource impact | Negligible | Noticeable drain makes it not worth using |

### Measurable Outcomes

1. **Reliability (Critical):** Near-zero missed notifications on watched items (> 99%)
2. **Latency:** 95%+ of notifications delivered within 2 minutes
3. **Usability:** App stays connected and functional without manual intervention

## Product Scope

### MVP - Minimum Viable Product

1. **GitLab Authentication** - Connect to self-hosted GitLab via personal access token (OAuth as Phase 1.5)
2. **Item List View** - Display issues and MRs the user is involved in (assigned, authored, tagged)
3. **Watch/Favorite Toggle** - Mark specific items to receive notifications for (optionally mirrored to GitLab subscription state)
4. **Mute/Snooze** - Temporarily suppress notifications for a tracked item without untracking it
5. **Push Notifications via ntfy** - Full context (who, what, which item) delivered within 1-2 minutes via ntfy.sh
6. **Deep Linking** - Tapping a notification opens the item directly in GitLab (browser)
7. **PWA Experience** - Installable web app with home screen icon, offline indicator, and mobile-optimized UI

### Growth Features (Post-MVP)

- **Smart Focus Detection:** Automatically suggest items to watch based on recent commit activity, review assignments, or thread participation
- **Noise Filtering:** Rules to suppress low-priority notifications (e.g., bot comments, CI status on non-focus branches)
- **Webhook Mode (Optional):** If user can configure project webhooks, server ingests events directly for near-instant notifications and reduced polling load (fallback to polling remains)

### Vision (Future)

- **Working Hours / DND:** Schedule when notifications are active vs. silenced
- **Quick Replies:** Respond to comments directly from the notification without opening GitLab

## User Journeys

### Journey 1: First-Time Setup - "Finally, Something That Might Actually Work"

It's Monday morning and Alex has just spent 15 minutes catching up on a thread in an MR where a teammate asked a blocking question on Friday at 4pm. The email notification got buried under weekend noise. Again. Alex mutters "there has to be a better way" and decides to finally set up iosGitlabNotifier.

Opening the app, Alex taps "Connect to GitLab" and pastes in the self-hosted instance URL. The app prompts for a personal access token - Alex generates one from GitLab settings with the right scopes (read_api, read_user) and pastes it in. The connection succeeds immediately.

The app automatically populates a list: 3 open MRs Alex authored, 7 issues assigned to Alex. Each has a toggle to Watch. Alex enables notifications for the 2 MRs currently in active review and 3 issues that are time-sensitive this sprint. The whole setup took 4 minutes.

That afternoon, Alex's phone buzzes. A teammate commented on the MR. The notification shows exactly who said what on which MR. Alex glances, sees it's a quick approval comment, and returns to the current task without breaking flow. The anxiety of "what am I missing?" is already quieter.

### Journey 2: Daily Notification Triage - "Flow State Protected"

It's Wednesday, 10:30am. Alex is deep in a complex refactoring task - the kind that requires holding a mental model of several interconnected components. The phone buzzes on the desk.

Without switching context to a browser, Alex glances at the notification: "**@maria** commented on **MR !847 - Auth refactor**: 'Can we use the existing token validator here instead?'" Alex immediately knows: this is the MR currently under review, Maria is the senior dev whose feedback matters, and the question is substantive but not blocking. Alex mentally notes "respond after this function is done" and returns to the code. Total interruption: 3 seconds.

Twenty minutes later, another buzz. "**@jordan** mentioned you in **Issue #234 - Login timeout bug**: 'Alex, can you confirm this is related to the session changes?'" This one needs a quick response - Jordan is blocked. Alex taps the notification, Safari opens directly to the issue, and Alex types a two-line response confirming yes, it's related, here's the commit hash. Done in 90 seconds.

At lunch, Alex realizes something new: there's been no compulsive email-checking all morning. No background anxiety about missing something. Just focused work with confident triage when needed.

### Journey 3: Focus Shift - "Priorities Change, Watch List Follows"

Thursday afternoon. A production incident pulls Alex onto a different project entirely. The sprint's MRs are now on hold - a critical hotfix in a separate repo needs immediate attention.

Alex opens the app, sees the current watch list, and quickly adjusts: unwatches the 2 paused MRs (no need for noise right now), opens the app's item list filtered to the hotfix repo, and watches the new incident issue plus the emergency MR just created. Three taps, done.

For the next 4 hours, Alex receives notifications only about the incident - teammate updates, CI status on the fix, the approval from the on-call lead. The paused work stays silent. When the hotfix is deployed Friday morning, Alex reverses the process: unwatches the resolved incident, re-enables the original MRs, and picks up exactly where things left off.

### Journey 4: Edge Case - "Wait, Did I Miss Something?"

Friday morning. Alex is reviewing the week's MR activity and notices something odd - a teammate merged an MR that Alex was supposed to review, but Alex never saw the review request.

Alex opens iosGitlabNotifier and checks: the MR isn't in the watch list. Right - it was a new MR where Alex was tagged as reviewer, not one Alex authored or was assigned to. The "New Tag Quick Action" notification came through on Tuesday, but Alex was in a meeting and dismissed it without watching.

This reveals a gap: Alex needs to be more intentional about the Watch/Dismiss decision on new tags. Alex adjusts the mental model: when a "you were tagged" notification appears, default to Watch unless it's clearly not relevant. The app did its job - it surfaced the tag. The miss was in the triage decision.

Alex also checks the item list, filters to "MRs where I'm reviewer," and watches two other pending reviews that slipped through. Lesson learned, gap closed.

### Journey Requirements Summary

These journeys reveal the following capability requirements:

**PWA (Web App):**
- GitLab authentication (PAT or OAuth)
- Item list with filtering (by repo, by involvement type)
- Watch/unwatch toggle per item
- Notification history view
- Deep linking to GitLab in browser
- Mobile-optimized responsive UI
- Installable as home screen app

**ntfy Push Notifications:**
- Rich notification content (who, what, which item)
- Deep links to GitLab when tapped
- Works on iOS via ntfy app from App Store

**Server:**
- GitLab polling and activity detection
- Watch list storage and management
- ntfy notification delivery

## PWA and Notification Requirements

### Project-Type Overview

iosGitlabNotifier is a Progressive Web App (PWA) paired with ntfy.sh for push notifications. The architecture includes a server component for GitLab polling and notification delivery, with the PWA providing watch list management and notification history.

### Platform Requirements

**PWA (Progressive Web App):**
- React + TypeScript + Vite
- Mobile-first responsive design with Tailwind CSS
- Installable via "Add to Home Screen" on iOS Safari
- Works on any modern browser (iOS Safari, Chrome, Firefox, etc.)

**ntfy App (for push notifications):**
- Free iOS app from App Store
- User subscribes to their unique notification topic
- Handles push notification display and deep linking

**Server:**
- Fastify + TypeScript + Prisma + SQLite
- Hosts PWA static files
- REST API for watch list management
- GitLab polling and ntfy notification delivery

### Push Notification Strategy

**Architecture:** Server-side polling with ntfy.sh delivery

**Server Component (Node.js/TypeScript):**
- Polls GitLab API every 1-2 minutes for watched items
- Detects new activity (comments, mentions, status changes)
- Sends push notifications via ntfy.sh HTTP API
- Stores watch list configuration

**Why ntfy.sh:**
- No Apple Developer account required ($0/year vs $99/year)
- Simple HTTP API for sending notifications
- Free tier sufficient for personal use
- iOS app available in App Store (they handle APNs)
- Can self-host ntfy server if desired

**ntfy Configuration:**
- Each user gets unique topic (e.g., `gitlab-notifier-{uuid}`)
- Notifications include title, message, priority, and click URL
- Tapping notification opens GitLab item in browser

### Offline Mode

**Behavior when offline:**
- Display cached watch list and last-known item states
- Show clear "offline" indicator banner
- Queue watch/unwatch actions for sync when connection returns

**Data caching:**
- React Query cache for API data (persisted to localStorage)
- Last-fetched item metadata (title, status, last activity)
- Notification history fetched from server

### Data Storage

**Server-side (canonical):**
- Watch list stored on notification server (single source of truth)
- All GitLab credentials stored server-side (encrypted)
- PWA only stores JWT token for API authentication

**Client-side (PWA):**
- JWT token in localStorage
- React Query cache for offline viewing
- No sensitive credentials stored in browser

### Distribution

**PWA Distribution:**
- No app store required
- Users access via URL and "Add to Home Screen"
- Updates deployed instantly (no review process)
- Works on any device with a modern browser

**ntfy App:**
- User installs from iOS App Store (free)
- No developer account needed (ntfy handles distribution)

### Implementation Considerations

**Monorepo Project Structure:**
```
iosGitlabNotifier/
├── packages/
│   ├── server/                  # Fastify notification server
│   │   ├── src/
│   │   │   ├── routes/          # API endpoints
│   │   │   ├── services/        # GitLab poller, ntfy sender
│   │   │   └── jobs/            # Background polling
│   │   ├── prisma/
│   │   └── package.json
│   │
│   ├── web/                     # React PWA
│   │   ├── src/
│   │   │   ├── components/      # React components
│   │   │   ├── pages/           # Route pages
│   │   │   ├── stores/          # Zustand stores
│   │   │   └── api/             # React Query hooks
│   │   ├── public/
│   │   │   └── manifest.json    # PWA manifest
│   │   └── package.json
│   │
│   └── shared/                  # Shared TypeScript
│       ├── src/
│       │   ├── types/           # API types
│       │   └── validation/      # Zod schemas
│       └── package.json
│
├── package.json                 # Workspace root
└── pnpm-workspace.yaml
```

**Development workflow:**
- All development on Linux (no Mac required)
- Run Vite dev server for hot reloading
- Run Vitest for unit tests
- Run Playwright for E2E tests
- Deploy to any server that can run Node.js

## Scoping Validation

### MVP Philosophy

**Approach:** Problem-Solving MVP
This project takes the leanest viable approach: solve the core notification reliability problem without feature expansion. Success is binary - either notifications arrive reliably within 2 minutes, or the app fails its purpose.

### Scope Boundaries

The MVP scope (7 features) directly maps to the 4 user journeys with no unnecessary additions. Each feature exists because a journey requires it:

| MVP Feature | Enables Journey |
|-------------|-----------------|
| GitLab Authentication | All journeys (prerequisite) |
| Item List View | Journeys 1, 3, 4 |
| Watch/Favorite Toggle | Journeys 1, 3 |
| ntfy Push Notifications | Journey 2 |
| Deep Linking | Journey 2 |
| PWA Experience | All journeys |

### Risk Mitigation

**Technical:** Server-side polling is proven. ntfy.sh is a mature, open-source notification service. PWA distribution eliminates all app store complexity.

**Scope Creep:** Post-MVP features (smart focus, noise filtering) are explicitly deferred. Working hours and quick replies are vision-only.

## Functional Requirements

### GitLab Connection

- FR1: User can connect to a self-hosted GitLab instance by providing the instance URL
- FR2: User can authenticate using a personal access token (PAT)
- FR3: (Phase 1.5) User can authenticate using OAuth (alternative to PAT)
- FR4: System validates connection and displays success/failure status
- FR5: User can disconnect from the current GitLab instance
- FR6: User can re-authenticate when credentials expire or are revoked

### Item Discovery

- FR7: User can view a list of issues assigned to them
- FR8: User can view a list of MRs they authored
- FR9: User can view a list of items where they are tagged/mentioned
- FR10: User can filter the item list by repository
- FR11: User can filter the item list by involvement type (assigned, authored, reviewer, mentioned)
- FR12: User can see item metadata including title, status, and last activity timestamp
- FR13: User can manually refresh the item list

### Watch List Management

- FR14: User can add an item to their watch list (toggle to "watched")
- FR15: User can remove an item from their watch list (toggle to "unwatched")
- FR16: User can view their current watch list
- FR17: Watch list stored on server and accessible from any device via PWA
- FR44: User can mute a tracked item until a chosen time (e.g., 30m, 2h, end of day)

### Auto-Watch Behavior

- FR48: When user is newly tagged in an issue/MR, item is auto-added to watch list with status "auto-watched"
- FR49: User can reverse a previous Dismiss action and add the item to watch list within the app (via notification history or deep link)

### GitLab Notification Alignment

- FR42: User can import GitLab subscription/notification-enabled items as tracked items (best-effort)
- FR43: User can choose whether tracked items should mirror GitLab subscription state (on/off per account)

### Device & Server Registration

- FR18: User's device registers with the notification server to receive push notifications
- FR19: User can view and manage their registered devices

### Push Notifications

- FR20: User receives push notification when activity occurs on a watched item
- FR21: Notification displays who performed the action (actor name/handle)
- FR22: Notification displays what action occurred (comment, mention, status change)
- FR23: Notification displays which item the activity is on (title and identifier)
- FR24: User receives notification when tagged in a new issue or MR they don't currently watch

### Notification Actions

- FR25: User can tap "Watch" quick action on new-tag notifications to add item to watch list
- FR26: User can tap "Dismiss" quick action on new-tag notifications to ignore the item
- FR27: User can tap notification to open the item directly in GitLab (browser)

### Notification History

- FR28: User can view notification history for watched items

### Notification Scheduling

- FR46: User can configure working hours / quiet hours per account (server-enforced)
- FR47: User can optionally allow urgent events (mentions/review requests) to bypass quiet hours

### Noise Control

- FR50: User can mute notifications from specific actors (e.g., bots) via deny-list

### System Status & Feedback

- FR29: User can see connection/sync status (connected, syncing, offline, error)
- FR30: User receives clear feedback when an operation fails
- FR31: Error messages include details to help user resolve the issue

### Sync & Offline Behavior

- FR32: User can manually trigger a sync
- FR33: User can view cached watch list when offline
- FR34: User can view last-known item states when offline
- FR35: System displays clear offline indicator when not connected
- FR36: System queues watch/unwatch changes made offline for sync when connection returns

### Diagnostics & Trust

- FR45: User can view polling/sync status details in app (last poll times, delivery stats) to verify the system is working
- FR51: User can trigger a test notification from settings to verify end-to-end push delivery

### Architecture Notes

> **Polling Scope:** Server polls:
> - `/todos` for items needing user attention (mentions, review requests, assignments), including FR24 cases
> - **Primary for watched items:** per-resource polling (notes/discussions) per watched Issue/MR
> - **Secondary for watched items:** per-resource "activity" endpoint(s) for non-comment state changes (e.g., approvals/merges)
> - **Last resort fallback:** global `/events?after=...` only when per-resource sources are unavailable or degraded
> This provides precise, scalable coverage without scanning all project activity.
>
> **Sync Conflict Resolution:** Last-write-wins (timestamp) - simple, predictable, sufficient for single-user.

## Non-Functional Requirements

### Performance

| Metric | Target | Failure Threshold |
|--------|--------|-------------------|
| Notification latency | < 2 minutes from GitLab activity | > 5 minutes |
| Server polling interval | 1-2 minutes | > 3 minutes degrades experience |

### Reliability

- NFR1: Near-zero missed notifications on watched items during working hours (> 99% reliability)
- NFR2: System maintains notification delivery during typical work schedule (no 24/7 uptime requirement)
- NFR3: Graceful degradation when GitLab API is unavailable (queue and retry)
- NFR4: Offline changes sync reliably when connection returns

### Security

- NFR5: GitLab credentials (PAT/OAuth tokens) stored encrypted on server (AES-256 or equivalent)
- NFR6: Credentials never logged or transmitted in plain text
- NFR7: Server-to-device communication secured via TLS
- NFR8: Device tokens and watch list data protected at rest on server

### Integration

- NFR9: GitLab API integration handles rate limiting gracefully (backoff, retry)
- NFR10: ntfy.sh integration handles delivery errors gracefully (retry with backoff, fallback to cached topic)
- NFR11: System detects and reports GitLab API authentication failures (credential expiry)

### Resource Efficiency

- NFR12: PWA minimizes client-side resource usage (server handles all polling)
- NFR13: Minimize background network activity (server handles polling, not client)
- NFR14: Local storage footprint appropriate for notification history scope

### Infrastructure & Deployment

- NFR15: Server supports multiple deployment options (self-hosted Docker, fly.io, DigitalOcean) for user flexibility
- NFR16: Health check and readiness endpoints enable container orchestration and uptime monitoring
- NFR17: Prometheus metrics and alerting rules enable proactive reliability monitoring
