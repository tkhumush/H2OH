# Phase 3 — DMs, Forums, and Member Management

## Prerequisites

- Phase 1 and Phase 2 are complete.
- Read `ARCHITECTURE.md`, `docs/NOSTR-PROTOCOL-REFERENCE.md`, `docs/DATA-MODELS.md`.
- The chat UI widgets from Phase 2 (message_bubble, message_input, markdown_body, reaction_bar) will be reused.

---

## Goal

At the end of Phase 3, the app should:

1. Send and receive encrypted direct messages (NIP-17 gift wrap).
2. Display a DM conversation list with last message preview.
3. Support group DMs (3+ participants).
4. Display forum channels with threaded discussion view.
5. Create new forum threads and reply to existing ones.
6. View and manage channel members (invite, remove, change roles).
7. Edit channel topic and purpose.

---

## Step 1: Implement NIP-17 Encrypted Direct Messages

**Files to create:**
- `lib/core/nostr/nip17.dart`
- `lib/core/nostr/nip44.dart`

**What to do:**

NIP-17 uses a 3-layer encryption scheme. See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 6.

### `nip44.dart` — Low-level encryption

NIP-44 encryption uses ECDH + HKDF + XChaCha20-Poly1305.

```dart
/// Derive shared secret from sender private key and recipient public key.
/// Uses ECDH on secp256k1, then HKDF with SHA-256.
Uint8List deriveSharedSecret(String senderPrivKeyHex, String recipientPubKeyHex) {
  // 1. Compute ECDH shared point: senderPrivKey * recipientPubKey
  // 2. Take the x-coordinate (32 bytes)
  // 3. Run HKDF-SHA256 with salt "nip44-v2" to derive a 32-byte key
  // Use pointycastle for secp256k1 and HKDF
}

/// Encrypt plaintext using NIP-44.
/// Returns base64-encoded ciphertext.
String nip44Encrypt(String senderPrivKeyHex, String recipientPubKeyHex, String plaintext) {
  final sharedKey = deriveSharedSecret(senderPrivKeyHex, recipientPubKeyHex);

  // 1. Pad plaintext to standard length (powers of 2, minimum 32 bytes)
  final padded = _padPlaintext(plaintext);

  // 2. Generate random 24-byte nonce
  final nonce = _randomBytes(24);

  // 3. Derive message key from shared key + nonce via HKDF
  final messageKey = _hkdfExpand(sharedKey, nonce, 76); // 32 chacha key + 12 chacha nonce + 32 hmac key

  // 4. Encrypt with XChaCha20-Poly1305
  final ciphertext = _xchacha20Poly1305Encrypt(
    key: messageKey.sublist(0, 32),
    nonce: messageKey.sublist(32, 44),
    plaintext: padded,
  );

  // 5. Compute HMAC-SHA256 over nonce + ciphertext
  final hmac = _hmacSha256(messageKey.sublist(44, 76), nonce + ciphertext);

  // 6. Return: version (1 byte: 0x02) + nonce + ciphertext + hmac, base64-encoded
  final payload = [0x02, ...nonce, ...ciphertext, ...hmac];
  return base64Encode(payload);
}

/// Decrypt NIP-44 ciphertext.
String nip44Decrypt(String recipientPrivKeyHex, String senderPubKeyHex, String ciphertextBase64) {
  final payload = base64Decode(ciphertextBase64);
  final version = payload[0]; // must be 0x02
  assert(version == 2);

  final nonce = payload.sublist(1, 25);       // 24 bytes
  final ciphertext = payload.sublist(25, payload.length - 32);
  final mac = payload.sublist(payload.length - 32);

  final sharedKey = deriveSharedSecret(recipientPrivKeyHex, senderPubKeyHex);
  final messageKey = _hkdfExpand(sharedKey, nonce, 76);

  // Verify HMAC
  final expectedMac = _hmacSha256(messageKey.sublist(44, 76), nonce + ciphertext);
  assert(constantTimeEquals(mac, expectedMac));

  // Decrypt
  final padded = _xchacha20Poly1305Decrypt(
    key: messageKey.sublist(0, 32),
    nonce: messageKey.sublist(32, 44),
    ciphertext: ciphertext,
  );

  // Unpad
  return _unpadPlaintext(padded);
}

/// NIP-44 padding: pad to next power of 2, minimum 32 bytes.
Uint8List _padPlaintext(String text) {
  final bytes = utf8.encode(text);
  final len = bytes.length;
  int paddedLen = 32;
  while (paddedLen < len) paddedLen *= 2;
  // Prefix with 2-byte big-endian length, then content, then zero padding
  final result = Uint8List(paddedLen + 2);
  result[0] = (len >> 8) & 0xff;
  result[1] = len & 0xff;
  result.setRange(2, 2 + len, bytes);
  return result;
}
```

