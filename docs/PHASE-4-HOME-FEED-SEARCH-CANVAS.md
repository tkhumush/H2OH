# Phase 4 — Home Feed, Search, and Canvas

## Prerequisites

- Phases 1-3 are complete.
- Read `ARCHITECTURE.md`, `docs/NOSTR-PROTOCOL-REFERENCE.md`, `docs/DATA-MODELS.md`.

---

## Goal

At the end of Phase 4, the app should:

1. Display a home feed with 4 categories (mentions, needs action, activity, agent activity).
2. Support global full-text search across all messages.
3. Support in-channel search.
4. Provide a rich-text canvas editor per channel.
5. Render code diffs with colored highlighting in messages.

---

## Step 1: Implement Home Feed Provider

**Files to create:**
- `lib/features/home/home_provider.dart`

```dart
@riverpod
class HomeFeed extends _$HomeFeed {
  @override
  Future<Map<FeedCategory, List<HomeFeedItem>>> build() async {
    final http = ref.watch(relayHttpProvider);
    final feed = await http.getHomeFeed();

    // Subscribe to live updates for new mentions/activity
    final relay = ref.watch(relayClientProvider);
    final keyPair = ref.watch(authProvider).valueOrNull!;
    final sub = relay.subscribe({
      'kinds': [1, 9],
      '#p': [keyPair.publicKeyHex], // Events mentioning me
    });

    sub.events.listen((event) {
      // Add to mentions category, refresh state
      _handleNewMention(event);
    });

    ref.onDispose(() => relay.unsubscribe(sub.subscriptionId));
    return feed;
  }

  void _handleNewMention(NostrEvent event) {
    final current = state.valueOrNull ?? {};
    final mentions = current[FeedCategory.mentions] ?? [];
    final newItem = HomeFeedItem(
      id: event.id,
      category: FeedCategory.mentions,
      title: 'New mention',
      snippet: event.content.length > 100
          ? '${event.content.substring(0, 100)}...'
          : event.content,
      timestamp: event.createdAt,
      channelId: _extractChannelId(event),
      channelName: '', // Resolve later
      authorPubkey: event.pubkey,
      eventId: event.id,
    );
    state = AsyncData({
      ...current,
      FeedCategory.mentions: [newItem, ...mentions],
    });
  }

  String _extractChannelId(NostrEvent event) {
    final hTag = event.tags.where((t) => t.isNotEmpty && t[0] == 'h').firstOrNull;
    return hTag != null && hTag.length > 1 ? hTag[1] : '';
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      return ref.read(relayHttpProvider).getHomeFeed();
    });
  }

  void markAsRead(String itemId) {
    final current = state.valueOrNull ?? {};
    final updated = current.map((category, items) => MapEntry(
      category,
      items.map((item) =>
        item.id == itemId ? item.copyWith(isRead: true) : item
      ).toList(),
    ));
    state = AsyncData(updated);
    // Persist read state to SharedPreferences
  }
}
```

---

## Step 2: Build Home Screen

**Files to modify:**
- `lib/features/home/home_screen.dart` (replace placeholder)

**Files to create:**
- `lib/features/home/widgets/mentions_card.dart`
- `lib/features/home/widgets/needs_action_card.dart`
- `lib/features/home/widgets/activity_card.dart`
- `lib/features/home/widgets/agent_activity_card.dart`

### `home_screen.dart`

