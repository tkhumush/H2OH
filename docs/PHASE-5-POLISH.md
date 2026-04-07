# Phase 5 — Polish: Presence, Notifications, Profile, Tokens, Theming

## Prerequisites

- Phases 1-4 are complete.
- Read `ARCHITECTURE.md`, `docs/NOSTR-PROTOCOL-REFERENCE.md`, `docs/DATA-MODELS.md`.

---

## Goal

At the end of Phase 5, the app should:

1. Show online/away/offline presence indicators on user avatars.
2. Publish presence status on app lifecycle changes.
3. Send push notifications for new messages (FCM + APNs).
4. Show local notifications for messages in non-active channels.
5. Allow full profile editing (name, avatar, bio, NIP-05).
6. Show other users' profiles on avatar tap.
7. Provide a polished settings screen with all configuration options.
8. Support API token creation and revocation.
9. Have a polished light + dark theme.
10. Handle errors, loading states, and edge cases gracefully.

---

## Step 1: Implement Presence System

**Files to create:**
- `lib/features/presence/presence_provider.dart`

See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 8 and `docs/DATA-MODELS.md` section 11.

```dart
@riverpod
class Presence extends _$Presence {
  final Map<String, Timer> _expiryTimers = {};

  @override
  Map<String, PresenceState> build() {
    final relay = ref.watch(relayClientProvider);
    final keyPair = ref.watch(authProvider).valueOrNull;
    if (keyPair == null) return {};

    // Subscribe to presence events (kind 20001)
    final sub = relay.subscribe({'kinds': [20001]});
    sub.events.listen(_handlePresenceEvent);

    // Publish own presence as online
    _publishPresence(PresenceState.online);

    // Listen to app lifecycle
    final observer = _LifecycleObserver(
      onResume: () => _publishPresence(PresenceState.online),
      onPause: () => _publishPresence(PresenceState.away),
      onDetach: () => _publishPresence(PresenceState.offline),
    );
    WidgetsBinding.instance.addObserver(observer);

    ref.onDispose(() {
      relay.unsubscribe(sub.subscriptionId);
      WidgetsBinding.instance.removeObserver(observer);
      for (final timer in _expiryTimers.values) timer.cancel();
      _publishPresence(PresenceState.offline);
    });

    return {};
  }

  void _handlePresenceEvent(NostrEvent event) {
    final pubkey = event.pubkey;
    final statusTag = event.tags.where((t) => t[0] == 'status').firstOrNull;
    if (statusTag == null || statusTag.length < 2) return;

    final presenceState = switch (statusTag[1]) {
      'online' => PresenceState.online,
      'away' => PresenceState.away,
      _ => PresenceState.offline,
    };

    // Update state
    state = {...state, pubkey: presenceState};

    // Set expiry timer (5 minutes → assume offline)
    _expiryTimers[pubkey]?.cancel();
    _expiryTimers[pubkey] = Timer(const Duration(minutes: 5), () {
      state = {...state, pubkey: PresenceState.offline};
      _expiryTimers.remove(pubkey);
    });
  }

  void _publishPresence(PresenceState status) {
    final keyPair = ref.read(authProvider).valueOrNull;
    if (keyPair == null) return;

    final event = NostrEvent.create(
      privateKeyHex: keyPair.privateKeyHex,
      pubkeyHex: keyPair.publicKeyHex,
      kind: 20001,
      content: '',
      tags: [['status', status.name]],
    );
    ref.read(relayClientProvider).publish(event);
  }

  /// Manually set presence (from settings).
  void setPresence(PresenceState status) {
    _publishPresence(status);
    final keyPair = ref.read(authProvider).valueOrNull;
    if (keyPair != null) {
      state = {...state, keyPair.publicKeyHex: status};
    }
  }
}

class _LifecycleObserver extends WidgetsBindingObserver {
  final VoidCallback onResume;
  final VoidCallback onPause;
  final VoidCallback onDetach;

  _LifecycleObserver({required this.onResume, required this.onPause, required this.onDetach});

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        onResume();
      case AppLifecycleState.paused:
      case AppLifecycleState.inactive:
        onPause();
      case AppLifecycleState.detached:
        onDetach();
      default:
        break;
    }
  }
}
```

---

## Step 2: Build Presence Indicator and Avatar Widget

**Files to create:**
- `lib/features/presence/presence_indicator.dart`
- `lib/shared/widgets/avatar.dart`

### `presence_indicator.dart`

A small colored dot to overlay on avatars.