**Important:** This is cryptographically sensitive. Use `pointycastle` for all primitives. Test against NIP-44 test vectors (available in the NIP-44 spec).

### `nip17.dart` — Gift Wrap Encryption

```dart
/// Encrypt and wrap a DM for sending.
NostrEvent createEncryptedDm({
  required String senderPrivKeyHex,
  required String senderPubKeyHex,
  required String recipientPubKeyHex,
  required String plaintext,
}) {
  // 1. Create inner event (kind 14)
  final innerEvent = NostrEvent.create(
    privateKeyHex: senderPrivKeyHex,
    pubkeyHex: senderPubKeyHex,
    kind: 14,
    content: plaintext,
    tags: [['p', recipientPubKeyHex]],
  );

  // 2. Create seal (kind 13) — encrypt inner event JSON
  final sealContent = nip44Encrypt(
    senderPrivKeyHex,
    recipientPubKeyHex,
    jsonEncode(innerEvent.toJson()),
  );
  final seal = NostrEvent.create(
    privateKeyHex: senderPrivKeyHex,
    pubkeyHex: senderPubKeyHex,
    kind: 13,
    content: sealContent,
    tags: [],
  );

  // 3. Create gift wrap (kind 1059) with random one-time key
  final randomKP = generateKeyPair();
  final wrapContent = nip44Encrypt(
    randomKP.privateKeyHex,
    recipientPubKeyHex,
    jsonEncode(seal.toJson()),
  );
  final giftWrap = NostrEvent.create(
    privateKeyHex: randomKP.privateKeyHex,
    pubkeyHex: randomKP.publicKeyHex,
    kind: 1059,
    content: wrapContent,
    tags: [['p', recipientPubKeyHex]],
  );

  return giftWrap;
}

/// Decrypt a received gift-wrapped DM.
NostrEvent decryptDm({
  required String recipientPrivKeyHex,
  required NostrEvent giftWrap,
}) {
  // 1. Decrypt gift wrap → seal
  final sealJson = nip44Decrypt(recipientPrivKeyHex, giftWrap.pubkey, giftWrap.content);
  final seal = NostrEvent.fromJson(jsonDecode(sealJson));

  // 2. Decrypt seal → inner event
  final innerJson = nip44Decrypt(recipientPrivKeyHex, seal.pubkey, seal.content);
  final innerEvent = NostrEvent.fromJson(jsonDecode(innerJson));

  // 3. Verify inner event signature
  assert(innerEvent.verifySignature());

  return innerEvent;
}
```

---

## Step 2: Implement DM Provider

**Files to create:**
- `lib/features/dm/dm_provider.dart`

**What to do:**

Manage DM conversations: listing, sending, receiving, decrypting.

