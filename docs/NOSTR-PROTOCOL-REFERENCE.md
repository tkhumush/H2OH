# Nostr Protocol Reference for H2OH

This document describes the Nostr protocol details that the H2OH app must implement
to communicate with a Sprout relay. It is the authoritative reference for any agent
implementing the `core/nostr/` layer.

---

## 1. Nostr Event Structure (NIP-01)

Every piece of data in Nostr is an **event**. An event is a JSON object:

```json
{
  "id": "32-byte hex SHA-256 of the serialized event",
  "pubkey": "32-byte hex public key of the event creator",
  "created_at": 1700000000,
  "kind": 1,
  "tags": [
    ["e", "<event-id>", "<relay-url>", "<marker>"],
    ["p", "<pubkey>"],
    ["t", "<hashtag>"]
  ],
  "content": "Hello world",
  "sig": "64-byte hex Schnorr signature"
}
```

### Computing the event ID

The `id` is the SHA-256 hash of the **canonical serialization**:

```
SHA256( UTF-8( JSON.serialize([
  0,                  // reserved
  <pubkey>,           // hex string
  <created_at>,       // integer
  <kind>,             // integer
  <tags>,             // array of arrays
  <content>           // string
]) ) )
```

The JSON serialization MUST:
- Use no whitespace (no spaces, no newlines).
- Serialize integers as numbers (not strings).
- Serialize strings with proper JSON escaping.

### Signing

The `sig` is a **BIP-340 Schnorr signature** of the `id` bytes using the secp256k1 curve.

```
sig = schnorr_sign(private_key, id_bytes)
```

Verification:
```
schnorr_verify(pubkey_bytes, id_bytes, sig_bytes) == true
```

### Implementation in Dart

```dart
// Pseudocode for core/nostr/event.dart

class NostrEvent {
  final String id;
  final String pubkey;
  final int createdAt;
  final int kind;
  final List<List<String>> tags;
  final String content;
  final String sig;

  /// Compute the event ID from fields.
  static String computeId({
    required String pubkey,
    required int createdAt,
    required int kind,
    required List<List<String>> tags,
    required String content,
  }) {
    final serialized = jsonEncode([0, pubkey, createdAt, kind, tags, content]);
    final bytes = utf8.encode(serialized);
    final hash = sha256.convert(bytes);
    return hash.toString(); // hex string
  }

  /// Create and sign a new event.
  static NostrEvent create({
    required String privateKeyHex,
    required String pubkeyHex,
    required int kind,
    required String content,
    List<List<String>> tags = const [],
  }) {
    final createdAt = DateTime.now().millisecondsSinceEpoch ~/ 1000;
    final id = computeId(
      pubkey: pubkeyHex,
      createdAt: createdAt,
      kind: kind,
      tags: tags,
      content: content,
    );
    final sig = schnorrSign(privateKeyHex, id); // BIP-340
    return NostrEvent(
      id: id,
      pubkey: pubkeyHex,
      createdAt: createdAt,
      kind: kind,
      tags: tags,
      content: content,
      sig: sig,
    );
  }
}
```

---

## 2. Client-Relay Communication (NIP-01)

Communication happens over a single WebSocket connection. Messages are JSON arrays.

### Client → Relay

#### EVENT (publish)
```json
["EVENT", <event-object>]
```
Publishes a signed event to the relay.

#### REQ (subscribe)
```json
["REQ", "<subscription-id>", <filter-1>, <filter-2>, ...]
```
Requests events matching the filters. The relay will:
1. Send all **stored events** matching the filters.
2. Then send **new events** in real-time as they arrive.
3. Send `EOSE` (End of Stored Events) after the initial batch.

**Filter object:**
```json
{
  "ids": ["<event-id-prefix>", ...],      // match event IDs (prefix match)
  "authors": ["<pubkey-prefix>", ...],     // match event pubkeys
  "kinds": [1, 7, ...],                    // match event kinds
  "#e": ["<event-id>", ...],              // match events with these e-tags
  "#p": ["<pubkey>", ...],                // match events with these p-tags
  "since": 1700000000,                     // match events created after this
  "until": 1700100000,                     // match events created before this
  "limit": 50                              // max number of stored events to return
}
```

All filter fields are optional. Multiple filters in one REQ are OR'd together.

#### CLOSE (unsubscribe)
```json
["CLOSE", "<subscription-id>"]
```
Tells the relay to stop sending events for this subscription.