```dart
class PresenceIndicator extends ConsumerWidget {
  final String pubkey;
  final double size;

  const PresenceIndicator({super.key, required this.pubkey, this.size = 12});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final presenceMap = ref.watch(presenceProvider);
    final status = presenceMap[pubkey] ?? PresenceState.offline;

    final color = switch (status) {
      PresenceState.online => Colors.green,
      PresenceState.away => Colors.amber,
      PresenceState.offline => Colors.grey,
    };

    return Container(
      width: size,
      height: size,
      decoration: BoxDecoration(
        color: color,
        shape: BoxShape.circle,
        border: Border.all(color: Theme.of(context).scaffoldBackgroundColor, width: 2),
      ),
    );
  }
}
```

### `shared/widgets/avatar.dart`

Reusable avatar with optional presence indicator.

```dart
class UserAvatar extends StatelessWidget {
  final String? avatarUrl;
  final String displayName;
  final String pubkey;
  final double radius;
  final bool showPresence;

  const UserAvatar({
    super.key,
    this.avatarUrl,
    required this.displayName,
    required this.pubkey,
    this.radius = 20,
    this.showPresence = false,
  });

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        CircleAvatar(
          radius: radius,
          backgroundImage: avatarUrl != null
              ? CachedNetworkImageProvider(avatarUrl!)
              : null,
          child: avatarUrl == null
              ? Text(
                  _initials(displayName),
                  style: TextStyle(fontSize: radius * 0.7),
                )
              : null,
        ),
        if (showPresence)
          Positioned(
            bottom: 0,
            right: 0,
            child: PresenceIndicator(pubkey: pubkey, size: radius * 0.6),
          ),
      ],
    );
  }

  String _initials(String name) {
    if (name.isEmpty) return '?';
    final parts = name.trim().split(' ');
    if (parts.length >= 2) return '${parts[0][0]}${parts[1][0]}'.toUpperCase();
    return name[0].toUpperCase();
  }
}
```

**Usage:** Replace all `CircleAvatar` instances in the app with `UserAvatar`:
- `MessageBubble`: show presence on author avatar.
- `ChannelTile`: show presence on last message author.
- `DmListScreen`: show presence on counterparty avatar.
- `MemberList`: show presence on each member.

---

## Step 3: Set Up Push Notifications

**Files to create:**
- `lib/core/services/notification_service.dart`

**What to do:**

Set up Firebase Cloud Messaging for push notifications on both Android and iOS.

**Pre-requisites (manual steps documented here for the developer):**
1. Create a Firebase project at console.firebase.google.com.
2. Add Android app (package: `com.h2oh.h2oh`), download `google-services.json` to `android/app/`.
3. Add iOS app (bundle ID: `com.h2oh.h2oh`), download `GoogleService-Info.plist` to `ios/Runner/`.
4. Enable Cloud Messaging in Firebase console.
5. For iOS: configure APNs key or certificate in Firebase console.

```dart
class NotificationService {
  static final NotificationService _instance = NotificationService._();
  factory NotificationService() => _instance;
  NotificationService._();

  final FlutterLocalNotificationsPlugin _local = FlutterLocalNotificationsPlugin();

  /// Initialize Firebase and notification channels.
  Future<void> initialize() async {
    await Firebase.initializeApp();

    // Request permissions (iOS)
    final messaging = FirebaseMessaging.instance;
    await messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // Get FCM token
    final token = await messaging.getToken();
    debugPrint('FCM token: $token');
    // TODO: Register token with relay if relay supports push endpoint

    // Initialize local notifications
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings();
    await _local.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
      onDidReceiveNotificationResponse: _onNotificationTap,
    );

    // Create Android notification channel
    const channel = AndroidNotificationChannel(
      'messages',
      'Messages',
      description: 'New message notifications',
      importance: Importance.high,
    );
    await _local.resolvePlatformSpecificImplementation<
        AndroidFlutterLocalNotificationsPlugin>()?.createNotificationChannel(channel);

    // Handle foreground messages
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // Handle background message tap
    FirebaseMessaging.onMessageOpenedApp.listen(_handleMessageTap);
  }

  /// Show a local notification (called when message arrives in non-active channel).
  Future<void> showMessageNotification({
    required String title,
    required String body,
    required String channelId,
    String? eventId,
  }) async {
    await _local.show(
      channelId.hashCode, // notification ID
      title,
      body,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'messages',
          'Messages',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      payload: jsonEncode({'channelId': channelId, 'eventId': eventId}),
    );
  }

  void _handleForegroundMessage(RemoteMessage message) {
    // Convert to local notification
    showMessageNotification(
      title: message.notification?.title ?? 'New message',
      body: message.notification?.body ?? '',
      channelId: message.data['channelId'] ?? '',
      eventId: message.data['eventId'],
    );
  }

  void _handleMessageTap(RemoteMessage message) {
    // Navigate to channel — need access to router
    final channelId = message.data['channelId'];
    if (channelId != null) {
      // Use a global navigator key or event bus to navigate
    }
  }

  void _onNotificationTap(NotificationResponse response) {
    if (response.payload != null) {
      final data = jsonDecode(response.payload!);
      final channelId = data['channelId'];
      // Navigate to channel
    }
  }
}
```

