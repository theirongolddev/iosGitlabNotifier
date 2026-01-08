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

iosGitlabNotifier is a minimal iOS app (with macOS menu bar companion) that delivers push notifications for GitLab activity from self-hosted instances within 1-2 minutes of events.

### The Problem

GitLab users on self-hosted instances have no reliable way to receive timely mobile notifications. Comments on merge requests, issue updates, and review requests go unseen for hours. Email notifications get buried in inbox noise, third-party apps are outdated or don't support self-hosted instances well, and there's no official GitLab mobile app for notifications.

### Target User

A developer working on a team using self-hosted GitLab as the central hub - someone who needs to protect focus while staying informed. The app enables triage at a glance without costly context-switching.

### What Makes This Special

- **Focused simplicity:** Does one thing well - no bloat, no in-app workflows
- **Self-hosted support:** Works with private GitLab instances, not just gitlab.com
- **Fast polling:** 1-2 minute notification latency vs. hours with email
- **Personal utility:** Built to solve a real workflow problem, not to be a product

## Project Classification

**Technical Type:** mobile_app
**Domain:** general (developer productivity)
**Complexity:** low
**Project Context:** Greenfield - new project

This is a straightforward mobile notification app focused on reliability and speed. No high-complexity domain concerns like regulatory compliance or safety-critical systems apply.

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
5. **Rich Push Notifications** - Full context (who, what, which item) delivered within 1-2 minutes
6. **New Tag Quick Action** - When tagged in a new issue/MR, notification includes quick action to Watch or Dismiss
7. **Deep Linking** - Tapping a notification opens the item directly in GitLab (browser)
8. **Dual Platform Support** - iOS app + macOS menu bar app with equivalent functionality

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

### Journey 5: macOS Menu Bar - "Desktop Flow, Same Trust"

Monday morning, Alex is at the desk with the MacBook - phone is charging across the room. The menu bar shows the iosGitlabNotifier icon, a small GitLab logo that stays unobtrusive until needed.

A macOS notification slides in from the top right: "**@chen** commented on **Issue #445 - API rate limiting**: 'Ready for your review when you have a moment.'" Alex sees it in peripheral vision, notes it's not urgent, and keeps typing. The notification disappears after a few seconds but the menu bar icon shows a subtle badge: 1 unread.

Later, during a compile wait, Alex clicks the menu bar icon. A compact dropdown shows recent notifications with the same info density as iOS: who, what, which item. Alex clicks the Issue #445 entry, browser opens to the exact issue, review happens.

The experience is identical to mobile - same watch list (synced), same notification content, same deep links. Just adapted to the desktop context where Alex might be actively coding rather than glancing at a phone.

### Journey 6: macOS Menu Bar - "Heads-Down Coding Session"

Wednesday afternoon. Alex has blocked 3 hours for deep work on a complex feature. macOS Focus mode is on, but iosGitlabNotifier is in the allowed apps list - it's the one interruption source Alex trusts.

During the session, three notifications arrive. The menu bar icon badge increments: 1... 2... 3. Alex doesn't look - the lack of sound/vibration (configured for this Focus mode) means it's not urgent. The badge is just ambient awareness.

At the 2-hour mark, Alex takes a stretch break and clicks the menu bar icon. Quick scan: two comments on a watched MR (approval + minor suggestion), one new mention on an issue. Alex processes all three in 4 minutes: acknowledges the approval, notes the suggestion for later, watches the new issue.

Back to coding. No anxiety about what accumulated. No email inbox to wade through. Just a clean queue that was handled in one batch.

### Journey Requirements Summary

These journeys reveal the following capability requirements:

**iOS App:**
- GitLab authentication (PAT or OAuth)
- Item list with filtering (by repo, by involvement type)
- Watch/unwatch toggle per item
- Rich push notifications (who, what, which item)
- New-tag quick action (Watch or Dismiss)
- Deep linking to GitLab in browser

**macOS Menu Bar App:**
- Same authentication and watch list (synced with iOS)
- Menu bar icon with unread badge
- Compact dropdown showing recent notifications
- macOS native notifications
- Deep linking to GitLab in browser
- Focus mode compatibility

**Cross-Platform:**
- Shared watch list state between iOS and macOS
- Consistent notification content format
- Unified item list and filtering

## Mobile App Specific Requirements

### Project-Type Overview

iosGitlabNotifier is a native Swift/SwiftUI application targeting iOS and macOS platforms. The architecture includes a lightweight server component for reliable push notification delivery, with client apps focused on watch list management and notification display.

### Platform Requirements

**iOS App:**
- SwiftUI-based interface
- Minimum iOS version: iOS 17+ (aligns with @Observable-based architecture)
- iPhone and iPad support (universal app)

**macOS Menu Bar App:**
- SwiftUI with MenuBarExtra
- Minimum macOS version: macOS 14+ (aligns with @Observable-based architecture)
- Menu bar utility with dropdown interface
- Native macOS notifications

**Code Sharing:**
- Shared Swift package for core logic (GitLab API client, data models, watch list management)
- Platform-specific UI layers
- Single Xcode workspace with iOS and macOS targets

### Push Notification Strategy

**Architecture:** Server-side polling with APNs delivery

**Server Component (Node.js/TypeScript):**
- Polls GitLab API every 1-2 minutes for watched items
- Detects new activity (comments, mentions, status changes)
- Sends push notifications via Apple Push Notification service (APNs)
- Stores device tokens and watch list configuration

