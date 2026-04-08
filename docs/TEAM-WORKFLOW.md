# H2OH Team Structure & Workflow Plan

## Context

The H2OH project is a Flutter mobile app (Android + iOS) with ~90 files, 14 data models,
9 Nostr protocol implementations, 20+ screens, and 5 implementation phases. The work spans
cryptography, real-time WebSocket communication, encrypted messaging, rich text editing,
and push notifications. This plan defines the team roles, responsibilities, and workflow
to build and ship the product efficiently.

## Recommended Team (5 Roles)

### Role 1: Core Protocol Engineer
**Focus:** `lib/core/nostr/` + `lib/core/relay/`

**Responsibilities:**
- Nostr keypair generation, signing, verification (secp256k1, BIP-340)
- NIP-01 WebSocket protocol (REQ/EVENT/CLOSE parsing, subscription management)
- NIP-42 relay authentication
- NIP-10 thread reply tag parsing
- NIP-25 reaction event creation/parsing
- NIP-17 + NIP-44 encrypted DM system (gift wrap, ECDH, XChaCha20-Poly1305)
- WebSocket relay client (connection lifecycle, reconnection, event batching)
- HTTP relay client (Dio wrapper, auth headers, all REST endpoints)
- Unit tests for all crypto and protocol code

**Phases active:** Phase 1 (steps 2-7), Phase 2 (step 11 NIP-25), Phase 3 (step 1 NIP-17/44)

**Why separate:** Crypto and protocol code is security-critical, low-level, and requires
deep understanding of Nostr specs. Bugs here break everything. This person works ahead
of the UI team, delivering tested protocol primitives they consume.

---

### Role 2: Feature Engineer — Channels & Messaging
**Focus:** `lib/features/channels/`, `lib/features/chat/`, `lib/features/forum/`

**Responsibilities:**
- Channel data model and provider (list, create, browse, join)
- Channel list screen, create screen, browse/discovery screen
- Chat provider (message history, live updates, pagination, send/edit/delete)
- Chat screen, message bubble, message input
- Thread screen and thread reply logic
- Forum provider (thread grouping, create thread, reply)
- Forum screen and forum thread screen
- Channel detail screen (info, topic, purpose)
- Member management UI (invite, remove, role changes)
- Typing indicators (send + receive + display)

**Phases active:** Phase 2 (steps 1-8, 12-13), Phase 3 (steps 5-8)

**Why separate:** This is the largest feature surface. Channels and messaging are the core
UX — the user spends 80% of their time here. Needs a dedicated engineer who owns the
full flow from channel list → chat → threads → forums.

---

### Role 3: Feature Engineer — DMs, Search, Home Feed
**Focus:** `lib/features/dm/`, `lib/features/search/`, `lib/features/home/`

**Responsibilities:**
- DM provider (encrypt/decrypt, conversation list, send/receive)
- DM list screen and DM chat screen
- Group DM support (multi-recipient gift wraps)
- Home feed provider (4 categories, live mention updates)
- Home screen with collapsible sections and unread badges
- Feed card widgets (mentions, needs action, activity, agent activity)
- Search provider (debounced, recent searches, scoped)
- Global search screen and in-channel search mode
- Search result tile with highlighting

**Phases active:** Phase 3 (steps 2-4, 9), Phase 4 (steps 1-5)

**Why separate:** DMs require integrating the NIP-17 crypto from Role 1 into a user-facing
flow. Search and home feed are distinct systems that benefit from one person owning
the full data flow (relay → provider → UI).

---

### Role 4: Feature Engineer — Canvas, Profile, Settings, Polish
**Focus:** `lib/features/channels/canvas/`, `lib/features/profile/`, `lib/features/settings/`, `lib/features/presence/`, `lib/shared/`

**Responsibilities:**
- Canvas provider (auto-save, optimistic updates) and canvas screen (flutter_quill)
- Profile provider and profile screen (edit own, view others)
- Avatar widget with presence dot overlay
- Settings screen (relay, presence, notifications, tokens, theme, security)
- Token management screen (create, list, revoke)
- Settings provider (theme mode, notification prefs persistence)
- Presence system (subscribe kind 20001, lifecycle observer, expiry timers)
- Presence indicator widget
- Theme finalization (light + dark Material 3)
- Responsive layout (phone vs tablet, NavigationRail)
- Error banners, loading skeletons, empty states

**Phases active:** Phase 4 (steps 6-7), Phase 5 (all steps)

**Why separate:** This role covers the "everything else" surface — settings, profile,
canvas, presence, theming, polish. These are largely independent features that one
person can own end-to-end without blocking others.

---

### Role 5: QA & Integration Lead
**Focus:** Testing, CI/CD, integration, app store prep

**Responsibilities:**
- Write widget tests for each screen
- Write integration tests for relay connection flow (mock WebSocket)
- Set up CI pipeline (GitHub Actions: lint, test, build)
- Manual testing on Android emulator + iOS simulator
- Test on physical devices (various screen sizes)
- Verify NIP compliance (test against real Sprout relay)
- App store preparation (icons, splash screens, metadata)
- Firebase project setup (FCM for push notifications)
- Performance profiling (message list scroll, WebSocket throughput)
- Security review (key storage, no key logging, DM encryption verification)

