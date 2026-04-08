# Phase 1 — Foundation

## Prerequisites

- Read `ARCHITECTURE.md` for overall structure, dependencies, and routing.
- Read `docs/NOSTR-PROTOCOL-REFERENCE.md` for protocol details.
- Read `docs/DATA-MODELS.md` for all model definitions.
- Flutter SDK installed (3.22+ / Dart 3.4+).

---

## Goal

Set up the Flutter project from scratch, implement Nostr key management, build the
WebSocket relay client with NIP-42 authentication, create the HTTP relay client, build
the relay URL configuration screen, and establish the app shell with bottom navigation
and GoRouter routing.

At the end of Phase 1, the app should:
1. Launch on Android and iOS.
2. Generate and securely store a Nostr keypair on first run.
3. Allow the user to enter a relay URL in Settings.
4. Connect to the relay over WebSocket.
5. Complete NIP-42 authentication.
6. Display a bottom navigation shell with placeholder screens for Home, Channels, DMs, Settings.

---

## Step 1: Create the Flutter Project

**What to do:**

Run `flutter create` in the repository root to scaffold the Flutter project.

```bash
cd /home/user/H2OH
flutter create --org com.h2oh --project-name h2oh --platforms android,ios .
```

**After creation:**
1. Replace the generated `lib/main.dart` with a minimal entry point.
2. Replace the generated `test/widget_test.dart` with a placeholder.
3. Update `pubspec.yaml` with all dependencies listed in `ARCHITECTURE.md` (the "Dependencies" section). Add them all now even if not used until later phases — this avoids dependency conflicts later.
4. Create `analysis_options.yaml` with recommended Flutter lints.
5. Run `flutter pub get` to install dependencies.
6. Run `dart run build_runner build` to generate freezed/json_serializable code (will be needed after Step 2).

**Expected files after this step:**
```
lib/main.dart
pubspec.yaml
analysis_options.yaml
android/
ios/
```

**`lib/main.dart` content:**
```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'app.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: H2OHApp()));
}
```

---

## Step 2: Implement Nostr Key Management

**Files to create:**
- `lib/core/nostr/keys.dart`
- `lib/core/nostr/event.dart`

### `lib/core/nostr/keys.dart`

This file handles keypair generation, derivation, bech32 encoding, and secure storage.

**Requirements:**
1. Generate a random 32-byte private key using a cryptographically secure RNG.
2. Derive the x-only public key (secp256k1 point, x-coordinate only, 32 bytes).
3. Encode/decode keys to/from bech32 format (`nsec1...` / `npub1...`).
4. Store the private key hex string in `flutter_secure_storage` under key `"nostr_private_key"`.
5. Load the private key from secure storage and derive the public key.
6. Support importing an existing `nsec` key.

**Functions to implement:**
```dart
/// Generate a new random keypair.
KeyPair generateKeyPair();

/// Derive public key from private key hex string.
KeyPair fromPrivateKeyHex(String privateKeyHex);

/// Decode an nsec bech32 string into a KeyPair.
KeyPair fromNsec(String nsec);

/// Save private key to flutter_secure_storage.
Future<void> saveKeyPair(KeyPair keyPair);

/// Load private key from flutter_secure_storage. Returns null if not found.
Future<KeyPair?> loadKeyPair();

/// Delete stored keypair (for logout/reset).
Future<void> deleteKeyPair();
```