### Relay → Client

#### EVENT (received)
```json
["EVENT", "<subscription-id>", <event-object>]
```
An event matching a subscription.

#### EOSE (end of stored events)
```json
["EOSE", "<subscription-id>"]
```
Signals that all stored events have been sent; future events are real-time.

#### OK (publish result)
```json
["OK", "<event-id>", true, ""]
["OK", "<event-id>", false, "error: reason"]
```
Acknowledges an EVENT publish. `true` = accepted, `false` = rejected with reason.

#### NOTICE (relay message)
```json
["NOTICE", "message"]
```
A human-readable message from the relay (usually errors or info).

#### AUTH (authentication challenge)
```json
["AUTH", "<challenge-string>"]
```
See NIP-42 below.

### Implementation in Dart

```dart
// Pseudocode for core/nostr/nip01.dart

/// Build a REQ message.
String buildReq(String subscriptionId, Map<String, dynamic> filter) {
  return jsonEncode(["REQ", subscriptionId, filter]);
}

/// Build an EVENT message for publishing.
String buildEvent(NostrEvent event) {
  return jsonEncode(["EVENT", event.toJson()]);
}

/// Build a CLOSE message.
String buildClose(String subscriptionId) {
  return jsonEncode(["CLOSE", subscriptionId]);
}

/// Parse an incoming relay message.
/// Returns a typed message object (EventMessage, EoseMessage, OkMessage, etc.)
RelayMessage parseRelayMessage(String raw) {
  final decoded = jsonDecode(raw) as List;
  switch (decoded[0] as String) {
    case 'EVENT':
      return EventMessage(
        subscriptionId: decoded[1],
        event: NostrEvent.fromJson(decoded[2]),
      );
    case 'EOSE':
      return EoseMessage(subscriptionId: decoded[1]);
    case 'OK':
      return OkMessage(
        eventId: decoded[1],
        accepted: decoded[2],
        message: decoded[3],
      );
    case 'NOTICE':
      return NoticeMessage(message: decoded[1]);
    case 'AUTH':
      return AuthMessage(challenge: decoded[1]);
    default:
      return UnknownMessage(raw);
  }
}
```

---

## 3. Authentication (NIP-42)

When connecting, the relay may send an `AUTH` challenge:
```json
["AUTH", "<challenge-string>"]
```

The client must respond with a signed event of **kind 22242**:
```json
{
  "kind": 22242,
  "content": "",
  "tags": [
    ["relay", "wss://relay.example.com"],
    ["challenge", "<challenge-string>"]
  ],
  ...signed fields...
}
```

Send it as:
```json
["AUTH", <signed-event>]
```

### Implementation in Dart

```dart
// Pseudocode for core/nostr/nip42.dart

NostrEvent createAuthEvent({
  required String privateKeyHex,
  required String pubkeyHex,
  required String relayUrl,
  required String challenge,
}) {
  return NostrEvent.create(
    privateKeyHex: privateKeyHex,
    pubkeyHex: pubkeyHex,
    kind: 22242,
    content: '',
    tags: [
      ['relay', relayUrl],
      ['challenge', challenge],
    ],
  );
}

/// Call this when you receive an AUTH message from the relay.
/// Sends the signed auth response back over the WebSocket.
void handleAuthChallenge(WebSocket ws, String challenge, KeyPair keys, String relayUrl) {
  final authEvent = createAuthEvent(
    privateKeyHex: keys.privateKeyHex,
    pubkeyHex: keys.publicKeyHex,
    relayUrl: relayUrl,
    challenge: challenge,
  );
  ws.add(jsonEncode(['AUTH', authEvent.toJson()]));
}
```

---

## 4. Thread Replies (NIP-10)

Thread structure is encoded in `e` tags with positional markers:

```json
{
  "kind": 1,
  "content": "This is a reply",
  "tags": [
    ["e", "<root-event-id>", "", "root"],
    ["e", "<parent-event-id>", "", "reply"],
    ["p", "<root-author-pubkey>"],
    ["p", "<parent-author-pubkey>"]
  ]
}
```

### Tag markers:
- `"root"` — the top-level event in the thread.
- `"reply"` — the event this is directly replying to.
- `"mention"` — an event referenced but not replied to.
- No marker — legacy positional format (first = root, last = reply).

