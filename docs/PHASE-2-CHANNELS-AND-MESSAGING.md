# Phase 2 — Channels and Messaging

## Prerequisites

- Phase 1 is complete (project scaffold, keys, relay client, auth, routing shell).
- Read `ARCHITECTURE.md` for project structure and dependencies.
- Read `docs/NOSTR-PROTOCOL-REFERENCE.md` for protocol details.
- Read `docs/DATA-MODELS.md` for all model definitions.

---

## Goal

Implement the channel system and real-time messaging. At the end of Phase 2, the app should:

1. Fetch and display a list of channels the user belongs to.
2. Allow browsing and joining public channels.
3. Allow creating new channels (stream or forum type).
4. Open a stream channel and display its message history.
5. Send and receive messages in real-time via WebSocket.
6. Render message content as Markdown with syntax highlighting.
7. Upload and display media (images) in messages.
8. Add emoji reactions to messages (NIP-25).
9. Reply in threads (NIP-10).
10. Show typing indicators.
11. Display code diffs with colored highlighting.

---

## Step 1: Implement Channel Data Model

**Files to create:**
- `lib/core/models/channel.dart`

**What to do:**

Create the `Channel` model, `ChannelType` enum, and `ChannelVisibility` enum using `freezed`.
See `docs/DATA-MODELS.md` section 3 for the exact definition.

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
    @Default([]) List<String> participants,
    @Default(false) bool isMember,
    String? createdBy,
  }) = _Channel;

  factory Channel.fromJson(Map<String, dynamic> json) =>
      _$ChannelFromJson(json);
}
```

**JSON key mapping:**
- `channelType` maps to `"channel_type"` in JSON.
- `memberCount` maps to `"member_count"`.
- `lastMessageAt` maps to `"last_message_at"`.
- `isMember` maps to `"is_member"`.
- `createdBy` maps to `"created_by"`.

Use `@JsonKey(name: 'channel_type')` annotations or configure `build.yaml` for snake_case.

**Run `dart run build_runner build`** after creating this file.

---

## Step 2: Implement Channel Provider

**Files to create:**
- `lib/features/channels/channel_provider.dart`

**What to do:**

Create a Riverpod `AsyncNotifierProvider` that manages the channel list.

```dart
@riverpod
class ChannelList extends _$ChannelList {
  @override
  Future<List<Channel>> build() async {
    final http = ref.watch(relayHttpProvider);
    final channels = await http.getChannels();
    return channels;
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final http = ref.read(relayHttpProvider);
      return await http.getChannels();
    });
  }
}
```

**Additional filtered providers:**

```dart
@riverpod
List<Channel> streamChannels(StreamChannelsRef ref) {
  final channels = ref.watch(channelListProvider).valueOrNull ?? [];
  return channels
      .where((c) => c.channelType == ChannelType.stream && c.isMember)
      .toList();
}

