# Migrating from `plaid_flutter` to `plaid_link_flutter`

`plaid_link_flutter` replaces the community `plaid_flutter` package's global
`PlaidLink.create()` and `PlaidLink.open()` calls and event streams with session
objects. You create a session for each flow, keep it in your widget state, and
pass callbacks in its configuration.

## What changed

| Area | `plaid_flutter` | `plaid_link_flutter` |
| --- | --- | --- |
| Package import | `package:plaid_flutter/plaid_flutter.dart` | `package:plaid_link_flutter/plaid_link_flutter.dart` |
| Primary API | Global `PlaidLink.create()` and `PlaidLink.open()` | `createPlaidLinkSession()` returns a `PlaidLinkSession` |
| Callbacks | Global streams: `PlaidLink.onSuccess`, `onExit`, `onEvent`, `onLoad` | Callbacks in the session's configuration |
| Event name | `event.name` | `event.eventName` |
| Layer | `PlaidLink.submit(...)` on the global object | `PlaidLayerSession.submit(...)` |
| Embedded Search | `PlaidEmbeddedView` | `PlaidEmbeddedSearchView` |
| FinanceKit | Not available | `syncFinanceKit(...)`, iOS only |
| OAuth resume | `PlaidLink.resumeAfterTermination(...)` | Handled by the native SDKs |
| Closing Link | `PlaidLink.close()` | No Flutter method; handle `onExit` |
| Flutter web | Supported | Not supported |

## Requirements

Replace the dependency in `pubspec.yaml`:

```yaml
dependencies:
  plaid_link_flutter: ^1.3.0
```

Then update your imports:

```dart
import 'package:plaid_link_flutter/plaid_link_flutter.dart';
```

The SDK requires Flutter 3.29.3 or later, Dart 3.7 or later, iOS 15.0 or later,
and Android `minSdk` 26 or later.

## Standard Link

Before, with global streams and static methods:

```dart
PlaidLink.onSuccess.listen((success) => print(success.publicToken));
PlaidLink.onExit.listen((exit) => print(exit.error?.errorMessage));
PlaidLink.onEvent.listen((event) => print(event.name));

await PlaidLink.create(
  configuration: LinkTokenConfiguration(token: linkToken),
);
await PlaidLink.open();
```

After, with a session that owns its callbacks:

```dart
PlaidLinkSession? _linkSession;
bool _linkReady = false;

Future<void> createSession(String linkToken) async {
  _linkSession = await createPlaidLinkSession(
    LinkTokenConfiguration(
      token: linkToken,
      onSuccess: (success) => print(success.publicToken),
      onExit: (exit) => print(exit.error?.errorMessage),
      onEvent: (event) => print(event.eventName),
      onLoad: () => setState(() => _linkReady = true),
    ),
  );
}

Future<void> openLink() async {
  await _linkSession?.open();
}
```

On iOS, `open(true)` presents Link full screen. On Android, the flag has no
effect.

## Callbacks and events

After `onExit`, or after the `HANDOFF` event that follows `onSuccess`, the SDK
removes the session's callbacks.

Only one Link or Layer session has callbacks at a time. Creating a new session
replaces the previous session's callbacks.

## Layer

Layer has its own configuration and session type. Create its token with
`/session/token/create`. Submit the user's phone number, then open Layer when
you receive `LAYER_READY`:

```dart
PlaidLayerSession? _layerSession;

Future<void> createLayerSession(String layerToken) async {
  _layerSession = await createPlaidLayerSession(
    LayerTokenConfiguration(
      token: layerToken,
      onSuccess: (success) => print(success.publicToken),
      onExit: (exit) => print(exit.error?.errorMessage),
      onEvent: (event) {
        if (event.eventName == LinkEventName.layerReady) {
          _layerSession?.open();
        }
      },
    ),
  );
}

Future<void> submitPhoneNumber(String phoneNumber) async {
  await _layerSession?.submit(SubmissionData(phoneNumber: phoneNumber));
}
```

If you receive `LAYER_NOT_AVAILABLE`, you can submit the user's date of birth in
a second `submit` call. For details, see the
[Layer docs](https://plaid.com/docs/layer/).

## Embedded Search

Replace `PlaidEmbeddedView` with `PlaidEmbeddedSearchView`:

```dart
SizedBox(
  height: 420,
  child: PlaidEmbeddedSearchView(
    token: linkToken,
    onSuccess: (success) => print(success.publicToken),
    onExit: (exit) => print(exit.error?.errorMessage),
    onEvent: (event) => print(event.eventName),
  ),
);
```

On Android, show only one Embedded Search view at a time.

## FinanceKit

FinanceKit requires iOS 17.4 or later. On Android, `syncFinanceKit` will throw a
`FinanceKitException` with the code `UNSUPPORTED_ANDROID`.

```dart
try {
  await syncFinanceKit(
    FinanceKitConfiguration(
      token: linkToken,
      requestAuthorizationIfNeeded: true,
      syncBehavior: FinanceKitSyncBehavior.live,
    ),
  );
} on FinanceKitException catch (error) {
  print('${error.code}: ${error.message}');
}
```

During development, you can use `FinanceKitSyncBehavior.simulated` instead of
`live`.

## OAuth

The SDK has no `resumeAfterTermination` method. The native Plaid SDKs resume
OAuth flows. On iOS, register your redirect URI as a Universal Link in the Plaid
Dashboard and your app's entitlements. On Android, register your app's package
name in the Plaid Dashboard.

## Closing Link

The SDK has no `close()` method. Link closes when the user exits, and your
`onExit` callback runs.

## Migration checklist

- Replace `plaid_flutter` with `plaid_link_flutter` in `pubspec.yaml`, and
  update your imports.
- Replace `PlaidLink.create()` and `PlaidLink.open()` with a session, and keep
  the session in your widget state.
- Move your stream listeners into the session's `onSuccess`, `onExit`,
  `onEvent`, and `onLoad` callbacks.
- Rename `event.name` to `event.eventName`.
- Use `createPlaidLayerSession()` for Layer and `PlaidEmbeddedSearchView` for
  Embedded Search.
- Remove calls to `resumeAfterTermination` and `close()`.
- Test OAuth redirects with your production redirect URI setup.