```dart
@riverpod
class DmConversations extends _$DmConversations {
  final Map<String, List<Message>> _conversations = {};

  @override
  Future<List<DmConversation>> build() async {
    final keyPair = ref.watch(authProvider).valueOrNull!;

    // Subscribe to incoming gift wraps via WebSocket
    final relay = ref.watch(relayClientProvider);
    final sub = relay.subscribe({
      'kinds': [1059],
      '#p': [keyPair.publicKeyHex],
    });

    sub.events.listen((giftWrap) {
      _handleIncomingDm(giftWrap, keyPair);
    });

    ref.onDispose(() => relay.unsubscribe(sub.subscriptionId));

    // Return empty initially; populate as events arrive
    return _buildConversationList();
  }

  void _handleIncomingDm(NostrEvent giftWrap, KeyPair keyPair) {
    try {
      final innerEvent = decryptDm(
        recipientPrivKeyHex: keyPair.privateKeyHex,
        giftWrap: giftWrap,
      );
      final senderPubkey = innerEvent.pubkey;
      final message = Message(
        id: innerEvent.id,
        pubkey: senderPubkey,
        content: innerEvent.content,
        createdAt: innerEvent.createdAt,
        kind: innerEvent.kind,
        channelId: '', // DMs don't have a channel
      );

      _conversations.putIfAbsent(senderPubkey, () => []);
      _conversations[senderPubkey]!.add(message);
      state = AsyncData(_buildConversationList());
    } catch (e) {
      // Log decryption failure, skip event
      debugPrint('Failed to decrypt DM: $e');
    }
  }

  List<DmConversation> _buildConversationList() {
    return _conversations.entries.map((entry) {
      final messages = entry.value..sort((a, b) => b.createdAt.compareTo(a.createdAt));
      return DmConversation(
        counterpartyPubkey: entry.key,
        lastMessage: messages.first,
        unreadCount: 0, // TODO: track read state
      );
    }).toList()
      ..sort((a, b) => b.lastMessage.createdAt.compareTo(a.lastMessage.createdAt));
  }

  Future<bool> sendDm(String recipientPubkey, String plaintext) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final giftWrap = createEncryptedDm(
      senderPrivKeyHex: keyPair.privateKeyHex,
      senderPubKeyHex: keyPair.publicKeyHex,
      recipientPubKeyHex: recipientPubkey,
      plaintext: plaintext,
    );
    final relay = ref.read(relayClientProvider);
    return await relay.publish(giftWrap);
  }
}

/// Helper model for DM conversation list.
class DmConversation {
  final String counterpartyPubkey;
  final Message lastMessage;
  final int unreadCount;

  DmConversation({
    required this.counterpartyPubkey,
    required this.lastMessage,
    required this.unreadCount,
  });
}
```

---

## Step 3: Build DM List Screen

**Files to create:**
- `lib/features/dm/dm_list_screen.dart` (replace placeholder)

**UI:**
1. App bar: "Messages" title.
2. Body: `ListView` of DM conversations, sorted by most recent.
3. Each tile:
   - Leading: counterparty avatar (with presence dot from Phase 5).
   - Title: counterparty display name (resolve via `profileProvider`).
   - Subtitle: last message snippet (decrypted, truncated to 50 chars).
   - Trailing: timestamp (timeago format) + unread badge if `unreadCount > 0`.
4. Tap → navigate to `/dm/{counterpartyPubkey}`.
5. FAB: "New message" button → opens user search dialog.
6. Pull-to-refresh.
7. Empty state: "No conversations yet. Tap + to start one."

**User search for new DM:**
```dart
void _startNewDm(BuildContext context) {
  showDialog(
    context: context,
    builder: (_) => AlertDialog(
      title: const Text('New Message'),
      content: TextField(
        decoration: const InputDecoration(
          hintText: 'Enter npub or search by name',
        ),
        controller: _searchController,
      ),
      actions: [
        TextButton(onPressed: () => Navigator.pop(context), child: const Text('Cancel')),
        TextButton(
          onPressed: () {
            final input = _searchController.text.trim();
            String pubkey;
            if (input.startsWith('npub1')) {
              pubkey = bech32Decode(input); // Convert npub to hex
            } else {
              pubkey = input; // Assume hex pubkey for now
            }
            Navigator.pop(context);
            context.push('/dm/$pubkey');
          },
          child: const Text('Chat'),
        ),
      ],
    ),
  );
}
```