**Initialize in `main.dart`:**
```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService().initialize();
  runApp(const ProviderScope(child: H2OHApp()));
}
```

---

## Step 4: Implement Local Notifications for WebSocket Messages

**What to do:**

When a new message arrives via WebSocket in a channel the user is NOT currently viewing,
show a local notification.

**Integration point:** In the `RelayClient` or a top-level provider, maintain the "active channel ID" state. When an EVENT arrives for a channel that is NOT the active one, trigger a notification.

```dart
@riverpod
class ActiveChannel extends _$ActiveChannel {
  @override
  String? build() => null;

  void setActive(String? channelId) {
    state = channelId;
  }
}

// In ChatScreen.initState():
ref.read(activeChannelProvider.notifier).setActive(channelId);

// In ChatScreen.dispose():
ref.read(activeChannelProvider.notifier).setActive(null);

// In a top-level message listener (e.g. in app.dart or a startup provider):
void _handleGlobalMessage(NostrEvent event, WidgetRef ref) {
  final activeChannel = ref.read(activeChannelProvider);
  final message = Message.fromNostrEvent(event);

  if (message.channelId != activeChannel && message.channelId.isNotEmpty) {
    final myPubkey = ref.read(authProvider).valueOrNull?.publicKeyHex;
    if (event.pubkey == myPubkey) return; // Don't notify for own messages

    NotificationService().showMessageNotification(
      title: '#${message.channelId}', // Resolve channel name if cached
      body: message.content.length > 100
          ? '${message.content.substring(0, 100)}...'
          : message.content,
      channelId: message.channelId,
      eventId: message.id,
    );
  }
}
```

---

## Step 5: Build Profile Screen

**Files to create:**
- `lib/features/profile/profile_screen.dart`
- `lib/features/profile/profile_provider.dart`

### `profile_provider.dart`

```dart
@riverpod
class MyProfile extends _$MyProfile {
  @override
  Future<UserProfile> build() async {
    final keyPair = ref.watch(authProvider).valueOrNull!;
    final http = ref.watch(relayHttpProvider);
    return await http.getProfile(keyPair.publicKeyHex);
  }

  Future<void> updateProfile({
    String? displayName,
    String? avatarUrl,
    String? about,
    String? nip05Handle,
  }) async {
    final http = ref.read(relayHttpProvider);
    await http.updateProfile(
      displayName: displayName,
      avatarUrl: avatarUrl,
      about: about,
      nip05Handle: nip05Handle,
    );
    ref.invalidateSelf(); // Reload profile
  }

  Future<String> uploadAvatar(File imageFile) async {
    final http = ref.read(relayHttpProvider);
    final media = await http.uploadMedia(imageFile);
    return media.url;
  }
}

/// Provider for viewing OTHER users' profiles.
@riverpod
Future<UserProfile> userProfile(UserProfileRef ref, String pubkey) async {
  final http = ref.watch(relayHttpProvider);
  return await http.getProfile(pubkey);
}
```

### `profile_screen.dart`

**Route:** `/profile` (own profile) or shown in dialog for other users.