### Parsing logic:
1. Look for e-tags with `"root"` marker → that's the thread root.
2. Look for e-tags with `"reply"` marker → that's the direct parent.
3. If no markers, fall back to positional: first e-tag = root, last e-tag = reply.

```dart
// Pseudocode for core/nostr/nip10.dart

class ThreadReference {
  final String? rootEventId;
  final String? replyEventId;
  final List<String> mentionedEventIds;
  final List<String> mentionedPubkeys;
}

ThreadReference parseThreadTags(List<List<String>> tags) {
  String? root, reply;
  final mentions = <String>[];
  final pubkeys = <String>[];

  final eTags = tags.where((t) => t[0] == 'e').toList();
  final pTags = tags.where((t) => t[0] == 'p').toList();

  for (final tag in eTags) {
    final marker = tag.length > 3 ? tag[3] : null;
    switch (marker) {
      case 'root':
        root = tag[1];
        break;
      case 'reply':
        reply = tag[1];
        break;
      case 'mention':
        mentions.add(tag[1]);
        break;
      default:
        // Positional fallback handled below
        break;
    }
  }

  // Positional fallback: no markers found
  if (root == null && reply == null && eTags.isNotEmpty) {
    root = eTags.first[1];
    if (eTags.length > 1) {
      reply = eTags.last[1];
    }
  }

  for (final tag in pTags) {
    pubkeys.add(tag[1]);
  }

  return ThreadReference(
    rootEventId: root,
    replyEventId: reply,
    mentionedEventIds: mentions,
    mentionedPubkeys: pubkeys,
  );
}

/// Build tags for a reply to a given event.
List<List<String>> buildReplyTags({
  required NostrEvent replyingTo,
  required String? threadRootId,
}) {
  final rootId = threadRootId ?? replyingTo.id;
  return [
    ['e', rootId, '', 'root'],
    if (rootId != replyingTo.id)
      ['e', replyingTo.id, '', 'reply'],
    ['p', replyingTo.pubkey],
  ];
}
```

---

## 5. Reactions (NIP-25)

A reaction is a **kind 7** event:

```json
{
  "kind": 7,
  "content": "+",
  "tags": [
    ["e", "<reacted-to-event-id>"],
    ["p", "<reacted-to-event-pubkey>"]
  ]
}
```

### Content values:
- `"+"` — like/upvote
- `"-"` — dislike/downvote
- Any emoji string (e.g. `"🔥"`, `"👍"`) — custom reaction

```dart
// Pseudocode for core/nostr/nip25.dart

NostrEvent createReaction({
  required String privateKeyHex,
  required String pubkeyHex,
  required String targetEventId,
  required String targetPubkey,
  String content = '+',
}) {
  return NostrEvent.create(
    privateKeyHex: privateKeyHex,
    pubkeyHex: pubkeyHex,
    kind: 7,
    content: content,
    tags: [
      ['e', targetEventId],
      ['p', targetPubkey],
    ],
  );
}

/// Parse a reaction event to get the target and content.
({String targetEventId, String targetPubkey, String reaction}) parseReaction(NostrEvent event) {
  assert(event.kind == 7);
  final eTag = event.tags.firstWhere((t) => t[0] == 'e');
  final pTag = event.tags.firstWhere((t) => t[0] == 'p');
  return (
    targetEventId: eTag[1],
    targetPubkey: pTag[1],
    reaction: event.content,
  );
}
```

---

## 6. Encrypted Direct Messages (NIP-17)

NIP-17 uses **gift wrap** encryption for DMs. The flow:

### Sending a DM:

1. Create the **inner event** (kind 14 — chat message):
   ```json
   {
     "kind": 14,
     "content": "Hello, this is a private message",
     "tags": [["p", "<recipient-pubkey>"]],
     ...signed by sender...
   }
   ```

2. Create a **seal** (kind 13): Encrypt the inner event JSON with a shared secret derived from sender's private key + recipient's public key (NIP-44 encryption). The seal's content is the ciphertext.
   ```json
   {
     "kind": 13,
     "content": "<nip44-encrypted-inner-event>",
     "tags": [],
     ...signed by sender...
   }
   ```

3. Create a **gift wrap** (kind 1059): Generate a random one-time keypair. Encrypt the seal with this random key + recipient's pubkey. The wrap looks like it came from a random pubkey, hiding the sender.
   ```json
   {
     "kind": 1059,
     "pubkey": "<random-one-time-pubkey>",
     "content": "<nip44-encrypted-seal>",
     "tags": [["p", "<recipient-pubkey>"]],
     ...signed by random key...
   }
   ```