---

## Step 4: Build DM Chat Screen

**Files to create:**
- `lib/features/dm/dm_chat_screen.dart`

**Route:** `/dm/:counterpartyPubkey`

**UI:**
Identical to the stream chat screen (Phase 2 Step 8) with these differences:
1. App bar: counterparty display name + presence status.
2. No channel-specific features (no topic, no canvas link).
3. Messages are decrypted on display.
4. Sending calls `dmProvider.sendDm()` instead of `chatProvider.sendMessage()`.

**Reuse from Phase 2:**
- `MessageBubble` — same widget, same markdown rendering.
- `MessageInput` — same widget, triggers DM send.
- `ReactionBar` — reactions work the same way.
- `MarkdownBody` — same rendering.

```dart
class DmChatScreen extends ConsumerWidget {
  final String counterpartyPubkey;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final conversations = ref.watch(dmConversationsProvider).valueOrNull ?? [];
    final convo = conversations.firstWhereOrNull(
      (c) => c.counterpartyPubkey == counterpartyPubkey,
    );
    final messages = convo?.messages ?? [];
    final profile = ref.watch(profileProvider(counterpartyPubkey)).valueOrNull;

    return Scaffold(
      appBar: AppBar(title: Text(profile?.displayName ?? 'Unknown')),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              reverse: true,
              itemCount: messages.length,
              itemBuilder: (_, i) => MessageBubble(
                message: messages[messages.length - 1 - i],
                isOwnMessage: messages[messages.length - 1 - i].pubkey ==
                    ref.read(authProvider).valueOrNull?.publicKeyHex,
              ),
            ),
          ),
          MessageInput(
            onSend: (text) => ref.read(dmConversationsProvider.notifier)
                .sendDm(counterpartyPubkey, text),
          ),
        ],
      ),
    );
  }
}
```

---

## Step 5: Implement Forum Provider

**Files to create:**
- `lib/features/forum/forum_provider.dart`

**What to do:**

Forum channels display messages as threads. A thread is a root post + its replies.

```dart
@freezed
class ForumThread with _$ForumThread {
  const factory ForumThread({
    required Message rootPost,
    required List<Message> replies,
    required int replyCount,
    required int lastActivityAt,
    required List<String> participantPubkeys,
  }) = _ForumThread;
}

@riverpod
class ForumThreads extends _$ForumThreads {
  @override
  Future<List<ForumThread>> build(String channelId) async {
    final http = ref.watch(relayHttpProvider);

    // Fetch all events for this forum channel
    final events = await http.getEvents(channelId: channelId, limit: 100);
    final messages = events.map((e) => Message.fromNostrEvent(e)).toList();

    // Separate root posts (no threadRootId) from replies
    final rootPosts = messages.where((m) => m.threadRootId == null).toList();
    final replies = messages.where((m) => m.threadRootId != null).toList();

    // Group replies by root
    final threads = rootPosts.map((root) {
      final threadReplies = replies
          .where((r) => r.threadRootId == root.id)
          .toList()
        ..sort((a, b) => a.createdAt.compareTo(b.createdAt));

      final participants = {root.pubkey, ...threadReplies.map((r) => r.pubkey)}.toList();
      final lastActivity = threadReplies.isNotEmpty
          ? threadReplies.last.createdAt
          : root.createdAt;

      return ForumThread(
        rootPost: root,
        replies: threadReplies,
        replyCount: threadReplies.length,
        lastActivityAt: lastActivity,
        participantPubkeys: participants,
      );
    }).toList();

    // Sort by last activity (most recent first)
    threads.sort((a, b) => b.lastActivityAt.compareTo(a.lastActivityAt));

    // Subscribe to live updates
    final relay = ref.watch(relayClientProvider);
    final sub = relay.subscribe({'kinds': [1, 9], '#h': [channelId]});
    sub.events.listen((event) { /* rebuild thread list */ });
    ref.onDispose(() => relay.unsubscribe(sub.subscriptionId));

    return threads;
  }

  Future<bool> createThread(String title, String content) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    // Title is the first line, content follows
    final fullContent = '$title\n\n$content';
    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 9,
      content: fullContent,
      tags: [['h', arg]], // arg = channelId
    );
    return ref.read(relayClientProvider).publish(event);
  }

  Future<bool> replyToThread(String rootEventId, String content) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final tags = [
      ['h', arg],
      ['e', rootEventId, '', 'root'],
    ];
    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 9,
      content: content,
      tags: tags,
    );
    return ref.read(relayClientProvider).publish(event);
  }
}
```