**Phases active:** All phases (starts with Phase 1 CI setup, ramps up in Phase 3+)

---

## Workflow

### Sprint Structure (2-week sprints)

```
Sprint 1 (Phase 1):  Foundation
Sprint 2 (Phase 2a): Channels + Models
Sprint 3 (Phase 2b): Chat + Messaging
Sprint 4 (Phase 3):  DMs + Forums + Members
Sprint 5 (Phase 4):  Home Feed + Search + Canvas
Sprint 6 (Phase 5a): Presence + Notifications + Profile
Sprint 7 (Phase 5b): Settings + Theme + Polish + QA
Sprint 8:            Bug fixes, performance, app store submission
```

### Parallel Work Lanes

```
Sprint    | Role 1 (Protocol)        | Role 2 (Channels)       | Role 3 (DMs/Search)     | Role 4 (Canvas/Polish)  | Role 5 (QA)
--------- | ------------------------ | ----------------------- | ----------------------- | ----------------------- | -------------------
1         | Keys, Event, NIP-01,     | —                       | —                       | —                       | CI/CD setup,
          | NIP-42, RelayClient,     |                         |                         |                         | lint config
          | RelayHttp, RelayConfig   |                         |                         |                         |
          |                          |                         |                         |                         |
2         | NIP-10, NIP-25           | Channel model/provider, | Data models (message,   | App shell, routing,     | Unit tests for
          |                          | Channel list/create/    | reaction, media)        | theme scaffold          | crypto layer
          |                          | browse screens          |                         |                         |
          |                          |                         |                         |                         |
3         | (buffer / support)       | Chat provider, chat     | Markdown body,          | Diff view, media        | Widget tests for
          |                          | screen, message bubble, | message input            | service                 | channels + chat
          |                          | thread screen           |                         |                         |
          |                          |                         |                         |                         |
4         | NIP-44, NIP-17           | Forum provider, forum   | DM provider, DM list,   | Member management,      | Integration tests,
          | (encryption)             | screen, forum thread    | DM chat, group DMs      | channel detail          | relay connection test
          |                          |                         |                         |                         |
5         | (buffer / support)       | Typing indicators,      | Home feed provider,     | Canvas provider,        | DM encryption
          |                          | reaction bar polish     | home screen, feed cards | canvas screen           | verification
          |                          |                         |                         |                         |
6         | (buffer / support)       | In-channel search mode  | Search provider, search | Presence system,        | End-to-end testing,
          |                          |                         | screen                  | presence indicator,     | device testing
          |                          |                         |                         | avatar widget           |
          |                          |                         |                         |                         |
7         | Security audit           | Bug fixes               | Bug fixes               | Settings, tokens,       | Performance testing,
          |                          |                         |                         | profile, notifications, | app store prep
          |                          |                         |                         | theme, responsive       |
          |                          |                         |                         |                         |
8         | —                        | Bug fixes               | Bug fixes               | Bug fixes               | Final QA, submission
```

### Definition of Done (per task)

1. Code compiles with no lint warnings.
2. Freezed/json_serializable code generated (`dart run build_runner build`).
3. Unit tests pass for any new logic (providers, models, protocol).
4. Widget renders correctly on both Android and iOS simulator.
5. PR reviewed by at least one other role.
6. Verification checklist items from the phase doc are checked off.

### Communication Rules

1. **Role 1 delivers protocol code first.** Other roles depend on it. Minimum viable:
   keys, event signing, relay client, HTTP client must be done before anyone else starts.
2. **Shared widgets go in `lib/shared/widgets/`.** Any widget used by 2+ features lives here.
   Role 4 owns this directory. Others submit PRs.
3. **Models go in `lib/core/models/`.** Role 2 and 3 both need these. Whoever gets there
   first creates them per DATA-MODELS.md. No duplicates.
4. **Router changes** are coordinated — `lib/shared/router.dart` is a merge conflict magnet.
   Each role adds their routes in a clearly marked section.
5. **Daily standup** focused on blockers, especially cross-role dependencies.

### Branch Strategy

```
main
 └── develop
      ├── feature/phase-1-foundation     (Role 1 + Role 4 + Role 5)
      ├── feature/phase-2-channels       (Role 2)
      ├── feature/phase-2-messaging      (Role 2 + Role 3)
      ├── feature/phase-3-dms            (Role 3)
      ├── feature/phase-3-forums         (Role 2)
      ├── feature/phase-3-members        (Role 4)
      ├── feature/phase-4-home-feed      (Role 3)
      ├── feature/phase-4-search         (Role 3)
      ├── feature/phase-4-canvas         (Role 4)
      ├── feature/phase-5-presence       (Role 4)
      ├── feature/phase-5-notifications  (Role 4)
      ├── feature/phase-5-profile        (Role 4)
      └── feature/phase-5-settings       (Role 4)
```

PRs merge to `develop`. `develop` merges to `main` at sprint boundaries after QA sign-off.

## Verification

- Each phase doc has a verification checklist — QA (Role 5) owns sign-off.
- Integration test: connect to a real Sprout relay, send a message, verify it arrives.
- Crypto test: NIP-44 test vectors from the spec, NIP-17 round-trip encrypt/decrypt.
- Cross-platform: test on Android 12+ and iOS 16+ minimum.
