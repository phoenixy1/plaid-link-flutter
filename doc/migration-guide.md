# Migration Guide: `plaid_flutter` to `plaid_link_flutter`

`plaid_link_flutter` replaces the community `plaid_flutter` package's global
`PlaidLink.create()` and `PlaidLink.open()` calls and event streams with session
objects. You create a session for each flow, keep it in your widget state, and
pass callbacks in its configuration.

## What Changed

| Area | `plaid_flutter` | `plaid_link_flutter` |
| --- | --- | --- |
| Package import | `package:plaid_flutter/plaid_flutter.dart` | `package:plaid_link_flutter/plaid_link_flutter.dart` |
| Native SDKs | iOS 6.x, Android 5.x | iOS LinkKit 7.4.0, Android Link SDK 6.4.0 |
| Primary API | Global `PlaidLink.create()` and `PlaidLink.open()` | `createPlaidLinkSession()` returns `PlaidLinkSession` |
| Callback model | Global streams: `PlaidLink.onSuccess`, `PlaidLink.onExit`, `PlaidLink.onEvent`, `PlaidLink.onLoad` | Per-session callbacks passed in the configuration |
| Link readiness | Wait for create completion or `onLoad` stream | Use `LinkTokenConfiguration.onLoad` |
| Event name | `event.name` | `event.eventName` |
| Layer | `PlaidLink.submit(...)` on the global Link object | `PlaidLayerSession.submit(...)` |
| Embedded | `PlaidEmbeddedView` | `PlaidEmbeddedSearchView` |
| FinanceKit | Not available | `syncFinanceKit(...)` on iOS |
| OAuth resume | `PlaidLink.resumeAfterTermination(...)` | Handled by the native SDKs |
| Closing Link | `PlaidLink.close()` | No Flutter method; handle `onExit` |
| Web | Supported by the community package | Not supported in this SDK |

## Package And Platform Requirements

Replace the dependency:

```yaml
dependencies:
  plaid_link_flutter: ^1.3.0
```

Then update imports:

```dart
import 'package:plaid_link_flutter/plaid_link_flutter.dart';
```

Requirements:

- Flutter `3.29.3+`
- Dart `3.7+`
- iOS `15.0+`
- Android `minSdk 26+`

## Standard Link

### Before

The community package creates a global handler and opens it through static
methods. Callbacks are usually wired through global streams.

```dart
final successSubscription = PlaidLink.onSuccess.listen((success) {
  print(success.publicToken);
});

final exitSubscription = PlaidLink.onExit.listen((exit) {
  print(exit.error?.errorMessage);
});

final eventSubscription = PlaidLink.onEvent.listen((event) {
  print(event.name);
});

await PlaidLink.create(
  configuration: LinkTokenConfiguration(token: linkToken),
);

await PlaidLink.open();
```

### After

Create a `PlaidLinkSession`, keep it alive for as long as you need to open Link,
and pass callbacks directly to the configuration.

```dart
PlaidLinkSession? _linkSession;
bool _linkReady = false;

Future<void> createSession(String linkToken) async {
  _linkSession = await createPlaidLinkSession(
    LinkTokenConfiguration(
      token: linkToken,
      onSuccess: (success) {
        print(success.publicToken);
      },
      onExit: (exit) {
        print(exit.error?.errorMessage);
      },
      onEvent: (event) {
        print(event.eventName);
      },
      onLoad: () {
        setState(() => _linkReady = true);
      },
    ),
  );
}

Future<void> openLink() async {
  await _linkSession?.open();
}
```

On iOS, `open(true)` presents Link full screen. On Android, the flag has no
effect.

## Callbacks And Events

After `onExit`, or after the `HANDOFF` event that follows `onSuccess`, the SDK
removes the session's callbacks.

Only one Link or Layer session has callbacks at a time. Creating a new session
replaces the previous session's callbacks.

## Layer

Layer now has its own configuration and session type. Use a Layer-compatible
link token, created with `/session/token/create`; a regular Link token is not
interchangeable. Submit the user's phone number, then open Layer when
you receive `LAYER_READY`:

```dart
PlaidLayerSession? _layerSession;

Future<void> createLayerSession(String layerToken) async {
  _layerSession = await createPlaidLayerSession(
    LayerTokenConfiguration(
      token: layerToken,
      onSuccess: (success) {
        print(success.publicToken);
      },
      onExit: (exit) {
        print(exit.error?.errorMessage);
      },
      onEvent: (event) {
        if (event.eventName == LinkEventName.layerReady) {
          _layerSession?.open();
        }
      },
    ),
  );
}

Future<void> submitPhoneNumber(String phoneNumber) async {
  await _layerSession?.submit(
    SubmissionData(
      phoneNumber: phoneNumber,
      params: <String, String>{'flow': 'signup'},
    ),
  );
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
    onSuccess: (success) {
      print(success.publicToken);
    },
    onExit: (exit) {
      print(exit.error?.errorMessage);
    },
    onEvent: (event) {
      print(event.eventName);
    },
    onLoad: () {
      print('Embedded Search loaded');
    },
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
  print(error.code);
  print(error.message);
}
```

During development, you can use `FinanceKitSyncBehavior.simulated` instead of
`live`.

## OAuth And Redirect Handling

The community package exposed `PlaidLink.resumeAfterTermination(...)`. The
official Flutter SDK does not expose a separate resume method. OAuth redirect
continuation is handled by the native Plaid SDKs and your app's platform
configuration.

For iOS, configure your Plaid redirect URI as a Universal Link in the Plaid
Dashboard and in your app entitlement setup. For Android, register the app
package name in the Plaid Dashboard and follow the native Android setup for the
Link SDK version used by this package.

## Close Handling

The community package exposed `PlaidLink.close()`. The official session API does
not expose a Flutter-level close method. Let the Link flow terminate through the
native UI and handle `onExit`.

## Migration Checklist

- Replace `plaid_flutter` with `plaid_link_flutter` in `pubspec.yaml`.
- Update imports to `package:plaid_link_flutter/plaid_link_flutter.dart`.
- Replace global `PlaidLink.create()` calls with a flow-specific create method.
- Store the returned session object in widget or state-management scope.
- Replace `PlaidLink.open()` with `PlaidLinkSession.open()`.
- Move global stream listeners into `onSuccess`, `onExit`, `onEvent`, and
  `onLoad` callbacks.
- Rename `LinkEvent.name` reads to `LinkEvent.eventName`.
- Use `LayerTokenConfiguration` and `PlaidLayerSession` for Layer.
- Use `PlaidEmbeddedSearchView` for Embedded Search.
- Gate FinanceKit behind iOS checks in your app code.
- Remove calls to `resumeAfterTermination` and `close()`.
- Confirm your backend creates the correct link token type for each flow.
- Re-test OAuth redirects using your app's production redirect URI setup.