---

## Step 6: Build Forum Screen

**Files to create:**
- `lib/features/forum/forum_screen.dart`

**Route:** `/channels/:channelId/forum`

**UI:**
1. App bar: channel name, action buttons (search, channel info).
2. Body: `ListView` of `ForumThread` cards.
3. Each thread card:
   - Title: first line of root post content (bold).
   - Author: avatar + display name.
   - Metadata: reply count, last activity timestamp, participant avatars (stacked).
   - Preview: first 2 lines of root post content.
4. Tap card → navigate to `/channels/{channelId}/thread/{rootEventId}`.
5. FAB: "New thread" button.
6. Sort options in app bar: "Most recent", "Most active", "Most replies".

**Thread card widget:**
```dart
class ForumThreadCard extends StatelessWidget {
  final ForumThread thread;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    final title = thread.rootPost.content.split('\n').first;
    final preview = thread.rootPost.content.split('\n').skip(1).take(2).join('\n');

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      child: ListTile(
        onTap: onTap,
        title: Text(title, style: const TextStyle(fontWeight: FontWeight.bold), maxLines: 1),
        subtitle: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            if (preview.isNotEmpty) Text(preview, maxLines: 2, overflow: TextOverflow.ellipsis),
            const SizedBox(height: 8),
            Row(
              children: [
                Icon(Icons.reply, size: 16),
                Text(' ${thread.replyCount}'),
                const SizedBox(width: 16),
                Text(timeago.format(DateTime.fromMillisecondsSinceEpoch(
                  thread.lastActivityAt * 1000))),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

**New thread dialog/screen:**
```dart
void _createNewThread(BuildContext context, WidgetRef ref, String channelId) {
  showModalBottomSheet(
    context: context,
    isScrollControlled: true,
    builder: (_) => Padding(
      padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          TextField(controller: _titleController, decoration: InputDecoration(hintText: 'Thread title')),
          TextField(controller: _contentController, decoration: InputDecoration(hintText: 'Write your post...'), maxLines: 5),
          ElevatedButton(
            onPressed: () async {
              await ref.read(forumThreadsProvider(channelId).notifier)
                  .createThread(_titleController.text, _contentController.text);
              Navigator.pop(context);
            },
            child: const Text('Post'),
          ),
        ],
      ),
    ),
  );
}
```

---

## Step 7: Build Forum Thread Screen

**Files to create:**
- `lib/features/forum/forum_thread_screen.dart`

**Route:** `/channels/:channelId/thread/:eventId` (shared with chat thread from Phase 2)

**UI:**
1. App bar: "Thread" title.
2. Top section: root post rendered as full `MessageBubble` (markdown, reactions, author info).
3. Divider with "N replies" label.
4. Reply list: chronological, each reply as `MessageBubble`.
5. Bottom: `MessageInput` for replying.

```dart
class ForumThreadScreen extends ConsumerWidget {
  final String channelId;
  final String rootEventId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final threads = ref.watch(forumThreadsProvider(channelId)).valueOrNull ?? [];
    final thread = threads.firstWhereOrNull((t) => t.rootPost.id == rootEventId);

    if (thread == null) return const Center(child: CircularProgressIndicator());