@riverpod
List<Channel> forumChannels(ForumChannelsRef ref) {
  final channels = ref.watch(channelListProvider).valueOrNull ?? [];
  return channels
      .where((c) => c.channelType == ChannelType.forum && c.isMember)
      .toList();
}
```

**WebSocket live updates:**

After initial HTTP fetch, subscribe to channel metadata events via WebSocket to update the list when channels are created/modified. Use the relay client to subscribe:

```dart
// In build() after HTTP fetch:
final relayClient = ref.watch(relayClientProvider);
final sub = relayClient.subscribe({'kinds': [39000]}); // group metadata events
sub.events.listen((event) {
  // Parse channel metadata from event, update state
  _handleChannelUpdate(event);
});
ref.onDispose(() => relayClient.unsubscribe(sub.subscriptionId));
```

---

## Step 3: Build Channel List Screen

**Files to create/modify:**
- `lib/features/channels/channel_list_screen.dart` (replace placeholder from Phase 1)
- `lib/features/channels/widgets/channel_tile.dart`

**What to do:**

Replace the placeholder `ChannelListScreen` with a real implementation.

### `channel_list_screen.dart`

**UI requirements:**
1. App bar: title "Channels", action button for "Browse channels" (navigates to `/channels/browse`).
2. Body: `RefreshIndicator` wrapping a `ListView`.
3. List is grouped into two sections with sticky headers:
   - "Stream Channels" — channels where `channelType == ChannelType.stream`
   - "Forum Channels" — channels where `channelType == ChannelType.forum`
4. Each channel renders as a `ChannelTile` widget.
5. Empty state: "No channels yet. Browse channels to join one."
6. FAB: FloatingActionButton with `+` icon, navigates to `/channels/create`.
7. Pull-to-refresh calls `ref.read(channelListProvider.notifier).refresh()`.

**Provider usage:**
```dart
class ChannelListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final streamChannels = ref.watch(streamChannelsProvider);
    final forumChannels = ref.watch(forumChannelsProvider);
    // Build the grouped list
  }
}
```

### `widgets/channel_tile.dart`

A `ListTile`-based widget displaying one channel.

**Props:** `Channel channel`, `VoidCallback onTap`

**Layout:**
- Leading: Icon based on type (`Icons.tag` for stream, `Icons.forum` for forum).
- Title: channel name.
- Subtitle: channel topic (if set) or description snippet.
- Trailing: member count badge + last message timestamp (formatted as "2m ago" using `timeago` package).

**On tap:** Navigate to `/channels/{channelId}/chat` for stream channels or `/channels/{channelId}/forum` for forum channels.

---

## Step 4: Build Channel Create Screen

**Files to create:**
- `lib/features/channels/channel_create_screen.dart`

**What to do:**

Build a form screen for creating new channels.

**UI:**
1. App bar: title "Create Channel", back button.
2. Form fields:
   - **Name** (`TextFormField`): required, validate non-empty, lowercase alphanumeric + hyphens only.
   - **Type** (`DropdownButtonFormField`): "Stream" or "Forum" (maps to `ChannelType`).
   - **Visibility** (`SwitchListTile`): "Private channel" toggle (maps to `ChannelVisibility`).
   - **Description** (`TextFormField`): optional, multiline, max 500 chars.
3. Submit button: "Create Channel".

**Behavior:**
```dart
Future<void> _createChannel() async {
  if (!_formKey.currentState!.validate()) return;
  setState(() => _isLoading = true);

  try {
    final http = ref.read(relayHttpProvider);
    final channel = await http.createChannel(
      name: _nameController.text.trim(),
      type: _selectedType,
      visibility: _isPrivate ? ChannelVisibility.private_ : ChannelVisibility.open,
      description: _descriptionController.text.trim(),
    );

    // Refresh channel list
    ref.read(channelListProvider.notifier).refresh();

    // Navigate to the new channel
    if (mounted) context.go('/channels/${channel.id}/chat');
  } catch (e) {
    // Show error snackbar
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Failed to create channel: $e')),
    );
  } finally {
    if (mounted) setState(() => _isLoading = false);
  }
}
```

---

## Step 5: Build Channel Browse/Discovery Screen

**Files to create:**
- `lib/features/channels/channel_browse_screen.dart`

**What to do:**

A screen to discover and join channels the user is NOT a member of.

**UI:**
1. App bar: title "Browse Channels", search field (filter by name as user types).
2. Body: list of ALL open channels from the relay.
3. Each tile: channel name, description, member count.
4. Trailing: "Join" button if `!channel.isMember`, or "Joined" chip if already a member.
5. Pull-to-refresh.

**Provider:**
```dart
@riverpod
Future<List<Channel>> allChannels(AllChannelsRef ref) async {
  final http = ref.watch(relayHttpProvider);
  return await http.getChannels(); // Returns all visible channels
}
```

**Join behavior:**
```dart
Future<void> _joinChannel(String channelId) async {
  final http = ref.read(relayHttpProvider);
  final keyPair = ref.read(authProvider).valueOrNull!;
  await http.addMember(channelId, keyPair.publicKeyHex);
  ref.invalidate(channelListProvider); // Refresh main channel list
  ref.invalidate(allChannelsProvider);  // Refresh browse list
}
```

**Search/filter:** Use a `TextEditingController` and filter the channel list locally:
```dart
final filtered = channels.where((c) =>
  c.name.toLowerCase().contains(query.toLowerCase()) ||
  c.description.toLowerCase().contains(query.toLowerCase())
).toList();
```

---

## Step 6: Implement Message Data Model

**Files to create:**
- `lib/core/models/message.dart`

**What to do:**

Create `Message`, `Reaction`, and `MediaAttachment` models using `freezed`.
See `docs/DATA-MODELS.md` sections 5, 6, 7 for exact definitions.

**Key implementation: `Message.fromNostrEvent(NostrEvent event)`**

This factory method parses a raw NostrEvent into a display-ready Message:

```dart
factory Message.fromNostrEvent(NostrEvent event) {
  // 1. Extract channelId from h-tag
  final hTag = event.tags.where((t) => t.isNotEmpty && t[0] == 'h').firstOrNull;
  final channelId = hTag != null && hTag.length > 1 ? hTag[1] : '';

  // 2. Parse NIP-10 thread references
  final threadRef = parseThreadTags(event.tags); // from nip10.dart

  // 3. Detect media URLs in content
  final mediaUrls = _extractMediaUrls(event.content);
  final attachments = mediaUrls.map((url) => MediaAttachment(
    url: url,
    mimeType: _guessMimeType(url),
  )).toList();

  return Message(
    id: event.id,
    pubkey: event.pubkey,
    content: event.content,
    createdAt: event.createdAt,
    kind: event.kind,
    channelId: channelId,
    threadRootId: threadRef.rootEventId,
    replyToId: threadRef.replyEventId,
    replyToAuthor: threadRef.mentionedPubkeys.isNotEmpty
        ? threadRef.mentionedPubkeys.first
        : null,
    attachments: attachments,
  );
}