### Receiving a DM:

1. Receive kind 1059 event (gift wrap) addressed to your pubkey.
2. Decrypt content with your private key + wrap's pubkey → get seal (kind 13).
3. Decrypt seal content with your private key + seal's pubkey → get inner event (kind 14).
4. Verify the inner event's signature.
5. Display the inner event's content.

### NIP-44 Encryption (used by NIP-17):

NIP-44 uses:
- **Key derivation:** ECDH shared secret from sender privkey × recipient pubkey, then HKDF.
- **Encryption:** XChaCha20-Poly1305 AEAD.
- **Padding:** Content is padded to standard lengths to prevent length analysis.

```dart
// Pseudocode for core/nostr/nip17.dart

/// Encrypt and wrap a DM for sending.
NostrEvent createEncryptedDm({
  required String senderPrivateKeyHex,
  required String senderPubkeyHex,
  required String recipientPubkeyHex,
  required String plaintext,
}) {
  // 1. Create inner event (kind 14)
  final innerEvent = NostrEvent.create(
    privateKeyHex: senderPrivateKeyHex,
    pubkeyHex: senderPubkeyHex,
    kind: 14,
    content: plaintext,
    tags: [['p', recipientPubkeyHex]],
  );

  // 2. Create seal (kind 13) — encrypt inner event
  final sealContent = nip44Encrypt(
    senderPrivateKeyHex,
    recipientPubkeyHex,
    jsonEncode(innerEvent.toJson()),
  );
  final seal = NostrEvent.create(
    privateKeyHex: senderPrivateKeyHex,
    pubkeyHex: senderPubkeyHex,
    kind: 13,
    content: sealContent,
    tags: [],
  );

  // 3. Create gift wrap (kind 1059) — random key, encrypt seal
  final randomKeyPair = generateKeyPair();
  final wrapContent = nip44Encrypt(
    randomKeyPair.privateKeyHex,
    recipientPubkeyHex,
    jsonEncode(seal.toJson()),
  );
  final giftWrap = NostrEvent.create(
    privateKeyHex: randomKeyPair.privateKeyHex,
    pubkeyHex: randomKeyPair.publicKeyHex,
    kind: 1059,
    content: wrapContent,
    tags: [['p', recipientPubkeyHex]],
  );

  return giftWrap;
}

/// Decrypt a received gift wrap DM.
NostrEvent decryptDm({
  required String recipientPrivateKeyHex,
  required NostrEvent giftWrap,
}) {
  // 1. Decrypt gift wrap → seal
  final sealJson = nip44Decrypt(
    recipientPrivateKeyHex,
    giftWrap.pubkey,
    giftWrap.content,
  );
  final seal = NostrEvent.fromJson(jsonDecode(sealJson));

  // 2. Decrypt seal → inner event
  final innerJson = nip44Decrypt(
    recipientPrivateKeyHex,
    seal.pubkey,
    seal.content,
  );
  final innerEvent = NostrEvent.fromJson(jsonDecode(innerJson));

  // 3. Verify inner event signature
  assert(innerEvent.verifySignature());

  return innerEvent;
}
```

---

## 7. NIP-29 Group Management (used by Sprout channels)

Sprout channels use NIP-29-style group events. Key kinds:

| Kind | Purpose | Content |
|------|---------|---------|
| 9 | Group chat message | Message text |
| 9000 | Add user to group | — |
| 9001 | Remove user from group | — |
| 9002 | Edit group metadata | JSON metadata |
| 9005 | Delete event | — |
| 9006 | Create group | JSON metadata |
| 9021 | Group member list request | — |
| 9022 | Group admin list request | — |
| 10009 | User's group list (pinned) | — |
| 39000 | Group metadata | JSON (name, about, picture) |

### Channel message (kind 9 or kind 1):
```json
{
  "kind": 9,
  "content": "Hello channel!",
  "tags": [
    ["h", "<channel-id>"]
  ]
}
```

The `h` tag identifies which channel the message belongs to.

### Message edit (kind 9005 with reference):
```json
{
  "kind": 9,
  "content": "Edited message content",
  "tags": [
    ["h", "<channel-id>"],
    ["e", "<original-event-id>", "", "root"]
  ]
}
```

Note: The exact edit/delete mechanism may vary by Sprout relay implementation. Consult the Sprout relay API docs.