    return Scaffold(
      appBar: AppBar(title: const Text('Thread')),
      body: Column(
        children: [
          Expanded(
            child: ListView(
              children: [
                // Root post
                MessageBubble(message: thread.rootPost, isOwnMessage: false),
                Divider(),
                Padding(
                  padding: const EdgeInsets.all(8),
                  child: Text('${thread.replyCount} replies',
                    style: Theme.of(context).textTheme.bodySmall),
                ),
                // Replies
                ...thread.replies.map((reply) =>
                  MessageBubble(message: reply, isOwnMessage: false),
                ),
              ],
            ),
          ),
          MessageInput(
            onSend: (text) => ref.read(forumThreadsProvider(channelId).notifier)
                .replyToThread(rootEventId, text),
          ),
        ],
      ),
    );
  }
}
```

---

## Step 8: Implement Channel Member Management

**Files to create:**
- `lib/features/channels/channel_detail_screen.dart`
- `lib/features/channels/widgets/member_list.dart`
- `lib/core/models/member.dart`

### `core/models/member.dart`

Create the `ChannelMember` and `MemberRole` models. See `docs/DATA-MODELS.md` section 4.

### `channel_detail_screen.dart`

**Route:** `/channels/:channelId`

**UI:**
1. App bar: channel name, edit button (if owner/admin).
2. Sections:
   - **Info:** channel type badge, visibility badge, description.
   - **Topic:** editable text (tap to edit if owner/admin).
   - **Purpose:** editable text (tap to edit if owner/admin).
   - **Canvas:** "Open canvas" button → navigates to `/channels/{channelId}/canvas` (Phase 4).
   - **Members:** member list with role badges.
3. For owner/admin: "Invite member" button at top of member section.

**Topic/Purpose editing:**
```dart
void _editTopic(WidgetRef ref, String channelId, String currentTopic) {
  showDialog(
    builder: (_) => AlertDialog(
      title: const Text('Edit Topic'),
      content: TextField(controller: TextEditingController(text: currentTopic)),
      actions: [
        TextButton(child: const Text('Cancel'), onPressed: () => Navigator.pop(context)),
        TextButton(
          child: const Text('Save'),
          onPressed: () async {
            await ref.read(relayHttpProvider).updateChannel(channelId, topic: _controller.text);
            ref.invalidate(channelListProvider);
            Navigator.pop(context);
          },
        ),
      ],
    ),
  );
}
```

### `widgets/member_list.dart`

**Props:** `String channelId`, `bool canManage` (true if current user is owner/admin)

```dart
class MemberList extends ConsumerWidget {
  final String channelId;
  final bool canManage;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final membersAsync = ref.watch(channelMembersProvider(channelId));

    return membersAsync.when(
      data: (members) => Column(
        children: [
          if (canManage)
            ListTile(
              leading: const Icon(Icons.person_add),
              title: const Text('Invite member'),
              onTap: () => _inviteMember(context, ref),
            ),
          ...members.map((member) => ListTile(
            leading: CircleAvatar(/* member avatar */),
            title: Text(member.displayName ?? member.pubkey.substring(0, 12)),
            subtitle: Text(member.role.name),
            trailing: canManage && member.role != MemberRole.owner
                ? PopupMenuButton(
                    itemBuilder: (_) => [
                      PopupMenuItem(value: 'role', child: Text('Change role')),
                      PopupMenuItem(value: 'remove', child: Text('Remove')),
                    ],
                    onSelected: (value) {
                      if (value == 'remove') _removeMember(ref, member.pubkey);
                      if (value == 'role') _changeRole(context, ref, member);
                    },
                  )
                : null,
          )),
        ],
      ),
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
    );
  }

  void _inviteMember(BuildContext context, WidgetRef ref) {
    // Show dialog to enter npub or search by name
    // On confirm: call ref.read(relayHttpProvider).addMember(channelId, pubkey)
  }

  void _removeMember(WidgetRef ref, String pubkey) {
    ref.read(relayHttpProvider).removeMember(channelId, pubkey);
    ref.invalidate(channelMembersProvider(channelId));
  }

  void _changeRole(BuildContext context, WidgetRef ref, ChannelMember member) {
    // Show dropdown to select new role
    // Call relay API to update role
  }
}