**Implementation notes:**
- Use `pointycastle` for secp256k1 operations.
- Use `bip340` package for BIP-340 Schnorr helpers.
- Use `dart:math` `Random.secure()` for random byte generation (or `pointycastle`'s `FortunaRandom`).
- Bech32 encoding: Use the bech32 codec. `nsec` prefix for private keys, `npub` prefix for public keys. The data is the 32-byte key converted to 5-bit groups.
- `flutter_secure_storage` instance should be created once and reused. Consider making it a Riverpod provider.

**KeyPair class** — see `docs/DATA-MODELS.md` section 2 for the exact definition.

### `lib/core/nostr/event.dart`

This file defines the `NostrEvent` model and implements event ID computation and signing.

**Requirements:**
1. Define the `NostrEvent` freezed class (see `docs/DATA-MODELS.md` section 1).
2. Implement `computeId()` — SHA-256 of the canonical serialization `[0, pubkey, created_at, kind, tags, content]`.
3. Implement `create()` — build a new event, compute its ID, sign with BIP-340 Schnorr.
4. Implement `verifySignature()` — verify a received event's signature.
5. Implement `toJson()` / `fromJson()` for relay communication.

**Critical details for `computeId()`:**
- The serialization MUST be: `jsonEncode([0, pubkey, createdAt, kind, tags, content])`
- No whitespace. Integers as numbers. Strings properly JSON-escaped.
- Hash the UTF-8 bytes of this string with SHA-256.
- Result is 32 bytes, represented as a 64-char lowercase hex string.

**Critical details for signing:**
- Use BIP-340 Schnorr signature over the event ID bytes (not the hex string — convert hex to 32 bytes first).
- The `bip340` Dart package provides `Bip340.sign(privateKey, message)` where both are hex strings.
- The signature is 64 bytes, represented as 128-char lowercase hex string.

**Run `dart run build_runner build`** after creating these files to generate the freezed code.

---

## Step 3: Implement NIP-01 Protocol Layer

**Files to create:**
- `lib/core/nostr/nip01.dart`

**Requirements:**
1. Build `REQ` messages: `["REQ", subscriptionId, filter]`.
2. Build `EVENT` messages: `["EVENT", eventObject]`.
3. Build `CLOSE` messages: `["CLOSE", subscriptionId]`.
4. Parse incoming relay messages into typed objects.

**Message types to define:**

```dart
sealed class RelayMessage {}

class EventMessage extends RelayMessage {
  final String subscriptionId;
  final NostrEvent event;
}

class EoseMessage extends RelayMessage {
  final String subscriptionId;
}

class OkMessage extends RelayMessage {
  final String eventId;
  final bool accepted;
  final String message;
}

class NoticeMessage extends RelayMessage {
  final String message;
}

class AuthMessage extends RelayMessage {
  final String challenge;
}

class UnknownMessage extends RelayMessage {
  final String raw;
}
```

**Parser function:**
```dart
RelayMessage parseRelayMessage(String rawJson) {
  final decoded = jsonDecode(rawJson) as List<dynamic>;
  final type = decoded[0] as String;
  // Switch on type and return appropriate message class.
  // See NOSTR-PROTOCOL-REFERENCE.md section 2 for exact formats.
}
```

**Builder functions:**
```dart
String buildReqMessage(String subscriptionId, Map<String, dynamic> filter);
String buildEventMessage(NostrEvent event);
String buildCloseMessage(String subscriptionId);
String buildAuthMessage(NostrEvent authEvent); // ["AUTH", event]
```

**See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 2** for exact message formats.

---

## Step 4: Implement NIP-42 Authentication

**Files to create:**
- `lib/core/nostr/nip42.dart`

**Requirements:**
1. When the relay sends `["AUTH", "<challenge>"]`, create a signed kind 22242 event.
2. The event must have tags: `["relay", relayUrl]` and `["challenge", challengeString]`.
3. Send it back as `["AUTH", signedEvent]`.

**Function to implement:**
```dart
NostrEvent createAuthEvent({
  required String privateKeyHex,
  required String pubkeyHex,
  required String relayUrl,
  required String challenge,
});
```

**See `docs/NOSTR-PROTOCOL-REFERENCE.md` section 3** for exact format.

---

## Step 5: Implement Relay Configuration

**Files to create:**
- `lib/core/relay/relay_config.dart`

**Requirements:**
1. Store the relay WebSocket URL in `SharedPreferences` under key `"relay_url"`.
2. Derive the HTTP base URL from the WebSocket URL:
   - `wss://` → `https://`
   - `ws://` → `http://`
3. Provide a Riverpod provider that exposes the current `RelayConfig`.
4. When the relay URL changes, all dependent providers (WebSocket client, HTTP client) must reinitialize.

**RelayConfig class** — see `docs/DATA-MODELS.md` section 12.

**Riverpod provider:**
```dart
@riverpod
class RelayConfigNotifier extends _$RelayConfigNotifier {
  @override
  Future<RelayConfig?> build() async {
    final prefs = await SharedPreferences.getInstance();
    final url = prefs.getString('relay_url');
    if (url == null) return null;
    return RelayConfig(websocketUrl: url);
  }

  Future<void> setRelayUrl(String url) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('relay_url', url);
    state = AsyncData(RelayConfig(websocketUrl: url));
  }

  Future<void> clearRelayUrl() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('relay_url');
    state = const AsyncData(null);
  }
}
```

---

## Step 6: Implement WebSocket Relay Client

**Files to create:**
- `lib/core/relay/relay_client.dart`

**Requirements:**

This is the most critical class in the app. It manages the WebSocket connection lifecycle
and all NIP-01 communication.

1. **Connect** to the relay WebSocket URL.
2. **Reconnect** with exponential backoff on disconnect (1s, 2s, 4s, 8s, 16s, max 30s).
3. **Handle NIP-42 auth** — when an `AUTH` challenge arrives, automatically sign and respond.
4. **Subscribe** — send `REQ` with a filter, return a `Stream<NostrEvent>` for that subscription.
5. **Publish** — send `EVENT`, await `OK` response, return success/failure.
6. **Unsubscribe** — send `CLOSE` for a subscription ID.
7. **Event batching** — buffer incoming events for 16ms before dispatching to listeners, to avoid per-event UI rebuilds.
8. **Connection state** — expose an observable connection state (connecting, connected, authenticated, disconnected, error).

**Class structure:**
```dart
enum RelayConnectionState { disconnected, connecting, connected, authenticated, error }

class RelayClient {
  final String websocketUrl;
  final KeyPair keyPair;

  RelayConnectionState _state = RelayConnectionState.disconnected;
  WebSocketChannel? _channel;
  final Map<String, StreamController<NostrEvent>> _subscriptions = {};

  /// Connect to the relay. Call after setting relay URL and loading keys.
  Future<void> connect();

  /// Disconnect gracefully.
  Future<void> disconnect();

  /// Subscribe to events matching the filter.
  /// Returns a stream of events for this subscription.
  /// The subscription ID is auto-generated (UUID v4).
  ({String subscriptionId, Stream<NostrEvent> events}) subscribe(
    Map<String, dynamic> filter,
  );

  /// Unsubscribe and close the stream.
  void unsubscribe(String subscriptionId);

  /// Publish a signed event to the relay.
  /// Returns true if the relay accepted it (OK true), false otherwise.
  Future<bool> publish(NostrEvent event);

  /// Observable connection state.
  Stream<RelayConnectionState> get connectionStateStream;

  // PRIVATE:
  void _handleMessage(String raw);      // parse + route to subscription or auth
  void _handleAuth(String challenge);    // create + send auth event
  void _scheduleReconnect();             // exponential backoff
  void _batchDispatch(String subId, NostrEvent event); // 16ms batching
}
```

**Riverpod provider:**
```dart
@riverpod
RelayClient relayClient(RelayClientRef ref) {
  final config = ref.watch(relayConfigNotifierProvider).valueOrNull;
  final keyPair = ref.watch(authProvider).valueOrNull;

  if (config == null || keyPair == null) {
    throw StateError('Relay not configured or keys not loaded');
  }

  final client = RelayClient(
    websocketUrl: config.websocketUrl,
    keyPair: keyPair,
  );
  client.connect();

  ref.onDispose(() => client.disconnect());
  return client;
}
```

**Important implementation notes:**
- Use `web_socket_channel` package's `WebSocketChannel.connect(uri)`.
- Listen to `_channel!.stream` for incoming messages.
- Send via `_channel!.sink.add(jsonString)`.
- On `WebSocketChannelException` or stream done event → trigger reconnect.
- During reconnect, re-establish all active subscriptions.
- The 16ms event batching: collect events in a list, use a `Timer` that fires every 16ms to flush the buffer to the stream controllers.

---

## Step 7: Implement HTTP Relay Client

**Files to create:**
- `lib/core/relay/relay_http.dart`

**Requirements:**
1. HTTP client wrapper using `dio`.
2. All requests include authentication headers:
   - `Authorization: Bearer <token>` if the user has an API token, OR
   - `X-Pubkey: <hex-pubkey>` as fallback.
3. Base URL derived from `RelayConfig.httpBaseUrl`.
4. Methods for all REST endpoints the app uses.

**Class structure:**
```dart
class RelayHttp {
  final Dio _dio;
  final String baseUrl;
  final String pubkeyHex;
  String? bearerToken;

  RelayHttp({
    required this.baseUrl,
    required this.pubkeyHex,
    this.bearerToken,
  }) : _dio = Dio(BaseOptions(baseUrl: baseUrl)) {
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        if (bearerToken != null) {
          options.headers['Authorization'] = 'Bearer $bearerToken';
        } else {
          options.headers['X-Pubkey'] = pubkeyHex;
        }
        handler.next(options);
      },
    ));
  }

  // -- Channels --
  Future<List<Channel>> getChannels();
  Future<Channel> getChannel(String channelId);
  Future<Channel> createChannel({
    required String name,
    required ChannelType type,
    ChannelVisibility visibility = ChannelVisibility.open,
    String? description,
  });
  Future<void> updateChannel(String channelId, {String? topic, String? purpose, String? description});

  // -- Members --
  Future<List<ChannelMember>> getMembers(String channelId);
  Future<void> addMember(String channelId, String pubkey, {MemberRole role = MemberRole.member});
  Future<void> removeMember(String channelId, String pubkey);

  // -- Events / Messages --
  Future<List<NostrEvent>> getEvents({
    required String channelId,
    int limit = 50,
    int? until,
    int? since,
  });

  // -- Profiles --
  Future<UserProfile> getProfile(String pubkey);
  Future<void> updateProfile({String? displayName, String? avatarUrl, String? about, String? nip05Handle});

  // -- Search --
  Future<List<SearchResult>> search(String query, {String? channelId});

  // -- Media --
  Future<MediaAttachment> uploadMedia(File file);

  // -- Tokens --
  Future<ApiToken> createToken(String name, {List<String>? scopes, int? expiresIn});
  Future<List<ApiToken>> listTokens();
  Future<void> revokeToken(String tokenId);

  // -- Home Feed --
  Future<Map<FeedCategory, List<HomeFeedItem>>> getHomeFeed();

  // -- Canvas --
  Future<CanvasDocument?> getCanvas(String channelId);
  Future<void> updateCanvas(String channelId, String content);
}
```

**Riverpod provider:**
```dart
@riverpod
RelayHttp relayHttp(RelayHttpRef ref) {
  final config = ref.watch(relayConfigNotifierProvider).valueOrNull;
  final keyPair = ref.watch(authProvider).valueOrNull;

  if (config == null || keyPair == null) {
    throw StateError('Relay not configured or keys not loaded');
  }

  return RelayHttp(
    baseUrl: config.httpBaseUrl,
    pubkeyHex: keyPair.publicKeyHex,
  );
}
```

---

## Step 8: Implement Auth Service

**Files to create:**
- `lib/core/services/auth_service.dart`

**Requirements:**
1. On app start: check if a keypair exists in secure storage.
2. If yes: load it and proceed.
3. If no: generate a new keypair and save it.
4. Expose the current `KeyPair` as a Riverpod provider.
5. Support key import (from nsec string) and key export (show nsec).
6. Support key deletion (logout/reset).

**Riverpod provider:**
```dart
@riverpod
class Auth extends _$Auth {
  @override
  Future<KeyPair> build() async {
    // Try to load existing keypair
    final existing = await loadKeyPair();
    if (existing != null) return existing;

    // Generate new keypair on first launch
    final newKeyPair = generateKeyPair();
    await saveKeyPair(newKeyPair);
    return newKeyPair;
  }

  Future<void> importKey(String nsec) async {
    final keyPair = fromNsec(nsec);
    await saveKeyPair(keyPair);
    state = AsyncData(keyPair);
  }

  Future<void> resetIdentity() async {
    await deleteKeyPair();
    final newKeyPair = generateKeyPair();
    await saveKeyPair(newKeyPair);
    state = AsyncData(newKeyPair);
  }
}
```

---

## Step 9: Build the App Shell and Routing

**Files to create:**
- `lib/app.dart`
- `lib/shared/router.dart`
- `lib/shared/theme.dart`
- `lib/features/home/home_screen.dart` (placeholder)
- `lib/features/channels/channel_list_screen.dart` (placeholder)
- `lib/features/dm/dm_list_screen.dart` (placeholder)
- `lib/features/settings/settings_screen.dart` (placeholder)
- `lib/features/settings/relay_config_screen.dart`

### `lib/app.dart`

```dart
class H2OHApp extends ConsumerWidget {
  const H2OHApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'H2OH',
      theme: lightTheme,
      darkTheme: darkTheme,
      themeMode: ThemeMode.system,
      routerConfig: router,
    );
  }
}
```

### `lib/shared/router.dart`

Define the GoRouter with a `ShellRoute` for the bottom navigation:

```dart
@riverpod
GoRouter router(RouterRef ref) {
  return GoRouter(
    initialLocation: '/',
    routes: [
      ShellRoute(
        builder: (context, state, child) => AppShell(child: child),
        routes: [
          GoRoute(path: '/', builder: (_, __) => const HomeScreen()),
          GoRoute(path: '/channels', builder: (_, __) => const ChannelListScreen()),
          GoRoute(path: '/dm', builder: (_, __) => const DmListScreen()),
          GoRoute(path: '/settings', builder: (_, __) => const SettingsScreen()),
          GoRoute(path: '/settings/relay', builder: (_, __) => const RelayConfigScreen()),
          // More routes added in later phases
        ],
      ),
    ],
  );
}
```

### AppShell widget

The `AppShell` provides a `Scaffold` with a `BottomNavigationBar`:

```dart
class AppShell extends StatelessWidget {
  final Widget child;
  const AppShell({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        type: BottomNavigationBarType.fixed,
        currentIndex: _calculateIndex(GoRouterState.of(context).uri.path),
        onTap: (index) => _onTap(context, index),
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
          BottomNavigationBarItem(icon: Icon(Icons.tag), label: 'Channels'),
          BottomNavigationBarItem(icon: Icon(Icons.chat_bubble), label: 'DMs'),
          BottomNavigationBarItem(icon: Icon(Icons.settings), label: 'Settings'),
        ],
      ),
    );
  }
}
```

### `lib/features/settings/relay_config_screen.dart`

This is a real screen (not placeholder). It lets the user enter a relay URL.

**UI requirements:**
1. A `TextFormField` pre-filled with the current relay URL (or empty).
2. Validation: must start with `ws://` or `wss://`.
3. A "Connect" button that saves the URL and triggers relay connection.
4. Display current connection state (disconnected, connecting, connected, authenticated, error).
5. A "Disconnect" button to clear the URL and disconnect.

**Behavior:**
- On "Connect": call `relayConfigNotifier.setRelayUrl(url)`.
- This triggers `relayClientProvider` to reinitialize with the new URL.
- Show a `CircularProgressIndicator` while connecting.
- Show a green checkmark when authenticated.
- Show error message if connection fails.

### Placeholder screens

Each placeholder screen should show the screen name as a centered `Text` widget and a brief description of what will be implemented in later phases. Example:

```dart
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(
      child: Text('Home Feed — Coming in Phase 4'),
    );
  }
}
```

### `lib/shared/theme.dart`

Define `lightTheme` and `darkTheme` as `ThemeData` objects. For now, use Material 3 defaults:

```dart
final lightTheme = ThemeData(
  useMaterial3: true,
  colorSchemeSeed: Colors.blue,
  brightness: Brightness.light,
);

final darkTheme = ThemeData(
  useMaterial3: true,
  colorSchemeSeed: Colors.blue,
  brightness: Brightness.dark,
);
```

---

## Step 10: Wire Up the Startup Flow

**Modify `lib/main.dart`** to orchestrate startup:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize secure storage and shared preferences early
  // (Riverpod providers handle actual loading)

  runApp(const ProviderScope(child: H2OHApp()));
}
```

**App startup sequence (handled by providers):**
1. `authProvider` loads/generates keypair from secure storage.
2. `relayConfigNotifierProvider` loads relay URL from SharedPreferences.
3. If relay URL exists: `relayClientProvider` connects to WebSocket.
4. If relay sends AUTH challenge: `RelayClient._handleAuth()` responds automatically.
5. UI displays connection state in Settings > Relay screen.

**First launch experience:**
1. User sees the app shell with bottom nav. Home shows placeholder.
2. User navigates to Settings → Relay.
3. User enters relay URL (e.g. `wss://my-relay.example.com`).
4. App connects, authenticates, shows green status.
5. User is ready for Phase 2 features.

