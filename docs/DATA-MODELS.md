# H2OH — Data Models Reference

This document defines every data model used in the H2OH app. All models live in
`lib/core/models/`. Use `freezed` for immutable data classes with `json_serializable`
for JSON serialization.

Any agent implementing features should use these exact models. If a model needs to be
extended, add fields here first, then implement.

---

## 1. NostrEvent (`core/nostr/event.dart`)

The fundamental unit of data. Everything sent to or received from the relay is a NostrEvent.

```dart
@freezed
class NostrEvent with _$NostrEvent {
  const factory NostrEvent({
    required String id,
    required String pubkey,
    required int createdAt,
    required int kind,
    required List<List<String>> tags,
    required String content,
    required String sig,
  }) = _NostrEvent;

  factory NostrEvent.fromJson(Map<String, dynamic> json) =>
      _$NostrEventFromJson(json);
}
```

**JSON mapping:**
- `createdAt` maps to JSON key `"created_at"`.

**Methods to implement on this class:**
- `String computeId()` — recompute the ID from fields (for verification).
- `bool verifySignature()` — verify `sig` against `id` and `pubkey`.
- `static NostrEvent create({...})` — create, compute ID, and sign a new event.
- `Map<String, dynamic> toJson()` — serialize for sending to relay.

---

## 2. KeyPair (`core/nostr/keys.dart`)

```dart
class KeyPair {
  final String privateKeyHex; // 32 bytes as 64-char hex string
  final String publicKeyHex;  // 32 bytes as 64-char hex string (x-only pubkey)

  KeyPair({required this.privateKeyHex, required this.publicKeyHex});

  /// Bech32-encoded private key for display/export.
  String get nsec => bech32Encode('nsec', privateKeyHex);

  /// Bech32-encoded public key for display/sharing.
  String get npub => bech32Encode('npub', publicKeyHex);
}
```

**Functions to implement in `keys.dart`:**
- `KeyPair generateKeyPair()` — generate random private key, derive public key.
- `KeyPair fromPrivateKeyHex(String hex)` — derive public key from private key.
- `KeyPair fromNsec(String nsec)` — decode bech32 nsec, derive public key.
- `Future<void> saveKeyPair(KeyPair kp)` — store private key in flutter_secure_storage.
- `Future<KeyPair?> loadKeyPair()` — load private key from flutter_secure_storage.

---

## 3. Channel (`core/models/channel.dart`)

```dart
enum ChannelType { stream, forum, dm }
enum ChannelVisibility { open, private_ }

@freezed
class Channel with _$Channel {
  const factory Channel({
    required String id,
    required String name,
    required ChannelType channelType,
    required ChannelVisibility visibility,
    @Default('') String description,
    @Default('') String topic,
    @Default('') String purpose,
    @Default(0) int memberCount,
    int? lastMessageAt,
    @Default([]) List<String> participants, // pubkeys
    @Default(false) bool isMember,
    String? createdBy, // pubkey of creator
  }) = _Channel;

  factory Channel.fromJson(Map<String, dynamic> json) =>
      _$ChannelFromJson(json);
}
```

**JSON mapping from Sprout relay `/api/channels` response:**
```json
{
  "id": "abc123",
  "name": "general",
  "channel_type": "stream",
  "visibility": "open",
  "description": "General discussion",
  "topic": "Current topic",
  "purpose": "A place for general chat",
  "member_count": 42,
  "last_message_at": 1700000000,
  "participants": ["pubkey1", "pubkey2"],
  "is_member": true,
  "created_by": "pubkey0"
}
```

---

## 4. ChannelMember (`core/models/member.dart`)

```dart
enum MemberRole { owner, admin, member, guest, bot }

@freezed
class ChannelMember with _$ChannelMember {
  const factory ChannelMember({
    required String pubkey,
    required MemberRole role,
    required int joinedAt,
    String? displayName,
    String? avatarUrl,
  }) = _ChannelMember;

  factory ChannelMember.fromJson(Map<String, dynamic> json) =>
      _$ChannelMemberFromJson(json);
}
```

**JSON mapping from `/api/channels/:id/members`:**
```json
{
  "pubkey": "hex-pubkey",
  "role": "admin",
  "joined_at": 1700000000,
  "display_name": "Alice",
  "avatar_url": "https://relay.example.com/media/abc.jpg"
}
```

---

## 5. Message (`core/models/message.dart`)

A parsed, display-ready message derived from a `NostrEvent`. This is the model the UI works with.

```dart
@freezed
class Message with _$Message {
  const factory Message({
    required String id,           // event ID
    required String pubkey,       // author pubkey
    required String content,      // message text (markdown)
    required int createdAt,       // Unix timestamp
    required int kind,            // event kind (1, 9, 14, etc.)
    required String channelId,    // from "h" tag
    @Default([]) List<Reaction> reactions,
    @Default([]) List<Message> threadReplies,
    String? threadRootId,         // from NIP-10 "root" e-tag
    String? replyToId,            // from NIP-10 "reply" e-tag
    String? replyToAuthor,        // pubkey from corresponding p-tag
    @Default(false) bool isEdited,
    String? editedEventId,        // ID of edit event (if edited)
    @Default([]) List<MediaAttachment> attachments,
    UserProfile? authorProfile,   // resolved after fetch (nullable until loaded)
  }) = _Message;

  factory Message.fromNostrEvent(NostrEvent event) {
    // Parse the event's tags to extract channelId, thread refs, etc.
    // See implementation notes below.
  }
}
```