/// Extract URLs that look like images/videos from message content.
static List<String> _extractMediaUrls(String content) {
  final regex = RegExp(r'https?://\S+\.(?:jpg|jpeg|png|gif|webp|mp4|mov)', caseSensitive: false);
  return regex.allMatches(content).map((m) => m.group(0)!).toList();
}

static String _guessMimeType(String url) {
  final ext = url.split('.').last.toLowerCase().split('?').first;
  return switch (ext) {
    'jpg' || 'jpeg' => 'image/jpeg',
    'png' => 'image/png',
    'gif' => 'image/gif',
    'webp' => 'image/webp',
    'mp4' => 'video/mp4',
    'mov' => 'video/quicktime',
    _ => 'application/octet-stream',
  };
}
```

**Run `dart run build_runner build`** after creating this file.

---

## Step 7: Implement Chat Provider

**Files to create:**
- `lib/features/chat/chat_provider.dart`

**What to do:**

Create a family provider parameterized by `channelId` that manages messages for a channel.

```dart
@riverpod
class ChatMessages extends _$ChatMessages {
  List<Message> _messages = [];
  String? _wsSubscriptionId;
  String? _reactionSubscriptionId;

  @override
  Future<List<Message>> build(String channelId) async {
    // 1. Fetch history via HTTP
    final http = ref.watch(relayHttpProvider);
    final events = await http.getEvents(channelId: channelId, limit: 50);
    _messages = events.map((e) => Message.fromNostrEvent(e)).toList();
    _messages.sort((a, b) => a.createdAt.compareTo(b.createdAt));

    // 2. Subscribe to live messages via WebSocket
    final relay = ref.watch(relayClientProvider);
    final sub = relay.subscribe({
      'kinds': [1, 9],
      '#h': [channelId],
    });
    _wsSubscriptionId = sub.subscriptionId;
    sub.events.listen(_handleNewMessage);

    // 3. Subscribe to reactions for visible messages
    _subscribeToReactions(relay);

    // 4. Cleanup on dispose
    ref.onDispose(() {
      if (_wsSubscriptionId != null) relay.unsubscribe(_wsSubscriptionId!);
      if (_reactionSubscriptionId != null) relay.unsubscribe(_reactionSubscriptionId!);
    });

    return _messages;
  }

  void _handleNewMessage(NostrEvent event) {
    final message = Message.fromNostrEvent(event);
    _messages = [..._messages, message];
    _messages.sort((a, b) => a.createdAt.compareTo(b.createdAt));
    state = AsyncData(_messages);
  }

  /// Load older messages (pagination).
  Future<void> loadMore() async {
    if (_messages.isEmpty) return;
    final oldestTimestamp = _messages.first.createdAt;
    final http = ref.read(relayHttpProvider);
    final older = await http.getEvents(
      channelId: arg, // the channelId from family
      limit: 50,
      until: oldestTimestamp,
    );
    final olderMessages = older.map((e) => Message.fromNostrEvent(e)).toList();
    _messages = [...olderMessages, ..._messages];
    state = AsyncData(_messages);
  }