```dart
class HomeScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final feedAsync = ref.watch(homeFeedProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Home'),
        actions: [
          IconButton(
            icon: const Icon(Icons.search),
            onPressed: () => context.push('/search'),
          ),
        ],
      ),
      body: feedAsync.when(
        data: (feed) => RefreshIndicator(
          onRefresh: () => ref.read(homeFeedProvider.notifier).refresh(),
          child: ListView(
            children: [
              _FeedSection(
                title: 'Mentions',
                items: feed[FeedCategory.mentions] ?? [],
                emptyText: 'No mentions yet.',
                cardBuilder: (item) => MentionsCard(item: item),
              ),
              _FeedSection(
                title: 'Needs Action',
                items: feed[FeedCategory.needsAction] ?? [],
                emptyText: 'Nothing needs your attention.',
                cardBuilder: (item) => NeedsActionCard(item: item),
              ),
              _FeedSection(
                title: 'Activity',
                items: feed[FeedCategory.activity] ?? [],
                emptyText: 'No recent activity.',
                cardBuilder: (item) => ActivityCard(item: item),
              ),
              _FeedSection(
                title: 'Agent Activity',
                items: feed[FeedCategory.agentActivity] ?? [],
                emptyText: 'No agent activity.',
                cardBuilder: (item) => AgentActivityCard(item: item),
              ),
            ],
          ),
        ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error: $e')),
      ),
    );
  }
}

class _FeedSection extends StatefulWidget {
  final String title;
  final List<HomeFeedItem> items;
  final String emptyText;
  final Widget Function(HomeFeedItem) cardBuilder;

  @override
  State<_FeedSection> createState() => _FeedSectionState();
}

class _FeedSectionState extends State<_FeedSection> {
  bool _expanded = true;

  @override
  Widget build(BuildContext context) {
    final unreadCount = widget.items.where((i) => !i.isRead).length;
    return Column(
      children: [
        ListTile(
          title: Row(children: [
            Text(widget.title, style: Theme.of(context).textTheme.titleMedium),
            if (unreadCount > 0) ...[
              const SizedBox(width: 8),
              Badge(label: Text('$unreadCount')),
            ],
          ]),
          trailing: Icon(_expanded ? Icons.expand_less : Icons.expand_more),
          onTap: () => setState(() => _expanded = !_expanded),
        ),
        if (_expanded) ...[
          if (widget.items.isEmpty)
            Padding(
              padding: const EdgeInsets.all(16),
              child: Text(widget.emptyText, style: Theme.of(context).textTheme.bodySmall),
            )
          else
            ...widget.items.map(widget.cardBuilder),
        ],
      ],
    );
  }
}
```

### Feed card widgets

Each card follows the same pattern — tapping navigates to the relevant channel/message:

```dart
class MentionsCard extends StatelessWidget {
  final HomeFeedItem item;

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 4),
      color: item.isRead ? null : Theme.of(context).colorScheme.primaryContainer.withOpacity(0.3),
      child: ListTile(
        leading: const Icon(Icons.alternate_email),
        title: Text(item.title, maxLines: 1, overflow: TextOverflow.ellipsis),
        subtitle: Text(item.snippet, maxLines: 2, overflow: TextOverflow.ellipsis),
        trailing: Text(timeago.format(
          DateTime.fromMillisecondsSinceEpoch(item.timestamp * 1000)),
          style: Theme.of(context).textTheme.bodySmall,
        ),
        onTap: () {
          // Mark as read and navigate
          context.push('/channels/${item.channelId}/chat');
        },
      ),
    );
  }
}
```

`NeedsActionCard`: same layout, leading icon `Icons.flag`, may include action buttons (approve/reject).
`ActivityCard`: same layout, leading icon `Icons.notifications`.
`AgentActivityCard`: same layout, leading icon `Icons.smart_toy`.

---

## Step 3: Implement Search Provider

**Files to create:**
- `lib/features/search/search_provider.dart`

```dart
@riverpod
class SearchState extends _$SearchState {
  Timer? _debounce;

  @override
  AsyncValue<List<SearchResult>> build() {
    ref.onDispose(() => _debounce?.cancel());
    return const AsyncData([]);
  }

  /// Search with debounce (300ms).
  void search(String query, {String? channelId}) {
    if (query.trim().isEmpty) {
      state = const AsyncData([]);
      return;
    }

    _debounce?.cancel();
    _debounce = Timer(const Duration(milliseconds: 300), () {
      _performSearch(query.trim(), channelId: channelId);
    });
  }

  Future<void> _performSearch(String query, {String? channelId}) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final http = ref.read(relayHttpProvider);
      final results = await http.search(query, channelId: channelId);
      // Save to recent searches
      _saveRecentSearch(query);
      return results;
    });
  }

  void clearResults() {
    state = const AsyncData([]);
  }

  Future<void> _saveRecentSearch(String query) async {
    final prefs = await SharedPreferences.getInstance();
    final recent = prefs.getStringList('recent_searches') ?? [];
    recent.remove(query); // Remove duplicate
    recent.insert(0, query); // Add to front
    if (recent.length > 10) recent.removeLast(); // Max 10
    await prefs.setStringList('recent_searches', recent);
  }
}

@riverpod
Future<List<String>> recentSearches(RecentSearchesRef ref) async {
  final prefs = await SharedPreferences.getInstance();
  return prefs.getStringList('recent_searches') ?? [];
}
```

