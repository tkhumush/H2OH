# H2OH — Architecture Document

## Overview

H2OH is a Flutter mobile application (Android + iOS) that serves as a client for a
[Sprout relay](https://github.com/block/sprout). It replicates the functionality of the
Sprout desktop app — minus agents and workflows — in a cross-platform mobile form factor.

The app communicates with a self-hosted Sprout relay over **WebSocket** (real-time events)
and **HTTP REST** (queries and uploads). All messages are **Nostr events** signed with
Schnorr signatures on the secp256k1 curve.

Users configure the relay URL in-app. There is no hardcoded relay — the app is relay-agnostic.

---

## Terminology

| Term | Definition |
|------|-----------|
| **Relay** | A Sprout relay server the app connects to. Speaks NIP-01 WebSocket + HTTP REST. |
| **Nostr event** | A JSON object with `id`, `pubkey`, `kind`, `content`, `tags`, `created_at`, `sig`. The universal data unit. |
| **NIP** | Nostr Implementation Possibility — a numbered spec (e.g. NIP-01, NIP-42). See `docs/NOSTR-PROTOCOL-REFERENCE.md`. |
| **Kind** | An integer identifying the event type (e.g. kind 1 = text note, kind 7 = reaction). |
| **Channel** | A conversation space. Can be "stream" (real-time chat), "forum" (threaded), or "dm" (direct message). |
| **Canvas** | A shared rich-text document attached to a channel. |
| **Subscription** | A WebSocket filter request (`REQ`) that tells the relay to stream matching events. |
| **nsec / npub** | Bech32-encoded Nostr secret key / public key. |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Flutter App (H2OH)                │
│                                                     │
│  ┌───────────┐  ┌───────────┐  ┌────────────────┐  │
│  │  Features  │  │  Features │  │   Features     │  │
│  │  (Home,    │  │  (Chat,   │  │  (Settings,    │  │
│  │   Search)  │  │   Forum,  │  │   Profile,     │  │
│  │           │  │   DM,     │  │   Presence)    │  │
│  │           │  │   Canvas) │  │               │  │
│  └─────┬─────┘  └─────┬─────┘  └──────┬─────────┘  │
│        │              │               │             │
│        ▼              ▼               ▼             │
│  ┌──────────────────────────────────────────────┐   │
│  │              Riverpod Providers              │   │
│  │   (State management, caching, reactivity)    │   │
│  └──────────────────┬───────────────────────────┘   │
│                     │                               │
│  ┌──────────────────▼───────────────────────────┐   │
│  │              Core Services                   │   │
│  │  ┌────────────┐  ┌────────────┐              │   │
│  │  │ RelayClient│  │ RelayHttp  │              │   │
│  │  │ (WebSocket)│  │  (REST)    │              │   │
│  │  └─────┬──────┘  └─────┬──────┘              │   │
│  │        │               │                     │   │
│  │  ┌─────▼───────────────▼──────┐              │   │
│  │  │    Nostr Protocol Layer    │              │   │
│  │  │  (Keys, Signing, NIPs)     │              │   │
│  │  └────────────────────────────┘              │   │
│  └──────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────┘
                       │
                       │  WebSocket (wss://) + HTTP (https://)
                       ▼
              ┌─────────────────┐
              │  Sprout Relay   │
              │  (self-hosted)  │
              └─────────────────┘
```

---

## Communication Pattern

The app uses a **dual-transport** pattern identical to the Sprout desktop app:

### WebSocket (Real-time)
- **Purpose:** Receive live events (new messages, reactions, presence, typing indicators).
- **Protocol:** NIP-01 — sends `REQ` (subscribe), receives `EVENT`, sends `CLOSE` (unsubscribe).
- **Auth:** NIP-42 — relay sends `AUTH` challenge, app responds with a signed auth event.
- **Publishing:** The app publishes new events (messages, reactions, etc.) by sending `EVENT` over WebSocket.
- **Reconnection:** Exponential backoff (1s → 2s → 4s → 8s → 16s → 30s max).
- **Event batching:** Buffer incoming events for 16ms before triggering UI updates, to avoid per-event rebuilds.

### HTTP REST (Queries)
- **Purpose:** Fetch channel lists, profiles, message history, search results, upload media.
- **Auth:** Bearer token header OR `X-Pubkey` header depending on endpoint.
- **Base URL:** Derived from the relay WebSocket URL (e.g. `wss://relay.example.com` → `https://relay.example.com`).

### When to use which transport

| Operation | Transport | Why |
|-----------|-----------|-----|
| Send a message | WebSocket `EVENT` | Real-time broadcast |
| Receive live messages | WebSocket `REQ` subscription | Push-based |
| Fetch channel list | HTTP GET `/api/channels` | One-shot query |
| Fetch message history | HTTP GET `/api/events` | Paginated query |
| Search messages | HTTP GET `/api/search` | Relay-side Typesense |
| Upload media | HTTP POST `/media/` | Binary upload |
| Fetch user profile | HTTP GET `/api/profiles` | One-shot query |
| Presence updates | WebSocket `EVENT` (kind 20001) | Real-time |
| Typing indicators | WebSocket `EVENT` | Real-time, ephemeral |

---

## Project Structure

```
h2oh/
├── android/                          # Android platform project
├── ios/                              # iOS platform project
├── lib/
│   ├── main.dart                     # Entry point
│   ├── app.dart                      # MaterialApp, GoRouter, ThemeData
│   │
│   ├── core/                         # Shared infrastructure (no UI)
│   │   ├── nostr/
│   │   │   ├── keys.dart             # Keypair generation, nsec/npub encoding, secure storage
│   │   │   ├── event.dart            # NostrEvent model, serialization, ID computation, signing
│   │   │   ├── nip01.dart            # REQ/EVENT/CLOSE message builders and parsers
│   │   │   ├── nip10.dart            # Thread reply tag parsing (root, reply, mention markers)
│   │   │   ├── nip17.dart            # NIP-17 encrypted direct messages (encrypt/decrypt)
│   │   │   ├── nip25.dart            # Reactions (kind 7 creation and parsing)
│   │   │   └── nip42.dart            # AUTH challenge handling and signed response
│   │   │
│   │   ├── relay/
│   │   │   ├── relay_client.dart     # WebSocket lifecycle, subscriptions, event dispatch
│   │   │   ├── relay_http.dart       # HTTP client wrapper with auth headers
│   │   │   └── relay_config.dart     # Relay URL read/write (SharedPreferences)
│   │   │
│   │   ├── models/                   # Pure Dart data classes (freezed or manual)
│   │   │   ├── channel.dart          # Channel, ChannelType enum
│   │   │   ├── message.dart          # Message (parsed from NostrEvent)
│   │   │   ├── profile.dart          # UserProfile
│   │   │   ├── member.dart           # ChannelMember, MemberRole enum
│   │   │   └── feed_item.dart        # HomeFeedItem, FeedCategory enum
│   │   │
│   │   └── services/
│   │       ├── auth_service.dart     # Orchestrates identity loading + NIP-42
│   │       ├── media_service.dart    # Image pick, compress, upload, thumbnail URL
│   │       └── notification_service.dart  # FCM/APNs setup, local notifications
│   │
│   ├── features/                     # Feature modules (vertical slices)
│   │   ├── home/
│   │   │   ├── home_screen.dart
│   │   │   ├── home_provider.dart
│   │   │   └── widgets/
│   │   │       ├── mentions_card.dart
│   │   │       ├── needs_action_card.dart
│   │   │       ├── activity_card.dart
│   │   │       └── agent_activity_card.dart
│   │   │
│   │   ├── channels/
│   │   │   ├── channel_list_screen.dart
│   │   │   ├── channel_detail_screen.dart
│   │   │   ├── channel_create_screen.dart
│   │   │   ├── channel_browse_screen.dart
│   │   │   ├── channel_provider.dart
│   │   │   ├── canvas/
│   │   │   │   ├── canvas_screen.dart
│   │   │   │   └── canvas_provider.dart
│   │   │   └── widgets/
│   │   │       ├── channel_tile.dart
│   │   │       └── member_list.dart
│   │   │
│   │   ├── chat/
│   │   │   ├── chat_screen.dart
│   │   │   ├── chat_provider.dart
│   │   │   ├── thread_screen.dart
│   │   │   └── widgets/
│   │   │       ├── message_bubble.dart
│   │   │       ├── message_input.dart
│   │   │       ├── reaction_bar.dart
│   │   │       ├── typing_indicator.dart
│   │   │       ├── markdown_body.dart
│   │   │       └── diff_view.dart
│   │   │
│   │   ├── forum/
│   │   │   ├── forum_screen.dart
│   │   │   ├── forum_thread_screen.dart
│   │   │   └── forum_provider.dart
│   │   │
│   │   ├── dm/
│   │   │   ├── dm_list_screen.dart
│   │   │   ├── dm_chat_screen.dart
│   │   │   └── dm_provider.dart
│   │   │
│   │   ├── search/
│   │   │   ├── search_screen.dart
│   │   │   ├── search_provider.dart
│   │   │   └── widgets/
│   │   │       └── search_result_tile.dart
│   │   │
│   │   ├── profile/
│   │   │   ├── profile_screen.dart
│   │   │   └── profile_provider.dart
│   │   │
│   │   ├── settings/
│   │   │   ├── settings_screen.dart
│   │   │   ├── relay_config_screen.dart
│   │   │   ├── token_management_screen.dart
│   │   │   └── settings_provider.dart
│   │   │
│   │   └── presence/
│   │       ├── presence_provider.dart
│   │       └── presence_indicator.dart
│   │
│   └── shared/
│       ├── theme.dart                # App-wide ThemeData (light + dark)
│       ├── router.dart               # GoRouter route definitions
│       └── widgets/
│           ├── avatar.dart           # User avatar with fallback
│           ├── loading.dart          # Shared loading indicators
│           └── error_banner.dart     # Error display widget
│
├── assets/                           # Fonts, images, icons
├── test/                             # Unit and widget tests
├── pubspec.yaml                      # Dependencies
├── analysis_options.yaml             # Lint rules
├── ARCHITECTURE.md                   # This file
└── docs/
    ├── PHASE-1-FOUNDATION.md
    ├── PHASE-2-CHANNELS-AND-MESSAGING.md
    ├── PHASE-3-DMS-FORUMS-MEMBERS.md
    ├── PHASE-4-HOME-FEED-SEARCH-CANVAS.md
    ├── PHASE-5-POLISH.md
    ├── NOSTR-PROTOCOL-REFERENCE.md
    └── DATA-MODELS.md
```

---

## State Management — Riverpod

All state is managed through `flutter_riverpod`. The pattern:

1. **Providers** live in each feature's `*_provider.dart` file.
2. **AsyncNotifierProvider** for data that loads from relay (channels, messages, profiles).
3. **StreamProvider** for WebSocket-driven live data (incoming events).
4. **StateProvider** for simple UI state (selected channel, search query).
5. **Core providers** in `core/` expose the relay client, auth state, and current user identity.

### Provider dependency graph (simplified)

```
relayConfigProvider (relay URL from SharedPreferences)
       │
       ▼
relayClientProvider (WebSocket connection, depends on relay URL)
       │
       ├──► channelProvider (subscribes to channel events)
       ├──► chatProvider (subscribes to messages for active channel)
       ├──► presenceProvider (subscribes to presence events)
       └──► homeFeedProvider (subscribes to home feed events)

relayHttpProvider (HTTP client, depends on relay URL)
       │
       ├──► channelListProvider (fetches channel list via HTTP)
       ├──► profileProvider (fetches/updates profiles via HTTP)
       ├──► searchProvider (queries search API)
       └──► mediaProvider (uploads media via HTTP)

authProvider (keypair from secure storage)
       │
       ├──► relayClientProvider (needs keys for NIP-42 auth + signing)
       └──► relayHttpProvider (needs pubkey for auth headers)
```

---

## Routing — GoRouter

The app uses `go_router` for declarative, URL-based navigation.

### Route tree

```
/                           → HomeScreen (default)
/channels                   → ChannelListScreen
/channels/browse            → ChannelBrowseScreen
/channels/create            → ChannelCreateScreen
/channels/:channelId        → ChannelDetailScreen
/channels/:channelId/chat   → ChatScreen (stream channel)
/channels/:channelId/forum  → ForumScreen
/channels/:channelId/canvas → CanvasScreen
/channels/:channelId/thread/:eventId → ThreadScreen
/dm                         → DmListScreen
/dm/:conversationId         → DmChatScreen
/search                     → SearchScreen
/profile                    → ProfileScreen
/settings                   → SettingsScreen
/settings/relay             → RelayConfigScreen
/settings/tokens            → TokenManagementScreen
```

### Shell route

A `ShellRoute` wraps the main screens with a persistent bottom navigation bar:
- **Home** (`/`)
- **Channels** (`/channels`)
- **DMs** (`/dm`)
- **Settings** (`/settings`)

---

## Dependencies

```yaml
dependencies:
  flutter:
    sdk: flutter

  # State management
  flutter_riverpod: ^2.5.0
  riverpod_annotation: ^2.3.0

  # Routing
  go_router: ^14.0.0

  # Networking
  web_socket_channel: ^3.0.0        # WebSocket to relay
  dio: ^5.4.0                       # HTTP REST client
  connectivity_plus: ^6.0.0         # Network state detection

  # Nostr / Crypto
  pointycastle: ^3.9.0              # Schnorr signatures (secp256k1)
  bip340: ^0.3.0                    # BIP-340 Schnorr helpers
  convert: ^3.1.0                   # Hex encoding
  crypto: ^3.0.0                    # SHA-256 for event IDs
  cryptography: ^2.7.0              # Additional crypto primitives

  # Storage
  flutter_secure_storage: ^9.2.0    # Private key storage
  shared_preferences: ^2.3.0        # Relay URL, app prefs

  # UI
  cached_network_image: ^3.3.0      # Image caching
  flutter_markdown: ^0.7.0          # Markdown rendering
  flutter_highlight: ^0.7.0         # Syntax highlighting in markdown
  emoji_picker_flutter: ^2.2.0      # Emoji picker for reactions
  flutter_quill: ^10.0.0            # Rich text editor for Canvas
  image_picker: ^1.1.0              # Pick images for upload
  image_cropper: ^8.0.0             # Crop before upload
  flutter_diff: ^0.1.0              # Diff view rendering

  # Notifications
  firebase_core: ^3.0.0
  firebase_messaging: ^15.0.0       # FCM push notifications
  flutter_local_notifications: ^18.0.0

  # Utilities
  uuid: ^4.4.0                      # Generate subscription IDs
  intl: ^0.19.0                     # Date/time formatting
  timeago: ^3.7.0                   # "2 minutes ago" formatting
  collection: ^1.18.0               # List utilities
  freezed_annotation: ^2.4.0        # Immutable data classes
  json_annotation: ^4.9.0           # JSON serialization

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.0
  flutter_lints: ^4.0.0
  mockito: ^5.4.0
  mocktail: ^1.0.0
```

---

## Security Considerations

1. **Private keys** are stored in `flutter_secure_storage` (Keychain on iOS, EncryptedSharedPreferences on Android). Never log or expose the nsec.
2. **All outgoing events** are signed with the user's private key. The relay rejects unsigned or improperly signed events.
3. **DM content** is encrypted with NIP-17 before sending. The relay only sees ciphertext.
4. **Relay URL** is stored in `SharedPreferences` (not sensitive). The relay connection uses `wss://` (TLS) in production.
5. **Bearer tokens** for API auth are stored in `flutter_secure_storage`.

---

## Testing Strategy

- **Unit tests** for all `core/nostr/` functions (key gen, signing, NIP parsing).
- **Unit tests** for all `core/models/` serialization/deserialization.
- **Widget tests** for each feature's main screen.
- **Integration tests** for the relay connection flow (mock WebSocket).
- **Golden tests** for critical UI components (message bubble, channel tile).

---

## Build & Run

```bash
# Create Flutter project (Phase 1, Step 1)
flutter create --org com.h2oh --project-name h2oh .

# Run on device
flutter run

# Run tests
flutter test

# Build for release
flutter build apk       # Android
flutter build ios        # iOS
```

---

## Phase Overview

| Phase | Focus | Doc |
|-------|-------|-----|
| 1 | Foundation — project setup, keys, relay connection, auth, routing shell | `docs/PHASE-1-FOUNDATION.md` |
| 2 | Channels & Messaging — channel list, chat, markdown, media, threads, reactions | `docs/PHASE-2-CHANNELS-AND-MESSAGING.md` |
| 3 | DMs, Forums, Members — encrypted DMs, forum threads, member management | `docs/PHASE-3-DMS-FORUMS-MEMBERS.md` |
| 4 | Home Feed, Search, Canvas — feed categories, full-text search, rich text canvas | `docs/PHASE-4-HOME-FEED-SEARCH-CANVAS.md` |
| 5 | Polish — presence, notifications, profile editing, tokens, theming | `docs/PHASE-5-POLISH.md` |

Each phase doc is a self-contained prompt. An agent can pick up any phase doc and implement it given the context in this architecture doc and the data models doc.