**Parsing a NostrEvent into a Message:**
1. Extract `channelId` from the `h` tag: `tags.firstWhere((t) => t[0] == 'h')[1]`.
2. Parse thread references using NIP-10 logic (see `NOSTR-PROTOCOL-REFERENCE.md` section 4).
3. Check for media URLs in content (URLs ending in `.jpg`, `.png`, `.gif`, `.mp4`, etc.) and extract as `MediaAttachment` objects.
4. `reactions` and `threadReplies` are populated separately by the provider, not from the event itself.

---

## 6. Reaction (`core/models/message.dart`)

```dart
@freezed
class Reaction with _$Reaction {
  const factory Reaction({
    required String eventId,      // reaction event ID
    required String pubkey,       // who reacted
    required String content,      // "+", "-", or emoji string
    required String targetEventId,
  }) = _Reaction;

  factory Reaction.fromNostrEvent(NostrEvent event) {
    // Parse kind 7 event — see NOSTR-PROTOCOL-REFERENCE.md section 5
  }
}
```

---

## 7. MediaAttachment (`core/models/message.dart`)

```dart
@freezed
class MediaAttachment with _$MediaAttachment {
  const factory MediaAttachment({
    required String url,
    required String mimeType,     // "image/jpeg", "image/png", "video/mp4", etc.
    String? sha256,               // hash from relay upload response
    int? size,                    // bytes
    String? thumbnailUrl,
    String? blurhash,             // BlurHash placeholder string
    int? width,
    int? height,
  }) = _MediaAttachment;

  factory MediaAttachment.fromJson(Map<String, dynamic> json) =>
      _$MediaAttachmentFromJson(json);
}
```

**JSON mapping from relay media upload response:**
```json
{
  "url": "https://relay.example.com/media/abc123",
  "sha256": "deadbeef...",
  "size": 102400,
  "type": "image/jpeg",
  "dim": "800x600",
  "thumb": "https://relay.example.com/media/abc123?thumb=1",
  "blurhash": "LKO2?U%2Tw=w]~RBVZRi};RPxuwH"
}
```

Note: `dim` is `"WxH"` string — split on `x` to get width/height integers.

---

## 8. UserProfile (`core/models/profile.dart`)

```dart
@freezed
class UserProfile with _$UserProfile {
  const factory UserProfile({
    required String pubkey,
    @Default('') String displayName,
    String? avatarUrl,
    @Default('') String about,
    String? nip05Handle,          // e.g. "alice@example.com"
    int? createdAt,
  }) = _UserProfile;

  factory UserProfile.fromJson(Map<String, dynamic> json) =>
      _$UserProfileFromJson(json);
}
```

**JSON mapping from `/api/profiles/:pubkey`:**
```json
{
  "pubkey": "hex-pubkey",
  "display_name": "Alice",
  "avatar_url": "https://relay.example.com/media/avatar.jpg",
  "about": "Flutter developer",
  "nip05_handle": "alice@example.com",
  "created_at": 1700000000
}
```

**Display name resolution priority:**
1. `displayName` if non-empty.
2. `nip05Handle` if set.
3. Truncated npub: `npub1abc...xyz` (first 8 + last 4 chars of npub).

---

## 9. HomeFeedItem (`core/models/feed_item.dart`)

```dart
enum FeedCategory { mentions, needsAction, activity, agentActivity }

@freezed
class HomeFeedItem with _$HomeFeedItem {
  const factory HomeFeedItem({
    required String id,
    required FeedCategory category,
    required String title,          // e.g. "Alice mentioned you in #general"
    required String snippet,        // truncated message content
    required int timestamp,
    required String channelId,
    required String channelName,
    String? authorPubkey,
    String? authorDisplayName,
    String? eventId,                // link to specific event
    @Default(false) bool isRead,
  }) = _HomeFeedItem;

  factory HomeFeedItem.fromJson(Map<String, dynamic> json) =>
      _$HomeFeedItemFromJson(json);
}
```

**JSON mapping from `/api/home-feed`:**
```json
{
  "feed": {
    "mentions": [
      {
        "id": "feed-item-1",
        "title": "Alice mentioned you in #general",
        "snippet": "Hey @you, can you review this?",
        "timestamp": 1700000000,
        "channel_id": "ch1",
        "channel_name": "general",
        "author_pubkey": "hex-pubkey",
        "author_display_name": "Alice",
        "event_id": "event-hex-id",
        "is_read": false
      }
    ],
    "needs_action": [...],
    "activity": [...],
    "agent_activity": [...]
  }
}
```

---