```dart
class ProfileScreen extends ConsumerStatefulWidget {
  @override
  ConsumerState<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends ConsumerState<ProfileScreen> {
  late TextEditingController _nameController;
  late TextEditingController _aboutController;
  late TextEditingController _nip05Controller;
  bool _isEditing = false;

  @override
  Widget build(BuildContext context) {
    final profileAsync = ref.watch(myProfileProvider);
    final keyPair = ref.watch(authProvider).valueOrNull;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Profile'),
        actions: [
          if (!_isEditing)
            IconButton(icon: const Icon(Icons.edit), onPressed: () => setState(() => _isEditing = true))
          else
            IconButton(icon: const Icon(Icons.check), onPressed: _saveProfile),
        ],
      ),
      body: profileAsync.when(
        data: (profile) {
          _initControllers(profile);
          return ListView(
            padding: const EdgeInsets.all(16),
            children: [
              // Avatar
              Center(
                child: GestureDetector(
                  onTap: _isEditing ? _changeAvatar : null,
                  child: Stack(
                    children: [
                      UserAvatar(
                        avatarUrl: profile.avatarUrl,
                        displayName: profile.displayName,
                        pubkey: profile.pubkey,
                        radius: 50,
                      ),
                      if (_isEditing)
                        Positioned(
                          bottom: 0, right: 0,
                          child: CircleAvatar(
                            radius: 16,
                            child: Icon(Icons.camera_alt, size: 16),
                          ),
                        ),
                    ],
                  ),
                ),
              ),
              const SizedBox(height: 24),

              // Display name
              _isEditing
                  ? TextField(controller: _nameController, decoration: InputDecoration(labelText: 'Display Name'))
                  : _infoTile('Display Name', profile.displayName),

              // About
              _isEditing
                  ? TextField(controller: _aboutController, decoration: InputDecoration(labelText: 'About'), maxLines: 3)
                  : _infoTile('About', profile.about),

              // NIP-05
              _isEditing
                  ? TextField(controller: _nip05Controller, decoration: InputDecoration(labelText: 'NIP-05 Handle'))
                  : _infoTile('NIP-05', profile.nip05Handle ?? ''),

              const Divider(height: 32),

              // Public key (always read-only)
              ListTile(
                title: const Text('Public Key'),
                subtitle: Text(keyPair?.npub ?? '', style: const TextStyle(fontFamily: 'monospace', fontSize: 12)),
                trailing: IconButton(
                  icon: const Icon(Icons.copy),
                  onPressed: () {
                    Clipboard.setData(ClipboardData(text: keyPair?.npub ?? ''));
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('Public key copied')),
                    );
                  },
                ),
              ),

              // Private key (hidden, show on tap with warning)
              ListTile(
                title: const Text('Private Key'),
                subtitle: const Text('Tap to reveal (keep this secret!)'),
                trailing: const Icon(Icons.visibility_off),
                onTap: () => _showPrivateKey(context, keyPair),
              ),

              const Divider(height: 32),

              // Import key
              OutlinedButton.icon(
                icon: const Icon(Icons.key),
                label: const Text('Import existing key (nsec)'),
                onPressed: () => _importKey(context, ref),
              ),
            ],
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error: $e')),
      ),
    );
  }

  void _saveProfile() async {
    await ref.read(myProfileProvider.notifier).updateProfile(
      displayName: _nameController.text.trim(),
      about: _aboutController.text.trim(),
      nip05Handle: _nip05Controller.text.trim(),
    );
    setState(() => _isEditing = false);
  }

  void _changeAvatar() async {
    final picker = ImagePicker();
    final xfile = await picker.pickImage(source: ImageSource.gallery, maxWidth: 512, maxHeight: 512);
    if (xfile == null) return;
    final url = await ref.read(myProfileProvider.notifier).uploadAvatar(File(xfile.path));
    await ref.read(myProfileProvider.notifier).updateProfile(avatarUrl: url);
  }

  void _showPrivateKey(BuildContext context, KeyPair? keyPair) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Private Key'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text('Never share this with anyone!', style: TextStyle(color: Colors.red)),
            const SizedBox(height: 16),
            SelectableText(keyPair?.nsec ?? '', style: const TextStyle(fontFamily: 'monospace', fontSize: 12)),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () {
              Clipboard.setData(ClipboardData(text: keyPair?.nsec ?? ''));
              Navigator.pop(context);
            },
            child: const Text('Copy'),
          ),
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('Close')),
        ],
      ),
    );
  }

  void _importKey(BuildContext context, WidgetRef ref) {
    final controller = TextEditingController();
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Import Key'),
        content: TextField(
          controller: controller,
          decoration: const InputDecoration(hintText: 'nsec1...'),
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('Cancel')),
          TextButton(
            onPressed: () async {
              try {
                await ref.read(authProvider.notifier).importKey(controller.text.trim());
                Navigator.pop(context);
                ScaffoldMessenger.of(context).showSnackBar(
                  const SnackBar(content: Text('Key imported successfully')),
                );
              } catch (e) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(content: Text('Invalid key: $e')),
                );
              }
            },
            child: const Text('Import'),
          ),
        ],
      ),
    );
  }
}
```

**Other user profile view (shown as bottom sheet):**
```dart
void showUserProfile(BuildContext context, String pubkey) {
  showModalBottomSheet(
    context: context,
    builder: (_) => Consumer(
      builder: (context, ref, _) {
        final profileAsync = ref.watch(userProfileProvider(pubkey));
        return profileAsync.when(
          data: (profile) => Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                UserAvatar(avatarUrl: profile.avatarUrl, displayName: profile.displayName, pubkey: pubkey, radius: 40, showPresence: true),
                const SizedBox(height: 12),
                Text(profile.displayName, style: Theme.of(context).textTheme.titleLarge),
                if (profile.about.isNotEmpty) Text(profile.about),
                const SizedBox(height: 16),
                ElevatedButton.icon(
                  icon: const Icon(Icons.chat),
                  label: const Text('Send Message'),
                  onPressed: () {
                    Navigator.pop(context);
                    context.push('/dm/$pubkey');
                  },
                ),
              ],
            ),
          ),
          loading: () => const SizedBox(height: 100, child: Center(child: CircularProgressIndicator())),
          error: (e, _) => Text('Error: $e'),
        );
      },
    ),
  );
}
```

---

## Step 6: Build Settings Screen