  /// Send a new message to the channel.
  Future<bool> sendMessage(String content) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 9,
      content: content,
      tags: [['h', arg]], // arg = channelId
    );
    final relay = ref.read(relayClientProvider);
    return await relay.publish(event);
  }

  /// Edit an existing message.
  Future<bool> editMessage(String eventId, String newContent) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 9,
      content: newContent,
      tags: [
        ['h', arg],
        ['e', eventId, '', 'root'], // reference to original
      ],
    );
    final relay = ref.read(relayClientProvider);
    return await relay.publish(event);
  }

  /// Delete a message.
  Future<bool> deleteMessage(String eventId) async {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 9005,
      content: '',
      tags: [
        ['h', arg],
        ['e', eventId],
      ],
    );
    final relay = ref.read(relayClientProvider);
    return await relay.publish(event);
  }

  void _subscribeToReactions(RelayClient relay) {
    final messageIds = _messages.map((m) => m.id).toList();
    if (messageIds.isEmpty) return;
    final sub = relay.subscribe({
      'kinds': [7],
      '#e': messageIds,
    });
    _reactionSubscriptionId = sub.subscriptionId;
    sub.events.listen(_handleReaction);
  }

  void _handleReaction(NostrEvent event) {
    final reaction = Reaction.fromNostrEvent(event);
    _messages = _messages.map((m) {
      if (m.id == reaction.targetEventId) {
        return m.copyWith(reactions: [...m.reactions, reaction]);
      }
      return m;
    }).toList();
    state = AsyncData(_messages);
  }
}
```

---

## Step 8: Build Chat Screen

**Files to create:**
- `lib/features/chat/chat_screen.dart`
- `lib/features/chat/widgets/message_bubble.dart`
- `lib/features/chat/widgets/message_input.dart`

### `chat_screen.dart`

**Route:** `/channels/:channelId/chat`

**UI structure:**
```dart
class ChatScreen extends ConsumerStatefulWidget {
  final String channelId;
  const ChatScreen({super.key, required this.channelId});
  // ...
}

// build():
Scaffold(
  appBar: AppBar(
    title: Text(channel.name),  // from channelProvider
    subtitle: Text('${channel.memberCount} members'),
    actions: [
      IconButton(icon: Icon(Icons.search), onPressed: () { /* Phase 4 */ }),
      IconButton(icon: Icon(Icons.info_outline), onPressed: () {
        context.push('/channels/${channelId}'); // channel detail screen (Phase 3)
      }),
    ],
  ),
  body: Column(
    children: [
      // Message list (expanded, scrollable)
      Expanded(
        child: _buildMessageList(),
      ),
      // Typing indicator
      TypingIndicator(channelId: channelId),
      // Message input
      MessageInput(onSend: _sendMessage, onAttach: _pickAndUploadMedia),
    ],
  ),
)
```

**Message list implementation:**
- Use `ListView.builder` with `reverse: true` (newest at bottom, natural scroll).
- Wrap in `NotificationListener<ScrollNotification>` to detect scroll-to-top for pagination:
  ```dart
  if (notification.metrics.pixels >= notification.metrics.maxScrollExtent - 200) {
    ref.read(chatMessagesProvider(channelId).notifier).loadMore();
  }
  ```
- Each item is a `MessageBubble` widget.
- Use `ScrollController` to auto-scroll to bottom when new messages arrive.

### `widgets/message_bubble.dart`

**Props:** `Message message`, `bool isOwnMessage`, `VoidCallback onReply`, `VoidCallback onReact`, `VoidCallback? onEdit`, `VoidCallback? onDelete`

**Layout:**
```
┌──────────────────────────────────┐
│ [Avatar]  Author Name    2:34 PM │
│                                  │
│  Message content rendered as     │
│  Markdown (see Step 9)           │
│                                  │
│  [Image attachment if any]       │
│                                  │
│  [Thread: 3 replies →]           │
│                                  │
│  👍 2  🔥 1  [+]                 │
└──────────────────────────────────┘
```

- Avatar: `CircleAvatar` with `CachedNetworkImage` or initials fallback. Tap to view profile.
- Author name: bold text. Use `UserProfile.displayName` or truncated npub.
- Timestamp: formatted with `timeago` package (e.g. "2m ago").
- Content: rendered via `MarkdownBody` widget (Step 9).
- Attachments: rendered as `CachedNetworkImage` with loading placeholder.
- Thread indicator: "N replies" text button, tap navigates to `/channels/{channelId}/thread/{message.id}`.
- Reactions: `ReactionBar` widget (Step 11).

**Long-press context menu:**
```dart
showModalBottomSheet(
  children: [
    ListTile(title: Text('Reply in thread'), onTap: onReply),
    ListTile(title: Text('React'), onTap: onReact),
    if (isOwnMessage) ListTile(title: Text('Edit'), onTap: onEdit),
    if (isOwnMessage) ListTile(title: Text('Delete'), onTap: onDelete),
  ],
);
```

### `widgets/message_input.dart`

**Props:** `Function(String) onSend`, `VoidCallback onAttach`

**Layout:**
```
┌─────────────────────────────────────────────┐
│ [📎]  Type a message...            [Send ➤] │
└─────────────────────────────────────────────┘
```

- `TextField` with `TextEditingController`, multiline (max 5 lines before scroll).
- Attachment button: calls `onAttach` to pick media (Step 10).
- Send button: enabled only when text is non-empty. Calls `onSend(text)`, clears field.
- On text change: trigger typing indicator (Step 13).

---

## Step 9: Implement Markdown Rendering

**Files to create:**
- `lib/features/chat/widgets/markdown_body.dart`

**What to do:**

Create a reusable widget that renders message content as Markdown.

**Requirements:**
1. Use `flutter_markdown` package.
2. Support: bold, italic, strikethrough, inline code, code blocks, links, lists, blockquotes, headings.
3. Code blocks: use `flutter_highlight` for syntax highlighting. Detect language from the fenced code block tag (e.g. ` ```dart `).
4. Links: open in external browser via `url_launcher`.
5. Images in markdown: render as `CachedNetworkImage`.