---

## 8. Presence (kind 20001)

Presence events indicate user online status:

```json
{
  "kind": 20001,
  "content": "",
  "tags": [
    ["status", "online"]
  ]
}
```

Status values: `"online"`, `"away"`, `"offline"`.

These are **ephemeral events** — the relay does not persist them long-term. Subscribe to them via WebSocket to see current user status.

---

## 9. Typing Indicators

Typing indicators are ephemeral events (the exact kind number is Sprout-specific). They indicate a user is typing in a channel:

```json
{
  "kind": 10001,
  "content": "",
  "tags": [
    ["h", "<channel-id>"]
  ]
}
```

These should:
- Be sent when the user starts typing (debounced, max once per 3 seconds).
- Expire after ~5 seconds if no new indicator is received.
- Never be persisted or displayed in message history.

---

## 10. Sprout Relay HTTP API

The Sprout relay exposes a REST API alongside the WebSocket. Base URL is derived from the WebSocket URL (same host, HTTP/HTTPS).

### Authentication headers

For HTTP requests, include one of:
```
Authorization: Bearer <api-token>
```
or:
```
X-Pubkey: <hex-pubkey>
```

### Key endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/channels` | List all channels the user can see |
| POST | `/api/channels` | Create a new channel |
| GET | `/api/channels/:id` | Get channel details |
| PUT | `/api/channels/:id` | Update channel metadata |
| GET | `/api/channels/:id/members` | List channel members |
| POST | `/api/channels/:id/members` | Add member to channel |
| DELETE | `/api/channels/:id/members/:pubkey` | Remove member |
| GET | `/api/events?channel=:id&limit=50&until=:timestamp` | Fetch events (paginated) |
| POST | `/api/events` | Publish event via HTTP (alternative to WebSocket) |
| GET | `/api/profiles/:pubkey` | Get user profile |
| PUT | `/api/profiles` | Update own profile |
| GET | `/api/search?q=:query&channel=:id` | Full-text search |
| POST | `/media/` | Upload media file (multipart) |
| GET | `/media/:hash` | Retrieve uploaded media |
| POST | `/api/tokens` | Create API token |
| GET | `/api/tokens` | List API tokens |
| DELETE | `/api/tokens/:id` | Revoke token |
| GET | `/api/home-feed` | Get home feed (mentions, activity, etc.) |
| GET | `/api/canvas/:channelId` | Get canvas content |
| PUT | `/api/canvas/:channelId` | Update canvas content |

### Pagination

History endpoints use cursor-based pagination:
- `limit` — max events to return (default 50).
- `until` — return events before this Unix timestamp.
- `since` — return events after this Unix timestamp.

To load older messages: send `until` = oldest message's `created_at` from current page.

---

## 11. Event Kinds Summary

| Kind | Name | NIP | Usage in H2OH |
|------|------|-----|---------------|
| 1 | Text note | NIP-01 | Channel messages (alternative to kind 9) |
| 7 | Reaction | NIP-25 | Emoji reactions to messages |
| 9 | Group chat message | NIP-29 | Channel messages |
| 13 | Seal | NIP-17 | DM encryption layer |
| 14 | Chat message (inner) | NIP-17 | DM plaintext |
| 1059 | Gift wrap | NIP-17 | DM outer envelope |
| 9000-9006 | Group management | NIP-29 | Channel admin operations |
| 10001 | Typing indicator | Custom | Ephemeral typing status |
| 20001 | Presence | Custom | Online/away/offline status |
| 22242 | Auth | NIP-42 | Relay authentication |
| 39000 | Group metadata | NIP-29 | Channel name/description |

---

## 12. Key Generation and Storage

### Generate a new keypair:

```dart
// Using secp256k1
final privateKey = generateRandomPrivateKey(); // 32 random bytes
final publicKey = derivePublicKey(privateKey);  // secp256k1 point, x-only (32 bytes)
```

### Bech32 encoding (for display/export):
- `nsec1...` — private key (bech32 with "nsec" prefix)
- `npub1...` — public key (bech32 with "npub" prefix)

### Storage:
- Private key → `flutter_secure_storage` with key `"nostr_private_key"`.
- Public key → derived from private key on load (never stored separately).
- On first launch: generate keypair, store private key.
- On subsequent launches: load private key from secure storage, derive pubkey.
- Optional: allow import of existing nsec key.