**Files to modify:**
- `lib/features/settings/settings_screen.dart` (replace placeholder)

**Files to create:**
- `lib/features/settings/settings_provider.dart`

### `settings_provider.dart`

```dart
@riverpod
class AppSettings extends _$AppSettings {
  @override
  Future<Map<String, dynamic>> build() async {
    final prefs = await SharedPreferences.getInstance();
    return {
      'themeMode': prefs.getString('theme_mode') ?? 'system', // light, dark, system
      'notificationsEnabled': prefs.getBool('notifications_enabled') ?? true,
      'notificationSound': prefs.getBool('notification_sound') ?? true,
    };
  }

  Future<void> setThemeMode(String mode) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('theme_mode', mode);
    state = AsyncData({...state.valueOrNull ?? {}, 'themeMode': mode});
  }

  Future<void> setNotificationsEnabled(bool enabled) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('notifications_enabled', enabled);
    state = AsyncData({...state.valueOrNull ?? {}, 'notificationsEnabled': enabled});
  }

  Future<void> setNotificationSound(bool enabled) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('notification_sound', enabled);
    state = AsyncData({...state.valueOrNull ?? {}, 'notificationSound': enabled});
  }
}
```

### `settings_screen.dart`

```dart
class SettingsScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final profile = ref.watch(myProfileProvider).valueOrNull;
    final relayConfig = ref.watch(relayConfigNotifierProvider).valueOrNull;
    final settings = ref.watch(appSettingsProvider).valueOrNull ?? {};
    final keyPair = ref.watch(authProvider).valueOrNull;

    return Scaffold(
      appBar: AppBar(title: const Text('Settings')),
      body: ListView(
        children: [
          // --- Profile Section ---
          _SectionHeader('Profile'),
          ListTile(
            leading: UserAvatar(
              avatarUrl: profile?.avatarUrl,
              displayName: profile?.displayName ?? '',
              pubkey: keyPair?.publicKeyHex ?? '',
              radius: 24,
            ),
            title: Text(profile?.displayName ?? 'Set up profile'),
            subtitle: Text(keyPair?.npub.substring(0, 20) ?? ''),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/profile'),
          ),

          // --- Relay Section ---
          _SectionHeader('Relay'),
          ListTile(
            leading: Icon(
              relayConfig != null ? Icons.cloud_done : Icons.cloud_off,
              color: relayConfig != null ? Colors.green : Colors.grey,
            ),
            title: Text(relayConfig?.websocketUrl ?? 'Not configured'),
            subtitle: Text(relayConfig != null ? 'Connected' : 'Tap to configure'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/settings/relay'),
          ),

          // --- Presence Section ---
          _SectionHeader('Presence'),
          ListTile(
            leading: const Icon(Icons.circle, color: Colors.green, size: 16),
            title: const Text('Status'),
            trailing: DropdownButton<PresenceState>(
              value: ref.watch(presenceProvider)[keyPair?.publicKeyHex] ?? PresenceState.online,
              items: PresenceState.values.map((s) => DropdownMenuItem(
                value: s,
                child: Text(s.name[0].toUpperCase() + s.name.substring(1)),
              )).toList(),
              onChanged: (value) {
                if (value != null) ref.read(presenceProvider.notifier).setPresence(value);
              },
            ),
          ),

          // --- Notifications Section ---
          _SectionHeader('Notifications'),
          SwitchListTile(
            title: const Text('Push Notifications'),
            value: settings['notificationsEnabled'] ?? true,
            onChanged: (v) => ref.read(appSettingsProvider.notifier).setNotificationsEnabled(v),
          ),
          SwitchListTile(
            title: const Text('Notification Sound'),
            value: settings['notificationSound'] ?? true,
            onChanged: (v) => ref.read(appSettingsProvider.notifier).setNotificationSound(v),
          ),

          // --- Tokens Section ---
          _SectionHeader('API Tokens'),
          ListTile(
            leading: const Icon(Icons.vpn_key),
            title: const Text('Manage Tokens'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/settings/tokens'),
          ),

          // --- Appearance Section ---
          _SectionHeader('Appearance'),
          ListTile(
            leading: const Icon(Icons.palette),
            title: const Text('Theme'),
            trailing: DropdownButton<String>(
              value: settings['themeMode'] ?? 'system',
              items: const [
                DropdownMenuItem(value: 'light', child: Text('Light')),
                DropdownMenuItem(value: 'dark', child: Text('Dark')),
                DropdownMenuItem(value: 'system', child: Text('System')),
              ],
              onChanged: (value) {
                if (value != null) ref.read(appSettingsProvider.notifier).setThemeMode(value);
              },
            ),
          ),

          // --- Security Section ---
          _SectionHeader('Security'),
          ListTile(
            leading: const Icon(Icons.key),
            title: const Text('Export Private Key'),
            subtitle: const Text('Show your nsec (keep secret!)'),
            onTap: () => _showPrivateKey(context, keyPair),
          ),
          ListTile(
            leading: const Icon(Icons.import_export),
            title: const Text('Import Key'),
            onTap: () => _importKey(context, ref),
          ),
          ListTile(
            leading: const Icon(Icons.refresh, color: Colors.red),
            title: const Text('Reset Identity', style: TextStyle(color: Colors.red)),
            subtitle: const Text('Generate a new keypair (irreversible!)'),
            onTap: () => _confirmReset(context, ref),
          ),

          // --- About Section ---
          _SectionHeader('About'),
          ListTile(
            leading: const Icon(Icons.info_outline),
            title: const Text('H2OH'),
            subtitle: const Text('v1.0.0'),
          ),
        ],
      ),
    );
  }

  void _confirmReset(BuildContext context, WidgetRef ref) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Reset Identity?'),
        content: const Text('This will generate a new keypair. Your old identity and messages will be lost. This cannot be undone.'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('Cancel')),
          TextButton(
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            onPressed: () async {
              await ref.read(authProvider.notifier).resetIdentity();
              Navigator.pop(context);
            },
            child: const Text('Reset'),
          ),
        ],
      ),
    );
  }
}

class _SectionHeader extends StatelessWidget {
  final String title;
  const _SectionHeader(this.title);

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.fromLTRB(16, 24, 16, 8),
      child: Text(title, style: Theme.of(context).textTheme.titleSmall?.copyWith(
        color: Theme.of(context).colorScheme.primary,
      )),
    );
  }
}
```

