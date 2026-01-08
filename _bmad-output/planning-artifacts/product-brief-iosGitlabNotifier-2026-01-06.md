---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments: []
date: 2026-01-06
author: Ubuntu
---

# Product Brief: iosGitlabNotifier

## Executive Summary

iosGitlabNotifier is a personal iOS utility that delivers timely push notifications for GitLab activity. It solves the problem of missing time-sensitive feedback on merge requests and issues due to unreliable email notifications. The app connects to a self-hosted GitLab instance, polls for user activity, and pushes notifications within 1-2 minutes - enabling quick triage without leaving the current context.

---

## Core Vision

### Problem Statement

GitLab users on self-hosted instances have no reliable way to receive timely mobile notifications for activity that requires their attention. Comments on merge requests, issue updates, and review requests can go unseen for hours until the user happens to check email.

### Problem Impact

- Delayed responses to teammates waiting on feedback
- Blocked merge requests and stalled code review cycles
- Mental overhead of constantly checking email/GitLab manually
- Missed time-sensitive discussions and decisions

### Why Existing Solutions Fall Short

- **No official GitLab mobile app** exists for notifications
- **Third-party apps** are outdated, poorly rated, or don't support self-hosted instances well
- **Email notifications** get buried in inbox noise and can't be configured to persist or alert distinctly
- **Existing tools** are bloated with features when all that's needed is reliable notification delivery

### Proposed Solution

A minimal iOS app that:
- Authenticates with a self-hosted GitLab instance
- Polls for activity relevant to the user (MR comments, issue updates, mentions, etc.)
- Delivers push notifications within 1-2 minutes of activity
- Shows enough context to triage on the spot (what happened, where)

### Key Differentiators

- **Focused simplicity:** Does one thing well - no bloat, no in-app workflows
- **Self-hosted support:** Works with private GitLab instances, not just gitlab.com
- **Fast polling:** 1-2 minute notification latency vs. hours with email
- **Personal utility:** Built to solve a real workflow problem, not to be a product

---

## Target Users

### Primary User

**Profile:** Developer on a team using self-hosted GitLab as the central hub for code, issues, and all work communication (not Slack/Teams). Typically juggling 2-3 active MRs while managing issues across requirements gathering and design feedback.

**Context:** Works at their desk during working hours. Needs a notification mechanism that doesn't require manually checking email or GitLab every few minutes to stay responsive.

**Key Consideration:** User has AuDHD and needs to protect focus while staying informed. Unnecessary context-switching is costly - notifications should help triage ("respond now" vs "this can wait") without creating cognitive overload.

**What They're Watching:**
- Issues assigned to them
- MRs they've opened
- Manually added items as focus shifts

**Success Criteria:** Knows within 1-2 minutes when something needs their attention on primary focus items, can triage importance at a glance, and doesn't have to interrupt flow to manually check for updates.

### Secondary Users

N/A - This is a personal utility, not a team product.

### User Journey

1. **Setup:** Connect to self-hosted GitLab instance, authenticate
2. **Configure:** Default watch list populates (assigned issues + authored MRs), adjust as needed
3. **Daily Use:** Notifications arrive on phone (or Mac) within 1-2 minutes of activity on watched items
4. **Triage:** Glance at notification, decide if it needs immediate response or can wait
5. **Iterate:** Add/remove items from watch list as focus changes throughout the day

### Platform Consideration

iOS is the primary target (user's device), though macOS support could serve the same purpose and may be explored as an alternative or complement.

---

## Success Metrics

### User Success

**Primary Success Indicator:** Reduced anxiety about missing time-sensitive GitLab activity

**Observable Behaviors:**
- No longer compulsively checking email/GitLab "just in case"
- Trusts that if something needs attention, the app will surface it
- Can focus on deep work without background worry about missed comments

### Technical Success Criteria

| Metric | Target | Failure Threshold |
|--------|--------|-------------------|
| Notification latency | < 2 minutes from GitLab activity | > 5 minutes is degraded experience |
| Notification reliability | > 99% of watched items | Frequent missed notifications undermine trust |
| Battery/resource impact | Negligible | Noticeable drain makes it not worth using |

### Key Performance Indicators

1. **Reliability (Critical):** Near-zero missed notifications on watched items (> 99%)
2. **Latency:** 95%+ of notifications delivered within 2 minutes
3. **Usability:** App stays connected and functional without manual intervention

### Business Objectives

N/A - This is a personal utility, not a commercial product. Success is measured entirely by whether it solves the user's workflow problem.

---

## MVP Scope

### Core Features

1. **GitLab Authentication** - Connect to a self-hosted GitLab instance via personal access token or OAuth
2. **Item List View** - Display issues and MRs the user is involved in (assigned, authored, tagged)
3. **Watch/Favorite Toggle** - Mark specific items to receive notifications for
4. **Mute/Snooze (lightweight)** - Temporarily suppress notifications for a tracked item without untracking it
5. **Rich Push Notifications** - Full context (who, what, which item) delivered within 1-2 minutes of activity on watched items
6. **New Tag Auto-Watch** - When tagged in a new issue/MR, item auto-adds to watch list (user can Unwatch or Dismiss; dismissed items can be re-added via notification history)
7. **Deep Linking** - Tapping a notification opens the item directly in GitLab (browser)
8. **Working Hours / Quiet Hours** - Schedule when notifications are active (optional urgency override for mentions/review requests)
9. **Basic Noise Control** - Mute notifications from selected actors (e.g., bots)
10. **Trust & Diagnostics** - View polling/delivery stats and trigger test notifications to verify the system works
11. **Dual Platform Support** - iOS app + macOS menu bar app with equivalent functionality

### Out of Scope for MVP

- Full activity feed (GitLab-like) within the app
  - (MVP includes a minimal "Recent Notifications" list for trust/debugging)
- Automatic focus detection via heuristics
- Full rules engine beyond simple quiet hours + urgency override + actor muting
- In-app actions (commenting, approving, merging, etc.)

### MVP Success Criteria

- Notifications arrive reliably within 2 minutes of GitLab activity
- Near-zero missed notifications on watched items (> 99% reliability)
- User can manage watch list without friction
- App runs without noticeable battery/resource impact

### Future Vision

- **Smart Focus Detection:** Automatically suggest items to watch based on recent commit activity, review assignments, or thread participation
- **Advanced Noise Filtering:** Rules engine to suppress low-priority notifications (e.g., CI status on non-focus branches, specific label patterns)
- **Quick Replies:** Respond to comments directly from the notification without opening GitLab