---

## Step 4: Build Global Search Screen

**Files to create:**
- `lib/features/search/search_screen.dart`
- `lib/features/search/widgets/search_result_tile.dart`

**Route:** `/search`

### `search_screen.dart`

```dart
class SearchScreen extends ConsumerStatefulWidget {
  @override
  ConsumerState<SearchScreen> createState() => _SearchScreenState();
}

class _SearchScreenState extends ConsumerState<SearchScreen> {
  final _controller = TextEditingController();
  final _focusNode = FocusNode();

  @override
  void initState() {
    super.initState();
    // Auto-focus the search field
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _focusNode.requestFocus();
    });
  }

  @override
  Widget build(BuildContext context) {
    final searchState = ref.watch(searchStateProvider);
    final recentSearches = ref.watch(recentSearchesProvider).valueOrNull ?? [];

    return Scaffold(
      appBar: AppBar(
        title: TextField(
          controller: _controller,
          focusNode: _focusNode,
          decoration: const InputDecoration(
            hintText: 'Search messages...',
            border: InputBorder.none,
          ),
          onChanged: (query) {
            ref.read(searchStateProvider.notifier).search(query);
          },
        ),
        actions: [
          if (_controller.text.isNotEmpty)
            IconButton(
              icon: const Icon(Icons.clear),
              onPressed: () {
                _controller.clear();
                ref.read(searchStateProvider.notifier).clearResults();
              },
            ),
        ],
      ),
      body: searchState.when(
        data: (results) {
          // Show recent searches when no query
          if (_controller.text.isEmpty) {
            return _buildRecentSearches(recentSearches);
          }
          if (results.isEmpty) {
            return const Center(child: Text('No results found'));
          }
          return ListView.builder(
            itemCount: results.length,
            itemBuilder: (_, i) => SearchResultTile(
              result: results[i],
              onTap: () {
                // Navigate to the message in its channel
                context.push('/channels/${results[i].channelId}/chat');
                // TODO: scroll to specific eventId
              },
            ),
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Search error: $e')),
      ),
    );
  }

  Widget _buildRecentSearches(List<String> searches) {
    if (searches.isEmpty) return const SizedBox.shrink();
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: Text('Recent searches', style: Theme.of(context).textTheme.titleSmall),
        ),
        ...searches.map((q) => ListTile(
          leading: const Icon(Icons.history),
          title: Text(q),
          onTap: () {
            _controller.text = q;
            ref.read(searchStateProvider.notifier).search(q);
          },
        )),
      ],
    );
  }
}
```

### `widgets/search_result_tile.dart`

```dart
class SearchResultTile extends StatelessWidget {
  final SearchResult result;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      onTap: onTap,
      leading: const Icon(Icons.message),
      title: Row(
        children: [
          Container(
            padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.secondaryContainer,
              borderRadius: BorderRadius.circular(4),
            ),
            child: Text('#${result.channelName}',
              style: Theme.of(context).textTheme.labelSmall),
          ),
          const SizedBox(width: 8),
          Text(result.authorDisplayName ?? 'Unknown',
            style: const TextStyle(fontWeight: FontWeight.bold)),
        ],
      ),
      subtitle: Text(
        result.content,
        maxLines: 2,
        overflow: TextOverflow.ellipsis,
      ),
      trailing: Text(
        timeago.format(DateTime.fromMillisecondsSinceEpoch(result.createdAt * 1000)),
        style: Theme.of(context).textTheme.bodySmall,
      ),
    );
  }
}
```

---