---

## Step 7: Build Token Management Screen

**Files to create:**
- `lib/features/settings/token_management_screen.dart`

**Route:** `/settings/tokens`

```dart
class TokenManagementScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tokensAsync = ref.watch(tokensProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('API Tokens')),
      floatingActionButton: FloatingActionButton(
        child: const Icon(Icons.add),
        onPressed: () => _createToken(context, ref),
      ),
      body: tokensAsync.when(
        data: (tokens) {
          if (tokens.isEmpty) {
            return const Center(child: Text('No tokens. Tap + to create one.'));
          }
          return ListView.builder(
            itemCount: tokens.length,
            itemBuilder: (_, i) {
              final token = tokens[i];
              return Dismissible(
                key: Key(token.id),
                direction: DismissDirection.endToStart,
                background: Container(color: Colors.red, alignment: Alignment.centerRight,
                  padding: const EdgeInsets.only(right: 16),
                  child: const Icon(Icons.delete, color: Colors.white)),
                confirmDismiss: (_) => _confirmRevoke(context),
                onDismissed: (_) {
                  ref.read(relayHttpProvider).revokeToken(token.id);
                  ref.invalidate(tokensProvider);
                },
                child: ListTile(
                  leading: const Icon(Icons.vpn_key),
                  title: Text(token.name),
                  subtitle: Text('Created ${timeago.format(
                    DateTime.fromMillisecondsSinceEpoch(token.createdAt * 1000))}'),
                  trailing: token.expiresAt != null
                      ? Text('Expires ${timeago.format(
                          DateTime.fromMillisecondsSinceEpoch(token.expiresAt! * 1000))}')
                      : const Text('No expiry'),
                ),
              );
            },
          );
        },
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error: $e')),
      ),
    );
  }

  void _createToken(BuildContext context, WidgetRef ref) {
    final nameController = TextEditingController();
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Create Token'),
        content: TextField(controller: nameController, decoration: InputDecoration(hintText: 'Token name')),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context), child: const Text('Cancel')),
          TextButton(
            onPressed: () async {
              final http = ref.read(relayHttpProvider);
              final token = await http.createToken(nameController.text.trim());
              Navigator.pop(context);
              ref.invalidate(tokensProvider);
              // Show the token string (only shown once!)
              _showTokenCreated(context, token);
            },
            child: const Text('Create'),
          ),
        ],
      ),
    );
  }

  void _showTokenCreated(BuildContext context, ApiToken token) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Token Created'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text('Copy this token now. It will not be shown again!',
              style: TextStyle(color: Colors.orange, fontWeight: FontWeight.bold)),
            const SizedBox(height: 16),
            SelectableText(token.token, style: const TextStyle(fontFamily: 'monospace', fontSize: 12)),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () {
              Clipboard.setData(ClipboardData(text: token.token));
              Navigator.pop(context);
            },
            child: const Text('Copy & Close'),
          ),
        ],
      ),
    );
  }

  Future<bool?> _confirmRevoke(BuildContext context) {
    return showDialog<bool>(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Revoke Token?'),
        content: const Text('This token will stop working immediately.'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('Cancel')),
          TextButton(
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            onPressed: () => Navigator.pop(context, true),
            child: const Text('Revoke'),
          ),
        ],
      ),
    );
  }
}

@riverpod
Future<List<ApiToken>> tokens(TokensRef ref) async {
  final http = ref.watch(relayHttpProvider);
  return await http.listTokens();
}
```