---

## Verification Checklist

After completing Phase 1, verify:

- [ ] `flutter run` launches on Android emulator and iOS simulator.
- [ ] A keypair is generated on first launch (check via debug print of npub).
- [ ] Keypair persists across app restarts (same npub on relaunch).
- [ ] Relay URL can be entered and saved in Settings > Relay.
- [ ] WebSocket connects to a running Sprout relay.
- [ ] NIP-42 auth challenge is handled automatically.
- [ ] Connection state updates are visible in the Relay config screen.
- [ ] Bottom navigation switches between 4 placeholder screens.
- [ ] GoRouter URLs work correctly (`/`, `/channels`, `/dm`, `/settings`, `/settings/relay`).
- [ ] App handles no-relay-configured state gracefully (no crashes).
- [ ] App handles relay-offline state gracefully (shows error, retries with backoff).

---

## Files Created in Phase 1

```
lib/main.dart
lib/app.dart
lib/core/nostr/keys.dart
lib/core/nostr/event.dart
lib/core/nostr/nip01.dart
lib/core/nostr/nip42.dart
lib/core/relay/relay_client.dart
lib/core/relay/relay_http.dart
lib/core/relay/relay_config.dart
lib/core/services/auth_service.dart
lib/shared/router.dart
lib/shared/theme.dart
lib/features/home/home_screen.dart              (placeholder)
lib/features/channels/channel_list_screen.dart   (placeholder)
lib/features/dm/dm_list_screen.dart              (placeholder)
lib/features/settings/settings_screen.dart       (placeholder)
lib/features/settings/relay_config_screen.dart   (real)
pubspec.yaml                                     (all dependencies)
analysis_options.yaml
```