```dart
class MarkdownBody extends StatelessWidget {
  final String content;
  const MarkdownBody({super.key, required this.content});

  @override
  Widget build(BuildContext context) {
    return MarkdownWidget(
      data: content,
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      styleSheet: MarkdownStyleSheet.fromTheme(Theme.of(context)).copyWith(
        codeblockDecoration: BoxDecoration(
          color: Theme.of(context).colorScheme.surfaceVariant,
          borderRadius: BorderRadius.circular(8),
        ),
        code: TextStyle(
          fontFamily: 'monospace',
          fontSize: 13,
          color: Theme.of(context).colorScheme.primary,
        ),
      ),
      builders: {
        'code': CodeBlockBuilder(), // Custom builder using flutter_highlight
      },
      onTapLink: (text, href, title) {
        if (href != null) launchUrl(Uri.parse(href));
      },
    );
  }
}
```

**CodeBlockBuilder** should:
1. Extract the language from the code block info string.
2. Use `HighlightView` from `flutter_highlight` with the appropriate language.
3. Fall back to plain monospace text if language is not recognized.
4. Wrap in a container with padding, rounded corners, and a "copy" button in the top-right.

---

## Step 10: Implement Media Upload and Display

**Files to create:**
- `lib/core/services/media_service.dart`

**What to do:**

Implement media picking, compression, upload, and display.

```dart
class MediaService {
  final RelayHttp _http;
  final ImagePicker _picker = ImagePicker();

  MediaService(this._http);

  /// Pick an image from gallery or camera and upload it.
  Future<MediaAttachment?> pickAndUpload({
    ImageSource source = ImageSource.gallery,
  }) async {
    // 1. Pick image
    final xfile = await _picker.pickImage(
      source: source,
      maxWidth: 1920,
      maxHeight: 1920,
      imageQuality: 85, // compress JPEG quality
    );
    if (xfile == null) return null;

    // 2. Upload to relay
    final file = File(xfile.path);
    final attachment = await _http.uploadMedia(file);
    return attachment;
  }
}
```

**Riverpod provider:**
```dart
@riverpod
MediaService mediaService(MediaServiceRef ref) {
  final http = ref.watch(relayHttpProvider);
  return MediaService(http);
}
```

**Usage in MessageInput:**
When user taps the attachment button:
```dart
final media = await ref.read(mediaServiceProvider).pickAndUpload();
if (media != null) {
  // Insert the media URL into the message content
  _controller.text += '\n${media.url}';
  // Or send immediately as a message with just the URL
}
```

**Display in MessageBubble:**
For each `MediaAttachment` in `message.attachments`:
```dart
ClipRRect(
  borderRadius: BorderRadius.circular(12),
  child: CachedNetworkImage(
    imageUrl: attachment.thumbnailUrl ?? attachment.url,
    placeholder: (_, __) => Container(
      width: 200, height: 150,
      color: Colors.grey[300],
      child: const Center(child: CircularProgressIndicator()),
    ),
    errorWidget: (_, __, ___) => const Icon(Icons.broken_image),
    fit: BoxFit.cover,
    maxWidth: 300,
  ),
)
```