---

## Step 8: Finalize Theming

**Files to modify:**
- `lib/shared/theme.dart`
- `lib/app.dart`

### `shared/theme.dart`

```dart
final lightTheme = ThemeData(
  useMaterial3: true,
  brightness: Brightness.light,
  colorSchemeSeed: const Color(0xFF1E88E5), // Blue
  fontFamily: 'Inter', // Or system default
  appBarTheme: const AppBarTheme(
    centerTitle: false,
    elevation: 0,
  ),
  cardTheme: CardTheme(
    elevation: 1,
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
  ),
  inputDecorationTheme: InputDecorationTheme(
    filled: true,
    border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
    contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
  ),
  bottomNavigationBarTheme: const BottomNavigationBarThemeData(
    type: BottomNavigationBarType.fixed,
    showUnselectedLabels: true,
  ),
);

final darkTheme = ThemeData(
  useMaterial3: true,
  brightness: Brightness.dark,
  colorSchemeSeed: const Color(0xFF1E88E5),
  fontFamily: 'Inter',
  appBarTheme: const AppBarTheme(
    centerTitle: false,
    elevation: 0,
  ),
  cardTheme: CardTheme(
    elevation: 1,
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
  ),
  inputDecorationTheme: InputDecorationTheme(
    filled: true,
    border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
    contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
  ),
  bottomNavigationBarTheme: const BottomNavigationBarThemeData(
    type: BottomNavigationBarType.fixed,
    showUnselectedLabels: true,
  ),
);

/// Text styles for code/monospace
const codeTextStyle = TextStyle(fontFamily: 'monospace', fontSize: 13);
```

### Wire theme to settings in `app.dart`

```dart
class H2OHApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    final settings = ref.watch(appSettingsProvider).valueOrNull ?? {};
    final themeModeStr = settings['themeMode'] ?? 'system';
    final themeMode = switch (themeModeStr) {
      'light' => ThemeMode.light,
      'dark' => ThemeMode.dark,
      _ => ThemeMode.system,
    };

    return MaterialApp.router(
      title: 'H2OH',
      theme: lightTheme,
      darkTheme: darkTheme,
      themeMode: themeMode,
      routerConfig: router,
    );
  }
}
```

---

## Step 9: Responsive Layout

**What to do:**

Add responsive layout support for tablets.

```dart
// lib/shared/widgets/responsive_layout.dart

class ResponsiveLayout extends StatelessWidget {
  final Widget phone;
  final Widget? tablet;

  const ResponsiveLayout({super.key, required this.phone, this.tablet});

  static bool isTablet(BuildContext context) =>
      MediaQuery.of(context).size.shortestSide >= 600;

  @override
  Widget build(BuildContext context) {
    if (isTablet(context) && tablet != null) return tablet!;
    return phone;
  }
}
```

**Channel list + chat master-detail on tablet:**
```dart
// In the channel list route, detect tablet and show split view:
class ChannelListScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    if (ResponsiveLayout.isTablet(context)) {
      return Row(
        children: [
          SizedBox(width: 320, child: _ChannelList()),
          const VerticalDivider(width: 1),
          Expanded(child: _selectedChannel != null
              ? ChatScreen(channelId: _selectedChannel!)
              : const Center(child: Text('Select a channel'))),
        ],
      );
    }
    return _ChannelList(); // Phone: full screen list
  }
}
```

**Bottom nav vs navigation rail:**
```dart
// In AppShell:
if (ResponsiveLayout.isTablet(context)) {
  return Row(
    children: [
      NavigationRail(
        selectedIndex: _currentIndex,
        onDestinationSelected: (i) => _onTap(context, i),
        destinations: const [
          NavigationRailDestination(icon: Icon(Icons.home), label: Text('Home')),
          NavigationRailDestination(icon: Icon(Icons.tag), label: Text('Channels')),
          NavigationRailDestination(icon: Icon(Icons.chat_bubble), label: Text('DMs')),
          NavigationRailDestination(icon: Icon(Icons.settings), label: Text('Settings')),
        ],
      ),
      const VerticalDivider(width: 1),
      Expanded(child: child),
    ],
  );
} else {
  return Scaffold(body: child, bottomNavigationBar: BottomNavigationBar(...));
}
```

---

## Step 10: Error Handling and Edge Cases

**Files to create:**
- `lib/shared/widgets/error_banner.dart`
- `lib/shared/widgets/loading.dart`

### `error_banner.dart`