## Step 5: Implement In-Channel Search

**What to do:**

Add a search mode to `ChatScreen` and `ForumScreen`.

In the app bar of both screens, the search icon (already added in Phase 2) should toggle
a search bar:

```dart
// In ChatScreen state:
bool _isSearching = false;
final _searchController = TextEditingController();

// In appBar:
actions: [
  IconButton(
    icon: Icon(_isSearching ? Icons.close : Icons.search),
    onPressed: () {
      setState(() {
        _isSearching = !_isSearching;
        if (!_isSearching) {
          _searchController.clear();
          ref.read(searchStateProvider.notifier).clearResults();
        }
      });
    },
  ),
],
title: _isSearching
  ? TextField(
      controller: _searchController,
      autofocus: true,
      decoration: InputDecoration(hintText: 'Search in channel...'),
      onChanged: (q) => ref.read(searchStateProvider.notifier)
          .search(q, channelId: channelId),
    )
  : Text(channel.name),
```

When in search mode, overlay results on top of the message list. Tapping a result scrolls to that message.

---

## Step 6: Implement Canvas Provider

**Files to create:**
- `lib/features/channels/canvas/canvas_provider.dart`
- `lib/core/models/canvas.dart`

### `core/models/canvas.dart`

See `docs/DATA-MODELS.md` section 13 for the `CanvasDocument` model.

### `canvas/canvas_provider.dart`

```dart
@riverpod
class Canvas extends _$Canvas {
  Timer? _autoSaveTimer;

  @override
  Future<CanvasDocument?> build(String channelId) async {
    final http = ref.watch(relayHttpProvider);
    ref.onDispose(() => _autoSaveTimer?.cancel());
    return await http.getCanvas(channelId);
  }

  /// Update canvas content with debounced auto-save (2 seconds).
  void updateContent(String content) {
    // Optimistic update
    final current = state.valueOrNull;
    state = AsyncData(current?.copyWith(
      content: content,
      lastModifiedAt: DateTime.now().millisecondsSinceEpoch ~/ 1000,
    ) ?? CanvasDocument(
      channelId: arg,
      content: content,
      lastModifiedAt: DateTime.now().millisecondsSinceEpoch ~/ 1000,
    ));

    // Debounced save
    _autoSaveTimer?.cancel();
    _autoSaveTimer = Timer(const Duration(seconds: 2), () {
      _saveToRelay(content);
    });
  }

  Future<void> _saveToRelay(String content) async {
    try {
      final http = ref.read(relayHttpProvider);
      await http.updateCanvas(arg, content);
    } catch (e) {
      debugPrint('Failed to save canvas: $e');
      // Could show a snackbar or retry
    }
  }

  /// Force save immediately (e.g. on back navigation).
  Future<void> saveNow() async {
    _autoSaveTimer?.cancel();
    final content = state.valueOrNull?.content;
    if (content != null) {
      await _saveToRelay(content);
    }
  }
}
```

---

## Step 7: Build Canvas Screen

**Files to create:**
- `lib/features/channels/canvas/canvas_screen.dart`

**Route:** `/channels/:channelId/canvas`