**Why server-side:**
- iOS Background App Refresh is throttled and unreliable (15+ min delays)
- Server polling guarantees consistent 1-2 minute latency
- Single polling source reduces GitLab API load vs. multiple clients

**APNs Configuration:**
- Rich notifications with full context (who, what, which item)
- Notification actions for Watch/Dismiss on new tags
- Silent pushes for watch list sync (optional)

### Offline Mode

**Behavior when offline:**
- Display cached watch list and last-known item states
- Show clear "offline" indicator
- Queue watch/unwatch actions for sync when connection returns
- No background polling attempts while offline

**Data caching:**
- Local persistence of watch list (Core Data or SwiftData)
- Last-fetched item metadata (title, status, last activity)
- Notification history for recent items

### Device Permissions

**Required:**
- Push notifications (critical for core functionality)
- Network access (GitLab API communication)

**Not required for MVP:**
- Background App Refresh (server handles polling)
- Location, camera, microphone, etc.

### Cross-Platform Sync

**Watch list synchronization between iOS and macOS:**

**Canonical approach (required):** Server-side storage
- Watch list stored on notification server (single source of truth for polling + notifications)
- Clients sync on app launch, foreground, and after local mutations
- More control, works without iCloud

**Optional optimization (non-authoritative):** iCloud Key-Value Store
- May cache "last UI state" for faster perceived sync between Apple devices
- Must never override server truth without reconciliation

### Store Compliance

**Initial approach:** Ad-hoc distribution (personal devices only)
- Free Apple ID sufficient for development
- Limited to registered device UDIDs
- No App Store review process

**Future path:** TestFlight → App Store (if desired)
- Requires Apple Developer Program ($99/year)
- TestFlight allows up to 10,000 testers
- App Store enables wider distribution

### Implementation Considerations

**Xcode Project Structure:**
```
iosGitlabNotifier/
├── Shared/                           # Swift package
│   ├── Models/                      # Data models
│   ├── GitLabAPI/                   # API client
│   └── WatchListManager/            # Core logic
├── iOS/                              # iOS app target
│   └── Views/                       # SwiftUI views
├── iOS-NotificationExtension/        # Notification Service Extension
│   └── NotificationService.swift    # Rich notification processing
├── macOS/                            # macOS app target
│   └── MenuBar/                     # Menu bar UI
└── Server/                           # Node.js server
    ├── src/
    │   ├── gitlab-poller.ts
    │   └── apns-sender.ts
    └── package.json
```

**Development workflow:**
- Swift code editable in any editor (VS Code with Swift extension)
- Build and test via `xcodebuild` CLI when possible
- Xcode required for: signing, device deployment, debugging
- Server component fully CLI-based (no Xcode dependency)

## Scoping Validation

### MVP Philosophy

**Approach:** Problem-Solving MVP
This project takes the leanest viable approach: solve the core notification reliability problem without feature expansion. Success is binary - either notifications arrive reliably within 2 minutes, or the app fails its purpose.

### Scope Boundaries

The MVP scope (7 features) directly maps to the 6 user journeys with no unnecessary additions. Each feature exists because a journey requires it:

| MVP Feature | Enables Journey |
|-------------|-----------------|
| GitLab Authentication | All journeys (prerequisite) |
| Item List View | Journeys 1, 3, 4 |
| Watch/Favorite Toggle | Journeys 1, 3 |
| Rich Push Notifications | Journeys 2, 5, 6 |
| New Tag Quick Action | Journey 4 |
| Deep Linking | Journeys 2, 5 |
| Dual Platform | Journeys 5, 6 |

### Risk Mitigation

**Technical:** Server-side polling with APNs is proven. Ad-hoc distribution eliminates App Store review complexity initially.

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
- FR17: Watch list synchronizes across user's iOS and macOS devices
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

### macOS Menu Bar (Platform-Specific)

- FR37: Menu bar displays app icon that remains unobtrusive when no notifications
- FR38: Menu bar icon displays unread notification badge count
- FR39: User can click menu bar icon to view compact dropdown of recent notifications and history
- FR40: User can click a notification entry in dropdown to open item in browser
- FR41: App integrates with macOS Focus mode (respects notification settings)

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

- NFR5: GitLab credentials (PAT/OAuth tokens) stored in platform Keychain (iOS Keychain / macOS Keychain)
- NFR6: Credentials never logged or transmitted in plain text
- NFR7: Server-to-device communication secured via TLS
- NFR8: Device tokens and watch list data protected at rest on server

### Integration

- NFR9: GitLab API integration handles rate limiting gracefully (backoff, retry)
- NFR10: APNs integration follows Apple best practices for token refresh and error handling
- NFR11: System detects and reports GitLab API authentication failures (credential expiry)

### Resource Efficiency

- NFR12: Follow iOS/macOS battery optimization best practices
- NFR13: Minimize background network activity (server handles polling, not client)
- NFR14: Local storage footprint appropriate for notification history scope

### Infrastructure & Deployment

- NFR15: Server supports multiple deployment options (self-hosted Docker, fly.io, DigitalOcean) for user flexibility
- NFR16: Health check and readiness endpoints enable container orchestration and uptime monitoring
- NFR17: Prometheus metrics and alerting rules enable proactive reliability monitoring