```dart
class ErrorBanner extends StatelessWidget {
  final String message;
  final String? actionLabel;
  final VoidCallback? onAction;
  final VoidCallback? onDismiss;

  @override
  Widget build(BuildContext context) {
    return MaterialBanner(
      content: Text(message),
      backgroundColor: Theme.of(context).colorScheme.errorContainer,
      leading: Icon(Icons.error_outline, color: Theme.of(context).colorScheme.error),
      actions: [
        if (actionLabel != null)
          TextButton(onPressed: onAction, child: Text(actionLabel!)),
        TextButton(onPressed: onDismiss, child: const Text('Dismiss')),
      ],
    );
  }
}
```

### `loading.dart`

```dart
/// Skeleton loader for list items.
class SkeletonListTile extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(backgroundColor: Colors.grey[300]),
      title: Container(height: 14, width: 120, color: Colors.grey[300]),
      subtitle: Container(height: 10, width: 200, color: Colors.grey[200]),
    );
  }
}

/// Loading list with N skeleton items.
class SkeletonList extends StatelessWidget {
  final int itemCount;
  const SkeletonList({this.itemCount = 8});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: itemCount,
      itemBuilder: (_, __) => const SkeletonListTile(),
    );
  }
}
```

### Error handling patterns to apply across the app:

**Network connection banner:**
In `AppShell`, listen to `relayClientProvider` connection state:
```dart
final connectionState = ref.watch(relayConnectionStateProvider);
if (connectionState == RelayConnectionState.disconnected ||
    connectionState == RelayConnectionState.error) {
  return Column(children: [
    ErrorBanner(
      message: 'Connection lost — retrying...',
      actionLabel: 'Retry now',
      onAction: () => ref.read(relayClientProvider).connect(),
    ),
    Expanded(child: child),
  ]);
}
```

**Empty states:** Add to every list screen (channels, DMs, search, feed):
```dart
if (items.isEmpty) {
  return Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.inbox, size: 64, color: Colors.grey[400]),
        const SizedBox(height: 16),
        Text('No messages yet', style: Theme.of(context).textTheme.bodyLarge),
      ],
    ),
  );
}
```

**Long messages:** In `MessageBubble`, truncate messages > 500 chars with "Show more":
```dart
final isLong = message.content.length > 500;
final displayContent = isLong && !_expanded
    ? '${message.content.substring(0, 500)}...'
    : message.content;
// ...
if (isLong) TextButton(
  onPressed: () => setState(() => _expanded = !_expanded),
  child: Text(_expanded ? 'Show less' : 'Show more'),
),
```

**Malformed events:** In `Message.fromNostrEvent()`, wrap parsing in try-catch. Return null or a placeholder message on failure. Log warnings.

---

## Step 11: Update Routing

**Modify:** `lib/shared/router.dart`

Add final routes:
```dart
GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
GoRoute(path: '/settings/tokens', builder: (_, __) => const TokenManagementScreen()),
```

---

## Verification Checklist

After completing Phase 5, verify:

- [ ] Presence dots show on avatars (green/yellow/gray).
- [ ] Own presence updates on app foreground/background/close.
- [ ] Presence expires after 5 minutes of no update.
- [ ] Manual presence setting works in Settings.
- [ ] Push notifications arrive for new messages (test with Firebase).
- [ ] Local notifications show for messages in non-active channels.
- [ ] Tapping notification navigates to the correct channel.
- [ ] No notification for currently active channel.
- [ ] Profile screen shows and edits display name, avatar, bio, NIP-05.
- [ ] Avatar upload works.
- [ ] npub can be copied, nsec can be revealed.
- [ ] Key import works with valid nsec.
- [ ] Identity reset generates new keypair.
- [ ] Settings screen shows all sections correctly.
- [ ] Theme toggle (light/dark/system) works and persists.
- [ ] Token creation shows token once, token listed in management.
- [ ] Token revocation works.
- [ ] Responsive layout: tablet shows split view, phone shows stacked.
- [ ] Error banner shows on connection loss.
- [ ] Empty states show on all empty lists.
- [ ] Long messages truncate with "Show more".
- [ ] Both themes look polished on all screens.

---

## Files Created in Phase 5

```
lib/features/presence/presence_provider.dart
lib/features/presence/presence_indicator.dart
lib/core/services/notification_service.dart
lib/features/profile/profile_screen.dart
lib/features/profile/profile_provider.dart
lib/features/settings/settings_screen.dart       (replaced placeholder)
lib/features/settings/settings_provider.dart
lib/features/settings/token_management_screen.dart
lib/shared/widgets/avatar.dart
lib/shared/widgets/error_banner.dart
lib/shared/widgets/loading.dart
lib/shared/widgets/responsive_layout.dart
lib/shared/theme.dart                            (modified — polished)
lib/shared/router.dart                           (modified — final routes)
lib/app.dart                                     (modified — theme from settings)
lib/main.dart                                    (modified — notification init)
```