@riverpod
Future<List<ChannelMember>> channelMembers(ChannelMembersRef ref, String channelId) async {
  final http = ref.watch(relayHttpProvider);
  return await http.getMembers(channelId);
}
```

---

## Step 9: Implement Group DMs

**What to do:**

Extend the DM system to support group conversations with 3+ participants.

**Sending a group DM:**
For each recipient in the group, create a separate gift wrap:
```dart
Future<void> sendGroupDm(List<String> recipientPubkeys, String plaintext) async {
  final keyPair = ref.read(authProvider).valueOrNull!;

  for (final recipientPubkey in recipientPubkeys) {
    // Include all participant pubkeys in the inner event tags
    final innerTags = recipientPubkeys.map((pk) => ['p', pk]).toList();

    final giftWrap = createEncryptedDm(
      senderPrivKeyHex: keyPair.privateKeyHex,
      senderPubKeyHex: keyPair.publicKeyHex,
      recipientPubKeyHex: recipientPubkey,
      plaintext: plaintext,
      additionalTags: innerTags, // All participants listed
    );
    await ref.read(relayClientProvider).publish(giftWrap);
  }
}
```

**Identifying group DMs:**
When decrypting a DM, check the inner event's `p` tags. If there are multiple `p` tags, it's a group DM. Group the conversation by the sorted set of all participant pubkeys (excluding self).

**DM list screen update:**
- Group DMs show multiple avatars (stacked) and names ("Alice, Bob, Carol").
- Tapping navigates to the same DM chat screen but with group context.

---

## Step 10: Update Routing

**Modify:** `lib/shared/router.dart`

Add new routes:
```dart
GoRoute(path: '/dm', builder: (_, __) => const DmListScreen()),
GoRoute(
  path: '/dm/:pubkey',
  builder: (_, state) => DmChatScreen(
    counterpartyPubkey: state.pathParameters['pubkey']!,
  ),
),
GoRoute(
  path: '/channels/:channelId',
  builder: (_, state) => ChannelDetailScreen(
    channelId: state.pathParameters['channelId']!,
  ),
),
GoRoute(
  path: '/channels/:channelId/forum',
  builder: (_, state) => ForumScreen(
    channelId: state.pathParameters['channelId']!,
  ),
),
```

---

## Verification Checklist

After completing Phase 3, verify:

- [ ] DM encryption/decryption works (send DM, receive and decrypt on other side).
- [ ] DM conversation list shows all conversations sorted by recency.
- [ ] New DM can be started by entering an npub.
- [ ] DM chat screen sends and receives encrypted messages.
- [ ] Forum channel displays threads as cards with title, reply count, activity.
- [ ] New forum thread can be created with title and content.
- [ ] Forum thread screen shows root post and replies.
- [ ] Replying in a forum thread works.
- [ ] Channel detail screen shows info, topic, purpose, members.
- [ ] Owner/admin can edit topic and purpose.
- [ ] Owner/admin can invite members.
- [ ] Owner/admin can remove members.
- [ ] Group DMs work with 3+ participants.
- [ ] All new routes navigate correctly.

---

## Files Created in Phase 3

```
lib/core/nostr/nip17.dart
lib/core/nostr/nip44.dart
lib/core/models/member.dart
lib/features/dm/dm_provider.dart
lib/features/dm/dm_list_screen.dart              (replaced placeholder)
lib/features/dm/dm_chat_screen.dart
lib/features/forum/forum_provider.dart
lib/features/forum/forum_screen.dart
lib/features/forum/forum_thread_screen.dart
lib/features/channels/channel_detail_screen.dart
lib/features/channels/widgets/member_list.dart
lib/shared/router.dart                           (modified — new routes)
```