```dart
class CanvasScreen extends ConsumerStatefulWidget {
  final String channelId;

  @override
  ConsumerState<CanvasScreen> createState() => _CanvasScreenState();
}

class _CanvasScreenState extends ConsumerState<CanvasScreen> {
  late QuillController _quillController;
  bool _initialized = false;

  @override
  void dispose() {
    // Save on exit
    ref.read(canvasProvider(widget.channelId).notifier).saveNow();
    _quillController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final canvasAsync = ref.watch(canvasProvider(widget.channelId));

    return Scaffold(
      appBar: AppBar(
        title: const Text('Canvas'),
        actions: [
          // Save status indicator
          canvasAsync.when(
            data: (_) => const Padding(
              padding: EdgeInsets.all(16),
              child: Icon(Icons.cloud_done, size: 18, color: Colors.green),
            ),
            loading: () => const Padding(
              padding: EdgeInsets.all(16),
              child: SizedBox(width: 18, height: 18, child: CircularProgressIndicator(strokeWidth: 2)),
            ),
            error: (_, __) => const Padding(
              padding: EdgeInsets.all(16),
              child: Icon(Icons.cloud_off, size: 18, color: Colors.red),
            ),
          ),
        ],
      ),
      body: canvasAsync.when(
        data: (canvas) {
          if (!_initialized) {
            _quillController = QuillController(
              document: canvas != null && canvas.content.isNotEmpty
                  ? Document.fromJson(jsonDecode(canvas.content))
                  : Document(),
              selection: const TextSelection.collapsed(offset: 0),
            );
            _quillController.addListener(() {
              final content = jsonEncode(_quillController.document.toDelta().toJson());
              ref.read(canvasProvider(widget.channelId).notifier).updateContent(content);
            });
            _initialized = true;
          }

          return Column(
            children: [
              // Toolbar
              QuillSimpleToolbar(
                controller: _quillController,
                configurations: const QuillSimpleToolbarConfigurations(
                  showBoldButton: true,
                  showItalicButton: true,
                  showUnderLineButton: false,
                  showHeaderStyle: true,
                  showListBullets: true,
                  showListNumbers: true,
                  showCodeBlock: true,
                  showLink: true,
                  showAlignmentButtons: false,
                ),
              ),
              const Divider(height: 1),
              // Editor
              Expanded(
                child: Padding(
                  padding: const EdgeInsets.all(16),
                  child: QuillEditor.basic(
                    controller: _quillController,
                    configurations: const QuillEditorConfigurations(
                      placeholder: 'Start writing...',
                    ),
                  ),
                ),
              ),
              // Last modified info
              if (canvas?.lastModifiedBy != null)
                Padding(
                  padding: const EdgeInsets.all(8),
                  child: Text(
                    'Last edited ${timeago.format(DateTime.fromMillisecondsSinceEpoch(canvas!.lastModifiedAt * 1000))}',
                    style: Theme.of(context).textTheme.bodySmall,
                  ),
                ),
            ],
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error loading canvas: $e')),
      ),
    );
  }
}
```

**Navigation:** Add a "Canvas" button in the channel detail screen (Phase 3 Step 8) and optionally in the chat screen app bar menu.

---

## Step 8: Update Routing

**Modify:** `lib/shared/router.dart`

Add new routes:
```dart
GoRoute(path: '/search', builder: (_, __) => const SearchScreen()),
GoRoute(
  path: '/channels/:channelId/canvas',
  builder: (_, state) => CanvasScreen(
    channelId: state.pathParameters['channelId']!,
  ),
),
```

---

## Verification Checklist

After completing Phase 4, verify:

- [ ] Home screen loads with 4 feed categories.
- [ ] Feed sections are collapsible with unread badges.
- [ ] Tapping a feed item navigates to the relevant channel.
- [ ] Pull-to-refresh reloads the home feed.
- [ ] New mentions appear in real-time.
- [ ] Global search returns results with channel name, author, snippet.
- [ ] Search debounces at 300ms.
- [ ] Recent searches are saved and displayed.
- [ ] In-channel search filters results to the active channel.
- [ ] Canvas loads existing content for a channel.
- [ ] Canvas rich text editing works (bold, italic, headings, lists, code).
- [ ] Canvas auto-saves 2 seconds after last edit.
- [ ] Canvas saves on back navigation.
- [ ] Save status indicator works (saved/saving/error).
- [ ] Empty canvas shows placeholder.

---

## Files Created in Phase 4

```
lib/core/models/canvas.dart
lib/features/home/home_provider.dart
lib/features/home/home_screen.dart               (replaced placeholder)
lib/features/home/widgets/mentions_card.dart
lib/features/home/widgets/needs_action_card.dart
lib/features/home/widgets/activity_card.dart
lib/features/home/widgets/agent_activity_card.dart
lib/features/search/search_provider.dart
lib/features/search/search_screen.dart
lib/features/search/widgets/search_result_tile.dart
lib/features/channels/canvas/canvas_provider.dart
lib/features/channels/canvas/canvas_screen.dart
lib/shared/router.dart                           (modified — new routes)
```