## 10. ApiToken (`core/models/token.dart`)

```dart
@freezed
class ApiToken with _$ApiToken {
  const factory ApiToken({
    required String id,
    required String name,
    required String token,          // the actual bearer token string (only on create)
    required int createdAt,
    int? expiresAt,
    @Default([]) List<String> scopes,
  }) = _ApiToken;

  factory ApiToken.fromJson(Map<String, dynamic> json) =>
      _$ApiTokenFromJson(json);
}
```

---

## 11. PresenceStatus (`core/models/presence.dart`)

```dart
enum PresenceState { online, away, offline }

@freezed
class PresenceStatus with _$PresenceStatus {
  const factory PresenceStatus({
    required String pubkey,
    required PresenceState state,
    required int lastSeen,          // Unix timestamp
  }) = _PresenceStatus;
}
```

Parsed from kind 20001 events:
- `state` = value of `["status", "online"]` tag.
- `lastSeen` = event's `created_at`.

---

## 12. RelayConfig (`core/relay/relay_config.dart`)

```dart
class RelayConfig {
  final String websocketUrl;    // e.g. "wss://relay.example.com"
  final String httpBaseUrl;     // derived: "https://relay.example.com"

  RelayConfig({required this.websocketUrl})
      : httpBaseUrl = websocketUrl
            .replaceFirst('wss://', 'https://')
            .replaceFirst('ws://', 'http://');
}
```

**Storage:** `SharedPreferences` key `"relay_url"`. Value is the WebSocket URL string.

---

## 13. CanvasDocument (`core/models/canvas.dart`)

```dart
@freezed
class CanvasDocument with _$CanvasDocument {
  const factory CanvasDocument({
    required String channelId,
    required String content,        // Rich text content (Delta JSON from flutter_quill)
    required int lastModifiedAt,
    String? lastModifiedBy,         // pubkey
  }) = _CanvasDocument;

  factory CanvasDocument.fromJson(Map<String, dynamic> json) =>
      _$CanvasDocumentFromJson(json);
}
```

**JSON mapping from `/api/canvas/:channelId`:**
```json
{
  "channel_id": "ch1",
  "content": "[{\"insert\":\"Hello canvas!\\n\"}]",
  "last_modified_at": 1700000000,
  "last_modified_by": "hex-pubkey"
}
```

---

## 14. SearchResult (`core/models/search.dart`)

```dart
@freezed
class SearchResult with _$SearchResult {
  const factory SearchResult({
    required String eventId,
    required String content,
    required String channelId,
    required String channelName,
    required String authorPubkey,
    String? authorDisplayName,
    required int createdAt,
    @Default([]) List<String> highlights,  // matched text fragments
  }) = _SearchResult;

  factory SearchResult.fromJson(Map<String, dynamic> json) =>
      _$SearchResultFromJson(json);
}
```

**JSON mapping from `/api/search`:**
```json
{
  "results": [
    {
      "event_id": "hex-id",
      "content": "Full message content",
      "channel_id": "ch1",
      "channel_name": "general",
      "author_pubkey": "hex-pubkey",
      "author_display_name": "Alice",
      "created_at": 1700000000,
      "highlights": ["matched <em>keyword</em> here"]
    }
  ]
}
```

---

## Enum Serialization

All enums should serialize to their snake_case string value:
- `ChannelType.stream` → `"stream"`
- `ChannelType.forum` → `"forum"`
- `ChannelType.dm` → `"dm"`
- `ChannelVisibility.open` → `"open"`
- `ChannelVisibility.private_` → `"private"`
- `MemberRole.owner` → `"owner"`
- `MemberRole.admin` → `"admin"`
- `MemberRole.member` → `"member"`
- `MemberRole.guest` → `"guest"`
- `MemberRole.bot` → `"bot"`
- `FeedCategory.mentions` → `"mentions"`
- `FeedCategory.needsAction` → `"needs_action"`
- `FeedCategory.activity` → `"activity"`
- `FeedCategory.agentActivity` → `"agent_activity"`
- `PresenceState.online` → `"online"`
- `PresenceState.away` → `"away"`
- `PresenceState.offline` → `"offline"`

Use `@JsonValue('snake_case_name')` annotation or a custom `JsonConverter` for enums with
Dart-reserved names (like `private_`).

---

## Relationships Between Models

```
Channel
  ├── has many ChannelMember
  ├── has many Message (via channelId)
  ├── has one CanvasDocument (via channelId)
  └── appears in HomeFeedItem (via channelId)

Message
  ├── has one UserProfile (via pubkey → authorProfile)
  ├── has many Reaction (via targetEventId)
  ├── has many Message as threadReplies (via threadRootId)
  ├── has many MediaAttachment
  └── references parent via replyToId (NIP-10)

UserProfile
  ├── referenced by Message.pubkey
  ├── referenced by ChannelMember.pubkey
  ├── referenced by Reaction.pubkey
  └── referenced by PresenceStatus.pubkey

HomeFeedItem
  ├── references Channel via channelId
  └── references NostrEvent via eventId
```