Tap on image → open full-size image in a dialog/fullscreen viewer.

---

## Step 11: Implement Reactions (NIP-25)

**Files to create:**
- `lib/core/nostr/nip25.dart`
- `lib/features/chat/widgets/reaction_bar.dart`

### `core/nostr/nip25.dart`

See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 5.

```dart
NostrEvent createReaction({
  required String privateKeyHex,
  required String pubkeyHex,
  required String targetEventId,
  required String targetPubkey,
  String content = '+', // "+" or emoji string like "🔥"
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
```

### `widgets/reaction_bar.dart`

**Props:** `List<Reaction> reactions`, `String messageId`, `String messagePubkey`

**Layout:**
```
[👍 2] [🔥 1] [+]
```

- Group reactions by `content` (emoji string).
- Each group: chip showing emoji + count. Highlighted if current user has reacted.
- Tap a chip: toggle your own reaction (add if not present, publish a new kind 7 event; if already reacted, the UI can indicate it but NIP-25 doesn't support un-reacting — just show it as toggled).
- `[+]` button: opens `emoji_picker_flutter` bottom sheet. On emoji select, publish reaction.

```dart
class ReactionBar extends ConsumerWidget {
  final List<Reaction> reactions;
  final String messageId;
  final String messagePubkey;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final grouped = _groupReactions(reactions);
    final myPubkey = ref.watch(authProvider).valueOrNull?.publicKeyHex;

    return Wrap(
      spacing: 4,
      children: [
        ...grouped.entries.map((entry) {
          final emoji = entry.key;
          final count = entry.value.length;
          final iReacted = entry.value.any((r) => r.pubkey == myPubkey);
          return ActionChip(
            label: Text('$emoji $count'),
            backgroundColor: iReacted ? Theme.of(context).colorScheme.primaryContainer : null,
            onPressed: () => _toggleReaction(ref, emoji),
          );
        }),
        ActionChip(
          label: const Text('+'),
          onPressed: () => _showEmojiPicker(context, ref),
        ),
      ],
    );
  }

  void _toggleReaction(WidgetRef ref, String emoji) {
    final keyPair = ref.read(authProvider).valueOrNull!;
    final event = createReaction(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      targetEventId: messageId,
      targetPubkey: messagePubkey,
      content: emoji,
    );
    ref.read(relayClientProvider).publish(event);
  }

  void _showEmojiPicker(BuildContext context, WidgetRef ref) {
    showModalBottomSheet(
      context: context,
      builder: (_) => SizedBox(
        height: 300,
        child: EmojiPicker(
          onEmojiSelected: (category, emoji) {
            Navigator.pop(context);
            _toggleReaction(ref, emoji.emoji);
          },
        ),
      ),
    );
  }
}
```

---

## Step 12: Implement Thread Replies (NIP-10)

**Files to create:**
- `lib/core/nostr/nip10.dart`
- `lib/features/chat/thread_screen.dart`

### `core/nostr/nip10.dart`

See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 4 for the full implementation.

Implement:
- `ThreadReference parseThreadTags(List<List<String>> tags)` — extract root, reply, mentions.
- `List<List<String>> buildReplyTags({required NostrEvent replyingTo, String? threadRootId})` — build tags for a reply.

### `thread_screen.dart`

**Route:** `/channels/:channelId/thread/:eventId`

**UI:**
1. App bar: "Thread" title, back button.
2. Top: the root message displayed as a full `MessageBubble` (non-tappable for thread).
3. Below: list of reply messages, sorted chronologically.
4. Bottom: `MessageInput` for posting a reply.

**Provider for thread messages:**
```dart
@riverpod
Future<List<Message>> threadMessages(ThreadMessagesRef ref, String rootEventId) async {
  final http = ref.watch(relayHttpProvider);
  final events = await http.getEvents(
    channelId: '', // Thread replies reference the root event, not channel
    // Use filter: #e tag matching rootEventId
  );
  // Filter to only replies that reference rootEventId as root
  return events
      .map((e) => Message.fromNostrEvent(e))
      .where((m) => m.threadRootId == rootEventId || m.replyToId == rootEventId)
      .toList()
    ..sort((a, b) => a.createdAt.compareTo(b.createdAt));
}
```

Alternatively, use WebSocket subscription:
```dart
relay.subscribe({
  'kinds': [1, 9],
  '#e': [rootEventId],
});
```

**Sending a thread reply:**
```dart
Future<bool> sendThreadReply(String content, NostrEvent rootEvent) async {
  final keyPair = ref.read(authProvider).valueOrNull!;
  final tags = buildReplyTags(replyingTo: rootEvent, threadRootId: rootEvent.id);
  // Add h-tag for channel context
  tags.add(['h', channelId]);

  final event = NostrEvent.create(
    privateKeyHex: keyPair.privateKeyHex,
    pubkeyHex: keyPair.publicKeyHex,
    kind: 9,
    content: content,
    tags: tags,
  );
  return ref.read(relayClientProvider).publish(event);
}
```

**Thread count in main chat:**
In `ChatProvider`, maintain a `Map<String, int> threadCounts` that counts replies per root event ID. Display in `MessageBubble` as "N replies" link.

---

## Step 13: Implement Typing Indicators

**Files to create:**
- `lib/features/chat/widgets/typing_indicator.dart`

**What to do:**

### Sending typing indicators:

In `MessageInput`, when user types:
```dart
Timer? _typingTimer;
DateTime? _lastTypingSent;

void _onTextChanged(String text) {
  final now = DateTime.now();
  // Debounce: only send every 3 seconds
  if (_lastTypingSent == null ||
      now.difference(_lastTypingSent!) > const Duration(seconds: 3)) {
    _sendTypingIndicator();
    _lastTypingSent = now;
  }

  // Reset expiry timer
  _typingTimer?.cancel();
  _typingTimer = Timer(const Duration(seconds: 5), () {
    _lastTypingSent = null; // Allow sending again after pause
  });
}

void _sendTypingIndicator() {
  final keyPair = ref.read(authProvider).valueOrNull!;
  final event = NostrEvent.create(
    privateKeyHex: keyPair.privateKeyHex,
    pubkeyHex: keyPair.publicKeyHex,
    kind: 10001,
    content: '',
    tags: [['h', channelId]],
  );
  ref.read(relayClientProvider).publish(event);
}
```

### Receiving typing indicators:

```dart
@riverpod
class TypingUsers extends _$TypingUsers {
  final Map<String, Timer> _expiryTimers = {};

  @override
  List<String> build(String channelId) {
    // Subscribe to typing events for this channel
    final relay = ref.watch(relayClientProvider);
    final sub = relay.subscribe({
      'kinds': [10001],
      '#h': [channelId],
    });

    sub.events.listen((event) {
      final pubkey = event.pubkey;
      final myPubkey = ref.read(authProvider).valueOrNull?.publicKeyHex;
      if (pubkey == myPubkey) return; // Ignore own typing

      // Add to typing list
      _expiryTimers[pubkey]?.cancel();
      _expiryTimers[pubkey] = Timer(const Duration(seconds: 5), () {
        _removeTyper(pubkey);
      });

      if (!state.contains(pubkey)) {
        state = [...state, pubkey];
      }
    });

    ref.onDispose(() {
      relay.unsubscribe(sub.subscriptionId);
      for (final timer in _expiryTimers.values) timer.cancel();
    });

    return [];
  }

  void _removeTyper(String pubkey) {
    _expiryTimers.remove(pubkey);
    state = state.where((p) => p != pubkey).toList();
  }
}
```

### `typing_indicator.dart` widget:

```dart
class TypingIndicator extends ConsumerWidget {
  final String channelId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final typingPubkeys = ref.watch(typingUsersProvider(channelId));
    if (typingPubkeys.isEmpty) return const SizedBox.shrink();

    // Resolve display names
    final names = typingPubkeys.map((pk) {
      final profile = ref.watch(profileProvider(pk)).valueOrNull;
      return profile?.displayName ?? 'Someone';
    }).toList();

    final text = switch (names.length) {
      1 => '${names[0]} is typing...',
      2 => '${names[0]} and ${names[1]} are typing...',
      _ => '${names[0]} and ${names.length - 1} others are typing...',
    };

    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: Text(text, style: Theme.of(context).textTheme.bodySmall),
    );
  }
}
```

---

## Step 14: Implement Diff Display

**Files to create:**
- `lib/features/chat/widgets/diff_view.dart`

**What to do:**

Detect and render unified diffs in message content.

**Detection:** A message contains a diff if:
1. It has a fenced code block with language `diff`, OR
2. Content lines start with `@@`, `+`, `-`, `---`, `+++` patterns.

```dart
class DiffView extends StatelessWidget {
  final String diffContent;

  @override
  Widget build(BuildContext context) {
    final lines = diffContent.split('\n');

    return Container(
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surfaceVariant,
        borderRadius: BorderRadius.circular(8),
      ),
      padding: const EdgeInsets.all(8),
      child: SingleChildScrollView(
        scrollDirection: Axis.horizontal,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: lines.map((line) => _buildDiffLine(context, line)).toList(),
        ),
      ),
    );
  }

  Widget _buildDiffLine(BuildContext context, String line) {
    Color? bgColor;
    Color textColor = Theme.of(context).colorScheme.onSurface;

    if (line.startsWith('+++') || line.startsWith('---')) {
      bgColor = Colors.grey.withOpacity(0.3);
      textColor = Theme.of(context).colorScheme.onSurfaceVariant;
    } else if (line.startsWith('+')) {
      bgColor = Colors.green.withOpacity(0.2);
    } else if (line.startsWith('-')) {
      bgColor = Colors.red.withOpacity(0.2);
    } else if (line.startsWith('@@')) {
      bgColor = Colors.blue.withOpacity(0.15);
      textColor = Colors.blue;
    }

    return Container(
      width: double.infinity,
      color: bgColor,
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 1),
      child: Text(
        line,
        style: TextStyle(
          fontFamily: 'monospace',
          fontSize: 12,
          color: textColor,
        ),
      ),
    );
  }
}
```

**Integration with MarkdownBody:**
In the custom `CodeBlockBuilder` (Step 9), detect when the language is `"diff"` and render a `DiffView` instead of a `HighlightView`.

---

## Step 15: Update Routing

**Modify:** `lib/shared/router.dart`

Add routes for the new screens:

```dart
GoRoute(
  path: '/channels',
  builder: (_, __) => const ChannelListScreen(),
),
GoRoute(
  path: '/channels/browse',
  builder: (_, __) => const ChannelBrowseScreen(),
),
GoRoute(
  path: '/channels/create',
  builder: (_, __) => const ChannelCreateScreen(),
),
GoRoute(
  path: '/channels/:channelId/chat',
  builder: (_, state) => ChatScreen(
    channelId: state.pathParameters['channelId']!,
  ),
),
GoRoute(
  path: '/channels/:channelId/thread/:eventId',
  builder: (_, state) => ThreadScreen(
    channelId: state.pathParameters['channelId']!,
    rootEventId: state.pathParameters['eventId']!,
  ),
),
```

---

## Verification Checklist

After completing Phase 2, verify:

- [ ] Channel list loads and displays grouped by type (stream/forum).
- [ ] Pull-to-refresh reloads channels.
- [ ] Channel create form validates input and creates a channel on the relay.
- [ ] Channel browse screen shows all channels with join/joined status.
- [ ] Joining a channel updates the channel list.
- [ ] Chat screen loads message history for a channel.
- [ ] Sending a message appears in the chat immediately.
- [ ] Messages from other users appear in real-time via WebSocket.
- [ ] Markdown renders correctly (bold, code, links, code blocks with highlighting).
- [ ] Images can be picked, uploaded, and displayed in messages.
- [ ] Reactions can be added via emoji picker and display below messages.
- [ ] Thread replies work: tap "Reply in thread", send reply, see thread count in main chat.
- [ ] Typing indicators show when other users type and when local user types.
- [ ] Diff content renders with green/red line coloring.
- [ ] Pagination loads older messages on scroll-to-top.
- [ ] All navigation routes work correctly.

---

## Files Created in Phase 2

```
lib/core/models/channel.dart
lib/core/models/message.dart              (Message, Reaction, MediaAttachment)
lib/core/nostr/nip10.dart
lib/core/nostr/nip25.dart
lib/core/services/media_service.dart
lib/features/channels/channel_provider.dart
lib/features/channels/channel_list_screen.dart   (replaced placeholder)
lib/features/channels/channel_create_screen.dart
lib/features/channels/channel_browse_screen.dart
lib/features/channels/widgets/channel_tile.dart
lib/features/chat/chat_provider.dart
lib/features/chat/chat_screen.dart
lib/features/chat/thread_screen.dart
lib/features/chat/widgets/message_bubble.dart
lib/features/chat/widgets/message_input.dart
lib/features/chat/widgets/markdown_body.dart
lib/features/chat/widgets/reaction_bar.dart
lib/features/chat/widgets/typing_indicator.dart
lib/features/chat/widgets/diff_view.dart
lib/shared/router.dart                           (modified — new routes)
```
